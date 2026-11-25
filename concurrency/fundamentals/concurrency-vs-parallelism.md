---
Status: 🌳 Evergreen
Created: 2026-11-25
Last Updated: 2026-11-25
---

# Concurrency vs Parallelism

## Table of Contents

1. What It Is
2. Why It Exists
3. Why It Is Called That
4. The Four Combinations
5. Processes, Threads, And Goroutines
6. CPU Bound vs I/O Bound Work
7. Amdahl's Law
8. GOMAXPROCS
9. What Problems It Brings
10. Go Template
11. Problems
12. References
13. Related Topics

## What It Is

**Concurrency** is a way of *structuring* a program: it is split into independently executing tasks whose lifetimes overlap. Whether those tasks ever run at the same instant is not part of the definition.

**Parallelism** is a property of *execution*: two or more computations are physically running at the same instant, on different cores (or different SIMD lanes, or different machines).

Rob Pike's framing, from his talk *Concurrency Is Not Parallelism*:

> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.

Concurrency is about the code. Parallelism is about the hardware running that code. A concurrent program *can* run in parallel if cores are available, but it is still correct (and still concurrent) on one core.

Two timelines for the same three tasks A, B, C:

```
Scenario 1: Concurrency without parallelism (1 core, GOMAXPROCS=1)

Core 0 | A A B B A C C B A C
         time ------------>

(only one task runs at any instant; the scheduler interleaves them)
```

```
Scenario 2: Concurrency with parallelism (3 cores, GOMAXPROCS=3)

Core 0 | A A A A
Core 1 | B B B B
Core 2 | C C C C
         time -->

(all three run at the same instant; wall time shrinks)
```

The program text is identical in both scenarios. Only the runtime environment changed.

## Why It Exists

They solve two different problems.

**Concurrency exists because the world is concurrent and most of a program's time is spent waiting.** A server has thousands of clients whose requests overlap. A disk read takes milliseconds, a network round trip tens of milliseconds, and a CPU instruction under a nanosecond. If the program handles one thing at a time, the CPU sits idle during every wait. Concurrency lets the program make progress on task B while task A is blocked. This pays off even on a single core.

**Parallelism exists because single-core speed stopped scaling.** Around the mid 2000s clock frequencies plateaued (power and heat limits), and chip makers started adding cores instead. A sequential program gets no faster on a 16-core machine. The only way to use the extra cores is to split work so pieces run simultaneously.

Go's design follows from this split. Goroutines and channels give you a cheap way to *express* concurrency; the runtime scheduler then *maps* that concurrency onto however many cores `GOMAXPROCS` allows. You write the structure once and get parallelism for free when the hardware has it.

## Why It Is Called That

- **Concurrent** comes from Latin *concurrere*, "to run together." The tasks run together over the same period, like runners on one track who take turns at a narrow gate.
- **Parallel** comes from Greek *parallēlos*, "beside one another." Like parallel lines, the computations proceed side by side, each on its own track.

"Run together over time" vs "side by side in space" is the whole distinction.

## The Four Combinations

| | Not Parallel | Parallel |
|---|---|---|
| **Not Concurrent** | A plain sequential program | One task split across hardware lanes: SIMD vector instructions, a GPU kernel adding two arrays |
| **Concurrent** | Goroutines on `GOMAXPROCS=1`; Node.js event loop; a single-core OS running many processes | Goroutines on a multi-core machine with `GOMAXPROCS > 1` |

The bottom-left and top-right cells are where interviewers probe.

- **Concurrent but not parallel:** many tasks in flight, one running at a time. Still needs synchronisation (see GOMAXPROCS).
- **Parallel but not concurrent:** one logical task with no independent sub-tasks, executed simultaneously at the hardware level. A single SIMD instruction that adds eight numbers at once has no interleaving, no scheduling, and no races. This cell proves the two ideas are genuinely different axes, not two names for one thing.

## Processes, Threads, And Goroutines

These are three units of concurrency, each cheaper and more tightly coupled than the last.

| | Process | OS Thread | Goroutine |
|---|---|---|---|
| Managed by | OS kernel | OS kernel | Go runtime |
| Address space | Its own, isolated | Shared with other threads of its process | Shared with all goroutines in the program |
| Stack | Per thread inside it | Fixed reservation, typically 8 MB of virtual memory on Linux (glibc follows `ulimit -s`) | Starts at 2 KB (since Go 1.4), grows and shrinks by copying |
| Creation cost | Highest (new page tables, file descriptor table) | Kernel call plus stack mapping | A small runtime allocation, no kernel call |
| Context switch | Kernel switch plus address space change (TLB effects) | Kernel switch, same address space | User-space switch by the Go scheduler, a few registers saved |
| Communication | IPC: pipes, sockets, shared memory segments | Shared memory plus locks | Channels, or shared memory plus `sync` |
| Failure blast radius | One process crashing leaves others alive | A crashing thread takes the whole process down | An unrecovered panic in any goroutine takes the whole program down |

Goroutines are not a replacement for threads. They run *on* threads: the runtime multiplexes many goroutines (M:N scheduling) onto a small pool of OS threads. How that multiplexing works (the GMP model) is the subject of `goroutines-and-scheduler.md`.

## CPU Bound vs I/O Bound Work

The single most useful question to ask about any task before adding goroutines: **while it runs, what is it waiting on?**

| | CPU Bound | I/O Bound |
|---|---|---|
| Spends its time | Executing instructions | Blocked on network, disk, a database, another service |
| Examples | Hashing, image resizing, compression, sorting, JSON encoding of large payloads | HTTP calls, DB queries, reading files, waiting on a queue |
| What helps | Parallelism: more cores | Concurrency: more tasks in flight while others wait |
| Useful goroutine count | About `GOMAXPROCS`; beyond that they just queue for a P | Far more than the core count; bounded by the downstream system (connection pool, rate limit, file descriptors), not by cores |
| Speedup ceiling | Number of cores, then Amdahl's law | Latency of the slowest call, then downstream capacity |

Why I/O-bound goroutines do not need cores: when a goroutine blocks on a network read, the Go runtime parks it and registers the socket with the netpoller (epoll on Linux, kqueue on BSD and macOS). The goroutine consumes no OS thread and no CPU until data arrives. Ten thousand goroutines waiting on sockets cost roughly ten thousand small stacks and nothing else. Blocking system calls such as file reads behave differently (they do hold an OS thread, but release their P so other goroutines keep running); `goroutines-and-scheduler.md` covers the handoff.

Most real work is mixed. A request handler that calls a database (I/O) and then renders a template (CPU) is I/O bound if the query dominates. Profile before deciding.

## Amdahl's Law

Amdahl's law (Gene Amdahl, 1967) answers: **if I add cores, how much faster can this get?**

### Derivation

Normalise the single-core runtime to 1. Split it into a fraction `p` that can be parallelised and a fraction `1 − p` that cannot (setup, a final merge, anything under a single lock).

On `N` cores, the serial part takes the same time; the parallel part divides by `N`:

```
T(N) = (1 − p) + p / N
```

Speedup is old time over new time:

```
S(N) = T(1) / T(N) = 1 / ((1 − p) + p / N)
```

Let `N` go to infinity. The `p / N` term vanishes and only the serial part remains:

```
S(∞) = 1 / (1 − p)
```

### What It Means

| Parallel Fraction `p` | Serial Fraction | Max Speedup, Infinite Cores | Speedup On 8 Cores |
|---|---|---|---|
| 50% | 50% | 2x | 1.8x |
| 90% | 10% | 10x | 4.7x |
| 95% | 5% | 20x | 5.9x |
| 99% | 1% | 100x | 7.5x |

The serial fraction sets the speed limit, and it bites early: a program that is 90% parallel already loses more than 40% of its theoretical gain on just 8 cores.

In Go code, the serial fraction hides in places that do not look serial: a single `sync.Mutex` around a shared map, one goroutine that merges every result, a channel with one consumer. Every goroutine that has to pass through one of these is executing the `1 − p` part.

### Two Caveats Interviewers Like

- **Amdahl is optimistic.** It assumes splitting work is free. Real coordination costs (goroutine creation, lock contention, cache traffic) grow with `N`, so a measured speedup curve often peaks and then *falls* as cores are added.
- **Amdahl assumes a fixed problem size.** Gustafson's law (1988) points out that with more cores we usually tackle *bigger* problems, and the serial part often stays roughly constant while the parallel part grows. Scaled speedup is then `s + p × N`, which grows linearly. Careful: here `s` and `p` are the serial and parallel fractions of the time *measured on the N-core run*, not the single-core fractions Amdahl uses. Mixing the two definitions is the usual source of "Gustafson disproves Amdahl" confusion; they describe different questions. Amdahl answers "how much faster is this job?"; Gustafson answers "how much more work can I do in the same time?"

## GOMAXPROCS

### What It Is

`GOMAXPROCS` is the maximum number of OS threads that can execute Go code *simultaneously*. In scheduler terms it is the number of Ps (logical processors). It is the dial that decides how much parallelism the runtime is allowed to use. Concurrency (how many goroutines exist) is unaffected by it.

The name is literal: GO MAXimum PROCessors.

Threads blocked in system calls do not count against the limit, which is why a Go program can have more OS threads than `GOMAXPROCS`.

### Reading And Setting It

```go
n := runtime.GOMAXPROCS(0) // any n < 1 queries without changing
prev := runtime.GOMAXPROCS(4) // sets to 4, returns the old value
```

It can also be set with the `GOMAXPROCS` environment variable. `runtime.NumCPU()` is different: it reports logical CPUs usable by the process (respecting the CPU affinity mask), checked once at startup.

### How The Default Evolved

| Go Version | Default |
|---|---|
| Before 1.5 | 1. Goroutines were concurrent but never parallel unless you opted in |
| 1.5 to 1.24 | Number of logical CPUs (`runtime.NumCPU()`) at startup |
| 1.25 onward | The minimum of logical CPUs, the CPU affinity mask count, and (on Linux) the cgroup CPU limit (quota / period). Never below 2 unless the logical CPU count or affinity mask count is below 2. Re-checked periodically, so it follows changes while the process runs |

The Go 1.25 change exists because of containers. Before it, a pod with a 2-CPU limit on a 64-core node got `GOMAXPROCS=64`. Problem 4 below walks through why that hurts.

The new behaviour turns off if you set `GOMAXPROCS` yourself (env var or `runtime.GOMAXPROCS`), and can be disabled with the GODEBUG settings `containermaxprocs=0` and `updatemaxprocs=0`. Kubernetes "CPU requests" are ignored; only the "CPU limit" becomes a cgroup bandwidth limit.

### GOMAXPROCS=1 Still Needs Synchronisation

Setting `GOMAXPROCS=1` removes parallelism, not concurrency. Only one goroutine runs at a time, but the scheduler can switch between them at almost any point; since Go 1.14 it preempts asynchronously, even inside a tight loop. `counter++` is a load, an add, and a store, and a switch between the load and the store loses an update. The Go memory model defines a data race without reference to cores, and the race detector reports it regardless of `GOMAXPROCS`.

The converse also holds: parallelism never introduces new *kinds* of correctness bugs. A program that synchronises properly (rather than relying on timing) is correct on any number of cores. More cores only make latent races show up more often.

## What Problems It Brings

### Concurrency Brings Correctness Problems

- **Nondeterminism:** the interleaving changes from run to run, so a bug may appear once in ten thousand runs.
- **Data races:** two goroutines touch the same memory, at least one writes, and nothing orders them (`memory-model.md`).
- **Deadlock, livelock, starvation, goroutine leaks** (`pitfalls.md`).
- **Harder testing:** a passing test proves one interleaving worked, not all of them. `go test -race` catches races only on paths actually exercised.

### Parallelism Brings Performance Problems

This list is also the answer to "when does adding goroutines make things slower?"

1. **Work per goroutine smaller than the cost of a goroutine.** Starting, scheduling, and joining a goroutine costs far more than incrementing a counter.
2. **Everyone queues on one lock.** Time spent inside a shared critical section is the `1 − p` term of Amdahl's law; adding goroutines grows the queue, not the throughput.
3. **More CPU-bound goroutines than Ps.** With `GOMAXPROCS=8`, the ninth CPU-bound goroutine waits for a P. Extra goroutines add scheduling and cache churn.
4. **False sharing.** Cores keep caches coherent per cache line (64 bytes on most x86 and ARM server CPUs). Two goroutines on different cores writing *different* variables in the *same* line keep invalidating each other's copy: no data race, yet the hardware serialises them. Fix: accumulate in a goroutine-local variable and write shared memory once.
5. **Misconfiguration.** `GOMAXPROCS` above a container's CPU limit leads to CPU throttling (Problem 4).

## Go Template

The shape every "do these N independent things concurrently" task starts from. Each goroutine owns its output slot, so no lock is needed.

```go
func doAll(inputs []Input) ([]Output, error) {
	results := make([]Output, len(inputs)) // one slot per task
	errs := make([]error, len(inputs))

	var wg sync.WaitGroup
	for i, in := range inputs {
		wg.Add(1) // Add before go, never inside the goroutine
		go func() { // i and in are per-iteration since Go 1.22
			defer wg.Done()
			results[i], errs[i] = process(in) // distinct index = distinct memory
		}()
	}
	wg.Wait() // after this, every write above is visible here

	return results, errors.Join(errs...) // nil if every err is nil
}
```

For CPU-bound work, replace one goroutine per task with `runtime.GOMAXPROCS(0)` chunks (Problem 2). For I/O-bound work at scale, bound the number in flight (`semaphores-and-barriers.md`, `concurrency-patterns.md`).

## Problems

### Problem 1: Fetch Many Slow Resources

[Go Concurrency Patterns: Google Search 2.0](https://go.dev/talks/2012/concurrency.slide#45) · Interview Classic

Given a list of IDs and a function `fetch(id) (string, error)` that takes about 100 ms (a network call), return all results in input order. With 10 IDs the whole call should take about 100 ms, not 1 second, and this must hold even with `GOMAXPROCS=1`.

**Brute Force:** Call `fetch` in a loop. Correct and ordered, but total time is the *sum* of all waits: O(n × latency). The first naive fix, launching goroutines that `append` to a shared slice, has a data race on the slice header (`go test -race` reports it) and returns results in completion order, not input order.

<details>
<summary>Hint 1</summary>

The CPU is idle during every wait. While one call is waiting, nothing stops the next call from starting.
</details>

<details>
<summary>Hint 2</summary>

If the output size is known up front, each concurrent task can be handed its own position in the output, so no two tasks ever write to the same place.
</details>

<details>
<summary>Solution</summary>

```go
func fetch(id int) (string, error) {
	time.Sleep(100 * time.Millisecond) // stands in for a network call
	return fmt.Sprintf("result-%d", id), nil
}

func FetchAll(ids []int) ([]string, error) {
	results := make([]string, len(ids))
	errs := make([]error, len(ids))

	var wg sync.WaitGroup
	for i, id := range ids {
		wg.Add(1)
		go func() {
			defer wg.Done()
			results[i], errs[i] = fetch(id)
		}()
	}
	wg.Wait()

	for _, err := range errs {
		if err != nil {
			return nil, err
		}
	}
	return results, nil
}
```

**Why this works:** total time becomes the *maximum* latency instead of the sum, because all waits overlap. A goroutine blocked on I/O needs no CPU, so this holds on one core: concurrency without parallelism.

There is no race: `results[i]` and `results[j]` are distinct memory locations, and `wg.Wait()` creates the happens-before edge that makes every goroutine's write visible to the caller.

Unbounded fan-out is fine for 10 IDs. For 100,000 IDs it would open 100,000 connections at once; bounding it is a Phase 3 topic.
</details>

### Problem 2: Parallel Letter Frequency

[Exercism: Parallel Letter Frequency](https://exercism.org/tracks/go/exercises/parallel-letter-frequency) · Medium

Implement `ConcurrentFrequency(texts []string) FreqMap`, which counts every rune across a list of texts, and make it faster than the sequential `Frequency` on a multi-core machine.

**Brute Force:** One goroutine per text, all incrementing one shared map. Without a lock the runtime aborts with `fatal error: concurrent map writes` (Go maps are not safe for concurrent writes, and the runtime detects it). With a mutex around each increment it is correct but slower than the sequential loop: every rune passes through the same lock, so `p` in Amdahl's law is effectively zero.

<details>
<summary>Hint 1</summary>

Counting is pure CPU work. More workers than there are cores to run them cannot make it faster, and each worker should do a lot of work per unit of coordination.
</details>

<details>
<summary>Hint 2</summary>

Counts from separate pieces of input can be computed independently and added together at the end. Shared state only needs to be touched once per worker.
</details>

<details>
<summary>Solution</summary>

```go
// FreqMap records the frequency of each rune in a given text.
type FreqMap map[rune]int

// Frequency counts the frequency of each rune in a given text.
func Frequency(text string) FreqMap {
	freq := FreqMap{}
	for _, r := range text {
		freq[r]++
	}
	return freq
}

// ConcurrentFrequency counts runes across texts with one worker per P.
func ConcurrentFrequency(texts []string) FreqMap {
	workers := min(runtime.GOMAXPROCS(0), len(texts)) // CPU bound: cap at Ps
	partials := make([]FreqMap, workers)

	var wg sync.WaitGroup
	for w := range workers {
		wg.Add(1)
		go func() {
			defer wg.Done()
			// Private map: no lock, no shared writes in the hot loop.
			// Worker w takes texts w, w+workers, w+2*workers, ...
			local := FreqMap{}
			for i := w; i < len(texts); i += workers {
				for _, r := range texts[i] {
					local[r]++
				}
			}
			partials[w] = local // one write per worker
		}()
	}
	wg.Wait()

	total := FreqMap{} // serial merge: workers × distinct runes, not input length
	for _, m := range partials {
		for r, n := range m {
			total[r] += n
		}
	}
	return total
}
```

**Why this works:** one worker per P means no queueing for Ps. Each worker fills a private map, so the hot loop has no lock and no shared writes. The only serial step is merging `workers` maps, whose size is bounded by the number of distinct runes, not by input length, so `p` is close to 1 and the speedup approaches the core count for large inputs.

The unit of work is one text. With a handful of short texts (as in the Exercism tests) the goroutine overhead outweighs the work, and with one huge text only one worker is busy; splitting that text into chunks would need care to cut on rune boundaries, not byte offsets. Exercism's tests check correctness only, so measure the speedup with `go test -bench .`.

`concurrency-patterns.md` revisits counting as a map-reduce pipeline over files, where the lens is I/O plus pipeline stages rather than sizing CPU workers.
</details>

### Problem 3: Estimate The Speedup Before Writing Code

[Amdahl's Law](https://en.wikipedia.org/wiki/Amdahl%27s_law) · Interview Classic

A batch job takes 100 seconds on one core. 10 seconds of it is reading configuration and merging results, which must run serially; the remaining 90 seconds parallelises perfectly.

1. How long does it take on 8 cores?
2. What is the best possible speedup on unlimited cores?
3. How many cores are needed for a 5x speedup?
4. Your manager asks for 10x. What do you say?

**Brute Force:** "8 cores means 8x faster, so 12.5 seconds." This treats the whole job as parallel and ignores the serial 10 seconds.

<details>
<summary>Hint 1</summary>

Split the runtime into the part that shrinks as cores are added and the part that does not. Only one of them is divided by the core count.
</details>

<details>
<summary>Hint 2</summary>

As the core count grows without bound, the shrinking part approaches zero. What is left is the floor on the runtime.
</details>

<details>
<summary>Solution</summary>

Here `p = 0.9`, so the serial fraction is `0.1`.

1. `T(8) = 10 + 90 / 8 = 21.25 s`, a speedup of `100 / 21.25 ≈ 4.7x`.
2. `S(∞) = 1 / 0.1 = 10x`. The job can never finish in less than the 10 serial seconds.
3. Solve `1 / (0.1 + 0.9 / N) = 5`: `0.1 + 0.9 / N = 0.2`, so `N = 9` cores.
4. 10x is the theoretical limit, reached only with infinitely many cores and zero overhead. In practice it is unreachable. The only route is to shrink the serial part, for example by parallelising the merge (a tree reduction) or overlapping config loading with the first batch of work.

The same formulas in Go:

```go
// Speedup returns Amdahl's predicted speedup for parallel fraction p on n cores.
func Speedup(p float64, n int) float64 {
	return 1 / ((1 - p) + p/float64(n))
}

// MaxSpeedup is the limit as n goes to infinity: only the serial part remains.
func MaxSpeedup(p float64) float64 {
	return 1 / (1 - p)
}

// CoresFor solves Amdahl's formula for n: the smallest core count that
// reaches the target speedup, or -1 if the target is at or past the ceiling.
func CoresFor(p, target float64) int {
	denom := 1/target - (1 - p) // time left for the parallel part, per unit
	if denom <= 1e-12 {
		return -1
	}
	return int(math.Ceil(p/denom - 1e-9))
}
```

`CoresFor` solves the equation directly rather than looping `n++` until the target is met. A loop never terminates for a target at the ceiling: in floating point, `1 / (1 - 0.9)` is slightly above 10, so the check passes and the loop runs forever.
</details>

### Problem 4: Worker Count In A CPU-Limited Container

[Go Blog: Container-Aware GOMAXPROCS](https://go.dev/blog/container-aware-gomaxprocs) · Interview Classic

A CPU-bound image-resizing service runs in Kubernetes with a CPU limit of 2 on a 64-core node. It sizes its worker pool like this:

```go
workers := runtime.NumCPU()
```

Under load, p99 latency spikes to around 100 ms even though average CPU use looks fine. What is wrong, and how do you fix it?

**Brute Force:** "Add more workers so requests do not queue." This makes the spikes worse.

<details>
<summary>Hint 1</summary>

A container CPU limit does not hide cores. It caps how much CPU *time* the process may use in each short accounting period, and pauses the process once that budget is spent.
</details>

<details>
<summary>Hint 2</summary>

Count how many threads can be burning CPU at the same instant, and divide the per-period CPU budget by that number.
</details>

<details>
<summary>Solution</summary>

`runtime.NumCPU()` respects the CPU affinity mask but not the cgroup CPU quota, so it returns 64. On Go versions before 1.25, `GOMAXPROCS` also defaulted to 64. A limit of 2 CPUs is enforced by the Linux CFS scheduler as a quota: 200 ms of CPU time per 100 ms period (the default period). With 64 threads all running CPU-bound work, the quota is used up after about 200 / 64 ≈ 3 ms of wall time. The kernel then throttles the whole container for the remaining ~97 ms of the period. Every request in flight stalls, which shows up as tail latency spikes, while average utilisation still reads "2 CPUs".

Fix: size CPU-bound pools by what the scheduler will actually run in parallel.

```go
// cpuBoundWorkers is the pool size for work that keeps a CPU busy.
// GOMAXPROCS(0) reads the setting without changing it.
func cpuBoundWorkers() int {
	return runtime.GOMAXPROCS(0)
}
```

- **Go 1.25 and later:** the default `GOMAXPROCS` already reads the cgroup limit, so here it becomes 2 and this function returns 2.
- **Before Go 1.25:** also set `GOMAXPROCS` to match the CPU limit, either through the environment variable or with `go.uber.org/automaxprocs`, which reads the cgroup quota at startup.

**Why this works:** with 2 Ps, at most 2 threads execute Go code at once, so the process uses its 200 ms budget evenly across the 100 ms period and is never throttled. Fewer workers, lower latency.
</details>

## References

- [Concurrency Is Not Parallelism](https://go.dev/blog/waza-talk): Rob Pike's talk, the source of the structure vs execution framing
- [runtime package: GOMAXPROCS and NumCPU](https://pkg.go.dev/runtime#GOMAXPROCS): exact definition of the limit, default calculation, and the minimum-of-2 rule
- [Go 1.25 Release Notes: Container-Aware GOMAXPROCS](https://go.dev/doc/go1.25#container-aware-gomaxprocs): cgroup CPU limit, periodic updates, GODEBUG switches
- [Container-Aware GOMAXPROCS](https://go.dev/blog/container-aware-gomaxprocs): Go team's explanation of CFS throttling and the Go 1.25 default
- [Go 1.5 Release Notes](https://go.dev/doc/go1.5): default `GOMAXPROCS` changed from 1 to the number of CPUs
- [Go 1.4 Release Notes](https://go.dev/doc/go1.4): goroutine starting stack reduced to 2 KB
- [The Go Memory Model](https://go.dev/ref/mem): why a data race is defined independently of the number of cores
- [Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities](https://doi.org/10.1145/1465482.1465560): Gene Amdahl's 1967 paper
- [Reevaluating Amdahl's Law](https://doi.org/10.1145/42411.42415): John Gustafson's 1988 scaled-speedup argument
- [automaxprocs](https://github.com/uber-go/automaxprocs): sets `GOMAXPROCS` from the cgroup quota on Go versions before 1.25

## Related Topics

- [Goroutines And Scheduler](goroutines-and-scheduler.md): how goroutines are multiplexed onto threads (GMP), syscall handoff, preemption
- [Memory Model](memory-model.md): happens-before and the definition of a data race
- [Sync Primitives](../primitives/sync-primitives.md): `WaitGroup` and `Mutex` used above
- [Concurrency Patterns](../patterns/concurrency-patterns.md): bounded parallelism and worker pools built on the sizing rules here
- [Roadmap](../ROADMAP.md)