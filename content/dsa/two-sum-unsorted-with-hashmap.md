---
title: Two Sum (Unsorted) — HashMap One-Pass Pattern
date: 2026-04-09
tags:
  - dsa
  - arrays
  - hashing
  - java
description: Solve Two Sum on an unsorted array in O(n) time using a one-pass HashMap (value → index) and complement lookups.
---

# Two Sum (Unsorted) — HashMap One-Pass Pattern

In [[dsa/two-pointers-sorted-arrays]] we used two pointers because the array was sorted. But for an **unsorted** array, moving pointers doesn’t give you a safe way to discard possibilities.

That’s where hashing shines. This is the classic “unsorted two sum” pattern: **store what you’ve seen**, and check if the **complement** exists.

---

## Problem

Given an integer array `nums` (unsorted) and an integer `target`, return indices `(i, j)` such that:

- `nums[i] + nums[j] == target`
- `i != j`

---

## Brute force (what we want to beat)

Check every pair:

- **Time:** \(O(n^2)\)
- **Space:** \(O(1)\)

Works, but slow.

---

## The key idea (complement lookup)

If `nums[i]` is the number we’re looking at, the value we *need* is:

\[
complement = target - nums[i]
\]

So the question becomes:

> Have we already seen `complement` earlier in the array?

If yes, we’re done.

---

## One-pass HashMap solution (Java)

We keep a map: **value → index** for values we’ve already processed.

Important detail: we check first, then insert. That prevents using the same element twice.

```java
int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> indexByValue = new HashMap<>();

    for (int i = 0; i < nums.length; i++) {
        int x = nums[i];
        int need = target - x;

        if (indexByValue.containsKey(need)) {
            return new int[] { indexByValue.get(need), i };
        }

        // store AFTER checking so we don't match the same element
        indexByValue.put(x, i);
    }

    return new int[] { -1, -1 };
}
```

### Complexity

- **Time:** \(O(n)\) average (each step does O(1) map operations)
- **Space:** \(O(n)\) for the map

---

## Subtle detail: duplicates

Example: `nums = [3, 3]`, `target = 6`

- At `i=0`, map is empty → store `3 → 0`
- At `i=1`, need is `3`, map contains it → return `(0, 1)`

So duplicates are handled naturally.

---

## When to use this vs sorting + two pointers

### Prefer HashMap when…

- The array is **unsorted**
- You must return **original indices**
- You want a clean one-pass solution

### Prefer sort + two pointers when…

- You don’t need original indices (or you can track them)
- You want **O(1)** extra space (ignoring sorting overhead)
- Sorting is allowed and doesn’t break the problem

This is the same tradeoff we saw in [[dsa/arrays-and-hashing-basics]]: **space to buy time**.

---

Next in [[dsa/index|DSA]]: “three sum” and why sorting becomes worth it there.

