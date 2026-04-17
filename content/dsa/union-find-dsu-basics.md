---
title: Union-Find (DSU) — Basics
date: 2026-04-16
tags:
  - dsa
  - graph
  - union-find
  - dsu
  - java
description: A simple intro to Disjoint Set Union (Union-Find) with path compression and union by size for near-constant connectivity checks.
---

# Union-Find (DSU) — Basics

After [[dsa/topological-sort-kahn-indegree]], another graph-friendly tool is **Union-Find** (also called **DSU**: Disjoint Set Union).

It answers a common question fast:  
**Are two nodes in the same connected component?**

---

## Core idea

Maintain groups of nodes. Each group has a representative called a **root**.

- `find(x)` returns x's root.
- `union(a, b)` merges the groups containing `a` and `b`.
- `connected(a, b)` is true if `find(a) == find(b)`.

If implemented with:

- **path compression** in `find`, and
- **union by size/rank** in `union`,

operations are effectively near O(1) in practice (amortized inverse Ackermann).

---

## Java implementation

```java
class DSU {
    private final int[] parent;
    private final int[] size;

    DSU(int n) {
        parent = new int[n];
        size = new int[n];
        for (int i = 0; i < n; i++) {
            parent[i] = i;
            size[i] = 1;
        }
    }

    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // path compression
        }
        return parent[x];
    }

    boolean union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return false; // already same component

        // union by size: attach smaller tree under larger
        if (size[ra] < size[rb]) {
            int tmp = ra;
            ra = rb;
            rb = tmp;
        }
        parent[rb] = ra;
        size[ra] += size[rb];
        return true;
    }

    boolean connected(int a, int b) {
        return find(a) == find(b);
    }
}
```

---

## Typical uses

- Dynamic connectivity: repeatedly add edges and ask if two nodes are connected.
- Cycle detection in an undirected graph:
  - for each edge `(u, v)`, if `find(u) == find(v)`, a cycle exists;
  - else `union(u, v)`.
- Kruskal's MST algorithm (pick edges by weight while avoiding cycles).

---

## Quick complexity

For `m` operations on `n` elements:

- **Time:** O(m * alpha(n)) amortized (very close to linear)
- **Space:** O(n)

---

Next in [[dsa/index|DSA]]: [[dsa/shortest-paths-bfs-vs-dijkstra-basics|Shortest paths — BFS (unweighted) vs Dijkstra (weighted, non-negative)]].

