---
title: Big O and Why It Matters
date: 2026-03-14
tags:
  - dsa
  - complexity
  - basics
description: A short intro to Big O notation and why we use it when talking about data structures and algorithms.
---

# Big O and Why It Matters

Before diving into specific data structures and algorithms, it helps to have a shared language for **how fast** or **how much memory** an approach uses. That’s what Big O notation is for.

---

## What is Big O?

**Big O** describes how time or space *grows* when the input size (e.g. length of an array, number of nodes in a tree) gets larger. We usually care about the **worst case** or the **typical case**, and we ignore constants and small terms so we can compare ideas clearly.

Examples:

- **O(1)** — constant. Same cost no matter how big the input (e.g. reading one element from an array by index).
- **O(n)** — linear. Cost grows in proportion to input size (e.g. scanning an array once).
- **O(n²)** — quadratic. Cost grows with the square of input size (e.g. two nested loops over the same array).
- **O(log n)** — logarithmic. Cost grows slowly as input grows (e.g. binary search on a sorted array).

---

## Why it matters

- **Interviews** — You’re often asked to improve an O(n²) solution to O(n) or O(n log n).
- **Real systems** — A slow algorithm can turn a small feature into a bottleneck when data grows.
- **Choosing structures** — Arrays give O(1) access by index but O(n) for search; hash sets give O(1) lookup by value. Big O helps you pick the right tool.

---

## One example

**Problem:** Check if a value exists in an array.

- **Unsorted array, one loop:** O(n) time, O(1) extra space.
- **Sorted array, binary search:** O(log n) time, O(1) extra space.
- **Put elements in a HashSet first, then lookup:** O(n) time to build, O(1) per lookup after; O(n) extra space.

So: no single “best” answer—it depends on how often you search, whether the array is sorted, and whether you can afford the extra space. Big O makes that tradeoff explicit.

---

This is the first note in my [[dsa/index|DSA]] section. Next I’ll add notes on arrays, hashing, and common patterns (two pointers, sliding window, etc.).
