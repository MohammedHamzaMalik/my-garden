---
title: Recursion, Memoization, and a First DP Pattern
date: 2026-04-18
tags:
  - dsa
  - dynamic-programming
  - recursion
  - memoization
  - java
description: From exponential recursion to memoized DP—Fibonacci and climbing stairs as the smallest useful examples, plus bottom-up tabulation.
---

# Recursion, Memoization, and a First DP Pattern

After [[dsa/shortest-paths-bfs-vs-dijkstra-basics]], graphs are covered for “shortest route” style problems. Many other questions are **optimization** or **counting** on a **smaller state** that you build up from subproblems—that is the **dynamic programming** family.

This note keeps one pattern only: **Fibonacci-shaped** recurrences (also **climbing stairs**).

---

## 1. Naive recursion (slow)

```java
long fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2);
}
```

The call tree **recomputes** the same `n` many times → time grows **exponentially** in `n`.

---

## 2. Memoization (top-down DP)

Cache answers by `n`. Each `n` is computed once.

```java
import java.util.Arrays;

long fibMemo(int n, long[] memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];
    memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    return memo[n];
}

long fibMemo(int n) {
    long[] memo = new long[n + 1];
    Arrays.fill(memo, -1);
    return fibMemo(n, memo);
}
```

**Time:** O(n). **Space:** O(n) for the array and recursion stack.

---

## 3. Tabulation (bottom-up DP)

Fill an array from small indices upward—no recursion stack depth.

```java
long fibTab(int n) {
    if (n <= 1) return n;
    long a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        long c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

**Time:** O(n). **Space:** O(1) if you only need the last two values.

---

## Same shape: climbing stairs

You can climb **1** or **2** steps at a time. Ways to reach step `n`:

`ways(n) = ways(n - 1) + ways(n - 2)`, with base cases `ways(0) = 1`, `ways(1) = 1` (or shift indices—problems vary).

So it is the same engine as Fibonacci after a change of bases.

---

## When this pattern shows up

- Counting paths with fixed step sizes.
- Strings or arrays where the answer at `i` depends only on `i-1` and `i-2` (or a small window).
- Any recursion where you notice **the same arguments** appearing again and again.

---

## Vocabulary (minimal)

- **Overlapping subproblems** — the same sub-result is needed many times (memo fixes this).
- **Optimal substructure** — optimal answer for the big problem uses optimal answers for smaller ones (more visible in “min cost” problems later).

---

Next in [[dsa/index|DSA]]: grid DP (unique paths, min path sum—two classic 2D examples).
