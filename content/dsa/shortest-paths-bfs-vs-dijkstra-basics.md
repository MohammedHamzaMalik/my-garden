---
title: Shortest Paths — BFS (Unweighted) vs Dijkstra (Weighted)
date: 2026-04-17
tags:
  - dsa
  - graph
  - bfs
  - dijkstra
  - shortest-path
  - java
description: When BFS already gives shortest paths, when you need Dijkstra with a min-heap, and a small Java sketch—building on graph BFS and heaps.
---

# Shortest Paths — BFS (Unweighted) vs Dijkstra (Weighted)

[[dsa/union-find-dsu-basics]] was about **connectivity**. This note is about **distance**: fewest edges, or smallest total edge weight.

---

## Unweighted (or “every edge costs 1”)

If every edge has the same positive cost, **shortest path = fewest edges** from a start node.

**BFS** from `start` explores nodes in increasing distance from `start`, so the **first time** you reach a node you have found a shortest path in edge count.

You already have the traversal pattern in [[dsa/graphs-adjacency-list-bfs-dfs]]; add a `dist[]` array (or `level` in the queue):

- `dist[start] = 0`
- when you first visit `v` from `u`, set `dist[v] = dist[u] + 1`

**Time:** O(V + E). **Space:** O(V).

---

## Weighted, non-negative edges — Dijkstra

If edges have **weights** and all weights are **≥ 0**, **Dijkstra’s algorithm** finds shortest **total weight** from one source.

**Idea:** always settle the not-yet-done node with the **smallest** tentative distance (a greedy step that is correct when weights are non-negative). A **min-heap** stores `(distance, node)`.

```java
import java.util.*;

List<List<int[]>> buildWeightedGraph(int n, int[][] edges) {
    List<List<int[]>> g = new ArrayList<>();
    for (int i = 0; i < n; i++) g.add(new ArrayList<>());
    for (int[] e : edges) {
        int u = e[0], v = e[1], w = e[2];
        g.get(u).add(new int[] { v, w });
    }
    return g;
}

int[] dijkstra(List<List<int[]>> g, int start) {
    int n = g.size();
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;

    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[] { 0, start });

    while (!pq.isEmpty()) {
        int[] cur = pq.poll();
        int d = cur[0], u = cur[1];
        if (d != dist[u]) continue; // stale entry

        for (int[] e : g.get(u)) {
            int v = e[0], w = e[1];
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.offer(new int[] { dist[v], v });
            }
        }
    }
    return dist;
}
```

With a **binary heap**, each edge relaxation costs O(log V) → **O((V + E) log V)** time in typical sparse graphs.

---

## Quick map

| Graph | Goal | Tool |
|-------|------|------|
| Unweighted | Fewest edges | **BFS** + `dist[]` |
| Non-negative weights | Min total weight | **Dijkstra** + min-heap |
| Negative edges allowed | Shortest paths (no negative cycles) | Bellman–Ford (not covered here) |

---

## Relation to earlier notes

- BFS layer order is the same spirit as [[dsa/binary-trees-dfs-and-bfs-traversals]] level-order, but on a general graph with a **visited** / `dist` check.
- Dijkstra’s heap is the same `PriorityQueue` pattern as [[dsa/heaps-and-priority-queue-basics]].

---

Next in [[dsa/index|DSA]]: [[dsa/recursion-memoization-and-dp-intro|Recursion, memoization, and a first DP pattern (Fibonacci / climbing stairs shape)]].
