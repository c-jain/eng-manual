---
Status: 🌳 Evergreen
Created: 2026-06-29
Last Updated: 2026-06-29
---

# Math for DSA

## Table of Contents

- [Why Math Belongs in DSA](#why-math-belongs-in-dsa)
- [Number Theory](#number-theory)
  - [Primes and Primality Testing](#primes-and-primality-testing)
  - [GCD and LCM](#gcd-and-lcm)
  - [Modular Arithmetic](#modular-arithmetic)
  - [Extended Euclidean Algorithm](#extended-euclidean-algorithm)
  - [Fast Exponentiation (Binary Exponentiation)](#fast-exponentiation-binary-exponentiation)
- [Combinatorics](#combinatorics)
  - [Factorials](#factorials)
  - [Permutations and Combinations](#permutations-and-combinations)
  - [Pascal's Triangle and Precomputation](#pascals-triangle-and-precomputation)
- [Sequences and Sums](#sequences-and-sums)
- [Pigeonhole Principle](#pigeonhole-principle)
- [Catalan Numbers](#catalan-numbers)
- [LeetCode Problems](#leetcode-problems)
- [How to Remember This](#how-to-remember-this)
- [Interview Cheat Sheet](#interview-cheat-sheet)
- [References](#references)

## Why Math Belongs in DSA

Many interview problems are math problems wearing a disguise. Without a math toolkit you'll brute-force what should be a one-liner, or fail to prove a guarantee the problem already hands you.

Math for DSA serves three roles:

- **Recognition** — identifying that "count the arrangements" is combinatorics, or "largest shared divisor" is GCD
- **Efficiency** — O(log n) exponentiation instead of O(n), O(n log log n) sieve instead of O(n√n)
- **Correctness** — handling large numbers via modular arithmetic instead of silently overflowing

The topics here cluster into: Number Theory (primes, GCD, modular arithmetic), Combinatorics (counting selections), and Structural Math (sums, pigeonhole, Catalan).

## Number Theory

### Primes and Primality Testing

A **prime** is an integer > 1 with no divisors other than 1 and itself. Every integer ≥ 2 factors uniquely into primes (Fundamental Theorem of Arithmetic).

**Etymology:** Latin *primus* = "first." Primes are the irreducible building blocks of all integers.

**Trial Division — O(√n) per number**

Only check divisors up to √n. If n = a × b and a > √n, then b < √n. So if no divisor exists up to √n, none can exist above it.

**Sieve of Eratosthenes — O(n log log n) time, O(n) space**

Named after Eratosthenes of Cyrene (~276 BC). Goal: find all primes up to n in bulk, not one at a time.

Algorithm:
- Mark all integers 2..n as prime
- For each prime p, mark all multiples of p starting from p² as composite
- Why start at p²? All multiples of p smaller than p² have a prime factor < p and were already marked by a prior iteration

```
n = 20

Initial: [2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20]

p=2: mark 4,6,8,10,12,14,16,18,20  (start at 2²=4)
     [2 3 ✗ 5 ✗ 7 ✗  9  ✗ 11  ✗ 13  ✗ 15  ✗ 17  ✗ 19  ✗]

p=3: mark 9,15  (6,12,18 already marked; start at 3²=9)
     [2 3 ✗ 5 ✗ 7 ✗  ✗  ✗ 11  ✗ 13  ✗  ✗  ✗ 17  ✗ 19  ✗]

p=5: 5²=25 > 20, stop

Primes: 2, 3, 5, 7, 11, 13, 17, 19
```

O(n log log n) comes from the sum of prime reciprocals: 1/2 + 1/3 + 1/5 + ... ≈ log log n.

**Pitfalls:**
- Sieve uses O(n) memory (`[]bool` in Go = 1 byte per element). For n=10⁸ that's ~100 MB; for n=10⁹ it's ~1 GB. Use segmented sieve when n is large (see below).
- The loop condition `i*i <= n` overflows int32 for large n; use `i <= n/i` or ensure int64

```go
// isPrime — trial division, O(√n) per number
func isPrime(n int) bool {
    if n < 2 {
        return false
    }
    for i := 2; i*i <= n; i++ {
        if n%i == 0 {
            return false
        }
    }
    return true
}

// sieve — returns isPrime[i]=true iff i is prime, for 0..n
// O(n log log n) time, O(n) space
func sieve(n int) []bool {
    isPrime := make([]bool, n+1)
    for i := 2; i <= n; i++ {
        isPrime[i] = true
    }
    for i := 2; i*i <= n; i++ {
        if isPrime[i] {
            for j := i * i; j <= n; j += i {
                isPrime[j] = false
            }
        }
    }
    return isPrime
}

// primesUpTo — collects all primes up to n into a slice
func primesUpTo(n int) []int {
    sieved := sieve(n)
    primes := []int{}
    for i := 2; i <= n; i++ {
        if sieved[i] {
            primes = append(primes, i)
        }
    }
    return primes
}
```

**Segmented Sieve — O(n log log n) time, O(√n) space**

When n is too large for the plain sieve's O(n) memory, process the range in chunks of size ~√n instead. The key insight: any composite number in a chunk [lo, hi] must have a prime factor ≤ √n — so first find all primes up to √n (cheap: O(√n) memory), then use them to mark composites in each chunk one at a time.

```
Example: n=30, √30≈5, small primes = {2, 3, 5}

Segment [5, 9]:
  p=2: mark 6, 8
  p=3: mark 6, 9
  Survivors: 5, 7  → primes {5, 7}

Segment [10, 14]:
  p=2: mark 10, 12, 14
  p=3: mark 12
  p=5: mark 10
  Survivors: 11, 13  → primes {11, 13}
```

```go
// segmentedSieve — finds all primes up to n
// O(n log log n) time, O(√n) space — use when plain sieve exceeds memory limits
func segmentedSieve(n int) []int {
    if n < 2 {
        return nil
    }
    sqrtN := int(math.Sqrt(float64(n))) + 1

    // Step 1: plain sieve up to √n — O(√n) memory
    smallSieve := sieve(sqrtN)
    smallPrimes := []int{}
    for i := 2; i <= sqrtN; i++ {
        if smallSieve[i] {
            smallPrimes = append(smallPrimes, i)
        }
    }

    primes := append([]int{}, smallPrimes...)
    segSize := sqrtN

    // Step 2: sieve each segment [lo, hi] — only O(√n) memory at a time
    for lo := sqrtN + 1; lo <= n; lo += segSize {
        hi := lo + segSize - 1
        if hi > n {
            hi = n
        }
        seg := make([]bool, hi-lo+1)
        for i := range seg {
            seg[i] = true
        }
        for _, p := range smallPrimes {
            // Ceiling division: smallest multiple of p that is >= lo.
            // Formula: ⌈lo/p⌉ × p. In integer arithmetic: ((lo+p-1)/p)*p.
            // Adding (p-1) before dividing forces Go's truncating / to round up.
            start := ((lo + p - 1) / p) * p
            // Safety guard: if start landed on p itself (happens when lo <= p),
            // skip to 2p — p is prime and must not be marked composite.
            // In this implementation lo >= sqrtN+1 > p always, so this never fires,
            // but protects a more general reuse of this block.
            if start == p {
                start += p
            }
            for j := start; j <= hi; j += p {
                seg[j-lo] = false
            }
        }
        for i, isPrime := range seg {
            if isPrime {
                primes = append(primes, lo+i)
            }
        }
    }
    return primes
}
```

### GCD and LCM

**GCD(a, b):** largest integer that divides both a and b.
**LCM(a, b):** smallest positive integer divisible by both a and b.

**Relationship:** `LCM(a, b) = (a / GCD(a, b)) × b` — divide first to avoid overflow before multiplying.

**Euclidean Algorithm — O(log min(a, b))**

Key identity: `GCD(a, b) = GCD(b, a % b)`

*Why this holds:* Any common divisor of a and b also divides a − k×b for any integer k, and in particular divides `a mod b = a − ⌊a/b⌋ × b`. Conversely, any common divisor of b and (a mod b) divides a. The set of common divisors is preserved at every step.

```
GCD(48, 18):

(a,  b)  →  (b,  a%b)
(48, 18) →  (18, 12)    48 mod 18 = 12
(18, 12) →  (12,  6)    18 mod 12 = 6
(12,  6) →  ( 6,  0)    12 mod  6 = 0
GCD = 6
```

Terminates because remainders strictly decrease. Worst case is consecutive Fibonacci numbers, giving O(log_φ n) ≈ O(log n) steps.

```go
// gcd — iterative Euclidean algorithm, O(log min(a,b))
func gcd(a, b int) int {
    for b != 0 {
        a, b = b, a%b
    }
    return a
}

// lcm — divide first to prevent overflow
func lcm(a, b int) int {
    return a / gcd(a, b) * b
}
```

### Modular Arithmetic

When intermediate values grow astronomically (factorials, exponentials), modular arithmetic keeps them bounded. Most competitive programming problems use mod = 10⁹+7 — a prime, which enables modular inverse.

**Etymology:** Latin *modulus* = "small measure." Mod wraps numbers into a fixed range, like a clock face.

**Core Properties:**
```
(a + b) % m  =  ((a%m) + (b%m)) % m
(a - b) % m  =  ((a%m) - (b%m) + m) % m   ← +m prevents negatives
(a × b) % m  =  ((a%m) × (b%m)) % m
(a / b) % m  ≠  ((a%m) / (b%m)) % m        ← division needs modular inverse
```

**Why +m for subtraction:** In Go, `(-1) % 7 = -1` (sign follows dividend, per the language spec). Writing `((a-b)%m + m) % m` absorbs the potential negative and returns the correct residue.

**Modular Inverse:**

a⁻¹ mod m is the number such that `a × a⁻¹ ≡ 1 (mod m)`.

When m is prime, **Fermat's Little Theorem** gives `a^(m-1) ≡ 1 (mod m)`, so `a⁻¹ = a^(m-2) mod m`.

This means: `(a / b) % m = (a × b^(m-2)) % m`

**Pitfalls:**
- Fermat's inverse only works when m is prime. For composite m, use the extended Euclidean algorithm
- a must not be divisible by m (the inverse doesn't exist otherwise)

```go
const MOD = 1_000_000_007

// safeMod — handles negative values from subtraction
func safeMod(a, m int) int {
    return ((a % m) + m) % m
}
```

### Extended Euclidean Algorithm

Plain Euclid finds GCD(a, b) but discards the intermediate equations used to get there. Extended Euclid keeps them, and uses them to find integers x, y satisfying **Bézout's Identity**:

```
a×x + b×y = gcd(a, b)
```

**Why such x, y must exist:** this isn't a convenient assumption — it's forced. Consider the set of every positive value reachable by combining a and b with integer multipliers: `S = {a×x + b×y > 0}`. S is non-empty (`a×1 + b×0 = a` is in it), so it has a smallest element d. If a divided by d left any nonzero remainder r, that r would itself be a smaller positive combination of a and b — contradicting that d was the smallest. So d divides a exactly, and by the same argument d divides b — making d a common divisor. Since gcd(a,b) also divides a and b, it divides any combination of them, including d itself — so `gcd(a,b) ≤ d` and `d ≤ gcd(a,b)`, forcing `d = gcd(a,b)`. The smallest reachable combination is always exactly the gcd, never anything else.

This is also the real reason the modular inverse pitfall above holds: every value reachable via `a×x + m×y` is a multiple of `gcd(a,m)`. If `gcd(a,m)=1`, then 1 is reachable — the inverse exists. If `gcd(a,m)>1`, 1 is never a multiple of it — the inverse cannot exist, for any modulus.

**Why we need it (vs. Fermat):** Fermat's inverse formula only works when the modulus is prime. Extended Euclid finds the modular inverse for *any* modulus m, provided `gcd(a, m) = 1` — a strictly more general tool.

**Why it gives the inverse:** when `gcd(a, m) = 1`, Bézout's identity becomes `a×x + m×y = 1`. Reducing both sides mod m makes the `m×y` term vanish (it's a multiple of m), leaving `a×x ≡ 1 (mod m)` — exactly the definition of x = a⁻¹.

**Manual derivation — forward divide, then backward substitute:**

```
GCD(8, 3):
  8 = 2×3 + 2     →  2 = 8 − 2×3        (i)
  3 = 1×2 + 1     →  1 = 3 − 1×2        (ii)
  2 = 2×1 + 0                            gcd = 1

Back-substitute (ii) using (i), one move at a time:
  1 = 3 − 1×2
  1 = 3 − 1×(8 − 2×3)      ← swap in (i) wherever "2" appears
  1 = 3 − 1×8 + 2×3        ← distribute the −1
  1 = 3×3 − 1×8            ← collect the two "3" terms

Match against a×x + b×y, using a=8, b=3 (the ORIGINAL GCD(8,3) labeling — don't flip):
  1 = 8×(−1) + 3×(3)
        ↑           ↑
       x=−1        y=3

Check: 8×(−1) + 3×3 = −8 + 9 = 1  ✓
```

This matches the recursive trace below exactly — `extGCD(8, 3)` returns `(g=1, x=−1, y=3)` — same a=8, b=3 labeling throughout, no relabeling mid-derivation.

**Deriving the recursive update line — `x, y = y1, x1-(a/b)*y1`:**

The recursive call `extGCD(b, a%b)` has already solved the smaller subproblem, returning `x1, y1` satisfying `b×x1 + (a%b)×y1 = g`. To turn this into a solution for the current problem `a×x + b×y = g`, substitute `a%b = a − (a/b)×b`:

```
b×x1 + (a − (a/b)×b)×y1 = g
b×x1 + a×y1 − (a/b)×b×y1 = g         ← distribute y1
a×y1 + b×(x1 − (a/b)×y1) = g          ← collect the two b-terms
```

Comparing this to the target `a×x + b×y = g` reads off the coefficients directly: `x = y1` and `y = x1 − (a/b)×y1`. This is the same distribute-then-collect move as the manual back-substitution above, just done once in general so it applies automatically at every level of the recursion.

The recursive implementation performs this same substitution automatically, bottom-up, instead of by hand:

```go
// extGCD — extended Euclidean algorithm
// Returns (g, x, y) such that a*x + b*y = g = gcd(a, b)
func extGCD(a, b int) (g, x, y int) {
    if b == 0 {
        return a, 1, 0 // base case: a*1 + 0*0 = a
    }
    g, x1, y1 := extGCD(b, a%b)
    x, y = y1, x1-(a/b)*y1
    return
}

// modInverseGeneral — works for any modulus m where gcd(a, m) = 1, prime or not
// Returns (inverse, true) if it exists, or (0, false) if it doesn't
func modInverseGeneral(a, m int) (int, bool) {
    g, x, _ := extGCD(a, m)
    if g != 1 {
        return 0, false // inverse doesn't exist — gcd(a, m) != 1
    }
    return ((x % m) + m) % m, true
}
```

**Complexity:** `extGCD(a, b)` recurses on `extGCD(b, a%b)` — the exact same call structure as plain Euclid — so it takes the same **O(log min(a, b))** time; tracking x and y only adds O(1) extra bookkeeping per call. Space differs from the iterative `gcd` above, though: each recursive call holds a stack frame until its child returns, so the call stack grows to depth O(log min(a, b)) — **O(log min(a, b)) space**, not O(1). `modInverseGeneral` inherits both bounds directly, since it's a single call to `extGCD` plus O(1) work.

**Pitfall:** the inverse exists if and only if `gcd(a, m) = 1`. If `gcd(a, m) > 1`, no inverse exists at all — not "hard to compute," genuinely absent. `extGCD` surfaces this directly via its returned `g`.

### Fast Exponentiation (Binary Exponentiation)

Naive x^n multiplies x by itself n times: O(n). Binary exponentiation achieves O(log n) by repeated squaring.

**Why "binary":** The exponent n in binary determines which power-of-two exponents to combine. Reading bits from LSB: a `1` bit means multiply result by the current base; always square the base and halve n.

```
3^13,  13 in binary = 1101₂ = 8 + 4 + 1

n=13 (bit=1): result = 1×3   = 3        base = 3²  = 9     n=6
n= 6 (bit=0): result = 3                base = 9²  = 81    n=3
n= 3 (bit=1): result = 3×81 = 243       base = 81² = 6561  n=1
n= 1 (bit=1): result = 243×6561 = 1594323                  n=0  stop

3^13 = 1,594,323  ✓   (4 multiplications instead of 12)
```

Rule: if n is odd, peel off one factor into result; always square base and halve n.

```go
// modPow — binary exponentiation with modular reduction
// Computes (base^exp) % mod in O(log exp) time
func modPow(base, exp, mod int) int {
    result := 1
    base %= mod
    for exp > 0 {
        if exp%2 == 1 { // current bit is 1
            result = result * base % mod
        }
        base = base * base % mod
        exp /= 2
    }
    return result
}

// modInverse — modular multiplicative inverse via Fermat's Little Theorem
// Requires mod to be prime. O(log mod) time (inherits modPow), O(1) space
func modInverse(a, mod int) int {
    return modPow(a, mod-2, mod)
}
```

**Pitfall (harmless):** on the final loop iteration — the one processing the most-significant bit — `base = base*base%mod` still executes, but `exp` becomes 0 right after, so that squared value is never read again. It's one wasted multiplication per call. This doesn't affect the O(log n) complexity (a constant added to a logarithmic loop stays logarithmic) and is conventionally left in, since avoiding it would need an extra branch to save a single multiply — not a good trade.

## Combinatorics

### Factorials

n! = n × (n-1) × ... × 2 × 1. Counts the number of orderings (permutations) of n distinct items.

**Overflow boundary — memorise:**
```
20! = 2,432,902,008,176,640,000  (~2.4 × 10^18) — fits in int64
21! = 51,090,942,171,709,440,000 — overflows int64
```

Always apply mod when factorials exceed n=20. For competitive programming, precompute a factorial table modulo a prime and derive all nCr queries from it.

### Permutations and Combinations

**Permutation (nPr):** ordered selection of r items from n.
```
nPr = n! / (n-r)!
```
Order matters: selecting {A, B} as AB ≠ BA → two distinct permutations.

**Combination (nCr, "n choose r"):** unordered selection of r items from n.
```
nCr = n! / (r! × (n-r)!)
```
Order doesn't matter: {A, B} = {B, A} → one combination.

**Key identity:** `nCr = nC(n-r)`. Choosing which r items are IN equals choosing which (n-r) items are OUT. Always reduce to the smaller side to minimise computation.

**Iterative nCr — O(r) time, O(1) space, no factorial overflow for moderate n:**

The product `n × (n-1) × ... × (n-r+1)` divided by `r!` is always an integer at each incremental step (Pascal's triangle property). Dividing step-by-step avoids computing large factorials outright.

```go
// nCr — iterative, O(r) time, O(1) space, avoids overflow for moderate n with small r
// Not suitable when n is very large; use precomputed factorials + mod instead
func nCr(n, r int) int {
    if r > n-r {
        r = n - r // always use the smaller side
    }
    result := 1
    for i := 0; i < r; i++ {
        result = result * (n - i) / (i + 1)
    }
    return result
}
```

**nCr with large n — precomputed factorials and modular inverse:**

**Why "propagate back" works:** factorials satisfy `fact[i+1] = fact[i] × (i+1)`. Taking the inverse of both sides (`inv(a×b) = inv(a)×inv(b) mod p`) gives `invFact[i+1] = invFact[i] × inv(i+1)` — stepping forward multiplies by `inv(i+1)`. To step backward, multiply both sides by `(i+1)`: the `inv(i+1)×(i+1)` term cancels to 1, leaving `invFact[i] = invFact[i+1] × (i+1)`. So each smaller index is derived from the next larger one with a single multiplication — undoing the forward step instead of recomputing it. This is why only **one** `modPow` call is needed (for `invFact[n]`); every other entry falls out in O(1), giving O(n) total instead of O(n log MOD) from calling Fermat's formula at every index.

```go
const MOD = 1_000_000_007

// precompute — builds factorial and inverse-factorial tables modulo MOD
// fact[i]    = i! mod MOD
// invFact[i] = modular inverse of i! mod MOD
// O(n) time (one modPow call, rest is O(1) per index), O(n) space (two tables)
func precompute(n int) (fact, invFact []int) {
    fact = make([]int, n+1)
    invFact = make([]int, n+1)
    fact[0] = 1
    for i := 1; i <= n; i++ {
        fact[i] = fact[i-1] * i % MOD
    }
    // invFact[n] via Fermat's Little Theorem — the one Fermat call needed
    invFact[n] = modPow(fact[n], MOD-2, MOD)
    // Propagate back: invFact[i] = invFact[i+1] * (i+1), derived above
    for i := n - 1; i >= 0; i-- {
        invFact[i] = invFact[i+1] * (i + 1) % MOD
    }
    return
}

// nCrMod — O(1) query after O(n) precomputation
func nCrMod(n, r int, fact, invFact []int) int {
    if r < 0 || r > n {
        return 0
    }
    return fact[n] * invFact[r] % MOD * invFact[n-r] % MOD
}
```

### Pascal's Triangle and Precomputation

Recurrence: `C(n, r) = C(n-1, r-1) + C(n-1, r)`

**Why:** To choose r from n items, either include item n (then pick r-1 from the rest: C(n-1, r-1)) or exclude it (pick r from the rest: C(n-1, r)). Sum both cases.

Named after Blaise Pascal (1623), though independently discovered by Yang Hui (China, 1261) and Omar Khayyam (Persia, ~1100).

```
n=0:       1
n=1:      1 1
n=2:     1 2 1
n=3:    1 3 3 1
n=4:   1 4 6 4 1
n=5:  1 5 10 10 5 1
```

Each interior value is the sum of the two values immediately above it. The entry at row n, column r is C(n, r).

```go
// buildPascal — O(n²) preprocessing, O(1) per query
// pascal[n][r] = C(n, r)
func buildPascal(maxN int) [][]int {
    pascal := make([][]int, maxN+1)
    for n := 0; n <= maxN; n++ {
        pascal[n] = make([]int, n+1)
        pascal[n][0] = 1
        pascal[n][n] = 1
        for r := 1; r < n; r++ {
            pascal[n][r] = pascal[n-1][r-1] + pascal[n-1][r]
        }
    }
    return pascal
}
```

## Sequences and Sums

Closed-form formulas replace O(n) summation loops with O(1) computation. Recognising which formula applies is the interview skill; the formula itself is a lookup.

| Sequence | Closed-Form Formula |
|---|---|
| 1 + 2 + ... + n | n(n+1) / 2 |
| 1² + 2² + ... + n² | n(n+1)(2n+1) / 6 |
| 1³ + 2³ + ... + n³ | [n(n+1) / 2]² |
| AP sum (n terms, first a, last l) | n(a + l) / 2 |
| GP sum (n terms, first a, ratio r ≠ 1) | a(rⁿ − 1) / (r − 1) |

- **AP (Arithmetic Progression):** fixed *additive* difference between consecutive terms
- **GP (Geometric Progression):** fixed *multiplicative* ratio between consecutive terms

**Gauss's trick for 1..n:** Pair 1 with n, 2 with n-1, ..., yielding n/2 pairs each summing to (n+1). Total: n(n+1)/2.

**Powers of 2 — memorise for scale estimation:**
```
2^10  ≈  10^3   (1 K)
2^20  ≈  10^6   (1 M)
2^30  ≈  10^9   (1 B)
2^32  ≈  4.3 × 10^9   (uint32 max)
2^63  ≈  9.2 × 10^18  (int64 max)
```

```go
// All O(1) time, O(1) space — closed forms, no loops
func sumN(n int) int       { return n * (n + 1) / 2 }
func sumSquares(n int) int { return n * (n + 1) * (2*n + 1) / 6 }
func sumCubes(n int) int   { s := sumN(n); return s * s }
func apSum(n, first, last int) int { return n * (first + last) / 2 }
```

## Pigeonhole Principle

**Statement:** If n+1 items are distributed into n containers, at least one container holds ≥ 2 items.

**Generalised form:** If n items go into k containers, at least one container holds ≥ ⌈n/k⌉ items.

Concretely: 7 items into 3 containers, `⌈7/3⌉ = 3`. Check by trying to avoid it — 3 containers holding at most 2 each cap out at `3×2=6` items total. There are 7 items, one more than that cap, so some container is forced past 2. The original Statement is this same formula with k=n containers and n+1 items: `⌈(n+1)/n⌉ = ⌈1 + 1/n⌉ = 2` for any n ≥ 1 — collapsing back to "at least one holds ≥ 2."

**Etymology:** Dovecotes (pigeon housing) were built with a grid of small individual nesting compartments — literally "pigeon-holes." Office desks later adopted the same grid shape for sorting letters into small compartments, borrowing the name despite having no birds involved. Mathematically, the principle was first stated by Dirichlet in 1834 under the German name *Schubfachprinzip* ("drawer principle") — English mathematicians later swapped "drawer" for the more recognizable "pigeonhole." Both senses are literal, physical embodiments of the same crowding effect: more pigeons than nesting holes, or more letters than desk compartments, forces a shared slot either way.

**Role in DSA — an existence argument, not a search.** Pigeonhole proves a collision, duplicate, or overlap *must* exist, before any scanning happens — like being told with certainty "a $20 bill is under one of these 10 tiles" without having lifted any of them. It never tells you *where*; finding the item still needs an actual algorithm.

```
LeetCode 287, concretely with n=5:

Array (6 numbers, each forced into [1,5]):  [3, 1, 4, 2, 5, 3]
Containers = the 5 possible values: 1, 2, 3, 4, 5
Items      = the 6 numbers in the array

6 items, 5 containers → duplicate guaranteed (here: 3 appears twice)

Contrast — range [1,6], 6 numbers: [3, 1, 4, 2, 5, 6] — every value used once,
zero duplicates possible. The guarantee only appears once items > range size.
```

**Common appearances:**
- Hash collisions: 10 buckets, 11 items inserted → at least one bucket gets 2, forced by arithmetic alone, regardless of hash function quality. Scaled up: hashing every possible string into a 64-bit table still guarantees collisions exist somewhere, since there are infinitely more possible strings than buckets — this is *why* every hash table needs collision handling (chaining, open addressing) as a structural feature, not an edge case.
- Birthday paradox — two unrelated numbers answering two different questions: **367 people** guarantees a shared birthday with 100% certainty via pure Pigeonhole (367 items > 366 possible birthdays). **23 people** crosses a 50% *probability* of a shared birthday — computed via the complement, `1 − P(all 23 different)`, where `P(all different) = (365/365)×(364/365)×...×(343/365) ≈ 0.493`, giving `≈50.7%`. This has nothing to do with Pigeonhole. Pigeonhole only ever produces guarantee-thresholds like 367, never probability-thresholds like 23.

## Catalan Numbers

The Catalan sequence: **1, 1, 2, 5, 14, 42, 132, 429, ...**

```
C_0=1  C_1=1  C_2=2  C_3=5  C_4=14  C_5=42
```

**Etymology:** Named after Eugène Catalan (Belgian, 1838), though first described by Minggantu (Chinese mathematician, ~1730).

**Visualising the concept — balanced parentheses as a balance path:**

Think of `(` as a step up and `)` as a step down, tracking a running balance left to right. A sequence of n pairs is valid exactly when the balance never goes negative and ends at 0:

```
()()()   balance: 1,0,1,0,1,0  — never negative, ends at 0  ✓
)(       balance: -1            — went negative immediately  ✗
(()      balance: 1,2,1         — ends at 1, not 0           ✗
```

For n=1: only `()` — C_1=1. For n=2: `()()` and `(())` — C_2=2. For n=3 there are exactly C_3=5 valid sequences: `()()()`, `(())()`, `(()())`, `()((  ))`, `((()))`.

**Deriving the recurrence:**

In any valid sequence, the opening `(` must match some `)`. Between them sit exactly i self-contained valid pairs (C_i ways), and after them sit the remaining n-i valid pairs (C_{n-i} ways). Summing over every possible split position i = 0..n-1:

```
C_n = Σ (i=0 to n-1) C_i × C_{n-1-i}      (equivalently written as C_{n+1} = Σ C_i × C_{n-i})
```

Concrete trace computing C_3, splitting over all i = 0..2:

```
i=0: 0 nested pairs (1 way: empty), 2 remaining (C_2=2: "()()", "(())")
     → "()" + "()()" = "()()()"  and  "()" + "(())" = "()(())"        [2 sequences]

i=1: 1 nested pair (C_1=1: "()"), 1 remaining (C_1=1: "()")
     → "(())" + "()" = "(())()"                                        [1 sequence]

i=2: 2 nested pairs (C_2=2: "()()", "(())"), 0 remaining (C_0=1: empty)
     → "(" + "()()" + ")" = "(()())"  and  "(" + "(())" + ")" = "((()))"   [2 sequences]

Total: 2 + 1 + 2 = 5 = C_3  ✓
```

**Deriving the closed form:**

Step 1 — count all arrangements. Each valid sequence is one way to place n `(` symbols in 2n positions. The total count of all placements (valid or not) is `C(2n, n)`.

Step 2 — subtract invalids via the reflection principle. An invalid sequence first touches balance −1 at some step t. Flip every step after t (up↔down). The flipped suffix turns the final balance from 0 into −2, so the reflected path ends with n+1 down-steps and n−1 up-steps. This creates an exact one-to-one match between invalid sequences and paths ending at −2, which total `C(2n, n-1)`. So:

```
valid = C(2n, n) − C(2n, n-1)
```

Step 3 — simplify. `C(2n, n-1) = C(2n, n) × n/(n+1)`, so:

```
valid = C(2n, n) − C(2n, n) × n/(n+1)
      = C(2n, n) × (1 − n/(n+1))
      = C(2n, n) × 1/(n+1)
      = C(2n, n) / (n+1)
```

**Closed form:**
```
C_n = C(2n, n) / (n+1)  =  (2n)! / ((n+1)! × n!)
```

Verify: C_3 = C(6,3) / (3+1) = 20/4 = 5  ✓

**What Catalan numbers count — the key interview appearances:**

- Valid sequences of n pairs of parentheses → C_3 = 5 (traced above)
- Structurally distinct BSTs with n nodes → C_3 = 5 (each split position of the root mirrors the i/n−i recurrence split above)
- Monotone lattice paths from (0,0) to (n,n) that never go above the diagonal → C_3 = 5 (each step right = `(`, step up = `)`, "above diagonal" = balance goes negative)
- Triangulations of a convex (n+2)-gon → C_3 = 5

**Overflow:** C_20 = 6,564,120,420 — already exceeds int32. Use int64 or mod.

```go
// catalan — dynamic programming via recurrence
// Outer loop runs n times; inner loop for step i runs i times.
// Total work: 1+2+...+n = n(n+1)/2  →  O(n²) time, O(n) space
func catalan(n int) int {
    c := make([]int, n+1)
    c[0] = 1
    for i := 1; i <= n; i++ {
        for j := 0; j < i; j++ {
            c[i] += c[j] * c[i-1-j]
        }
    }
    return c[n]
}

// catalanMod — O(1) query using precomputed factorials (requires precompute() from Combinatorics)
// C_n = C(2n, n) / (n+1) = fact[2n] × invFact[n+1] × invFact[n]  (mod prime)
func catalanMod(n int, fact, invFact []int) int {
    return fact[2*n] * invFact[n+1] % MOD * invFact[n] % MOD
}
```

## LeetCode Problems

### 204 — Count Primes

Return the count of primes strictly less than n.

Link: https://leetcode.com/problems/count-primes/

<details>
<summary>Hint 1 — Approach</summary>

Checking each number individually up to n is O(n√n) — far too slow for large n. Use the Sieve of Eratosthenes to mark all composites in O(n log log n).

</details>

<details>
<summary>Hint 2 — Boundaries</summary>

The problem asks for primes strictly less than n, so the sieve array only needs size n (indices 0..n-1), and the outer loop runs while `i*i < n` (strict), not `<=`. Count primes in range [2, n-1].

</details>

```go
func countPrimes(n int) int {
    if n < 2 {
        return 0
    }
    isPrime := make([]bool, n)
    for i := 2; i < n; i++ {
        isPrime[i] = true
    }
    for i := 2; i*i < n; i++ {
        if isPrime[i] {
            for j := i * i; j < n; j += i {
                isPrime[j] = false
            }
        }
    }
    count := 0
    for i := 2; i < n; i++ {
        if isPrime[i] {
            count++
        }
    }
    return count
}
```

### 50 — Pow(x, n)

Implement `pow(x, n)` — x raised to the power n, where n can be negative.

Link: https://leetcode.com/problems/powx-n/

<details>
<summary>Hint 1 — Negative exponent</summary>

For negative n, compute `(1/x)^|n|`. Handle by setting `x = 1/x` and `n = -n` upfront, then run the same algorithm.

</details>

<details>
<summary>Hint 2 — Algorithm</summary>

Use binary exponentiation: if n is odd, multiply result by x and decrement n by 1; always square x and halve n. This runs in O(log n).

</details>

<details>
<summary>Hint 3 — Edge case</summary>

In Go, `int` is 64-bit on most platforms, so `n = math.MinInt32` negating safely. If using int32 explicitly, negating MinInt32 overflows — cast to int64 first.

</details>

```go
func myPow(x float64, n int) float64 {
    if n < 0 {
        x = 1 / x
        n = -n
    }
    result := 1.0
    for n > 0 {
        if n%2 == 1 {
            result *= x
        }
        x *= x
        n /= 2
    }
    return result
}
```

### 1071 — Greatest Common Divisor of Strings

For two strings str1 and str2, return the longest string x such that both str1 and str2 are repetitions of x. Return `""` if no such string exists.

Link: https://leetcode.com/problems/greatest-common-divisor-of-strings/

<details>
<summary>Hint 1 — Existence check</summary>

If a common divisor string exists, then str1 + str2 must equal str2 + str1 (both would produce the same repeated pattern regardless of concatenation order). If they differ, return `""`.

</details>

<details>
<summary>Hint 2 — Length of result</summary>

When a divisor exists, its length is exactly GCD(len(str1), len(str2)) — a direct analogy to integer GCD. Return `str1[:gcdLen]`.

</details>

<details>
<summary>Hint 3 — Complexity</summary>

The concatenation check `str1+str2 != str2+str1` costs O(n+m) to build and compare the two strings. The gcd computation afterward is O(log min(n,m)), dominated by the string work. Overall: **O(n+m) time, O(n+m) space** (for the concatenated strings).

</details>

```go
func gcdOfStrings(str1, str2 string) string {
    if str1+str2 != str2+str1 {
        return ""
    }
    g := gcd(len(str1), len(str2))
    return str1[:g]
}

func gcd(a, b int) int {
    for b != 0 {
        a, b = b, a%b
    }
    return a
}
```

### 62 — Unique Paths

A robot starts at the top-left of an m×n grid and can only move right or down. How many distinct paths reach the bottom-right corner?

Link: https://leetcode.com/problems/unique-paths/

<details>
<summary>Hint 1 — Frame as combinatorics</summary>

Every path makes exactly (m-1) downward moves and (n-1) rightward moves, totalling (m+n-2) moves. The number of distinct paths is the number of ways to choose which (m-1) of those moves go down: `C(m+n-2, m-1)`.

</details>

<details>
<summary>Hint 2 — Avoid overflow in nCr</summary>

Compute `C(N, K)` iteratively where `N = m+n-2`, `K = min(m-1, n-1)`. At each step, `result = result*(N-i)/(i+1)`. Integer division is exact here — the intermediate value is always a whole number (Pascal's triangle property), so no mod or floating point needed for the given constraints.

</details>

<details>
<summary>Hint 3 — Complexity</summary>

The loop runs K times, where K = min(m-1, n-1) — that's why the smaller side is chosen. **O(min(m, n)) time, O(1) space.** Compare to the DP-grid approach (also valid, O(m×n) time and space) — the combinatorial formula is strictly better on both fronts.

</details>

```go
func uniquePaths(m, n int) int {
    // C(m+n-2, m-1) total paths
    N, K := m+n-2, m-1
    if K > N-K {
        K = N - K // use the smaller side
    }
    result := 1
    for i := 0; i < K; i++ {
        result = result * (N - i) / (i + 1)
    }
    return result
}
```

### 287 — Find the Duplicate Number

Given an array of n+1 integers where each integer is in [1, n], find the duplicate. Must use O(1) extra space and must not modify the array.

Link: https://leetcode.com/problems/find-the-duplicate-number/

<details>
<summary>Hint 1 — Existence via Pigeonhole</summary>

n+1 values all drawn from [1, n] → n+1 items in n "containers." By Pigeonhole, at least one value appears twice. Existence is proven; the challenge is finding it efficiently under the space constraint.

</details>

<details>
<summary>Hint 2 — Binary search on the answer</summary>

Binary search over the candidate range [1, n]. For a midpoint `mid`, count how many numbers in the array are ≤ mid. If that count exceeds mid, by Pigeonhole the duplicate is in [1, mid]. Otherwise it's in [mid+1, n]. O(n log n) time, O(1) space.

</details>

<details>
<summary>Hint 3 — O(n) solution exists</summary>

Floyd's cycle detection (treating array values as "next" pointers) finds the duplicate in O(n) time and O(1) space. The binary search approach shown here uses Pigeonhole reasoning explicitly and is easier to derive in an interview.

</details>

```go
func findDuplicate(nums []int) int {
    lo, hi := 1, len(nums)-1
    for lo < hi {
        mid := lo + (hi-lo)/2
        count := 0
        for _, n := range nums {
            if n <= mid {
                count++
            }
        }
        // If count > mid, more values exist in [1,mid] than slots → duplicate there
        if count > mid {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}
```

## How to Remember This

**GCD — "Bigger mod Smaller, until zero"**
Take the larger number mod the smaller; the remainder becomes the new smaller. When the remainder hits zero, the other value is the GCD.

**Sieve — "Start marking at p²"**
All smaller multiples of p have a factor < p and were already handled. Fresh marking begins at p².

**Modular subtraction — "Add m before mod"**
`((a - b) % m + m) % m` — the +m absorbs any potential negative, always landing in [0, m-1].

**Fermat's inverse — "prime minus two"**
`a⁻¹ ≡ a^(p-2) mod p`. Why p-2? Fermat says a^(p-1) ≡ 1, so a × a^(p-2) ≡ 1. Peeling off one a gives a⁻¹.

**Extended Euclid — "Forward divide, backward substitute"**
Run plain Euclid forward, keeping every equation. Then walk backward, swapping each leftover remainder back in, until only a and m remain: `a×x + m×y = 1`. The x is the inverse.

**Binary exponentiation — "Odd: take one; Even: just square"**
If n is odd, pull one factor into result and make n even. If n is even, square base and halve n.

**nCr = nC(n-r)**
Choosing who's IN is the same problem as choosing who's OUT. Always work with the smaller side.

**Pascal's recurrence — "Sum of the two above"**
`C(n, r) = C(n-1, r-1) + C(n-1, r)`. Include item n, or exclude it — sum both cases.

**Gauss sum — "n(n+1)/2"**
Pair opposite ends (1+n, 2+(n-1), ...) → n/2 pairs, each summing to (n+1).

**Powers of 2 — "Kilo, Mega, Giga"**
2^10 ≈ K, 2^20 ≈ M, 2^30 ≈ G. Each decade adds 3 zeros (roughly).

**Pigeonhole — "n+1 in n boxes → collision guaranteed"**
More items than containers → at least one container holds two. Pure existence proof, no algorithm needed.

**Catalan — "1, 1, 2, 5, 14 → parentheses / BSTs / paths"**
Recognise the sequence. Formula: C(2n, n) / (n+1). Three canonical appearances: valid bracket sequences, distinct BSTs, lattice paths below diagonal.

## Interview Cheat Sheet

**Signal phrases → math recognition:**

- "How many ways to arrange / select / order..." → Permutation or Combination
- "Return the answer mod 10^9+7" → Modular arithmetic; precompute fact and invFact tables
- "Compute x^n efficiently" → Binary exponentiation
- "Find all primes up to n" → Sieve of Eratosthenes
- "Largest shared divisor / smallest common multiple" → GCD / LCM via Euclidean
- "Must a duplicate exist?" → Pigeonhole as existence proof
- "Count valid structures with n elements (brackets, BSTs, paths...)" → Check if sequence is Catalan

**Red flags — common mistakes:**

- Using `(a - b) % m` without `+m` → negative residues in Go (-1 instead of m-1)
- Computing `a * b` before `a / gcd(a,b)` in LCM → integer overflow
- Checking primality up to n instead of √n → O(n) per check instead of O(√n)
- Using division inside mod arithmetic without modular inverse → silent wrong answer
- Applying Fermat's inverse when the modulus is composite → invalid; use extended GCD
- Forgetting `21! overflows int64` → wrong answer without mod
- Using `float64` for large-integer exponentiation → precision loss

**Common interview probes:**

- "What's the complexity of your primality test?" → O(√n) trial division; O(n log log n) sieve
- "What if n = 10^9 and you need all primes?" → Segmented sieve (the plain sieve uses ~1 GB)
- "Can you do Pow(x, n) iteratively?" → Yes, binary exponentiation is naturally iterative; recursion not needed
- "What if the modulus is not prime?" → Extended Euclidean algorithm for modular inverse
- "GCD of three numbers?" → `gcd(a, gcd(b, c))` — GCD is associative
- "How many distinct BSTs with 4 nodes?" → C_4 = 14

## References

- Donald Knuth, *The Art of Computer Programming, Vol. 2: Seminumerical Algorithms* — authoritative reference on number theory for computing
- Sanjoy Dasgupta et al., *Algorithms* — Chapter 1 covers Euclidean GCD and modular arithmetic with proofs
- cp-algorithms.com — https://cp-algorithms.com/algebra/ — Sieve, binary exponentiation, modular inverse, extended GCD, all with proofs
- OEIS A000108 (Catalan numbers) — https://oeis.org/A000108
- Striver's A2Z DSA Sheet — Math section (supplementary reference)