---
title: Topological Sort — Kahn’s Algorithm (Indegree)
date: 2026-04-15
tags:
  - dsa
  - graph
  - topological-sort
  - bfs
  - java
description: A simple intro to topological sorting on DAGs using Kahn’s algorithm with indegree counts and a queue.
---

# Topological Sort — Kahn’s Algorithm (Indegree)

After [[dsa/graphs-adjacency-list-bfs-dfs]], a natural next step is **topological sort**.

A topological order is a linear ordering of nodes where every directed edge `u -> v` means `u` appears before `v`.

It only exists for a **DAG** (Directed Acyclic Graph).

---

## Intuition

If a node has **indegree 0** (no prerequisites), it can come next in the ordering.

Kahn's algorithm repeats:

1. Put all indegree-0 nodes in a queue.
2. Pop one, add it to answer.
3. "Remove" its outgoing edges (decrease indegree of neighbors).
4. Any neighbor that becomes indegree 0 goes into queue.

If you process all nodes, you have a valid order.  
If some nodes are never processed, there is a cycle.

---

## Java implementation

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

List<Integer> topoSortKahn(int n, int[][] edges) {
    List<List<Integer>> g = new ArrayList<>();
    for (int i = 0; i < n; i++) g.add(new ArrayList<>());

    int[] indegree = new int[n];
    for (int[] e : edges) {
        int u = e[0], v = e[1];
        g.get(u).add(v);
        indegree[v]++;
    }

    Deque<Integer> q = new ArrayDeque<>();
    for (int i = 0; i < n; i++) {
        if (indegree[i] == 0) q.addLast(i);
    }

    List<Integer> order = new ArrayList<>();
    while (!q.isEmpty()) {
        int u = q.removeFirst();
        order.add(u);
        for (int v : g.get(u)) {
            indegree[v]--;
            if (indegree[v] == 0) q.addLast(v);
        }
    }

    // cycle check
    if (order.size() != n) return new ArrayList<>();
    return order;
}
```

---

## Complexity

- **Time:** O(V + E)
- **Space:** O(V + E)

Each node enters/leaves the queue once, and each edge is considered once.

---

## Common uses

- Course scheduling (prerequisites)
- Build/dependency ordering
- Task pipelines with dependency edges

---

Next in [[dsa/index|DSA]]: union-find (disjoint set union) for connectivity and cycle checks.

