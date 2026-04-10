---
title: Monotonic Deque — Sliding Window Min and Max
date: 2026-04-10
tags:
  - dsa
  - deque
  - monotonic-queue
  - sliding-window
  - java
description: Using a monotonic deque to get min or max in every fixed-size window in O(n)—the natural follow-up to sliding window + monotonic stack.
---

# Monotonic Deque — Sliding Window Min and Max

[[dsa/sliding-window-fixed-and-variable]] showed sliding windows with sums and frequency maps. [[dsa/stacks-and-monotonic-stack]] introduced **monotonic stacks** for “next greater” style problems. A **monotonic deque** (often implemented with `ArrayDeque`) is the usual tool when you need the **minimum or maximum inside every window of size `k`** in **O(n)** total time.

---

## The problem

Given an array `nums` and window size `k`, for each contiguous window of length `k`, output the **minimum** (or **maximum**) value in that window.

- **Naive:** for each window, scan `k` elements → **O(n·k)**.
- **Heap of size k:** insert/remove → **O(n log k)**.
- **Monotonic deque:** each index pushed and popped at most once → **O(n)** time, **O(k)** space.

---

## Idea: deque of indices, monotonic by value

Keep a deque of **indices** `i` such that corresponding values `nums[i]` are **strictly increasing** from front to back (for **window minimum**).

- **Front** = index of the **smallest** value among candidates still inside the current window.
- Before adding index `i`, **pop from the back** while `nums[back] >= nums[i]` — those indices can never be the minimum while `i` is in the window.
- After moving `right`, **pop from the front** if that index is **left of** the window (`index <= right - k`).

When `right >= k - 1`, the front of the deque is the answer for the window ending at `right`.

For **window maximum**, use a **decreasing** deque: pop from the back while `nums[back] <= nums[i]`.

---

## Window minimum (Java)

```java
import java.util.ArrayDeque;
import java.util.Deque;

static int[] minSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] ans = new int[n - k + 1];
    Deque<Integer> dq = new ArrayDeque<>(); // indices, nums[dq] increasing

    for (int i = 0; i < n; i++) {
        while (!dq.isEmpty() && nums[dq.peekLast()] >= nums[i]) {
            dq.pollLast();
        }
        dq.addLast(i);
        if (dq.peekFirst() <= i - k) {
            dq.pollFirst();
        }
        if (i >= k - 1) {
            ans[i - k + 1] = nums[dq.peekFirst()];
        }
    }
    return ans;
}
```

**Window maximum:** change the `while` condition to `nums[dq.peekLast()] <= nums[i]` and keep the rest the same (deque stores indices with **decreasing** values).

---

## Why it’s O(n)

Each index is **added to the deque once** and **removed at most once** from the front and at most once from the back over the whole scan. So the total number of deque operations is linear in `n`.

---

## How this relates to the monotonic stack

- **Stack:** you only pop when the new element “wins” from one direction (often left-to-right).
- **Deque:** same **monotonicity** invariant, plus you **drop indices that fell out of the window** from the **front**—because the window slides.

---

## When to use it

- “**Sliding window maximum/minimum**” with fixed `k`.
- Variants that need the min/max in a range that advances in one direction (streaming).

When you only need “top k” or arbitrary order, a **heap** is often simpler even if it’s **O(log k)** per step.

---

Next in [[dsa/index|DSA]]: heaps / priority queues (k largest, merge k sorted lists, scheduling).
