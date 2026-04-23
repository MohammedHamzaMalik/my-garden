---
title: Prefix Sums — Range Query and Subarray Sum Equals K
date: 2026-04-22
tags:
  - dsa
  - arrays
  - prefix-sum
  - hashing
  - java
description: "A simple prefix-sum toolkit for O(1) range sums and O(n) counting of subarrays with sum k using a hash map."
---

# Prefix Sums — Range Query and Subarray Sum Equals K

After [[dsa/intervals-merge-meeting-rooms-sweepline-basics]], here is a very common array trick: **prefix sums**.

Prefix sums turn repeated range-sum work from “sum every time” into “subtract two precomputed values.”

---

## 1) Range sum query in O(1)

For array `nums`, define:

`pref[i] = nums[0] + nums[1] + ... + nums[i - 1]`

So `pref` has length `n + 1`, and `pref[0] = 0`.

Then sum of `nums[l..r]` is:

`pref[r + 1] - pref[l]`

### Java

```java
int[] buildPrefix(int[] nums) {
    int n = nums.length;
    int[] pref = new int[n + 1];
    for (int i = 0; i < n; i++) {
        pref[i + 1] = pref[i] + nums[i];
    }
    return pref;
}

int rangeSum(int[] pref, int l, int r) {
    return pref[r + 1] - pref[l];
}
```

Build once in O(n), answer each query in O(1).

---

## 2) Subarray Sum Equals K (count)

Count how many subarrays have sum exactly `k`.

### Key equation

If current prefix sum is `sum`, and a previous prefix was `sum - k`, then the subarray between them has sum `k`.

So while scanning, keep a frequency map:

- key = prefix sum value
- value = how many times seen

### Java

```java
import java.util.HashMap;
import java.util.Map;

int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1); // empty prefix

    int sum = 0, ans = 0;
    for (int x : nums) {
        sum += x;
        ans += freq.getOrDefault(sum - k, 0);
        freq.put(sum, freq.getOrDefault(sum, 0) + 1);
    }
    return ans;
}
```

---

## Why this works with negatives too

Sliding window sum tricks often fail with negative numbers.  
Prefix sum + hash map still works, because it is based on exact algebra (`sum - previous = k`), not monotonic movement.

---

## Complexity

- Range-sum queries: build O(n), each query O(1)
- Subarray-sum-equals-k: O(n) time, O(n) space

---

## Quick pattern

When you see:

- many range-sum queries, or
- “count subarrays with sum = target”

think **prefix sum**, often combined with a **hash map**.

---

Next in [[dsa/index|DSA]]: [[dsa/bit-manipulation-xor-masks-basics|bit manipulation basics (xor tricks, bit masks, and common operations)]].

