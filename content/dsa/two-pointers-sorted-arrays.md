---
title: Two Pointers — Sorted Arrays
date: 2026-04-05
tags:
  - dsa
  - arrays
  - two-pointers
  - java
description: The two-pointer technique on sorted arrays—O(n) time and O(1) extra space by moving left and right indices instead of nested loops.
---

# Two Pointers — Sorted Arrays

In [[dsa/arrays-and-hashing-basics]] we used **hashing** to get O(1) lookups at the cost of O(n) extra space. When the array is **sorted** (or you can sort it once), **two pointers** often give you O(n) time with **O(1) extra space**—no map needed.

---

## The idea

Place one pointer at the **start** (`left`) and one at the **end** (`right`) of the array. Each step, look at `arr[left]` and `arr[right]`, decide something (sum too small? too big?), and move **one** pointer. Because the array is ordered, each move throws away a chunk of impossible answers—similar to binary search, but with two indices.

---

## Classic shape: target sum in a sorted array

**Problem:** Given sorted `nums` and `target`, find two indices such that `nums[i] + nums[j] == target` (or report none).

```java
int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            return new int[] { left, right };
        }
        if (sum < target) {
            left++;      // need a larger sum
        } else {
            right--;     // need a smaller sum
        }
    }
    return new int[] { -1, -1 };
}
```

- **Time:** O(n) — each pointer moves at most n steps.
- **Space:** O(1) — only two indices.

Compare to checking every pair: O(n²). Sorting first costs O(n log n); if the input is already sorted, the scan is linear.

---

## When to think “two pointers”

Signals:

- **Sorted** sequence (or you sort once and the problem still makes sense).
- You need a **pair**, **subarray**, or **window** whose property depends on both ends.
- You can make a **local decision** (move left or right) that never invalidates a future optimal answer.

Examples families: two-sum / three-sum variants, merging sorted arrays, some palindrome checks, container-with-water style problems.

---

## When hashing is still better

- Input is **not** sorted and sorting would **break** the problem (e.g. original indices matter and you cannot reorder).
- You need **many arbitrary lookups** by value, not a walking pair from both ends.

---

## Relation to the last note

| Approach | Typical time | Extra space | Best when |
|----------|--------------|-------------|-----------|
| Hash map ([[dsa/arrays-and-hashing-basics]]) | O(n) | O(n) | Unsorted, one-pass complement lookup |
| Two pointers (this note) | O(n) on sorted | O(1) | Sorted array, pair from both ends |

---

Next in [[dsa/index|DSA]]: sliding window on arrays (fixed and variable size).
