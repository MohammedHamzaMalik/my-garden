---
title: 1D DP — House Robber and Maximum Subarray
date: 2026-04-20
tags:
  - dsa
  - dynamic-programming
  - arrays
  - java
description: "Two simple 1D DP patterns on arrays: House Robber (pick non-adjacent max sum) and Kadane's algorithm (maximum subarray sum)."
---

# 1D DP — House Robber and Maximum Subarray

After [[dsa/grid-dp-unique-paths-and-min-path-sum]], here is a simpler DP shape: **1D DP on arrays**.

This note covers two classics:

- **House Robber** (choose non-adjacent elements for max sum)
- **Maximum Subarray** (best contiguous sum, Kadane)

---

## 1) House Robber

At each index `i`, you decide:

- **skip** current house -> best stays `dp[i - 1]`
- **take** current house -> `dp[i - 2] + nums[i]`

So:

`dp[i] = max(dp[i - 1], dp[i - 2] + nums[i])`

### O(1) space Java

```java
int rob(int[] nums) {
    int prev2 = 0; // dp[i - 2]
    int prev1 = 0; // dp[i - 1]
    for (int x : nums) {
        int cur = Math.max(prev1, prev2 + x);
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```

---

## 2) Maximum Subarray (Kadane)

At each index, best subarray ending at `i` is either:

- start fresh at `nums[i]`, or
- extend previous best ending at `i - 1`

Transition:

`endHere = max(nums[i], endHere + nums[i])`

Track global best while scanning.

### Java

```java
int maxSubArray(int[] nums) {
    int endHere = nums[0];
    int best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        endHere = Math.max(nums[i], endHere + nums[i]);
        best = Math.max(best, endHere);
    }
    return best;
}
```

---

## Complexity

Both patterns run in:

- **Time:** O(n)
- **Space:** O(1)

---

## Quick takeaway

For 1D DP, define what your running state means:

- House Robber: best up to index `i`
- Kadane: best subarray that **must end at** index `i`

Once the state meaning is clear, transitions become straightforward.

---

Next in [[dsa/index|DSA]]: interval problems (merge intervals, meeting rooms, and sweep-line basics).

