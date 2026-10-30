---
Status: 🌳 Evergreen
Created: 2026-10-30
Last Updated: 2026-10-30
---

# Bit Manipulation

## Table of Contents

1. [What It Is and Why It Exists](#what-it-is-and-why-it-exists)
2. [Why "Bit Manipulation"](#why-bit-manipulation)
3. [Binary Representation and Two's Complement](#binary-representation-and-twos-complement)
4. [The Six Bitwise Operators](#the-six-bitwise-operators)
5. [Shifts: Arithmetic vs Logical](#shifts-arithmetic-vs-logical)
6. [Single-Bit Operations](#single-bit-operations)
7. [Bit Tricks: Isolate, Remove, Test the Lowest Set Bit](#bit-tricks-isolate-remove-test-the-lowest-set-bit)
8. [Power of Two](#power-of-two)
9. [Brian Kernighan's Algorithm](#brian-kernighans-algorithm)
10. [XOR Properties and Tricks](#xor-properties-and-tricks)
11. [Bitmask for Subsets](#bitmask-for-subsets)
12. [Go Template Code](#go-template-code)
13. [Where JavaScript Differs](#where-javascript-differs)
14. [LeetCode Problems](#leetcode-problems)
15. [References](#references)

## What It Is and Why It Exists

Bit manipulation is treating an integer as a fixed-length array of bits and operating on those bits directly rather than on the number's arithmetic value. On a 64-bit machine every `int` is already 64 bits laid out contiguously in a CPU register; bit manipulation just exposes that layout to the programmer.

It exists because a single CPU instruction (AND, OR, XOR, shift) touches all 64 bits at once. This gives you a free 64-way parallelism for anything that fits into a bit-per-item encoding: a set of at most 64 elements becomes one integer, a subset enumeration over 20 items becomes a loop from 0 to `1<<20`, a "have I seen this" flag over the letters 'a' to 'z' becomes a single 32-bit int. The alternative (a `[]bool` or `map[int]bool`) allocates on the heap, chases pointers, and misses cache. A bitmask lives in a register.

### The Problems It Introduces

Bit tricks are dense: `n & (n-1)` clears the lowest set bit, but nothing in that expression looks like "clear the lowest set bit". Code becomes unreadable to anyone who has not memorized the vocabulary. That is the real cost, not correctness.

The second problem is overflow and sign. On a signed integer, the top bit is the sign bit, so `1 << 63` in Go's `int64` is the smallest negative number, not a large positive one. Right-shifting a negative signed integer sign-extends (fills with 1s), which is usually not what a bit-manipulation algorithm wants. Working in `uint` or converting explicitly is often necessary.

Concrete example. The bit pattern `11111000` is `-8` when read as `int8` and `248` when read as `uint8`. Same eight bits, different type:

```go
var s int8  = -8   // bits: 11111000
var u uint8 = 248  // bits: 11111000

s >> 1  // -4,  bits: 11111100   (top bit copied down, "sign-extended")
u >> 1  // 124, bits: 01111100   (top bit filled with 0, "zero-filled")
```

Both shifts move every bit one position right and drop bit 0. The only difference is what fills the vacated top bit, and that decision is made by the operand's declared type, not by the shift operator. If you wanted to treat `-8` as raw bits and isolate its high half, `s >> 1` silently returns the wrong pattern; `int8(uint8(s) >> 1)` gives the intended `01111100` (124), matching the unsigned shift.

The third problem is portability. Bit widths differ across languages and platforms. Go's `int` is 32 or 64 bits depending on the target, JavaScript's bitwise operators silently truncate to 32-bit signed, and C leaves shift-out-of-range as undefined behaviour. Code that "just works" in one language can silently return wrong answers in another.

## Why "Bit Manipulation"

"Bit" is short for **binary digit**, a term coined by Claude Shannon's colleague John Tukey in 1946 for the smallest unit of information: a single 0 or 1. "Manipulation" is a hands-on word borrowed from the manipulator arms on early industrial machinery. Together the phrase captures the idea of reaching into the number's internal representation and rearranging its bits directly, as opposed to arithmetic which treats the number as an abstract value.

Older names for the same techniques include **bit twiddling** (from Henry Warren's *Hacker's Delight*, the canonical reference) and **bitwise operations** (the mathematical operator family, AND / OR / XOR / NOT). "Bit hacks" usually refers specifically to the clever one-liners like `n & (n-1)`.

## Binary Representation and Two's Complement

An integer is stored as a fixed-width sequence of bits. In an 8-bit example:

```
Decimal 13 in 8-bit binary:

  bit index:  7  6  5  4  3  2  1  0
  bit value:  0  0  0  0  1  1  0  1
  place val: 128 64 32 16  8  4  2  1

  13 = 8 + 4 + 1 = 2^3 + 2^2 + 2^0
```

Bit index 0 is the **least significant bit** (LSB, worth 1), and the highest index is the **most significant bit** (MSB, worth `2^(width-1)`). This is the same base-2 positional notation used in decimal, just with two digits instead of ten.

Negative numbers use **two's complement**, which is the encoding almost every modern CPU uses. To get the two's complement representation of `-n`:

1. Take `n`'s binary form.
2. Flip every bit (this is the **one's complement**).
3. Add 1.

```
+5 in 8-bit: 0000 0101
Flip bits:   1111 1010
Add 1:       1111 1011  = -5 in two's complement
```

Two's complement has two useful properties that fall out of this encoding.

**One adder circuit handles both signed and unsigned.** The hardware just adds bit-by-bit with carry and truncates to the word width. The same output bits are correct under either interpretation. Concretely, in 8-bit:

```
  00000011      (3)
+ 11111110      (-2 as signed, 254 as unsigned)
----------
1 00000001      truncate to 8 bits -> 00000001

Signed reading:   3 + (-2) = 1     correct
Unsigned reading: 3 + 254 = 257    which mod 256 is 1, also correct
```

Both readings agree because two's complement arithmetic *is* arithmetic mod `2^n`, and modular arithmetic doesn't care which representatives you pick for the values. The CPU can ship a single `ADD` (and a single `SUB`) that serves both `int` and `uint` code. Sign-magnitude cannot do this: to add `+3` and `-2` you would have to inspect signs, decide whether the operation is really a subtract, compare magnitudes to pick which operand goes on top, and compute the result sign separately. That is much more logic, and a lot more transistors.

**Zero has one bit pattern.** In two's complement, only `00000000` means zero. In sign-magnitude, `00000000` (+0) and `10000000` (-0) both mean zero, so `x == 0` has to check both patterns, a hash of `x` needs a canonicalization step, and one of the `2^n` available bit patterns is wasted representing a value you already had. Two's complement spends the freed slot on one extra negative number instead. That is why `int8` runs from `-128` to `+127` (asymmetric range, with `-128` having no positive counterpart) rather than the `-127` to `+127` you would get from sign-magnitude.

The consequence for bit manipulation: the sign bit (the MSB) is not a separate flag, it is a regular bit with a large negative place value. In signed 32-bit, bit 31 is worth `-2^31` instead of `+2^31`. This is why `1 << 31` overflows to a negative number in signed `int32`.

## The Six Bitwise Operators

Go has six bitwise operators: AND, OR, XOR, AND NOT, left shift, right shift. Plus the unary NOT (complement). Truth tables applied bit-by-bit.

### AND (`&`)

Result bit is 1 only if **both** input bits are 1.

```
  1010
& 1100
------
  1000
```

Uses: **masking** (keep only some bits, zero the rest). `n & 0xFF` isolates the low 8 bits.

The word "mask" is borrowed from painters and photographers, who tape or overlay a physical mask onto a surface so paint, light, or etchant reaches only the exposed regions. A bit mask does the same job to a number: you build a value with 1s at the positions you want to "see through" and 0s at the positions you want to block, then AND. Bits at the 1-positions pass through unchanged (since `x & 1 = x`), and bits at the 0-positions are forced to 0 (`x & 0 = 0`). That is the whole meaning of the term. The same word appears in "netmask" (which bits of an IP address are network vs host), "signal mask" in POSIX (which signals are blocked), and in chip fabrication (photomasks that determine which regions of silicon get etched). It is the same metaphor every time.

### OR (`|`)

Result bit is 1 if **either** input bit is 1.

```
  1010
| 1100
------
  1110
```

Uses: **setting** bits. `n | 1` forces the lowest bit to 1 regardless of what it was.

### XOR (`^`)

Result bit is 1 if the input bits **differ**. Also written "exclusive OR" because it excludes the case where both are 1.

```
  1010
^ 1100
------
  0110
```

Uses: **toggling** bits and cancellation (see the XOR section below). `n ^ n == 0` for any `n`.

### NOT (`^` as unary in Go)

Flips every bit. Go reuses `^` as the unary operator (most languages use `~`). The result depends on the type width: `^uint8(0b1010) == 0b11110101`.

Uses: creating masks that clear specific bits.

### AND NOT / Bit Clear (`&^`)

**Go-specific.** Equivalent to `a & (^b)`: keep bits from `a` that are 0 in `b`. In other languages you write it as `a & ~b`. Go added it as a single operator to save the reader an explicit negation.

```
  1010
&^ 1100    (equivalent to 1010 & 0011)
------
  0010
```

Uses: **clearing** specific bits. `n &^ (1 << i)` clears bit `i` regardless of what it was.

### Left Shift (`<<`)

Shift all bits left by `k` positions, filling the low bits with zeros. Equivalent to multiplying by `2^k` when no overflow.

```
0b0011 << 2 == 0b1100   (3 becomes 12)
```

Uses: constructing masks (`1 << i` is a mask with only bit `i` set), and fast multiplication by powers of 2.

### Right Shift (`>>`)

Shift all bits right by `k` positions. Fill of the high bits depends on the type (see next section). Equivalent to integer division by `2^k` for non-negative numbers.

```
0b1100 >> 2 == 0b0011   (12 becomes 3)
```

Uses: extracting a bit (`(n >> i) & 1` is bit `i`), fast division by powers of 2, and iterating over bits.

## Shifts: Arithmetic vs Logical

Right shift comes in two flavours, and the language chooses for you based on the operand's type.

**Logical right shift** fills the high bits with 0. Used for unsigned types. This is what you almost always want for bit manipulation.

**Arithmetic right shift** fills the high bits with the sign bit (copies the MSB downward). Used for signed types. This preserves the sign: `-8 >> 1 == -4`, not `+2^62-something` that a zero-fill would give.

```
Unsigned (logical) right shift:
  1111 1011 >> 1  ->  0111 1101

Signed (arithmetic) right shift:
  1111 1011 >> 1  ->  1111 1101   (top bit copied down)
```

In Go, the compiler picks based on operand type. If you have `n int` and want logical shift, convert first: `int(uint(n) >> 1)`. Left shift has no such split (both fill with 0 on the right).

Practical rule: **use `uint` (or an explicit `uintN`) for anything that treats an integer as a bit pattern**. Three separate traps hit signed integers here.

**Signed shifts fill the wrong bit.** As above, `>>` on a signed value sign-extends, so bit patterns with a 1 in the top position never lose it under right shift:

```go
var si int8  = -1   // bits: 11111111
var ui uint8 = 255  // bits: 11111111 (same pattern)

si >> 4  // -1,   bits: 11111111   (nothing changed, sign kept extending)
ui >> 4  // 15,   bits: 00001111   (zeros shifted in, what you actually wanted)
```

**Signed left shift silently produces negative numbers.** `1 << (width-1)` puts a 1 in the sign bit, which on a signed type flips the value to the smallest negative number. Nothing warns you.

```go
var s int8  = 1
var u uint8 = 1

s << 7   // -128, bits: 10000000   (looks like a huge negative, was meant to be a mask)
u << 7   //  128, bits: 10000000   (positive integer, as expected)
```

If you then compare that "mask" against something with `>` or `<`, or use it as a loop bound, the negative sign silently breaks the code.

**`fmt` prints signed values with a minus sign, not their bit pattern.** The `%b`, `%x`, and `%o` verbs all format the *value*, not the raw bits. On a negative signed integer they emit a leading `-` and the magnitude, which is almost never what you want when debugging a bit trick.

```go
var n int8 = -1
fmt.Printf("%b\n", n)          // -1        (not "11111111")
fmt.Printf("%x\n", n)          // -1        (not "ff")
fmt.Printf("%b\n", uint8(n))   // 11111111  (correct bit pattern)
fmt.Printf("%x\n", uint8(n))   // ff        (correct hex)
```

Convert to the matching unsigned type before printing, and you see the actual bits. Debug output that looks reasonable but hides the sign bit is how bit bugs stay hidden through a whole afternoon.

## Single-Bit Operations

Four one-liners that come up in almost every bit problem. Assume `n` is the integer, `i` is the bit index (0 = LSB).

```go
// Set bit i to 1 (leaves other bits alone).
n | (1 << i)

// Clear bit i to 0 (leaves other bits alone).
n &^ (1 << i)          // in Go
n & ^(1 << i)          // portable equivalent

// Toggle bit i (0 becomes 1, 1 becomes 0).
n ^ (1 << i)

// Check bit i: true if it is 1.
n & (1 << i) != 0
```

Each of these constructs a **single-bit mask** with `1 << i`, then combines it with `n` using the operator that matches the desired truth table:

| Goal   | Truth table | Operator |
|--------|-------------|----------|
| Set    | force to 1  | OR       |
| Clear  | force to 0  | AND NOT  |
| Toggle | flip        | XOR      |
| Check  | isolate     | AND      |

Memorize the four-operator mapping and you have re-derived every single-bit trick.

## Bit Tricks: Isolate, Remove, Test the Lowest Set Bit

Three tricks around the **lowest set bit** (the rightmost 1 in the binary form) appear so often they are worth learning as vocabulary.

### Isolate the Lowest Set Bit

```go
lowest := n & (-n)
```

Why it works: `-n` in two's complement is `^n + 1`. All bits below the lowest set bit of `n` are 0 in `n`, so after complement they are 1, and adding 1 propagates a carry all the way up to that lowest set bit position. The result is a number where **only** the lowest set bit of `n` is set.

```
n     = 0b0010 1100
^n    = 0b1101 0011
^n+1  = 0b1101 0100    (this is -n)
n & -n= 0b0000 0100    (isolates the lowest set bit)
```

Used to find the position or value of the lowest set bit without a loop. `math/bits.TrailingZeros(n)` gives the index of the lowest set bit directly.

### Remove the Lowest Set Bit

```go
n & (n - 1)
```

Why it works: subtracting 1 from `n` flips the lowest set bit to 0 and turns all the 0s below it into 1s. ANDing with the original `n` keeps only the bits above the lowest set bit, which are unchanged by the subtraction.

```
n     = 0b0010 1100
n-1   = 0b0010 1011
n&n-1 = 0b0010 1000    (lowest set bit cleared)
```

This is the core of Brian Kernighan's popcount (below) and of the power-of-two check.

## Power of Two

A positive integer is a power of two iff it has **exactly one bit set**.

```go
func isPowerOfTwo(n int) bool {
    return n > 0 && n&(n-1) == 0
}
```

Why: for a power of two `n = 1 << k`, subtracting 1 gives `k` ones in positions `0` to `k-1` with position `k` cleared. ANDing them gives 0. For any other positive `n`, at least two bits are set, and only the lowest gets cleared by the trick, so bits above it survive and the AND is nonzero.

The `n > 0` guard rules out zero (has no bits set, but is not a power of two) and negative numbers (in two's complement, `-1` is all 1s, and `-1 & -2 == -2 != 0`, so the trick fails on negatives in ways that are not obvious).

## Brian Kernighan's Algorithm

Counting the set bits (aka **population count**, **popcount**, or **Hamming weight**) using the "remove the lowest set bit" trick in a loop.

```go
func popcount(n uint) int {
    c := 0
    for n != 0 {
        n &= n - 1
        c++
    }
    return c
}
```

Complexity is **O(k)** where `k` is the number of set bits in the input. Since a value `n` fits in about `log₂(n)` bits, `k` is at most `⌊log₂(n)⌋ + 1`, so the worst case in terms of the value is **O(log n)** (hit when every bit is 1, e.g. `n = 2^m - 1`). Sparse inputs run in much less: a power of two finishes in 1 iteration, and `n` with 3 set bits finishes in 3, regardless of how large `n` itself is. The naive shift-and-mask loop by contrast always does `w` iterations (word width, usually 32 or 64) no matter the input.

Kernighan published this in 1988. For a fixed word width the compiler will usually turn `bits.OnesCount(n)` into the CPU's dedicated `POPCNT` instruction (single cycle on modern x86 and ARM), so use `math/bits` in real code and reach for Kernighan only in an interview or on hardware without POPCNT.

## XOR Properties and Tricks

XOR has four algebraic properties that make it uniquely useful.

1. `a ^ 0 == a` (0 is the identity)
2. `a ^ a == 0` (every value is its own inverse)
3. `a ^ b == b ^ a` (commutative)
4. `(a ^ b) ^ c == a ^ (b ^ c)` (associative)

Consequence: XORing a list of numbers gives a result that is invariant under reordering and unaffected by any value that appears an even number of times. This is the entire toolkit behind XOR problems: reduce the array with XOR and duplicates cancel while the odd-count element survives. The LeetCode section below works through this on **Single Number** (#136) and its index-pair variant **Missing Number** (#268).

### Swap Without a Temporary

```go
a ^= b
b ^= a  // b is now a^b^b = a
a ^= b  // a is now a^b^a = b
```

A party trick, not something to write in production (the compiler already generates optimal swap code from `a, b = b, a`), but worth knowing that XOR can express it.

## Bitmask for Subsets

If a set has `n` elements, it has `2^n` subsets. Number them `0` to `2^n - 1`. In each number's binary form, bit `i` is 1 iff the `i`th element is in the subset. This turns subset enumeration into a counting loop from `0` to `(1 << n) - 1`, testing each bit of `mask` with `mask & (1 << i)` to decide inclusion. Total O(n * 2^n) time and output size, matching the information-theoretic lower bound. Full Go code lives with **Subsets** (#78) in the LeetCode section.

```
Bitmask enumeration for nums = [a, b, c]:

  mask   binary   subset
  ----   ------   -------
  0      000      {}
  1      001      {a}
  2      010      {b}
  3      011      {a, b}
  4      100      {c}
  5      101      {a, c}
  6      110      {b, c}
  7      111      {a, b, c}
```

This trick has an upper limit at around `n = 62`. Past that, `1 << n` no longer fits in a 64-bit integer. The exact ceiling depends on the type:

- `int64` (Go's default `int` on 64-bit machines): `1 << 62` is `4611686018427387904`, still positive and safe. `1 << 63` puts a 1 in the sign bit, so the value flips to `MinInt64` (`-9223372036854775808`). A loop `for mask := 0; mask < (1 << 63); mask++` starts with `mask < some huge negative`, which is false immediately, so the loop runs zero times and you silently enumerate nothing.
- `uint64`: gains one bit because there is no sign bit to steal. `1 << 63` is `9223372036854775808`, still fine. `1 << 64` shifts the 1 clean out of the word and leaves 0, so a `mask < (1 << 64)` bound is again broken.

Practically the loop time limit hits far earlier. `2^20` is about a million iterations (roughly a second of work in the inner body shown here), `2^30` is a billion (about a minute), and `2^40` is a trillion (around eighteen hours). By the time you are anywhere near `n = 62` you would need the age of the universe to finish, so the overflow number is a language-level ceiling more than a real one. In practice, keep bitmask enumeration to `n <= 20` for exhaustive listing and `n <= 25` for bitmask DP where each state is O(1) work.

Related pattern: **bitmask DP**. Store a set-of-visited-nodes as a bitmask key in a DP table. Travelling Salesman on up to 20 nodes fits in `2^20` states. That extension lives with the DP notes; the enumeration trick above is the entry point.

## Go Template Code

The single-bit ops, the power-of-two check, Kernighan's popcount, and the XOR reduction are all one-liners already shown in their teaching sections above. The one thing worth committing to muscle memory beyond those is the `math/bits` cheat sheet: it is Go's standard-library home for bit primitives and the compiler lowers most of it to single CPU instructions.

### The `math/bits` Cheat Sheet

| Function                              | What it returns                                        |
|---------------------------------------|--------------------------------------------------------|
| `bits.OnesCount(x uint)`              | Number of 1-bits (popcount)                            |
| `bits.LeadingZeros(x uint)`           | Number of leading 0-bits (0s before the first 1 from MSB) |
| `bits.TrailingZeros(x uint)`          | Number of trailing 0-bits (index of lowest set bit)    |
| `bits.Len(x uint)`                    | Minimum bits to represent `x` (`0` for `x == 0`)       |
| `bits.Reverse(x uint)`                | Reverse the bit order                                  |
| `bits.RotateLeft(x uint, k int)`      | Rotate left by `k` bits (circular shift)               |
| `bits.Add`, `bits.Sub`, `bits.Mul`, `bits.Div` | Arithmetic with carry / borrow / overflow output  |

Sized variants exist (`OnesCount32`, `LeadingZeros64`, etc.) when the width matters.

## Where JavaScript Differs

JavaScript's bitwise operators silently truncate operands to **32-bit signed integers**, regardless of whether the source value is a `Number` (64-bit float) or a `BigInt`. That is the single biggest gotcha.

```js
// 1 << 31 is -2147483648 (top bit is the sign bit).
(1 << 31)       // -2147483648
(1 << 31) >>> 0 // 2147483648  (unsigned right shift converts to uint32)
```

`>>>` is JavaScript's **unsigned right shift** and is the standard trick for "convert to uint32 view". Go has no direct equivalent; you convert types explicitly (`int(uint(n) >> 1)`).

For 64-bit bit manipulation in JavaScript, use `BigInt` and its bitwise operators (`&`, `|`, `^`, `<<`, `>>`). BigInt has no fixed width, so there is no `>>>` for BigInt (nothing to be unsigned about).

## LeetCode Problems

### 1. Single Number [#136 (Easy)](https://leetcode.com/problems/single-number/)

Every element in a nonempty array appears twice except for one. Find that one in O(n) time and O(1) space.

<details>
<summary>Brute Force</summary>

Sort the array and scan for the element whose neighbours differ. O(n log n) time, O(1) extra space (or O(n) if the sort is not in place). A hash map of counts is O(n) time and O(n) space. Neither hits the O(1) space bound.
</details>

<details>
<summary>Hint 1</summary>

You need an operation that erases every value that appears twice while preserving the value that appears once, without using extra memory. What binary operator satisfies `x op x = 0` and `x op 0 = x` for every `x`?
</details>

<details>
<summary>Hint 2</summary>

XOR is commutative and associative, so reducing the whole array with XOR does not depend on order. Every pair cancels to 0; the lone element survives.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(n) time, O(1) space
func singleNumber(nums []int) int {
    x := 0
    for _, v := range nums {
        x ^= v
    }
    return x
}
```
</details>

### 2. Number of 1 Bits [#191 (Easy)](https://leetcode.com/problems/number-of-1-bits/)

Return the number of set bits in an unsigned 32-bit integer.

<details>
<summary>Brute Force</summary>

Loop 32 times, extract the LSB with `n & 1`, shift right, accumulate. O(w) time where w = 32 regardless of input. Correct but does redundant work on sparse inputs.
</details>

<details>
<summary>Hint 1</summary>

You can do fewer than w iterations if each iteration guarantees you remove exactly one set bit. Is there an expression whose result has one fewer set bit than the input?
</details>

<details>
<summary>Hint 2</summary>

`n & (n - 1)` clears the lowest set bit. Count how many times you can apply it before `n` reaches zero, and that count is the number of set bits.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(k) time where k = set bits (so O(log n) worst case), O(1) space
func hammingWeight(n uint32) int {
    c := 0
    for n != 0 {
        n &= n - 1
        c++
    }
    return c
}
```

In production use `bits.OnesCount32(n)`, which compiles to the CPU's POPCNT instruction.
</details>

<details>
<summary>Follow-Up (official): If this function is called many times, how would you optimize it?</summary>

Precompute the popcount of every possible byte (256 values) into a lookup table. Each 32-bit input splits into 4 bytes, so one call is 4 table reads plus 3 adds. Trades 256 bytes of memory for a constant 4-step evaluation, with no branches and no shift loop.

```go
var popcountTable [256]uint8

func init() {
    for i := 0; i < 256; i++ {
        popcountTable[i] = popcountTable[i>>1] + uint8(i&1)
    }
}

func hammingWeight(n uint32) int {
    return int(popcountTable[n&0xff] +
        popcountTable[(n>>8)&0xff] +
        popcountTable[(n>>16)&0xff] +
        popcountTable[(n>>24)&0xff])
}
```

The table itself builds in O(256) at startup using the same DP recurrence as Counting Bits (#338): a byte's popcount equals its right-shifted byte's popcount plus its low bit. On real hardware `bits.OnesCount32` beats this (one POPCNT instruction), but when POPCNT is unavailable the byte-table version is the standard answer.
</details>

### 3. Power of Two [#231 (Easy)](https://leetcode.com/problems/power-of-two/)

Return true iff `n` is a power of two. `n` may be any int32, including zero or negative.

<details>
<summary>Brute Force</summary>

Divide by 2 repeatedly while the number is even; report whether you end at 1. O(log n) time. A loop of `1 << i` compared to `n` for `i` from 0 to 30 also works.
</details>

<details>
<summary>Hint 1</summary>

A power of two has exactly one bit set. What single-expression test rejects everything else?
</details>

<details>
<summary>Hint 2</summary>

`n & (n - 1) == 0` is true iff `n` has zero or one set bits. Guard against zero and negatives with `n > 0` and you have the full check.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(1) time and space
func isPowerOfTwo(n int) bool {
    return n > 0 && n&(n-1) == 0
}
```
</details>

<details>
<summary>Follow-Up (official): Could you solve it without loops/recursion?</summary>

The solution shown above already satisfies this. `n > 0 && n&(n-1) == 0` is a straight expression with no loop and no recursive call. The classic "divide by 2 while even" approach does loop, so the official follow-up exists to push the reader toward the bit-trick form.
</details>

### 4. Missing Number [#268 (Easy)](https://leetcode.com/problems/missing-number/)

Given an array of `n` distinct numbers taken from `[0, n]`, return the missing one. O(n) time, O(1) space.

<details>
<summary>Brute Force</summary>

Sum `0 + 1 + ... + n` using `n*(n+1)/2` and subtract the array sum. Correct in O(n) time and O(1) space, but the intermediate sum can overflow for large `n` in fixed-width integers. Sorting is O(n log n). A hash set is O(n) space.
</details>

<details>
<summary>Hint 1</summary>

The array is missing exactly one number from a known range. If you could pair every value in the array with its matching index from `[0, n]`, only the missing number and its index would remain unpaired.
</details>

<details>
<summary>Hint 2</summary>

XOR of an index with the value at that index cancels the pair when they match. Start the accumulator at `n` (the index that has no array entry) and XOR in every `i ^ nums[i]`. What survives is the missing number.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(n) time, O(1) space, no risk of overflow
func missingNumber(nums []int) int {
    x := len(nums)
    for i, v := range nums {
        x ^= i ^ v
    }
    return x
}
```
</details>

<details>
<summary>Follow-Up (official): Could you implement a solution using only O(1) extra space complexity and O(n) runtime complexity?</summary>

The XOR solution above already meets both bounds: one pass through the array (O(n) time) and a single accumulator variable (O(1) space). The sum-formula alternative also fits but risks overflow for large `n` in fixed-width integers; XOR has no such risk since each bit result stays within the type.
</details>

### 5. Counting Bits [#338 (Easy)](https://leetcode.com/problems/counting-bits/)

Given `n`, return an array `ans` of length `n+1` where `ans[i]` is the number of set bits in `i`, computed in O(n) total time.

<details>
<summary>Brute Force</summary>

Call popcount on each `i` from 0 to n. With Brian Kernighan or POPCNT this is O(n log n) or O(n) respectively, but the DP solution below beats both in practice with a single addition per element.
</details>

<details>
<summary>Hint 1</summary>

Every integer `i > 0` is either even or odd. If you already know the bit count of every smaller integer, can you express `ans[i]` in terms of `ans` at a smaller index?
</details>

<details>
<summary>Hint 2</summary>

`i >> 1` is `i` with its lowest bit dropped. So `ans[i] = ans[i>>1] + (i & 1)`: the bit count of the rest, plus 1 if the dropped bit was set.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(n) time, O(n) output space
func countBits(n int) []int {
    dp := make([]int, n+1)
    for i := 1; i <= n; i++ {
        dp[i] = dp[i>>1] + (i & 1)
    }
    return dp
}
```
</details>

<details>
<summary>Follow-Up (official): Can you do it in linear time O(n) in a single pass, without any built-in popcount function?</summary>

The DP solution above satisfies all three constraints at once. It runs a single `for i := 1; i <= n; i++` loop (one pass, O(n) total), and the recurrence `dp[i] = dp[i>>1] + (i & 1)` uses only shift and AND, no language-level popcount intrinsic. The alternative of calling `bits.OnesCount(uint(i))` inside the loop would be O(n) wall-clock but violates the "no built-in" part of the follow-up.
</details>

### 6. Reverse Bits [#190 (Easy)](https://leetcode.com/problems/reverse-bits/)

Reverse the bits of a 32-bit unsigned integer. Bit 0 becomes bit 31, bit 1 becomes bit 30, and so on.

<details>
<summary>Brute Force</summary>

Loop 32 times, extract the LSB of the input, shift it into the result. O(32) time, O(1) space. This is already the natural bit-by-bit solution; there is no "worse" version worth explaining.
</details>

<details>
<summary>Hint 1</summary>

You need to place each input bit at the mirrored output position. Build the result one bit at a time by shifting it left and OR-ing in the next input bit.
</details>

<details>
<summary>Hint 2</summary>

At each step: shift `result` left by 1, OR in `n & 1`, then shift `n` right by 1. After 32 iterations `result` holds the reversed bits.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(32) = O(1) time and space
func reverseBits(n uint32) uint32 {
    var r uint32
    for i := 0; i < 32; i++ {
        r = (r << 1) | (n & 1)
        n >>= 1
    }
    return r
}
```

`math/bits.Reverse32(n)` does this in one instruction on hardware that supports it.
</details>

<details>
<summary>Follow-Up (official): If this function is called many times, how would you optimize it?</summary>

Same shape of answer as the popcount follow-up: precompute the bit-reversal of every byte (256 entries), then reversing a 32-bit value is four lookups and three shift-OR assemblies in the opposite byte order. Trades 256 bytes of memory for a constant-time call with no 32-iteration loop.

```go
var reverseByteTable [256]uint8

func init() {
    for i := 0; i < 256; i++ {
        b := uint8(i)
        var r uint8
        for j := 0; j < 8; j++ {
            r = (r << 1) | (b & 1)
            b >>= 1
        }
        reverseByteTable[i] = r
    }
}

func reverseBits(n uint32) uint32 {
    b0 := uint32(reverseByteTable[n&0xff])
    b1 := uint32(reverseByteTable[(n>>8)&0xff])
    b2 := uint32(reverseByteTable[(n>>16)&0xff])
    b3 := uint32(reverseByteTable[(n>>24)&0xff])
    return (b0 << 24) | (b1 << 16) | (b2 << 8) | b3
}
```

Note the byte order on reassembly: the input's low byte (`n & 0xff`) holds bits 0 to 7, which after reversal must land in bit positions 24 to 31 of the output — hence `b0 << 24`. The high input byte lands in the low output byte. Modern x86 and ARM also expose a native bit-reverse instruction (`RBIT` on ARM, nothing in base x86 but AVX has `VPSHUFB` tricks); `bits.Reverse32` picks the best available.
</details>

### 7. Subsets [#78 (Medium)](https://leetcode.com/problems/subsets/)

Given a set of distinct integers, return all `2^n` subsets. Order does not matter.

<details>
<summary>Brute Force</summary>

Backtracking: at each element choose "include" or "exclude", recurse. O(n * 2^n) time (matches the output size), O(n) recursion depth. This is a valid solution; the bitmask version below is the iterative alternative that some interviewers prefer to see.
</details>

<details>
<summary>Hint 1</summary>

The subsets of an n-element set are in bijection with binary numbers of length n. Each number represents one subset by "is bit `i` set?"
</details>

<details>
<summary>Hint 2</summary>

Loop `mask` from `0` to `(1 << n) - 1`. For each `mask`, walk the bits: if bit `i` is set, include `nums[i]` in this subset.
</details>

<details>
<summary>Solution (Go)</summary>

```go
// O(n * 2^n) time and output space
func subsets(nums []int) [][]int {
    n := len(nums)
    out := make([][]int, 0, 1<<n)
    for mask := 0; mask < (1 << n); mask++ {
        sub := []int{}
        for i := 0; i < n; i++ {
            if mask&(1<<i) != 0 {
                sub = append(sub, nums[i])
            }
        }
        out = append(out, sub)
    }
    return out
}
```
</details>

### 8. Single Number II [#137 (Medium)](https://leetcode.com/problems/single-number-ii/)

Every element in a nonempty array appears **three** times except one, which appears once. Find that one in O(n) time and O(1) space.

<details>
<summary>Brute Force</summary>

Hash map of counts, return the key with count 1. O(n) time, O(n) space. Sort and scan a group of three at a time is O(n log n) time and O(1) space. Neither is the required O(n) time with O(1) space.
</details>

<details>
<summary>Hint 1</summary>

XOR alone does not work: XOR is "mod 2 counting" (a bit set an even number of times cancels). You need a construction that is "mod 3 counting" per bit. Think about one bit position at a time: how would you tell whether the unique number has a 1 there, given that triples contribute 0 or 3 and the singleton contributes 0 or 1?
</details>

<details>
<summary>Hint 2</summary>

The sum of a single bit position across the whole array, taken mod 3, equals the unique number's bit at that position (triples contribute 0 or 3 which drop out, the singleton contributes 0 or 1 which survives). Doing this for all 32 positions gives one straightforward solution. To avoid the inner 32-iteration loop, note that each bit position only ever needs 3 states (0, 1, 2 mod 3), so 2 bits of state per position suffice, which can be stored across two parallel bitmasks.
</details>

<details>
<summary>Solution 1: Count Bits Mod 3 (the one you can invent in an interview)</summary>

For each of 32 bit positions, sum that bit across the array and take the result mod 3. That number is the unique element's bit at that position. Reassemble.

```go
// O(32 * n) = O(n) time, O(1) space
func singleNumber(nums []int) int {
    var ans int32
    for i := 0; i < 32; i++ {
        var count int32
        for _, v := range nums {
            count += int32(v>>i) & 1
        }
        ans |= (count % 3) << i
    }
    return int(ans)
}
```

This already satisfies the problem's O(n) time, O(1) space constraints and is accepted by LeetCode. It is the solution most interviewers expect, because it comes from a plainly stated observation about the structure of the input rather than from pattern memorization.
</details>

<details>
<summary>Solution 2: Two-Accumulator State Machine (the elegant one)</summary>

Tighten the constant factor by doing all 32 bit positions in parallel. Each bit position independently needs to count 0, 1, 2 mod 3, which is 3 states, which fits in 2 bits of state. Store those 2 bits across two parallel bitmasks `ones` and `twos`:

- `(twos, ones) = (0, 0)` means that bit position's count is 0 mod 3.
- `(twos, ones) = (0, 1)` means the count is 1 mod 3.
- `(twos, ones) = (1, 0)` means the count is 2 mod 3.
- `(1, 1)` is unreachable and never occurs.

When value `v` arrives, each bit position's state should either stay put (if `v`'s bit is 0) or advance `0 -> 1 -> 2 -> 0` (if `v`'s bit is 1). From this we derive two Boolean expressions, one for the new `ones` and one for the new `twos`.

**Deriving `new_ones`.** List every valid starting state through v = 0 and v = 1 and read off what the next ones-bit must be:

```
  twos  ones  v    new_ones
   0     0    0       0
   0     1    0       1
   1     0    0       0
   0     0    1       1       (0 -> 1)
   0     1    1       0       (1 -> 2, so ones goes to 0)
   1     0    1       0       (2 -> 0, stays 0)
```

The rows with `new_ones = 1` share two properties: `ones XOR v = 1` and `twos = 0`. So:

```
new_ones = (ones XOR v) AND NOT twos
```

**Deriving `new_twos`.** Now the state is `(twos, new_ones, v)` because `ones` has already been updated. Run the same six starts through to their final states and read off what the twos-bit must be:

```
  twos  new_ones  v    new_twos
   0      0       0       0
   0      1       0       0
   1      0       0       1     (count 2 staying 2)
   0      0       1       0
   0      1       1       1     (count 1 getting a 1 -> count 2)
   1      0       1       0     (count 2 getting a 1 -> count 0)
```

The two rows where `new_twos = 1` share: `twos XOR v = 1` and `new_ones = 0`. So:

```
new_twos = (twos XOR v) AND NOT new_ones
```

Both formulas have the same shape (`(X ^ v) AND NOT Y`); the `AND NOT` factor is what blocks the unreachable `(1, 1)` state. Translated to Go (where `&^` is AND NOT):

```go
// O(n) time, O(1) space
func singleNumber(nums []int) int {
    ones, twos := 0, 0
    for _, v := range nums {
        ones = (ones ^ v) &^ twos    // uses old twos
        twos = (twos ^ v) &^ ones    // uses just-updated ones
    }
    return ones
}
```

Order matters: `twos` must be updated after `ones` because its formula references the new `ones`. Swap the two lines and the second bit position in a dry run on `[2,2,3,2]` produces the wrong answer.

After processing every number, each bit position's state is its count mod 3. The unique element contributes 1 to the count at every bit position where it has a 1, so those positions end up in state `(0, 1)`. Return `ones` to read them out.
</details>

<details>
<summary>Interview Strategy</summary>

Walk in with Solution 1. It is derivable on the spot from the observation that triples sum to a multiple of 3. Only if asked to remove the inner 32-iteration loop should you reach for Solution 2, and derive it in front of the interviewer by writing the truth table rather than quoting the formula from memory. The general shape also tells you how to generalize: for a problem saying "appears k times except one," you need `⌈log₂ k⌉` parallel bitmasks. k = 2 collapses to plain XOR (Single Number #136); k = 3 needs two bitmasks (this problem); k = 5 would need three.
</details>

## References

- Henry Warren, *Hacker's Delight* (2nd ed.) - the canonical reference for bit-twiddling algorithms with proofs
- Sean Anderson, [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html) - Stanford's collection of one-liner bit tricks with derivations
- [Go source: math/bits](https://pkg.go.dev/math/bits) - standard library documentation for every bit-level primitive
- Brian Kernighan and Dennis Ritchie, *The C Programming Language* (2nd ed.), Exercise 2-9 - the original 1988 popcount trick
- [Wikipedia: Two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) - full derivation of the negative-number encoding
- [MDN: Bitwise operators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators#bitwise_operators) - JavaScript's 32-bit truncation rules
- Internal: `dsa/fundamentals/math-for-dsa.md` - powers of 2, modular arithmetic, logarithms
- Internal: `dsa/data-structures/arrays.md` - array-based alternatives when the bit count exceeds one machine word