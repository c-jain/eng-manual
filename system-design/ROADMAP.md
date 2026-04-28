# System Design Roadmap — Interview Prep (2 Months)

> **Goal:** Structured, topic-by-topic coverage of High-Level Design (HLD) and Low-Level Design (LLD) for software engineering interviews.

> **Approach:** One topic per day, deep notes per session.

## Table of Contents

- [High-Level Design (HLD)](#high-level-design-hld)
- [Low-Level Design (LLD)](#low-level-design-lld)
- [Study Plan](#study-plan)
- [Resources](#resources)

## High-Level Design (HLD)

HLD focuses on the **architecture** of a system — how components interact at a large scale.
Think: distributed systems, scalability, availability, reliability.

### Module 1 — Foundations (Week 1)

| # | Topic | File |
|---|-------|------|
| 1 | How to approach an HLD interview (framework & template) | `hld/hld-interview-approach.md` |
| 2 | Client-Server model, IP, DNS, HTTP/HTTPS basics | `hld/networking-basics.md` |
| 3 | APIs — REST, GraphQL, gRPC; API Gateway | `hld/api-design.md` |
| 4 | Latency vs Throughput vs Availability vs Consistency | `hld/core-tradeoffs.md` |
| 5 | CAP Theorem & PACELC Theorem | `hld/cap-theorem.md` |
| 6 | Back-of-the-envelope estimation (capacity planning) | `hld/estimation.md` |
| 7 | SLAs, SLOs, SLIs — reliability engineering basics | `hld/reliability-sla.md` |

---

### Module 2 — Scalability Patterns (Week 2)

| # | Topic | File |
|---|-------|------|
| 8 | Horizontal vs Vertical Scaling | `hld/scaling.md` |
| 9 | Load Balancing — algorithms, L4 vs L7 | `hld/load-balancing.md` |
| 10 | Caching — strategies (write-through, write-back, write-around), eviction policies (LRU, LFU), CDN | `hld/caching.md` |
| 11 | Content Delivery Networks (CDN) | `hld/cdn.md` |
| 12 | Database Scaling — replication, sharding, partitioning | `hld/database-scaling.md` |
| 13 | SQL vs NoSQL — when to use which, trade-offs | `hld/sql-vs-nosql.md` |
| 14 | Consistent Hashing — deep dive | `hld/consistent-hashing.md` |

---

### Module 3 — Core Infrastructure Concepts (Week 3)

| # | Topic | File |
|---|-------|------|
| 15 | Message Queues & Event Streaming — Kafka, RabbitMQ, SQS | `hld/message-queues.md` |
| 16 | Pub/Sub pattern vs Message Queue pattern | `hld/pubsub-vs-queue.md` |
| 17 | Service Discovery & Coordination — Zookeeper, etcd | `hld/service-discovery.md` |
| 18 | Microservices Architecture — principles, pitfalls, service mesh | `hld/microservices.md` |
| 19 | Monolith vs Microservices — decision framework | `hld/monolith-vs-microservices.md` |
| 20 | Rate Limiting — token bucket, leaky bucket, sliding window | `hld/rate-limiting.md` |
| 21 | Proxies — forward proxy, reverse proxy, sidecar pattern | `hld/proxies.md` |

---

### Module 4 — Storage & Data Systems (Week 4)

| # | Topic | File |
|---|-------|------|
| 22 | Relational Databases deep dive — indexes, transactions, ACID | `hld/relational-databases.md` |
| 23 | NoSQL databases — Key-Value, Document, Wide-Column, Graph | `hld/nosql-types.md` |
| 24 | Distributed File Storage — S3, HDFS, GFS concepts | `hld/distributed-storage.md` |
| 25 | Blob Storage vs Object Storage vs Block Storage | `hld/storage-types.md` |
| 26 | Search Systems — Elasticsearch, inverted index | `hld/search-systems.md` |
| 27 | Time-Series Databases | `hld/time-series-db.md` |
| 28 | Data Warehousing & OLAP vs OLTP | `hld/olap-vs-oltp.md` |

---

### Module 5 — HLD Case Studies / Classic System Designs (Week 5–6)

| # | System | Concepts Covered | File |
|---|--------|-----------------|------|
| 29 | URL Shortener (TinyURL) | Hashing, DB design, redirection | `hld/design-url-shortener.md` |
| 30 | Rate Limiter | Token bucket, distributed counters | `hld/design-rate-limiter.md` |
| 31 | Key-Value Store (like Redis/DynamoDB) | Consistent hashing, replication, WAL | `hld/design-key-value-store.md` |
| 32 | Distributed Cache | Cache invalidation, eviction, consistency | `hld/design-distributed-cache.md` |
| 33 | Notification System | Push/pull, fan-out, WebSockets vs SSE | `hld/design-notification-system.md` |
| 34 | News Feed / Social Media Feed | Fan-out-on-write vs on-read, ranking | `hld/design-news-feed.md` |
| 35 | Chat System (WhatsApp) | WebSockets, message ordering, delivery receipts | `hld/design-chat-system.md` |
| 36 | Search Autocomplete / Typeahead | Trie, ranking, caching | `hld/design-typeahead.md` |
| 37 | Web Crawler | BFS, politeness, deduplication | `hld/design-web-crawler.md` |
| 38 | YouTube / Video Streaming | CDN, transcoding, chunked upload | `hld/design-video-streaming.md` |
| 39 | Uber / Ride-Sharing | Geospatial indexing, matching, real-time | `hld/design-ride-sharing.md` |
| 40 | Google Drive / Dropbox | File sync, chunking, delta encoding | `hld/design-file-storage.md` |
| 41 | Twitter | Timeline generation, trending topics | `hld/design-twitter.md` |
| 42 | Distributed Message Queue (like Kafka) | Partitions, offsets, consumer groups | `hld/design-message-queue.md` |

---

## Low-Level Design (LLD)

LLD focuses on **code design** — class hierarchies, relationships, design patterns, and clean APIs.
Think: object-oriented design, SOLID principles, design patterns, coding clean abstractions.

### Module 1 — OOP & Design Foundations (Week 1)

| # | Topic | File |
|---|-------|------|
| 1 | OOP Pillars — Encapsulation, Abstraction, Inheritance, Polymorphism | `lld/oop-pillars.md` |
| 2 | SOLID Principles — deep dive with examples | `lld/solid-principles.md` |
| 3 | DRY, KISS, YAGNI — design heuristics | `lld/design-heuristics.md` |
| 4 | UML Diagrams — class, sequence, activity diagrams | `lld/uml-diagrams.md` |
| 5 | How to approach an LLD interview (framework & template) | `lld/lld-interview-approach.md` |

---

### Module 2 — Design Patterns: Creational (Week 2)

| # | Pattern | Intent | File |
|---|---------|--------|------|
| 6 | Singleton | One instance, global access | `lld/patterns/singleton.md` |
| 7 | Factory Method | Delegate instantiation to subclass | `lld/patterns/factory-method.md` |
| 8 | Abstract Factory | Family of related objects | `lld/patterns/abstract-factory.md` |
| 9 | Builder | Step-by-step object construction | `lld/patterns/builder.md` |
| 10 | Prototype | Clone existing objects | `lld/patterns/prototype.md` |

---

### Module 3 — Design Patterns: Structural (Week 2–3)

| # | Pattern | Intent | File |
|---|---------|--------|------|
| 11 | Adapter | Bridge incompatible interfaces | `lld/patterns/adapter.md` |
| 12 | Decorator | Add behavior dynamically | `lld/patterns/decorator.md` |
| 13 | Facade | Simplified interface to subsystem | `lld/patterns/facade.md` |
| 14 | Proxy | Placeholder / access control | `lld/patterns/proxy.md` |
| 15 | Composite | Tree structures of objects | `lld/patterns/composite.md` |
| 16 | Bridge | Decouple abstraction from implementation | `lld/patterns/bridge.md` |
| 17 | Flyweight | Share objects to save memory | `lld/patterns/flyweight.md` |

---

### Module 4 — Design Patterns: Behavioral (Week 3)

| # | Pattern | Intent | File |
|---|---------|--------|------|
| 18 | Observer | Event subscription & notification | `lld/patterns/observer.md` |
| 19 | Strategy | Swap algorithms at runtime | `lld/patterns/strategy.md` |
| 20 | Command | Encapsulate requests as objects | `lld/patterns/command.md` |
| 21 | State | Alter behavior as state changes | `lld/patterns/state.md` |
| 22 | Chain of Responsibility | Pass request along a handler chain | `lld/patterns/chain-of-responsibility.md` |
| 23 | Iterator | Traverse without exposing internals | `lld/patterns/iterator.md` |
| 24 | Template Method | Skeleton algorithm, fill in steps | `lld/patterns/template-method.md` |
| 25 | Mediator | Central communication hub | `lld/patterns/mediator.md` |

---

### Module 5 — LLD Case Studies / Classic Class Designs (Week 4–6)

| # | System | Key Concepts | File |
|---|--------|-------------|------|
| 26 | Parking Lot | OOP modeling, enums, state pattern | `lld/design-parking-lot.md` |
| 27 | Library Management System | Entities, relationships, search | `lld/design-library-management.md` |
| 28 | ATM Machine | State pattern, transactions | `lld/design-atm.md` |
| 29 | Elevator System | Scheduling, state machine | `lld/design-elevator.md` |
| 30 | Vending Machine | State pattern, inventory | `lld/design-vending-machine.md` |
| 31 | Hotel Booking System | Availability, concurrency, pricing | `lld/design-hotel-booking.md` |
| 32 | Movie Ticket Booking (BookMyShow) | Seats, booking states, payments | `lld/design-movie-booking.md` |
| 33 | Chess Game | Board, pieces, move validation | `lld/design-chess.md` |
| 34 | Snake and Ladder | Game loop, dice, board state | `lld/design-snake-ladder.md` |
| 35 | Cab Booking (Ola/Uber LLD) | Driver matching, trip states | `lld/design-cab-booking.md` |
| 36 | Food Delivery (Swiggy/Zomato LLD) | Order lifecycle, restaurant, cart | `lld/design-food-delivery.md` |
| 37 | Logger System | Singleton, chain of responsibility | `lld/design-logger.md` |
| 38 | Pub/Sub Event Bus | Observer, decoupled messaging | `lld/design-pub-sub.md` |
| 39 | In-Memory Cache (like LRU Cache) | Data structures + OOP together | `lld/design-lru-cache.md` |
| 40 | Splitwise | Expense splitting, balance tracking | `lld/design-splitwise.md` |

---

## Study Plan

### 2-Month Overview

| Week | Focus | Topics |
|------|-------|--------|
| Week 1 | HLD Foundations + LLD OOP Foundations | HLD #1–7, LLD #1–5 |
| Week 2 | HLD Scalability + LLD Creational Patterns | HLD #8–14, LLD #6–10 |
| Week 3 | HLD Infrastructure + LLD Structural & Behavioral Patterns | HLD #15–21, LLD #11–25 |
| Week 4 | HLD Storage & Data + LLD Case Studies begin | HLD #22–28, LLD #26–32 |
| Week 5 | HLD Case Studies (Part 1) + LLD Case Studies (Part 2) | HLD #29–36, LLD #33–40 |
| Week 6 | HLD Case Studies (Part 2) + Buffer / Revision | HLD #37–42, Revise weak areas |
| Week 7 | Mock Designs — timed practice (HLD) | 2 full designs/day with time-boxing |
| Week 8 | Mock Designs — timed practice (LLD) + Final Revision | 2 full designs/day with time-boxing |

## Resources

These are the canonical references to use — avoids the noise of scattered GitHub repos.

### Books
| Resource | What it covers |
|----------|---------------|
| *Designing Data-Intensive Applications* — Martin Kleppmann | Best book on HLD internals — databases, replication, consistency |
| *System Design Interview* Vol. 1 & 2 — Alex Xu | Great structured walkthroughs of HLD case studies |
| *Clean Code* — Robert C. Martin | LLD foundations, naming, functions |
| *Head First Design Patterns* | Most approachable intro to design patterns |
| *Design Patterns* — GoF (Gang of Four) | The canonical reference for all 23 patterns |

### Free Online
| Resource | Link |
|----------|------|
| ByteByteGo Blog (Alex Xu) | https://blog.bytebytego.com |
| Gaurav Sen YouTube | https://www.youtube.com/@gkcs |
| sudoCODE YouTube | https://www.youtube.com/@sudocode |
| Concept && Coding (Shreyansh Jain) | https://www.youtube.com/@ConceptAndCoding |
| Refactoring.Guru (Design Patterns) | https://refactoring.guru/design-patterns |
| GitHub: donnemartin/system-design-primer | https://github.com/donnemartin/system-design-primer |
| GitHub: ashishps1/awesome-system-design-resources | https://github.com/ashishps1/awesome-system-design-resources |

> **Tip:** You don't need all of these. For HLD case studies → Alex Xu Vol 1 & 2 + ByteByteGo.
> For LLD → Refactoring.Guru + Concept && Coding. For internals → Kleppmann.
> Everything else is supplementary.