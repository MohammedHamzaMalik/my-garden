---
title: Bit Manipulation — XOR, Masks, and Common Operations
date: 2026-04-23
tags:
  - dsa
  - bit-manipulation
  - math
  - java
description: "A simple bit manipulation starter: set/check/clear/toggle bits, XOR tricks, and common interview-style bit patterns."
---

# Bit Manipulation — XOR, Masks, and Common Operations

After [[dsa/prefix-sums-range-query-and-subarray-sum-k]], here is a short note on **bit manipulation**.

Bits can look low-level, but a few operations solve many interview problems cleanly.

---

## 1) Core operators

For integer bits in Java:

- `&` AND
- `|` OR
- `^` XOR
- `~` NOT
- `<<` left shift
- `>>` signed right shift
- `>>>` unsigned right shift

---

## 2) Bit mask basics (index `i`)

Use `1 << i` as a mask for bit `i`.

```java
int mask = 1 << i;

boolean isSet = (x & mask) != 0; // check bit i
int setBit = x | mask;           // set bit i to 1
int clearBit = x & ~mask;        // clear bit i to 0
int toggleBit = x ^ mask;        // flip bit i
```

---

## 3) XOR properties worth remembering

- `a ^ a = 0`
- `a ^ 0 = a`
- XOR is commutative and associative

So if every number appears twice except one, XOR of all numbers leaves the single one.

```java
int singleNumber(int[] nums) {
    int x = 0;
    for (int v : nums) x ^= v;
    return x;
}
```

---

## 4) Check if power of two

A positive power of two has exactly one set bit:

```java
boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

Reason: subtracting 1 flips that single `1` bit to `0` and turns lower bits to `1`, so AND becomes zero only in that case.

---

## 5) Count set bits (Brian Kernighan trick)

`x & (x - 1)` removes the lowest set bit.

```java
int popcount(int x) {
    int cnt = 0;
    while (x != 0) {
        x &= (x - 1);
        cnt++;
    }
    return cnt;
}
```

Time is proportional to number of set bits, not total bit width.

---

## Quick pattern

When a problem says:

- “exactly one element appears once,”
- “subset/state compression,”
- “toggle/check flags,”

think in terms of masks and XOR first.

---

Next in [[dsa/index|DSA]]: [[dsa/backtracking-subsets-permutations-basics|backtracking basics (subsets, permutations, and decision tree thinking)]].

