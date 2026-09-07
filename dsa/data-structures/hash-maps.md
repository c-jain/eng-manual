---
Status: 🌳 Evergreen
Created: 2026-09-07
Last Updated: 2026-09-07
---

# Hash Maps

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Hash Map"](#why-hash-map)
3. [The Hash Function](#the-hash-function)
4. [Collision Resolution](#collision-resolution)
5. [Load Factor and Resizing](#load-factor-and-resizing)
6. [Go Map Internals](#go-map-internals)
7. [Hash Sets](#hash-sets)
8. [Frequency Counting: The Interview Workhorse](#frequency-counting-the-interview-workhorse)
9. [Complexity Reference](#complexity-reference)
10. [Go Template Code](#go-template-code)
11. [Where JavaScript Differs](#where-javascript-differs)
12. [LeetCode Problems](#leetcode-problems)
13. [References](#references)

## What It Is and Why It Exists

A hash map is a data structure that stores key-value pairs and answers `get`, `put`, and `delete` in **expected O(1)** time. It sits between an array (O(1) access but only by integer index in a dense range) and a search tree (O(log n) access by any comparable key). It gives you array-like speed while accepting any key you can hash: strings, structs, tuples, anything.

The reason it exists is a specific gap. Suppose you need to answer "have I seen this URL before?" over a stream of URLs. An array indexed by URL is impossible because URLs are strings, not integers in `[0, n)`. A sorted list gives you O(log n) lookup but O(n) insert. A binary search tree gives O(log n) for both, but every operation touches log n cache lines and involves pointer chasing. A hash map compresses any key down to an array index via a hash function, then does array-speed lookup. That is the trick, all of it.

### The Problems It Introduces

Compression from a huge key space (all possible strings) down to a small index range (say, 16 buckets) is not injective. Two different keys can compress to the same index. This is a **collision**, and every hash map's design is dominated by how it handles them. There are two families of solutions — chaining and open addressing — and each has its own quirks.

The other cost is **loss of ordering**. Keys hash to arbitrary buckets, so iteration order is unrelated to insertion order or key order. If you need ordering, you need either a different structure (tree map, LinkedHashMap) or a parallel data structure. Go goes one step further and **deliberately randomizes iteration order** per run to stop code from depending on any particular order (see the Go section).

Third: **worst-case O(n)**. If every key hashes to the same bucket (a pathological input or a poor hash function under adversarial conditions), every operation degrades to O(n). Serious implementations defend against this with cryptographic-strength seeds or by switching bucket structures to trees under high collision counts.

### Cache Lines and Cache Locality

A **cache line** is the unit of data the CPU moves between main memory and its cache. On most modern CPUs it is **64 bytes**. You never fetch a single byte or a single int from RAM — the hardware fetches the entire 64-byte line containing it and caches that line, betting that you will access nearby bytes next.

This matters because RAM is slow. A cache hit costs roughly 1 ns; a cache miss (going out to main memory) costs roughly 100 ns. That is a 100x gap, so real-world speed depends less on operation count and more on **how many cache misses each operation triggers**.

**Cache locality** is a design property: data structures that keep related items packed into the same cache line win, and structures that scatter items across the heap lose. Two direct consequences relevant to hash maps:

- A binary search tree node lives at whatever heap address the allocator picks. Its children are separate allocations at unrelated addresses. A depth-`log n` lookup makes `log n` pointer hops, each likely a cold cache line — that is what "touches log n cache lines" means. `O(log n)` operations with a per-operation constant of one cache miss can be slower than `O(1)` operations that fit inside a single already-loaded line.
- A hash map with open addressing (or Go's 8-entries-per-bucket layout) packs many keys contiguously. One cache line typically holds several keys at once. A lookup usually costs a single cache miss to fetch the bucket, then scans it in-cache. That is why open addressing often beats chaining in practice even at similar load factors, and why hash maps often beat trees despite the same asymptotic bound in the worst case.

The same principle explains why arrays beat linked lists for sequential access, why Go slices beat linked structures for iteration-heavy code, and why "asymptotically slower" algorithms sometimes win on modern hardware. Keep this in mind wherever "pointer chasing" appears as a criticism: the real cost is usually not the pointer, it is the cache miss the pointer causes.

## Why "Hash Map"

"Hash" comes from the culinary sense — chop and mix. A hash function chops the key's bits up and mixes them together to produce a small integer. The output looks nothing like the input; that scrambling is the whole point (an unscrambled function like "first character" gives everything starting with 'a' the same slot).

"Map" is the mathematical term: a function from a set of keys to a set of values. Together, "hash map" is exactly what it says — a mapping backed by hashing. Older names for the same idea include **hash table** (emphasizing the underlying array), **dictionary** (Python), and **associative array** (PHP, awk). They all refer to the same data structure.

## The Hash Function

A hash function takes a key of arbitrary size (a string, a struct, an integer) and returns a fixed-size integer in `[0, 2^64)`. That integer is then reduced to a bucket index by `hash(key) % numBuckets` (or a bitmask when `numBuckets` is a power of two, which is faster).

Three properties matter for a hash map:

1. **Deterministic.** The same key must always produce the same hash within one process. Otherwise, you would never find what you stored.
2. **Uniform.** The hash values should spread across the output range as evenly as possible so that keys distribute evenly across buckets. Clustering wastes buckets and inflates collision chains.
3. **Fast.** The hash is computed on every operation. Cryptographic hashes (SHA-256) are uniform but too slow for a hash map's hot path; hash maps use non-cryptographic hashes like FNV, xxHash, or SipHash.

Runtime-seeded hashes (Go uses a per-process random seed for its map hash) defend against **hash flooding**: an attacker who knows the exact hash function can craft inputs that all collide, degrading operations to O(n) and creating a denial-of-service vector. Randomizing the seed makes that attack infeasible without inside knowledge.

## Collision Resolution

Two keys `k1` and `k2` collide when `hash(k1) % numBuckets == hash(k2) % numBuckets`. The map must still store both and retrieve each unambiguously. Two schemes dominate.

### Chaining (Separate Chaining)

Each bucket holds a small collection (linked list, dynamic array, or a small tree) of all entries whose keys hashed to that bucket. On `get`, hash to the bucket, then linearly scan the collection comparing full keys.

```
Chaining: buckets are heads of small collections

bucket 0 -> [ (k7,v7) ]
bucket 1 -> [ (k3,v3) -> (k9,v9) ]      collision resolved here
bucket 2 -> (empty)
bucket 3 -> [ (k1,v1) -> (k5,v5) -> (k2,v2) ]
```

**Pros:** simple to implement, degrades gracefully as it fills, deletion is trivial (unlink from the chain).
**Cons:** every bucket entry is a pointer dereference. Poor cache locality once chains grow.

### Open Addressing

Every entry lives directly inside the bucket array. On collision, probe forward (linear probing: `+1, +2, +3, ...`; quadratic probing: `+1, +4, +9, ...`; double hashing: step size determined by a second hash) until an empty slot is found.

```
Open addressing (linear probing): everything lives in one flat array

index:  0    1    2    3    4    5    6    7
        [k7] [k3] [k9] [ - ] [k1] [k5] [k2] [ - ]

k3 hashed to 1. k9 also hashed to 1 -> probed to 2 (next empty).
k1 hashed to 4. k5 also hashed to 4 -> probed to 5. k2 to 4 -> probed to 6.
```

**Pros:** excellent cache locality (single array, no pointer chasing). Often the fastest choice at modest load factors.
**Cons:** deletion is tricky. You cannot just clear a slot, because a later probe for a different key might stop early at the empty slot even though its target lives further along the probe chain. Solutions: tombstones (mark slot as "deleted but keep probing past it") or shift-back deletion. Also more sensitive to load factor: performance falls off a cliff past ~70% full because probe chains explode.

### Where This Lives in Memory

Framed against the four segments:

- **Stack:** the map header (pointer or handle) when declared as a local variable. In Go, this is the `*hmap` pointer that `make(map[K]V)` returns.
- **Heap:** all bucket arrays, overflow buckets, and stored key/value pairs. Maps grow, so their storage cannot live on the stack.
- **Data/BSS Segment:** not typically used for maps. A global `var m = map[string]int{...}` still allocates the underlying storage on the heap at init time; only the header pointer sits in Data.
- **Text/Code Segment:** the hash function and map runtime routines (`runtime.mapaccess`, `runtime.mapassign` in Go). Fixed code, shared across all maps in the process.

## Load Factor and Resizing

**Load factor** is `count / numBuckets` — how full the map is. High load factor means longer collision chains (chaining) or longer probe sequences (open addressing), both of which degrade operations toward O(n). Every hash map defines a threshold (commonly ~0.75 for chaining, ~0.5 to 0.7 for open addressing) beyond which it **resizes**: allocates a new bucket array of roughly double the size, then rehashes every existing key into the new array (because `hash % newSize` differs from `hash % oldSize`).

The resize is O(n), but it happens rarely enough that the **amortized** cost per insert stays O(1). Ten thousand inserts might trigger a few resizes; averaging their cost across all inserts keeps the per-insert cost constant in expectation.

The catch: individual inserts can spike to O(n) latency when a resize fires. Latency-sensitive systems either presize the map (`make(map[K]V, expectedN)` in Go, which allocates enough buckets up front to avoid growth) or use **incremental rehashing** where the resize work is split across many subsequent operations. Go's runtime uses incremental rehashing since Go 1.9: after growth is triggered, each subsequent map operation migrates one or two old buckets into the new array until the old array is fully drained.

## Go Map Internals

Go's `map` is a hash table with chaining, but its bucket layout is unusual and worth knowing because it directly explains several observable behaviors.

### Buckets Hold Eight Entries

Each bucket in Go's map is not a single slot; it holds up to **8 key-value pairs** in a flat layout. The bucket header stores the top 8 bits of each key's hash (an array of 8 `tophash` bytes). On lookup, the runtime first compares the incoming key's tophash bits against each of the 8 tophash bytes — a fast integer scan — and only falls back to a full key comparison for slots where the tophash matches.

```
One bucket (bmap):
+----------------------------------------------------------+
|  tophash[0..7]  |  keys[0..7]  |  values[0..7]  |  overflow  |
+----------------------------------------------------------+
      8 bytes         8 x K          8 x V           *bmap
```

Keys and values are stored in separate contiguous groups within the bucket (all 8 keys together, then all 8 values together), not interleaved. This layout keeps keys packed for cache-friendly scanning during lookup.

### Overflow Buckets

When a bucket's 8 slots fill and another key hashes to that bucket, an **overflow bucket** is allocated and linked from the parent bucket's `overflow` pointer. Long overflow chains are the visible symptom of a bad hash distribution or an undersized initial capacity. The runtime uses overflow chain length as one of the triggers for a growth cycle.

### Iteration Order Is Deliberately Randomized

Every `for k, v := range m` in Go picks a random starting bucket and a random offset within that bucket. This is **intentional** and enforced by the runtime — Go's designers wanted to prevent code from silently depending on any specific iteration order, because that order is not part of the language contract and could change between versions.

If you need a stable order, collect keys into a slice and sort:

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### Comparable Keys Only

Map keys must be **comparable** in Go's sense: types on which `==` is defined. That excludes slices, maps, and functions. Structs are comparable if all their fields are, so a struct of two ints works as a key; a struct containing a slice does not. This is why composite keys (multi-field lookups) in Go typically use structs or fixed-size arrays (`[26]int` for a letter frequency signature), not slices.

### Nil Map Reads vs Writes

Reading from a nil map returns the zero value silently (`m[k]` returns `0` for an int-valued nil map). **Writing to a nil map panics** at runtime with "assignment to entry in nil map". This trips up code that declares `var m map[K]V` (which produces a nil map) instead of `m := make(map[K]V)`.

### Presizing Matters

`make(map[K]V, hint)` allocates enough buckets up front for roughly `hint` entries without triggering growth. When you know the expected size, always presize — it avoids multiple grow cycles and reduces GC pressure from discarded old bucket arrays. For a map you are going to fill with `n` items, `make(map[K]V, n)` is a real optimization, not a micro-tweak.

## Hash Sets

A hash **set** is a hash map that only stores keys, no values. Every hash map implementation gives you a set for free: use the map with a zero-byte value type.

In Go, the idiomatic zero-byte type is `struct{}`, which occupies **exactly zero bytes** in memory (verified by `unsafe.Sizeof(struct{}{}) == 0`). Compared to `map[int]bool`, using `map[int]struct{}` saves one byte per entry (a bool is one byte in Go), which adds up on large sets and makes the intent obvious to readers: this map is being used as a set.

```go
seen := make(map[int]struct{})
seen[42] = struct{}{}      // insert
_, exists := seen[42]      // lookup — the value doesn't matter
delete(seen, 42)           // remove
```

Some languages ship a dedicated `Set` type (Java's `HashSet`, Python's `set`, JavaScript's `Set`). Go does not, so `map[K]struct{}` is the standard-library idiom.

## Frequency Counting: The Interview Workhorse

The single most common interview use of a hash map is **counting occurrences**. `map[K]int` where the value increments on every occurrence. This solves an enormous class of problems: anagram detection, most-frequent element, first non-repeating character, valid parentheses with variants, subarray-sum-equals-K (combined with prefix sum), and many others.

The recurring subtlety when the map is used as a **distinct-count check** (i.e., `len(m)` is your signal): after decrementing a count to zero, you must `delete` the key, not leave a zero entry. `len(m)` counts keys, not non-zero values.

```go
freq[c]--
if freq[c] == 0 {
    delete(freq, c)   // required if len(freq) is used as the distinct-value count
}
```

If `len(m)` is not your signal (you only ever read specific counts), leaving zero entries is harmless. Know which mode you are in.

For small, dense key spaces (lowercase letters, ASCII bytes, digit characters), replace the map with a **fixed-size array**: `[26]int`, `[128]int`, `[10]int`. Arrays skip the hashing overhead entirely and give better cache behavior. This is not premature optimization; it is the standard move for anagram-family problems where keys are known bounded characters.

## Complexity Reference

| Operation | Average | Worst Case | Notes |
|---|---|---|---|
| Insert (`m[k] = v`) | O(1) | O(n) | Worst case on catastrophic collisions or during a resize |
| Lookup (`m[k]`) | O(1) | O(n) | Worst case when a bucket's chain holds every key |
| Delete (`delete(m, k)`) | O(1) | O(n) | Same reasoning as lookup |
| Iterate (`for range m`) | O(n) | O(n + buckets) | Empty buckets are still scanned; a sparse map after many deletes iterates slowly |
| Presized `make(map[K]V, n)` fill | O(n) total | O(n) | No growth cycles fired |
| Unsized `make(map[K]V)` fill of n items | O(n) amortized | O(n log n) worst | Multiple resizes, but each key is rehashed O(log n) times worst case |

Space is O(n) but with a constant factor typically **1.5x to 2x** the raw key/value bytes — bucket arrays are kept at a load factor below 1.0, plus per-bucket tophash bytes and overflow pointers add overhead.

## Go Template Code

```go
package hashmaps

// FrequencyCount returns a value -> count map over the input.
// Time: O(n) | Space: O(n)
func FrequencyCount(nums []int) map[int]int {
    freq := make(map[int]int, len(nums))
    for _, v := range nums {
        freq[v]++
    }
    return freq
}

// FirstDuplicate returns the first value seen twice while scanning left to right,
// or -1 if all values are unique. Uses a set (map[int]struct{}) so the value
// itself carries no payload; struct{} occupies zero bytes.
// Time: O(n) | Space: O(n)
func FirstDuplicate(nums []int) int {
    seen := make(map[int]struct{}, len(nums))
    for _, v := range nums {
        if _, ok := seen[v]; ok {
            return v
        }
        seen[v] = struct{}{}
    }
    return -1
}

// GroupBy groups strings by an equivalence key derived from each string.
// Composite keys (strings, arrays of comparable types, structs of comparables)
// work directly as map keys; slices, maps, and functions do not because they
// are not comparable in Go.
// Time: O(n * cost(keyFn)) | Space: O(n)
func GroupBy(items []string, keyFn func(string) string) map[string][]string {
    groups := make(map[string][]string)
    for _, s := range items {
        k := keyFn(s)
        groups[k] = append(groups[k], s)
    }
    return groups
}
```

## Where JavaScript Differs

JavaScript has two hash-map-like structures with different tradeoffs.

**Plain objects (`{}`)** use string keys only. Numeric keys are silently coerced to strings (`obj[1]` and `obj["1"]` hit the same slot). Objects also inherit from `Object.prototype`, so keys like `"toString"` or `"hasOwnProperty"` collide with inherited properties. For general-purpose maps, avoid plain objects.

**`Map`** is the modern hash map: any value can be a key (including objects, where identity is the hash), insertion order is preserved and iterated deterministically (opposite of Go's randomized order), and there is no prototype pollution risk.

```javascript
const m = new Map();
m.set("a", 1);
m.set(42, "n");
m.set({id: 1}, "obj");  // object identity as key
m.get("a");             // 1
m.has(42);              // true
m.delete("a");
m.size;                 // 2

// Iteration is insertion order
for (const [k, v] of m) { /* ... */ }
```

For sets, use `Set` (analogous to `Map` but keys only).

The key mental switch coming from Go: JS `Map` **guarantees insertion order**, while Go maps **guarantee randomization**. Code portable between the two must not depend on either behavior.

## LeetCode Problems

### 1. Two Sum — [#1 (Easy)](https://leetcode.com/problems/two-sum/)

Given an integer array and a target, return indices of the two numbers that add up to target. Assume exactly one solution and no element used twice.

<details>
<summary>Brute Force</summary>

For each index `i`, scan every `j > i` and check whether `nums[i] + nums[j] == target`.

Time: O(n²). Space: O(1).

The waste: for each `i`, the inner scan asks "does the value `target - nums[i]` appear anywhere in the rest of the array?" That is a membership question, which is exactly what a hash map answers in O(1).
</details>

<details>
<summary>Hint 1</summary>

For each element `nums[i]`, what specific value would complete the pair? If you had a data structure that answered "does value X exist and where?" in O(1), how many passes would you need?
</details>

<details>
<summary>Hint 2</summary>

Store each value's index in a map as you iterate. Before storing the current value, first check whether its complement (target minus current value) already sits in the map.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums))
    for i, v := range nums {
        if j, ok := seen[target-v]; ok {
            return []int{j, i}
        }
        seen[v] = i
    }
    return nil
}
```

Single pass. The check happens **before** the store, so an element cannot pair with itself.
</details>

### 2. Contains Duplicate — [#217 (Easy)](https://leetcode.com/problems/contains-duplicate/)

Return true if any value appears at least twice in the array.

<details>
<summary>Brute Force</summary>

Two nested loops comparing every pair: O(n²) time, O(1) space.

Alternative: sort first, then scan adjacent pairs — O(n log n) time, O(1) or O(n) space depending on whether sort is in-place.

The hash set version wins on time complexity by trading in-place work for O(n) space.
</details>

<details>
<summary>Hint 1</summary>

Scan the array once. What do you need to remember about earlier elements to detect a repeat when you see the current one?
</details>

<details>
<summary>Hint 2</summary>

A set of previously seen values. Check membership before inserting.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func containsDuplicate(nums []int) bool {
    seen := make(map[int]struct{}, len(nums))
    for _, v := range nums {
        if _, ok := seen[v]; ok {
            return true
        }
        seen[v] = struct{}{}
    }
    return false
}
```

`struct{}` value avoids the one-byte cost of `map[int]bool`.
</details>

### 3. Valid Anagram — [#242 (Easy)](https://leetcode.com/problems/valid-anagram/)

Given strings `s` and `t`, return true if `t` is an anagram of `s` (same multiset of characters).

<details>
<summary>Brute Force</summary>

Sort both strings and compare. O(n log n) time, O(n) space (Go's sort on strings requires converting to `[]byte` or `[]rune`).

The counting version drops to O(n) time by observing that anagrams have identical character frequencies, and frequency comparison does not need sorted order.
</details>

<details>
<summary>Hint 1</summary>

Two anagrams must have identical character frequencies. What does that suggest about the data you need to compare?
</details>

<details>
<summary>Hint 2</summary>

Since the input is constrained to lowercase English letters, a fixed 26-slot integer array is faster than a map — no hashing overhead. Increment for one string, decrement for the other; if every slot ends at zero, they are anagrams.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(1) — the 26-slot array is bounded
func isAnagram(s, t string) bool {
    if len(s) != len(t) {
        return false
    }
    var freq [26]int
    for i := 0; i < len(s); i++ {
        freq[s[i]-'a']++
        freq[t[i]-'a']--
    }
    for _, c := range freq {
        if c != 0 {
            return false
        }
    }
    return true
}
```

For the follow-up "what if the input contains Unicode?", replace `[26]int` with `map[rune]int` and iterate with `for _, r := range s` (which yields runes, not bytes).
</details>

### 4. Group Anagrams — [#49 (Medium)](https://leetcode.com/problems/group-anagrams/)

Given an array of strings, group the anagrams together. Order of groups and within groups does not matter.

<details>
<summary>Brute Force</summary>

For each string, scan every already-formed group and check whether it is an anagram of the group's first element (using the counting trick from #242). O(n² · k) where n is the number of strings and k is average string length.

The optimized version replaces the "scan every group" step with an O(1) map lookup by using an anagram-signature as the map key.
</details>

<details>
<summary>Hint 1</summary>

Two anagrams share the same character-frequency profile. If you can turn that profile into a map key, every anagram lands in the same bucket in one pass.
</details>

<details>
<summary>Hint 2</summary>

A sorted version of the string works as a key, but sorting is O(k log k) per string. A `[26]int` frequency array is O(k) and, being a fixed-size comparable array, is a valid map key in Go directly.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n * k) | Space: O(n * k)
func groupAnagrams(strs []string) [][]string {
    groups := make(map[[26]int][]string)
    for _, s := range strs {
        var key [26]int
        for i := 0; i < len(s); i++ {
            key[s[i]-'a']++
        }
        groups[key] = append(groups[key], s)
    }
    out := make([][]string, 0, len(groups))
    for _, g := range groups {
        out = append(out, g)
    }
    return out
}
```

Fixed-size arrays are comparable in Go and hash directly, so `[26]int` is a valid map key. Slices are not comparable, which is why `[]int` would fail to compile as a key type. This is the cleanest Go-specific application of "arrays are comparable, slices are not."
</details>

### 5. Longest Consecutive Sequence — [#128 (Medium)](https://leetcode.com/problems/longest-consecutive-sequence/)

Given an unsorted array of integers, return the length of the longest sequence of consecutive integers. Solve in O(n) time.

<details>
<summary>Brute Force</summary>

Sort, then scan adjacent pairs counting consecutive runs. O(n log n) time.

The O(n) version requires giving up sorted order and instead using a hash set to jump around by value, plus a clever check that avoids re-walking the same run from every starting point.
</details>

<details>
<summary>Hint 1</summary>

Put every value in a set for O(1) membership. Then for each value, count the length of the run starting there by walking `v, v+1, v+2, ...` while each is in the set. But naively doing this for every value is O(n²) in the worst case — each element in a run of length L gets walked from L different starting points.
</details>

<details>
<summary>Hint 2</summary>

Only start walking from values that are **run starts**. A value `v` is a run start if and only if `v-1` is not in the set. That check is O(1), and it guarantees each run is walked exactly once across the entire scan, keeping total work O(n).
</details>

<details>
<summary>Solution (Go)</summary>

```go
// Time: O(n) | Space: O(n)
func longestConsecutive(nums []int) int {
    set := make(map[int]struct{}, len(nums))
    for _, v := range nums {
        set[v] = struct{}{}
    }
    best := 0
    for v := range set {
        if _, ok := set[v-1]; ok {
            continue // v is not a run start; skip
        }
        length := 1
        for {
            if _, ok := set[v+length]; !ok {
                break
            }
            length++
        }
        if length > best {
            best = length
        }
    }
    return best
}
```

The `v-1` check is the whole trick. Without it, the inner walk from every value would be O(n²) worst case. With it, the sum of inner walks across all run starts is exactly n, because each value is walked past exactly once (as part of exactly one run).
</details>

### 6. Insert Delete GetRandom O(1) — [#380 (Medium)](https://leetcode.com/problems/insert-delete-getrandom-o1/)

Design a data structure supporting `Insert(val)`, `Remove(val)`, and `GetRandom()`, all in O(1) average time.

<details>
<summary>Brute Force</summary>

- Array only: `Insert` O(1), `Remove` O(n) (need to find and shift), `GetRandom` O(1).
- Hash set only: `Insert` O(1), `Remove` O(1), `GetRandom` O(n) (need to iterate to a random position; sets have no indexed access).

Neither structure alone gives O(1) for all three. Combining them does, but the combination requires one non-obvious trick during removal.
</details>

<details>
<summary>Hint 1</summary>

A slice supports O(1) random pick by index. A map supports O(1) membership and lookup. Store the values in a slice for `GetRandom`, and use a map to remember each value's index in the slice.
</details>

<details>
<summary>Hint 2</summary>

The problem is removal: removing an arbitrary index from a slice is O(n) because everything after shifts. Fix this by swapping the target with the **last** element, then popping the last element — that keeps removal O(1) at the cost of not preserving order (which the problem does not require). Remember to update the map's index for the value that got swapped in.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// All operations: O(1) average
type RandomizedSet struct {
    idx  map[int]int // value -> index in vals
    vals []int
}

func Constructor() RandomizedSet {
    return RandomizedSet{idx: make(map[int]int)}
}

func (r *RandomizedSet) Insert(val int) bool {
    if _, ok := r.idx[val]; ok {
        return false
    }
    r.idx[val] = len(r.vals)
    r.vals = append(r.vals, val)
    return true
}

func (r *RandomizedSet) Remove(val int) bool {
    i, ok := r.idx[val]
    if !ok {
        return false
    }
    last := len(r.vals) - 1
    if i != last {
        lastVal := r.vals[last]
        r.vals[i] = lastVal      // swap last into the removed slot
        r.idx[lastVal] = i       // update the moved value's index
    }
    r.vals = r.vals[:last]       // pop
    delete(r.idx, val)
    return true
}

func (r *RandomizedSet) GetRandom() int {
    return r.vals[rand.Intn(len(r.vals))]
}
```

The swap-with-last-then-pop trick is the whole design. It appears in several other problems (Insert Delete GetRandom O(1) - Duplicates allowed #381, and any "remove arbitrary element from a bag in O(1)" scenario).
</details>

## References

- [Go source: runtime/map.go](https://github.com/golang/go/blob/master/src/runtime/map.go) — the actual bucket layout, tophash trick, incremental rehashing
- [Go blog: Go maps in action](https://go.dev/blog/maps) — official reference for the language-level behavior
- Sedgewick and Wayne, *Algorithms* (4th ed.), Chapter 3.4 — chaining, open addressing, load factor analysis with proofs
- [Wikipedia: Hash table](https://en.wikipedia.org/wiki/Hash_table) — comprehensive survey of collision schemes and their tradeoffs
- [MDN: Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) — JavaScript `Map` reference
- Internal: `dsa/patterns/prefix-sum.md` — prefix sum plus hash map for subarray-sum problems
- Internal: `dsa/patterns/sliding-window.md` — frequency-map maintenance across a moving window