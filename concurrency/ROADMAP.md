---
Status: 🌳 Evergreen
Created: 2026-11-24
Last Updated: 2026-11-24
---

# Concurrency Interview Preparation Roadmap

## Overview

A long-term reference for concurrency and parallelism interview rounds. Go is the only language. The schedule is about 6 hours a week for roughly 8 weeks, running alongside the DSA track. Phases 1 to 4 cover the concepts and problems asked in most concurrency rounds; Phase 5 is mixed practice and mocks.

Scope is interview preparation only. Topics that matter in production but rarely in an interview (profiling, circuit breakers, retries, deployment tuning) are left out.

## What This Covers

The material is split into three layers. Each answers a different question.

| Layer | Question It Answers | Example |
|---|---|---|
| Fundamentals | What is actually running, and what guarantees do I get? | Scheduler, memory model |
| Primitives | Which building block coordinates these goroutines? | Channels, mutex, semaphore, atomics |
| Patterns | How do I compose blocks into a known solution shape? | Worker pool, pipeline, lock ordering |

Mixing the layers is the common mistake: memorising a worker pool without knowing why the channel close ends the loop, or knowing every `sync` type without recognising which problem shape calls for which.

## Directory Structure

```
concurrency/
├── ROADMAP.md
├── fundamentals/
│   ├── concurrency-vs-parallelism.md
│   ├── goroutines-and-scheduler.md
│   └── memory-model.md
├── primitives/
│   ├── channels.md
│   ├── select-and-timers.md
│   ├── context.md
│   ├── sync-primitives.md
│   ├── semaphores-and-barriers.md
│   └── atomics.md
└── patterns/
    ├── concurrency-patterns.md
    ├── locking-strategies.md
    ├── backpressure-and-buffering.md
    └── pitfalls.md
```

13 files in total.

## Session Format

Each topic follows four steps:

1. Teach the concept: what it is, why it exists, why it is named that, and what problems it brings.
2. Show an annotated Go template for the core technique.
3. Work through 4 to 8 curated problems.
4. Save the material as a markdown file in this folder.

Every solution must pass `go test -race` before it goes into a file.

## Hint Discipline

1. Attempt the problem for at least 15 minutes.
2. Read Hint 1. Retry for at least 10 minutes.
3. Read Hint 2. Retry.
4. Only then open the solution.

Never open the solution before writing down some approach, even a wrong one. For concurrency, "an approach" means naming which goroutines exist and which primitive hands control between them.

## Problem Format

Each problem entry has a title with link and difficulty, a Brute Force section, two collapsible hints, and a collapsible Go solution.

````markdown
**1115. Print FooBar Alternately** · [LeetCode](https://leetcode.com/problems/print-foobar-alternately/) · Medium

**Brute Force:** Spin on a shared boolean flag. Without synchronisation this is a data race,
and with a mutex it burns CPU busy-waiting.

<details>
<summary>Hint 1</summary>
Each goroutine must wait for permission before printing, then grant permission to the other.
</details>

<details>
<summary>Hint 2</summary>
Two unbuffered channels, one per direction, act as the permission tokens.
</details>

<details>
<summary>Solution</summary>

```go
// Go solution
```

**Why this works:** ...
</details>
````

Brute Force for concurrency problems means the naive version (unsynchronised, busy-waiting, or sleep-based) and exactly how it breaks: race, deadlock, wasted CPU, or nondeterministic ordering.

## Classical Problems

Famous problems live inside the file that teaches their core primitive, not in separate files.

| Problem | Difficulty | Home File |
|---|---|---|
| [Parallel Letter Frequency](https://exercism.org/tracks/go/exercises/parallel-letter-frequency) | Medium | `concurrency-vs-parallelism.md` |
| Odd and even printer with two goroutines | Interview Classic | `channels.md` |
| [1114. Print in Order](https://leetcode.com/problems/print-in-order/) | Easy | `channels.md` |
| [1115. Print FooBar Alternately](https://leetcode.com/problems/print-foobar-alternately/) | Medium | `channels.md` |
| Timeout wrapper around a slow call | Interview Classic | `select-and-timers.md` |
| Query N replicas, return the first answer, cancel the rest | Interview Classic | `context.md` |
| [1195. Fizz Buzz Multithreaded](https://leetcode.com/problems/fizz-buzz-multithreaded/) | Medium | `sync-primitives.md` |
| Bounded blocking queue | Interview Classic | `sync-primitives.md` |
| Readers-writers with writer preference | Interview Classic | `sync-primitives.md` |
| [1116. Print Zero Even Odd](https://leetcode.com/problems/print-zero-even-odd/) | Medium | `semaphores-and-barriers.md` |
| [1117. Building H2O](https://leetcode.com/problems/building-h2o/) | Medium | `semaphores-and-barriers.md` |
| Lock-free counter and Treiber stack | Interview Classic | `atomics.md` |
| Worker pool that preserves input order | Interview Classic | `concurrency-patterns.md` |
| Concurrent web crawler ([Go Tour exercise](https://go.dev/tour/concurrency/10)) | Interview Classic | `concurrency-patterns.md` |
| Parallel merge sort with depth cutoff | Interview Classic | `concurrency-patterns.md` |
| Word frequency across many files | Interview Classic | `concurrency-patterns.md` |
| [1226. The Dining Philosophers](https://leetcode.com/problems/the-dining-philosophers/) | Medium | `locking-strategies.md` |
| Bank transfer between accounts without deadlock | Interview Classic | `locking-strategies.md` |
| Thread-safe LRU cache with TTL | Interview Classic | `locking-strategies.md` |
| Token bucket rate limiter | Interview Classic | `backpressure-and-buffering.md` |
| Pub/sub broker with slow subscribers | Interview Classic | `backpressure-and-buffering.md` |
| Spot-the-bug drills (race, deadlock, leak) | Interview Classic | `pitfalls.md` |

**Excluded:** Santa Claus, Cigarette Smokers, and Sleeping Barber puzzles (rarely asked), lock-free queues (too long for an interview slot), garbage collector internals, profiling and tracing tools, and resilience patterns such as circuit breakers and retries.

## Phase 1: Foundations (Week 1, ~6h)

*What a goroutine is, physically, and why Go can run a million of them.*

| File | Topic | Type |
|---|---|---|
| `fundamentals/concurrency-vs-parallelism.md` | Definitions, processes vs threads vs goroutines, CPU bound vs I/O bound work, Amdahl's law, `GOMAXPROCS` | Concept |
| `fundamentals/goroutines-and-scheduler.md` | Goroutine vs OS thread, GMP model, run queues, work stealing, blocking syscall handoff, preemption | Concept |

## Phase 2: Channels And Context (Weeks 2–3, ~12h)

*Predict the behaviour of any channel operation without running the code.*

| File | Topic | Type |
|---|---|---|
| `primitives/channels.md` | Unbuffered vs buffered, ownership (only the sender closes), nil and closed channel behaviour, signaling by close, `range` over channel | Primitive |
| `primitives/select-and-timers.md` | `select` semantics, `default` for non-blocking ops, nil-channel trick, for-select loop, timeouts, `time.After` vs `time.NewTimer` | Primitive |
| `primitives/context.md` | Cancellation tree, `WithCancel`, `WithTimeout`, `WithDeadline`, propagation rules, `WithValue` misuse | Primitive |

## Phase 3: Sharing Memory Safely (Weeks 4–5, ~12h)

*Choose between a channel, a mutex, a semaphore, and an atomic with a reason.*

| File | Topic | Type |
|---|---|---|
| `primitives/sync-primitives.md` | `Mutex`, `RWMutex`, `WaitGroup`, `Once`, `Cond` (lost and spurious wakeup, why `Wait` sits in a loop), `sync.Map`, channel vs mutex decision rule | Primitive |
| `primitives/semaphores-and-barriers.md` | Counting semaphore as a buffered channel, weighted semaphore, barrier, countdown latch | Primitive |
| `primitives/atomics.md` | Typed atomics, compare-and-swap loops, ABA problem | Primitive |
| `fundamentals/memory-model.md` | Happens-before, definition of a data race, which operations synchronise, race detector | Concept |

## Phase 4: Patterns (Weeks 6–7, ~12h)

*Build any common concurrent component from memory in under 20 minutes.*

| File | Topic | Type |
|---|---|---|
| `patterns/concurrency-patterns.md` | Generator, pipeline, fan-out and fan-in, worker pool, producer-consumer, bounded parallelism, `errgroup` | Pattern |
| `patterns/locking-strategies.md` | Coarse vs fine-grained locking, lock striping, lock ordering, optimistic vs pessimistic locking, why Go has no reentrant mutex | Pattern |
| `patterns/backpressure-and-buffering.md` | What a buffer buys you, block vs drop, bounded queues, token bucket and leaky bucket | Pattern |
| `patterns/pitfalls.md` | Deadlock (four Coffman conditions), livelock, starvation, goroutine leaks, loop variable capture, misplaced `WaitGroup.Add`, copying a mutex, concurrent map writes, double close | Pattern |

## Phase 5: Mixed Practice And Mocks (Week 8, ~6h)

*Solve under a timer while explaining out loud.*

- Re-solve one problem from each file without notes.
- Three 45-minute mocks: one synchronization puzzle, one concurrent data structure (blocking queue or LRU cache), one component build (worker pool, crawler, or rate limiter).
- One spot-the-bug round: five unseen snippets, two minutes each.

## Priority If Time Is Short

Complete phases in order and stop when time runs out.

| Phases Completed | What You Can Handle |
|---|---|
| 1–2 | Conceptual questions on goroutines and channels; simple ordering puzzles |
| 1–3 | All LeetCode concurrency puzzles and race-focused spot-the-bug questions |
| 1–4 | Concurrent data structures, component builds, and deadlock reasoning |
| 1–5 | Timed performance under interview conditions |

## References

- [The Go Memory Model](https://go.dev/ref/mem): the official happens-before specification
- [Concurrency Is Not Parallelism](https://go.dev/blog/waza-talk): Rob Pike's talk framing the core distinction
- [Go Concurrency Patterns](https://go.dev/talks/2012/concurrency.slide): generator, fan-in, timeout, and quit patterns
- [Advanced Go Concurrency Patterns](https://go.dev/blog/io2013-talk-concurrency): select loops and the nil-channel trick
- [Go Concurrency Patterns: Pipelines and Cancellation](https://go.dev/blog/pipelines): pipeline stages and cancellation
- [Go Concurrency Patterns: Context](https://go.dev/blog/context): how context propagates
- [Scalable Go Scheduler Design Doc](https://golang.org/s/go11sched): the original GMP design
- [A Tour of Go: Concurrency](https://go.dev/tour/concurrency/1): exercises including the web crawler
- [LeetCode Concurrency Problem Set](https://leetcode.com/problemset/concurrency/): synchronization puzzles
- Concurrency in Go, Katherine Cox-Buday (O'Reilly): book-length treatment of Go concurrency patterns
- [go-concurrency-exercises](https://github.com/loong/go-concurrency-exercises): test-verified Go exercises (crawler limits, producer-consumer, cache race) for Phases 2 to 4