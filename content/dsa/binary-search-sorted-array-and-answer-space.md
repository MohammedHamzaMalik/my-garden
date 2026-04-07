---
title: Binary Search — Sorted Arrays and Answer Space
date: 2026-04-07
tags:
  - dsa
  - binary-search
  - java
description: Classic binary search on sorted data, avoiding off-by-one bugs, and searching on an implicit “answer” range when the problem isn’t a plain array lookup.
---

# Binary Search — Sorted Arrays and Answer Space

[[dsa/stacks-and-monotonic-stack]] teased **binary search**. It’s not only “find `x` in a sorted array”—the same halving idea applies whenever you can **discard half** of a candidate range and keep the correct answer in the other half.

---

## 1. Classic: find index in a sorted array

**Goal:** Given sorted `nums` and `target`, return an index `i` with `nums[i] == target`, or `-1` if missing.

**Idea:** Compare `target` to the **middle** element. If `target` is smaller, search the **left** half; if larger, search the **right** half; if equal, done.

```java
int binarySearch(int[] nums, int target) {
    int lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2; // avoid overflow vs (lo + hi) / 2
        if (nums[mid] == target) return mid;
        if (nums[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

- **Time:** O(log n) — the range halves each step.
- **Space:** O(1) for this iterative version.

**Invariant:** If `target` exists, it always lies in `[lo, hi]` until found or the range is empty.

---

## 2. Variants (same skeleton)

- **First / last position** of `target` — after finding one hit, keep searching left or right.
- **Insert position** (lower bound) — smallest `i` with `nums[i] >= target`; often solved with `while (lo < hi)` and careful `mid` rounding.

The details change; the pattern is still **narrow the interval** using comparisons at `mid`.

---

## 3. Answer-space binary search

Sometimes there is **no sorted array** of candidates—only a **numeric answer** in a range `[minAnswer, maxAnswer]`.

**Example sketch:** “What is the **minimum** capacity `K` such that some feasibility check passes?” If:

- larger `K` always makes the check **easier** (or always passes), and  
- smaller `K` makes it **harder**,

then feasibility is **monotonic** in `K`. You can binary search on `K`:

```text
lo = min possible answer, hi = max possible answer
while lo < hi:
    mid = (lo + hi) / 2   // or lower/upper mid depending on problem
    if feasible(mid):
        hi = mid          // try smaller (or lo = mid if searching max)
    else:
        lo = mid + 1
return lo
```

You still need a correct `feasible(mid)` and careful **mid** handling so the loop terminates (many bugs are off-by-one here).

---

## 4. When *not* to use binary search

- No **order** or **monotonicity**—you can’t discard half the range safely.
- Data is **unsorted** and sorting doesn’t help the question (e.g. you need all pairs).

---

## Quick map

| Flavor | What you halve | Typical use |
|--------|----------------|-------------|
| Index BS | Index range in sorted array | Find value, bounds |
| Answer BS | Integer / real range of “answer” | Min max capacity, min days, etc. |

---

Next in [[dsa/index|DSA]]: linked lists (dummy head, fast/slow pointers).
