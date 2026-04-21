---
title: Intervals — Merge, Meeting Rooms, Sweep Line Basics
date: 2026-04-21
tags:
  - dsa
  - intervals
  - sorting
  - sweep-line
  - java
description: "A simple interval toolkit: sort-by-start + merge, meeting room overlap check, and the sweep-line/counting-events idea."
---

# Intervals — Merge, Meeting Rooms, Sweep Line Basics

After [[dsa/one-dimensional-dp-house-robber-and-kadane]], here is a common non-DP pattern: **interval problems**.

Most interval tasks begin with one move:
**sort intervals by start time**.

---

## 1) Merge intervals

Given intervals like `[start, end]`, combine overlapping ranges.

### Idea

1. Sort by `start`.
2. Keep a result list.
3. If current interval overlaps with the last merged one, extend the end.
4. Otherwise, start a new merged interval.

### Java

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    List<int[]> out = new ArrayList<>();

    for (int[] cur : intervals) {
        if (out.isEmpty() || out.get(out.size() - 1)[1] < cur[0]) {
            out.add(new int[] { cur[0], cur[1] });
        } else {
            out.get(out.size() - 1)[1] =
                Math.max(out.get(out.size() - 1)[1], cur[1]);
        }
    }
    return out.toArray(new int[out.size()][]);
}
```

---

## 2) Meeting rooms (can one person attend all?)

Given meeting intervals, return false if any two overlap.

### Idea

Sort by start; if `current.start < previous.end`, overlap exists.

```java
import java.util.Arrays;

boolean canAttendAll(int[][] meetings) {
    Arrays.sort(meetings, (a, b) -> Integer.compare(a[0], b[0]));
    for (int i = 1; i < meetings.length; i++) {
        if (meetings[i][0] < meetings[i - 1][1]) return false;
    }
    return true;
}
```

---

## 3) Sweep-line basics (count active intervals)

For “maximum simultaneous meetings” or “minimum rooms required”:

- Turn each interval into two events:
  - `(start, +1)`
  - `(end, -1)`
- Sort events by time (with tie rules as needed).
- Scan left to right, keep running count of active intervals.
- Track maximum count.

This “events + running total” trick appears in many scheduling problems.

---

## Complexity

Most interval solutions are dominated by sorting:

- **Time:** O(n log n)
- **Space:** O(n) for output/events (or O(1) extra if in-place checks only)

---

## Quick pattern

When you see time ranges:

1. sort by start (or event time),
2. scan once,
3. maintain current merged/active state.

That solves a large chunk of interval questions.

---

Next in [[dsa/index|DSA]]: prefix sums (range sum queries, subarray sum equals k).

