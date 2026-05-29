# Message Queues & Event Streaming — Kafka, RabbitMQ, SQS

## Table of Contents

- [Why Message Queues Exist](#why-message-queues-exist)
- [Core Vocabulary](#core-vocabulary)
- [The Fundamental Split: Queue vs. Stream](#the-fundamental-split-queue-vs-stream)
- [Kafka](#kafka)
  - [What It Is](#what-it-is)
  - [Core Internals](#core-internals)
  - [Producers](#producers)
  - [Consumer Groups](#consumer-groups)
  - [Replication and Durability](#replication-and-durability)
  - [Delivery Semantics](#delivery-semantics)
  - [Zookeeper vs. KRaft](#zookeeper-vs-kraft)
- [RabbitMQ](#rabbitmq)
  - [What It Is](#what-it-is-1)
  - [Core Internals](#core-internals-1)
  - [Exchange Types](#exchange-types)
  - [Dead Letter Exchange](#dead-letter-exchange)
- [SQS](#sqs)
  - [What It Is](#what-it-is-2)
  - [Visibility Timeout](#visibility-timeout)
  - [Standard vs. FIFO Queues](#standard-vs-fifo-queues)
  - [Long Polling](#long-polling)
  - [SQS + SNS Fan-out](#sqs--sns-fan-out)
- [Three-Way Comparison](#three-way-comparison)
- [When To Use What](#when-to-use-what)
- [Common Design Patterns](#common-design-patterns)
- [Trade-offs](#trade-offs)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

---

## Why Message Queues Exist

When Service A calls Service B directly over HTTP, three failure modes surface immediately:

- **Latency coupling** — A waits for B. B being slow makes A slow.
- **Availability coupling** — B being down makes A fail.
- **Load coupling** — A traffic spike propagates directly to B.

A message queue places a durable buffer between producer and consumer. A publishes a message and continues; B processes it when ready. The two services are now **temporally decoupled** (don't need to be alive simultaneously) and **spatially decoupled** (don't need each other's address).

**Memory anchor:** Think of a message queue as a postal system. The sender drops a letter in the box and walks away. The post office (broker) stores and delivers it. The receiver picks it up on their own schedule.

---

## Core Vocabulary

- **Producer / Publisher** — The service that sends messages.
- **Consumer / Subscriber** — The service that reads and processes messages.
- **Message** — The unit of data in transit (payload + metadata).
- **Broker** — The server that stores and routes messages.
- **Queue** — Point-to-point buffer; one consumer gets each message.
- **Topic** — Named channel; multiple consumers can read the same message independently.
- **Acknowledgement (ACK)** — Consumer telling the broker "I processed this successfully."
- **Dead Letter Queue (DLQ)** — Destination for messages that repeatedly fail processing.
- **Offset** — A sequential position marker within a Kafka partition log.
- **Consumer Group** — A set of consumer instances sharing the work of consuming a topic.

---

## The Fundamental Split: Queue vs. Stream

This distinction is the most important mental model for interviews.

**Queue model — point-to-point (RabbitMQ, SQS):**

```
Producer --> [Queue] --> Consumer A  (message is deleted after consumed)
```

Each message is consumed by exactly one consumer. Once consumed, it is gone. Designed for **task distribution** — each job must be done once by one worker.

**Stream model — log-based (Kafka):**

```
Producer --> [Partition Log] --> Consumer Group A reads at offset 12
                             --> Consumer Group B reads at offset 7
                  (message persists; consumers track their own position)
```

Messages are appended to an immutable log. Each consumer group independently tracks its position (offset). The message remains until retention expires, regardless of whether it has been consumed. Designed for **event broadcasting, replay, and audit trails**.

---

## Kafka

### What It Is

Kafka is a **distributed, partitioned, replicated commit log**. It is not a traditional queue — it is a log. Every message is an ordered, immutable record appended to a partition. Kafka is optimised for high throughput (millions of messages per second) and horizontal scalability.

**Why it is called Kafka:** Named after Franz Kafka by creator Jay Kreps, who thought it was "a system optimised for writing" — and Kafka wrote prolifically. Mostly trivia; useful for recall.

### Core Internals

A **topic** is divided into **partitions**. Each partition is an ordered, append-only log stored on disk.

```
Topic: "orders"

Partition 0:  [offset 0] [offset 1] [offset 2] [offset 3] ...
Partition 1:  [offset 0] [offset 1] [offset 2] [offset 3] ...
Partition 2:  [offset 0] [offset 1] [offset 2] [offset 3] ...
```

- Each partition is an **independent ordered sequence** — ordering is guaranteed within a partition, not across partitions.
- Each partition is replicated across multiple brokers (one leader + N followers).
- More partitions = more parallelism but more overhead (more open file handles, longer leader election).

### Producers

Producers publish messages to a topic. The partition a message lands in is determined by:

- **Key present:** `hash(key) % num_partitions` — same key always goes to the same partition, guaranteeing per-key ordering.
- **No key:** round-robin across partitions.

The producer can also explicitly specify a partition.

### Consumer Groups

A **consumer group** is the unit of parallelism on the read side. Each partition is assigned to **at most one consumer instance** within a group.

```
Topic "orders" — 3 partitions

Consumer Group "payments-service":
  Partition 0 --> Instance C1
  Partition 1 --> Instance C2
  Partition 2 --> Instance C3

Consumer Group "analytics-service":
  Partition 0 --> Instance D1
  Partition 1 --> Instance D1   (fewer instances than partitions — one handles two)
  Partition 2 --> Instance D2
```

Key rules:
- Max useful parallelism = number of partitions. Adding more consumer instances than partitions leaves some idle.
- Two different consumer groups consume the same topic **independently** — true pub-sub.
- Each consumer group stores its current offset per partition in the internal `__consumer_offsets` topic.

**Replay:** Reset a consumer group's offset to an earlier position (or the beginning) and re-read all messages from that point. This is impossible in a traditional queue — it is one of Kafka's defining advantages.

### Replication and Durability

Each partition has one **Leader** broker (serves all reads and writes) and N **follower** brokers that replicate the leader.

- **ISR (In-Sync Replicas):** The set of replicas that are fully caught up with the leader. A replica falls out of ISR if it falls too far behind.
- **`acks` producer config controls durability:**

```
acks=0  --> fire and forget; no confirmation; highest throughput, data loss possible
acks=1  --> leader confirms; fast; loss possible if leader dies before followers catch up
acks=all (or -1) --> all ISR confirm; strongest durability; highest latency
```

- **`min.insync.replicas`:** Minimum number of ISR replicas that must acknowledge before write succeeds (used with `acks=all`).

### Delivery Semantics

| Semantic | How | Risk |
|---|---|---|
| At-most-once | Commit offset before processing | Message lost on crash |
| At-least-once | Commit offset after processing | Message reprocessed on crash (duplicate) |
| Exactly-once | Idempotent producers + transactions | Complex; Kafka supports natively since 0.11 |

**At-least-once is the default** for most real-world systems. Consumers are designed to be idempotent (processing the same message twice has no additional effect).

### Zookeeper vs. KRaft

- **Classic Kafka (pre-3.x):** Relied on Apache ZooKeeper for broker metadata, leader election, and cluster coordination.
- **KRaft (Kafka Raft, 3.x+):** Kafka now manages its own metadata using the Raft consensus algorithm, stored in Kafka itself. ZooKeeper dependency removed.
- **Interview signal:** Mentioning KRaft shows awareness of Kafka's current trajectory.

---

## RabbitMQ

### What It Is

RabbitMQ is a traditional message broker implementing the **AMQP** (Advanced Message Queuing Protocol). Unlike Kafka's log-based model, RabbitMQ is a **smart routing layer** between producers and queues. Messages are deleted once consumed and acknowledged.

**Why "MQ":** Stands for Message Queue — exactly what it does.

### Core Internals

```
Producer --> [Exchange] --> [Queue A] --> Consumer 1
                       --> [Queue B] --> Consumer 2
                       --> [Queue C] --> Consumer 3

Routing rules (bindings) connect the exchange to queues.
```

The **Exchange** is the defining feature of RabbitMQ's model. The producer never publishes directly to a queue — it publishes to an exchange with a **routing key**. The exchange inspects the key (or headers) and routes the message to the correct queue(s) based on **binding rules**.

**No concept of offsets** — once consumed and ACKed, the message is deleted from the queue. There is no replay.

**Push model:** RabbitMQ pushes messages to consumers. The `prefetch` count controls how many unacknowledged messages a consumer can hold at once (prevents overwhelming slow consumers).

### Exchange Types

| Exchange Type | Routing Logic | Use Case |
|---|---|---|
| **Direct** | Exact match on routing key | Route `orders.created` to `orders-queue` |
| **Topic** | Wildcard match (`*` = one word, `#` = zero or more) | Route `orders.*` to all order event queues |
| **Fanout** | Ignore routing key; broadcast to all bound queues | Notifications sent to multiple services |
| **Headers** | Match on message header attributes | Rare; flexible but verbose |

**Memory anchor:** Exchange types are just different sorting strategies in the postal sorting office. Direct = exact address. Topic = postcode wildcard. Fanout = flyer drop to every mailbox.

### Dead Letter Exchange

When a message cannot be delivered (rejected, expired via TTL, or queue at max length), it is routed to a configured **Dead Letter Exchange (DLX)**, which routes it to a **Dead Letter Queue (DLQ)**. This enables inspection, alerting, and manual retry of failed messages.

---

## SQS

### What It Is

**Amazon Simple Queue Service** is a fully managed, distributed message queue service on AWS. No brokers to provision, no clusters to manage — AWS handles all replication, scaling, and durability internally.

**Why "Simple":** Compared to self-managed brokers (Kafka, RabbitMQ), there is no operational surface area. Simple refers to the ops model, not the feature set.

### Visibility Timeout

The single most-tested SQS concept in interviews.

When a consumer calls `ReceiveMessage`, the message is **not deleted**. Instead, it becomes **invisible** to all other consumers for a configurable period (default: 30 seconds). This is the visibility timeout.

```
Scenario A — Success:
[Message visible in queue]
      |
  Consumer receives (message hidden, 30s clock starts)
      |
  Consumer processes successfully
      |
  Consumer calls DeleteMessage
      |
[Message permanently removed]

Scenario B — Consumer Crash:
[Message visible in queue]
      |
  Consumer receives (message hidden, 30s clock starts)
      |
  Consumer crashes (no DeleteMessage call)
      |
  Visibility timeout expires
      |
[Message becomes visible again --> another consumer picks it up]
```

This mechanism gives SQS **at-least-once delivery** for Standard queues. If the consumer processes the message but crashes before calling `DeleteMessage`, the message will be reprocessed.

**Extending the timeout:** If processing takes longer than the visibility timeout, the consumer should call `ChangeMessageVisibility` to extend the window before it expires.

### Standard vs. FIFO Queues

| Dimension | Standard | FIFO |
|---|---|---|
| **Ordering** | Best-effort (not guaranteed) | Strict first-in-first-out |
| **Delivery** | At-least-once (duplicates possible) | Exactly-once processing |
| **Throughput** | Nearly unlimited | 300 TPS (3000 with batching) |
| **Use case** | High-volume, order-insensitive tasks | Payment processing, ordered workflows |

### Long Polling

- **Short polling (default):** `ReceiveMessage` returns immediately even if the queue is empty. Wastes API calls and incurs cost.
- **Long polling:** `ReceiveMessage` waits up to 20 seconds for a message to arrive before returning. Preferred for cost efficiency and reduced empty-response noise.

Set `WaitTimeSeconds` to enable long polling.

### SQS + SNS Fan-out

SQS has no native broadcast. To publish one event and deliver it to multiple SQS queues (multiple services), use **SNS (Simple Notification Service)** as a fan-out layer:

```
Event Source
    |
   SNS Topic
    |
   / | \
SQS  SQS  SQS
 A    B    C
(payments) (analytics) (notifications)
```

This is the standard AWS pattern for decoupled microservices event distribution.

---

## Three-Way Comparison

| Dimension | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| **Model** | Distributed log / stream | AMQP broker / queue | Managed cloud queue |
| **Message retention** | Time-based or size-based | Until consumed + ACKed | Up to 14 days |
| **Replay** | Yes — reset offset | No | No |
| **Ordering guarantee** | Per partition (by key) | Per queue | FIFO queue only |
| **Peak throughput** | Millions/sec | Tens of thousands/sec | Very high (Standard) |
| **Routing** | By partition key | Exchange routing rules | None (SNS for fan-out) |
| **Delivery** | At-least-once; exactly-once available | At-least-once | Standard: at-least-once; FIFO: exactly-once |
| **Ops burden** | High (self-managed) or managed (Confluent/MSK) | Medium | Zero |
| **Consumer model** | Pull (poll) | Push (with prefetch) | Pull (`ReceiveMessage`) |
| **Best for** | Event sourcing, audit logs, high-throughput pipelines | Complex routing, task queues, priority queues | AWS-native, simple decoupled queuing |

---

## When To Use What

**Choose Kafka when:**
- You need high throughput (millions of events/sec).
- Multiple independent services need to consume the same event stream.
- You need event replay or audit trail (e.g., rebuilding a projection).
- You are building an event-sourced system or a real-time analytics pipeline.

**Choose RabbitMQ when:**
- You need complex routing logic (multiple exchange types, wildcards).
- You need per-message TTL, priority queues, or sophisticated retry policies.
- You are building task queues where each task is processed by exactly one worker.

**Choose SQS when:**
- You are on AWS and want zero operational overhead.
- You need a reliable decoupled buffer between two AWS services.
- You want FIFO semantics with exactly-once processing (SQS FIFO).
- The fan-out requirement is met by combining with SNS.

---

## Common Design Patterns

**Work Queue (competing consumers):**
Multiple consumer instances consume from the same queue. Messages are distributed across consumers. Used for parallel task processing (e.g., thumbnail generation, email sending).

```
Producer --> [Queue] --> Consumer 1
                    --> Consumer 2
                    --> Consumer 3
(each message processed by exactly one consumer)
```

**Pub-Sub (fan-out):**
One event consumed by multiple independent subscriber systems.

```
Producer --> [Topic / Exchange / SNS]
                    --> Service A (payments)
                    --> Service B (analytics)
                    --> Service C (notifications)
(each service gets every message, independently)
```

**Event Sourcing with Kafka:**
All state changes are published as immutable events to Kafka. Services rebuild their own state by replaying the event log from the beginning. The log is the source of truth.

**Outbox Pattern (ensuring message delivery):**
Write the event to a local `outbox` table in the same DB transaction as the business operation. A separate process reads the outbox and publishes to the queue. Prevents the dual-write problem (operation succeeds but message lost, or vice versa).

---

## Trade-offs

**Kafka:**
- Operationally complex (brokers, partitions, replication, ZooKeeper/KRaft).
- Adding partitions later is possible but disruptive (rebalancing).
- Not suitable for message-by-message TTL or priority queues.
- Consumer lag (falling behind the log) can be hard to debug.

**RabbitMQ:**
- No replay — once consumed, messages are gone.
- Not designed for very high throughput (Kafka-scale).
- Exchange/binding configuration can become complex to manage.
- Horizontal scaling is harder than Kafka.

**SQS:**
- Vendor lock-in (AWS).
- FIFO queues have a hard throughput ceiling (300–3000 TPS).
- No native replay; no wildcard routing without SNS.
- Visibility timeout requires consumers to be designed carefully.

---

## Interview Cheat Sheet

**Common Questions**

- *"How does Kafka guarantee ordering?"*
  Ordering is guaranteed **within a partition**. Use the same message key for all events that must be ordered — they will hash to the same partition.

- *"What is the difference between a queue and a topic?"*
  Queue: point-to-point, one consumer gets each message, message deleted after consumption. Topic: one message readable by multiple independent consumer groups, message retained by time/size.

- *"How does SQS achieve at-least-once delivery?"*
  Via the **visibility timeout** mechanism. The message is hidden (not deleted) after receipt. If the consumer does not delete it before the timeout expires, it becomes visible again for another consumer.

- *"What happens if a Kafka consumer crashes mid-processing?"*
  With at-least-once semantics (offset committed after processing), the message will be reprocessed by the next consumer assignment. Consumers should be idempotent.

- *"How do you scale Kafka consumers?"*
  Add more consumer instances to the consumer group, up to the number of partitions. To scale beyond that, increase the partition count first.

- *"Kafka vs. RabbitMQ — how do you choose?"*
  Kafka for high-throughput streaming, event sourcing, and replay. RabbitMQ for complex routing, task queues, and per-message TTL.

**Strong Signal Phrases**
- "Kafka partitions are the unit of parallelism — I'd size partitions to the expected peak consumer parallelism."
- "With at-least-once delivery, I'd ensure consumer idempotency using a deduplication key or an idempotency table."
- "The outbox pattern eliminates the dual-write problem — write the event to the DB and publish to the queue atomically."
- "I'd use the visibility timeout to bound worst-case reprocessing, and set it to 2× the expected processing time."
- "Consumer group offsets give Kafka its replay capability — something neither SQS nor RabbitMQ can do natively."

---

## References

- [Kafka Documentation — Core Concepts](https://kafka.apache.org/documentation/)
- [Kafka Design — The Log: What every software engineer should know](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [RabbitMQ — AMQP Concepts](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [RabbitMQ — Exchange Types](https://www.rabbitmq.com/tutorials/tutorial-four-go)
- [AWS SQS — Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [AWS SQS — Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Confluent — Kafka vs. RabbitMQ](https://www.confluent.io/learn/kafka-vs-rabbitmq/)