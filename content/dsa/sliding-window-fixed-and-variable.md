---
title: Sliding Window — Fixed and Variable Size
date: 2026-04-06
tags:
  - dsa
  - arrays
  - sliding-window
  - two-pointers
description: Sliding window on arrays and strings—fixed-size windows for sums, variable windows with expand/shrink for constraints like “at most K distinct.”
---

# Sliding Window — Fixed and Variable Size

[[dsa/two-pointers-sorted-arrays]] used two pointers on **sorted** data. **Sliding window** is another two-pointer pattern: `left` and `right` bound a **contiguous segment** (window) in an array or string. You grow or shrink the window so some property stays valid or so you optimize a metric.

---

## Fixed-size window

**Problem sketch:** Given an array, find the **maximum sum** among all contiguous subarrays of length `k`.

**Idea:** Sum the first `k` elements. Then slide: add the element entering the window, subtract the one leaving.

```java
int maxSumSubarrayOfSizeK(int[] nums, int k) {
    if (nums.length < k) return 0;
    int sum = 0;
    for (int i = 0; i < k; i++) sum += nums[i];
    int best = sum;
    for (int right = k; right < nums.length; right++) {
        sum += nums[right] - nums[right - k];
        best = Math.max(best, sum);
    }
    return best;
}
```

- **Time:** O(n) — each index touched a constant number of times.
- **Space:** O(1).

---

## Variable-size window

**Problem sketch:** Longest substring with **at most** `K` distinct characters (or: smallest subarray with sum ≥ target).

**Idea:**

1. Expand `right` until the window is **valid** (or invalid, depending on the problem).
2. When it becomes **invalid**, shrink from `left` until it is valid again.
3. Track the best answer whenever the window satisfies the goal.

You often keep a **frequency map** (or array of counts) for the current window so you know in O(1) whether shrinking is needed.

Pseudo-shape:

```text
left = 0
for right = 0 .. n-1:
    add nums[right] to window state
    while window is invalid:
        remove nums[left] from window state
        left++
    update answer with [left, right]
```

---

## Fixed vs variable — quick comparison

| Style | `right` moves | `left` moves | Typical use |
|-------|---------------|--------------|-------------|
| Fixed k | Always +1 each step | Follows `right - k + 1` | Rolling sum, averages |
| Variable | Expand until constraint | Shrink while invalid | Longest valid substring, min length with sum ≥ S |

---

## Relation to hashing

For “distinct count in window” you combine **sliding window** with the **frequency map** idea from [[dsa/arrays-and-hashing-basics]]: the map holds counts only for indices inside `[left, right]`.

---

Next in [[dsa/index|DSA]]: stacks and monotonic stacks for “next greater element” style problems.
