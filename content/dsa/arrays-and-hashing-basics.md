---
title: Arrays & Hashing — First Pattern
date: 2026-04-04
tags:
  - dsa
  - arrays
  - hashing
  - java
description: Using arrays with hash maps and sets for O(1) lookups, frequency counts, and complement problems—building on Big O.
---

# Arrays & Hashing — First Pattern

After [[dsa/big-o-and-why-it-matters]], the next practical step is the **arrays + hashing** combo: keep data in an array (or list), use a **hash map** or **hash set** when you need fast “have I seen this?” or “how many times?” answers.

---

## Why pair arrays with hashing?

- **Array by index** — O(1) access if you know the index.
- **Search by value in an unsorted array** — O(n) if you scan every time.
- **Hash set / map** — O(1) average for insert, lookup, and delete (assuming a good hash).

So: walk the array once, build a set or map, then answer questions in constant time per query.

---

## Pattern 1: “Have I seen this before?”

**Idea:** Use a `Set` (or `HashSet` in Java).

Example: detect if an array has a duplicate.

```java
boolean hasDuplicate(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    for (int x : nums) {
        if (!seen.add(x)) {
            return true; // already in the set
        }
    }
    return false;
}
```

- **Time:** O(n) — one pass.
- **Space:** O(n) — the set.

Compare to nested loops: O(n²) time, O(1) extra space. Trading space for time is the usual tradeoff here.

---

## Pattern 2: “How many times?”

**Idea:** Use a `Map` from value → count (`HashMap` in Java).

Example: character or number frequencies.

```java
Map<Integer, Integer> freq = new HashMap<>();
for (int x : nums) {
    freq.merge(x, 1, Integer::sum);
}
```

Useful for anagrams, “top k frequent,” and many string problems.

---

## Pattern 3: Complement / target sum

**Idea:** For each `x`, check if `target - x` was already seen (set or map).

Classic shape: two-sum style thinking with one pass and O(n) extra space instead of O(n²) pairs.

---

## When *not* to reach for a hash map

- Input is **sorted** and you only need pairs or ranges — **two pointers** can be O(n) with O(1) space.
- Keys are **small integers in a range** — a **count array** can beat a map (no hashing overhead).

---

This is today’s note in my [[dsa/index|DSA]] section. Next I’ll write about **two pointers** on sorted arrays and when it beats hashing on space.
