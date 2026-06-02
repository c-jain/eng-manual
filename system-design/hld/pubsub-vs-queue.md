# Pub/Sub Pattern vs Message Queue Pattern

## Table of Contents

- [Why Async Messaging Exists](#why-async-messaging-exists)
- [Message Queue (Point-to-Point)](#message-queue-point-to-point)
- [Pub/Sub (Publish-Subscribe)](#pubsub-publish-subscribe)
- [Side-by-Side Comparison](#side-by-side-comparison)
- [The Kafka Nuance](#the-kafka-nuance)
- [When to Use Which](#when-to-use-which)
- [Real Systems and Where They Sit](#real-systems-and-where-they-sit)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why Async Messaging Exists

Synchronous calls (HTTP, gRPC) couple the caller to the callee: both services must be up simultaneously, the caller blocks while waiting, and the callee's address must be known. Under load, a slow callee stalls the entire call chain.

Async messaging breaks this by inserting a **broker** — a durable intermediary that accepts messages from producers and delivers them to consumers independently. Producers and consumers do not know about each other; they only know the broker.

Two dominant patterns have emerged from this idea: **message queues** and **pub/sub**. Both use a broker. The difference is in how many consumers receive each message and what happens to the message after delivery.

---

## Message Queue (Point-to-Point)

### What It Is

A producer sends a message to a named **queue**. Exactly one consumer dequeues and processes that message. Once processed and acknowledged (ACKed), the message is permanently removed.

```
Producer
   |
   v
[Queue: process-payment]
   |
   +---> Consumer A  <-- gets this message, ACKs it, message deleted
   |
   +---> Consumer B  <-- idle, gets the *next* message
```

### Why It's Called a Queue

The name is structural. Underneath, a queue is a FIFO data structure — messages are produced at one end and consumed from the other. The name reflects the data structure, not a metaphor.

### Competing Consumers

Multiple consumer instances can listen on the same queue. The broker distributes messages across them, naturally load-balancing. Each message still goes to exactly one consumer. This is the standard horizontal scaling strategy for queue-based workers.

```
[Queue: resize-image]
   |
   +---> Worker 1  <-- msg A
   +---> Worker 2  <-- msg B
   +---> Worker 3  <-- msg C
```

### Key Behaviours

- **Delivery guarantee:** at-least-once by default (broker retries if no ACK received within a timeout); exactly-once requires idempotent consumers.
- **Message fate:** deleted after ACK. No replay by default.
- **Dead-letter queue (DLQ):** if a consumer repeatedly fails to ACK (NACK or timeout), the broker moves the message to a DLQ for inspection. DLQs must be actively monitored.
- **Ordering:** guaranteed per queue when there is a single consumer; breaks under multiple concurrent consumers because processing finishes in non-deterministic order.

### Problems It Brings

- Only one consumer receives each message — unsuitable when multiple independent systems must react to the same event.
- Message ordering is hard to preserve at scale.
- DLQ management is operational overhead — poison messages can pile up silently.
- Re-processing (replay) requires manual re-queueing.

---

## Pub/Sub (Publish-Subscribe)

### What It Is

A publisher emits an event to a named **topic**. Every service that has subscribed to that topic receives its own independent copy of the message. The broker manages the subscription registry.

```
Publisher: "order placed"
   |
   v
[Topic: order.created]
   |
   +---> Email Service      (subscriber 1) -- sends confirmation
   +---> Inventory Service  (subscriber 2) -- decrements stock
   +---> Analytics Service  (subscriber 3) -- records event
```

Each subscriber processes the message independently. One subscriber failing does not affect the others.

### Why It's Called Pub/Sub

The two roles are named after their actions: publishers **pub**lish events, subscribers **sub**scribe to topics. The name describes the interaction contract, not the data structure.

### Fan-Out Is the Defining Characteristic

Fan-out means one message produces N independent deliveries, one per subscriber. This is fundamentally different from a queue, where one message produces exactly one delivery.

### Key Behaviours

- **Fan-out:** every subscriber gets its own copy.
- **Subscription model:** subscribers register interest in a topic; the broker routes accordingly. New subscribers can be added without touching the publisher.
- **Message retention:** depends on the broker. Kafka retains messages by offset (configurable retention period, supports replay). Redis Pub/Sub does not retain — a subscriber that is offline misses messages permanently.
- **Delivery guarantee:** at-least-once is standard; exactly-once across all subscribers is harder to guarantee end-to-end.

### Problems It Brings

- More complex operationally — topics, subscriptions, and consumer group offsets all require management.
- Offline subscribers can miss events unless the broker retains messages.
- Exactly-once semantics across all subscribers require each subscriber to be individually idempotent.
- Harder to reason about ordering when multiple subscribers process concurrently.

---

## Side-by-Side Comparison

```
Dimension            Message Queue              Pub/Sub
─────────────────    ───────────────────────    ──────────────────────────
Consumer count       One per message            All subscribers per message
Fan-out              No                         Yes
Message after ACK    Deleted                    Retained (broker-dependent)
Replay               Manual re-queue            Native (Kafka, GCP Pub/Sub)
Load balancing       Built-in (competing)       Per subscriber group
Primary use case     Task distribution          Event broadcasting
Coupling             Consumer knows queue name  Subscriber knows topic name
```

---

## The Kafka Nuance

Kafka is architecturally a **distributed commit log** (append-only, ordered, partitioned). It is not a traditional queue, but it can implement both patterns depending on consumer group configuration.

### Within a Consumer Group — Queue Semantics

Each partition is assigned to exactly one consumer instance in the group. Messages in that partition go to one consumer at a time.

```
Topic: orders (3 partitions)
Consumer Group: billing (2 instances)

  Partition 0 ──> billing-instance-1
  Partition 1 ──> billing-instance-2
  Partition 2 ──> billing-instance-1

Result: each message processed by exactly one billing instance.
```

### Across Consumer Groups — Pub/Sub Semantics

Each consumer group independently tracks its own offset. A message is delivered to each group, and each group processes it independently.

```
Topic: orders

  Group: billing    ──> processes all messages (for invoicing)
  Group: analytics  ──> processes all messages (for reporting)

Result: every message is delivered to both groups — fan-out.
```

**The rule to remember:** consumer group = unit of fan-out. One group = queue-like. Multiple groups = pub/sub-like. This is why Kafka is called a "unified streaming platform" — it supports both models.

---

## When to Use Which

**Use a message queue when:**

- One worker should do a job — image resizing, email sending, payment processing.
- You need natural load balancing across worker instances.
- You want backpressure — producers slow down automatically when the queue fills up.
- Processing must be idempotent but only one consumer needs to act.
- You are building a pipeline where step A hands off to step B sequentially.

**Use pub/sub when:**

- Multiple independent services must react to the same event — order placed triggers email, inventory, and analytics simultaneously.
- You want publishers and subscribers to be fully decoupled — publishers do not know who is listening.
- You need event replay — re-process historical events for a new subscriber or for debugging.
- You are implementing an event-driven architecture where services communicate exclusively through events.

**Common mistake to avoid:** using a queue when you actually need fan-out leads to duplicating the queue and consuming the same message multiple times through separate queues. Use pub/sub instead and let the broker handle delivery to each subscriber.

---

## Real Systems and Where They Sit

- **RabbitMQ** — supports both. Exchange types control routing: `direct`/`fanout` exchanges implement pub/sub; classic queues implement point-to-point.
- **Apache Kafka** — distributed commit log; implements both via consumer group model. Default choice for high-throughput event streaming.
- **AWS SQS** — pure message queue (standard and FIFO variants). Pair with AWS SNS for pub/sub fan-out.
- **AWS SNS** — pub/sub fan-out. Typically SNS fan-outs to multiple SQS queues (the SNS+SQS fan-out pattern).
- **Google Cloud Pub/Sub** — managed pub/sub with at-least-once delivery and message retention.
- **Redis Pub/Sub** — lightweight, fire-and-forget pub/sub. No persistence; offline subscribers miss messages. Suitable for ephemeral notifications, not durable events.
- **NATS** — lightweight, high-performance; supports both queue groups (queue semantics) and core pub/sub.

---

## Interview Cheat Sheet

**One-line distinction:**

> Queue: one message, one consumer. Pub/Sub: one message, every subscriber.

**Common probes and strong answers:**

- *"How do you fan-out to multiple services?"* — Pub/Sub. Each service subscribes to the same topic and receives its own copy. With SQS, you'd need SNS fan-out to multiple queues.
- *"How do you scale consumers without duplicating work?"* — Message queue with competing consumers. Multiple workers on the same queue; broker distributes.
- *"What happens if a subscriber is offline?"* — Depends on the broker. Kafka retains by offset — the subscriber catches up when it reconnects. Redis Pub/Sub drops the message.
- *"How does Kafka behave?"* — Within a consumer group: queue semantics (one consumer per partition). Across groups: pub/sub semantics (each group gets all messages).
- *"What is a DLQ?"* — Dead-letter queue. Messages that fail repeatedly (no ACK after N retries) are routed here for inspection. Critical to monitor; unmonitored DLQs silently swallow failed work.
- *"How do you guarantee exactly-once processing?"* — At the broker level, use idempotency keys and deduplication (SQS FIFO, Kafka transactions). At the consumer level, make processing idempotent so re-delivery is safe.

**Red flags to avoid:**

- Conflating fan-out with load balancing — they are opposites. Fan-out sends to all; load balancing sends to one.
- Calling Kafka "just a queue" — it is a commit log; the queue vs pub/sub behaviour is an emergent property of consumer group configuration.
- Forgetting DLQ when describing queue-based systems — interviewers notice.

---

## References

- [AWS: Message Queuing vs Pub/Sub](https://aws.amazon.com/compare/the-difference-between-pub-sub-messaging-and-message-queuing/)
- [Kafka Documentation — Consumer Groups](https://kafka.apache.org/documentation/#intro_consumers)
- [Google Cloud Pub/Sub Overview](https://cloud.google.com/pubsub/docs/overview)
- [RabbitMQ: AMQP Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)