---
title: Heaps and Priority Queues — Basics (Java)
date: 2026-04-11
tags:
  - dsa
  - heap
  - priority-queue
  - java
description: What a min-heap is, how Java’s PriorityQueue works, and two classic uses—k largest elements and “merge k sorted” at a high level.
---

# Heaps and Priority Queues — Basics (Java)

After [[dsa/monotonic-deque-sliding-window-min-max]] mentioned heaps for “top k” style work, here is a **small** note on the same idea: a **priority queue** backed by a **binary heap**.

---

## What you get

A **min-heap** keeps the **smallest** element at the top. In Java, `PriorityQueue` is a min-heap by default (for `Integer`, smaller numbers come out first).

Typical costs (average / amortized for a binary heap):

- **Insert (`offer`)** — O(log n)
- **Peek smallest** — O(1)
- **Remove smallest (`poll`)** — O(log n)

So it is slower per operation than a stack or deque, but you gain **global order** among everything you inserted.

---

## Min vs max in Java

```java
import java.util.Comparator;
import java.util.PriorityQueue;

// Min-heap (default for Integer)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
```

---

## Pattern 1: K largest numbers in an array

**Idea:** Keep a **min-heap** of size at most `k`. If the heap has more than `k` elements, remove the smallest — what remains are the `k` largest values seen so far.

```java
import java.util.PriorityQueue;

static int[] kLargest(int[] nums, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>(); // min-heap
    for (int x : nums) {
        pq.offer(x);
        if (pq.size() > k) {
            pq.poll();
        }
    }
    return pq.stream().mapToInt(i -> i).toArray();
}
```

**Time:** O(n log k) if you never let the heap grow past `k`. **Space:** O(k).

(If you need the **k smallest**, use a **max-heap** of size `k` instead — mirror image.)

---

## Pattern 2: Merge K sorted lists (sketch)

You have several sorted lists (or arrays). Always take the **smallest head** among lists, append it to the result, advance that list’s pointer, and **push the new head** into a min-heap keyed by value.

That is the usual interview shape; the details are bookkeeping with indices or nodes. Same heap API: `offer`, `peek`, `poll`.

---

## When a heap is a good fit

- **Top k** / **kth** largest or smallest.
- **Scheduling** — “process the job with smallest deadline next.”
- **Merging** many sorted streams by always picking the current minimum.

When the problem is **strictly a sliding window min/max** on a line, [[dsa/monotonic-deque-sliding-window-min-max]] is often O(n) and nicer than a heap — but heaps are easier when order is not “window-shaped.”

---

Next in [[dsa/index|DSA]]: trees and traversals (DFS pre/in/post-order, BFS level-order).
