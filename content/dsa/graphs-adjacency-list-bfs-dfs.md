---
title: Graphs — Adjacency List, BFS, and DFS
date: 2026-04-14
tags:
  - dsa
  - graph
  - bfs
  - dfs
  - java
description: Representing a graph with an adjacency list, breadth-first search (shortest hops), and depth-first search (recursive or stack)—the usual interview starting point.
---

# Graphs — Adjacency List, BFS, and DFS

[[dsa/binary-search-tree-basics]] was still a tree (one parent, no back edges). A **graph** is a set of **vertices** (nodes) and **edges** between them; interviews usually start with **unweighted** graphs and either **undirected** or **directed** edges.

---

## Representation: adjacency list

For `n` vertices labeled `0 .. n-1`, keep a list of neighbors for each vertex:

```java
import java.util.ArrayList;
import java.util.List;

List<List<Integer>> buildGraph(int n, int[][] edges, boolean undirected) {
    List<List<Integer>> g = new ArrayList<>();
    for (int i = 0; i < n; i++) g.add(new ArrayList<>());
    for (int[] e : edges) {
        int u = e[0], v = e[1];
        g.get(u).add(v);
        if (undirected) g.get(v).add(u);
    }
    return g;
}
```

**Space:** O(V + E) to store all edges once (twice if undirected). This beats a dense **adjacency matrix** O(V²) when the graph is sparse.

---

## BFS (breadth-first)

Use a **queue** and a **visited** set (or boolean array). Visit start, then neighbors layer by layer.

- **Unweighted shortest path** (fewest edges from `start` to `target`): BFS is the standard tool — the first time you reach a node is via a shortest route.

```java
import java.util.ArrayDeque;
import java.util.Deque;

boolean bfsReachable(List<List<Integer>> g, int start, int target) {
    int n = g.size();
    boolean[] seen = new boolean[n];
    Deque<Integer> q = new ArrayDeque<>();
    q.add(start);
    seen[start] = true;
    while (!q.isEmpty()) {
        int u = q.removeFirst();
        if (u == target) return true;
        for (int v : g.get(u)) {
            if (!seen[v]) {
                seen[v] = true;
                q.add(v);
            }
        }
    }
    return false;
}
```

**Time:** O(V + E) — each vertex enters the queue once; each edge is relaxed once from its tail.

---

## DFS (depth-first)

Explore as far as possible along one branch, then **backtrack**. Two common styles:

**Recursive** — short to write; uses the call stack O(h) where `h` can be V in a long path.

```java
boolean dfsRecursive(List<List<Integer>> g, int u, int target, boolean[] seen) {
    if (u == target) return true;
    seen[u] = true;
    for (int v : g.get(u)) {
        if (!seen[v] && dfsRecursive(g, v, target, seen)) return true;
    }
    return false;
}
```

**Iterative** — explicit `Deque` as a stack: push `u`, pop, push unvisited neighbors.

DFS is great for **connectivity**, **cycle detection** (with extra state), and many grid problems that are graphs in disguise.

---

## BFS vs DFS (quick)

| Goal | Often use |
|------|-----------|
| Fewest edges / levels from a start | **BFS** |
| “Is there a path?” / count components | Either; DFS is often less code |
| Grid flood fill | DFS or BFS both common |

---

## Pitfalls

- **Disconnected graph** — outer loop over vertices: if not `seen[i]`, start a new BFS/DFS from `i` (counts **connected components**).
- **Self-loops / multi-edges** — clarify the problem; adjacency list may list duplicates unless you dedupe.

---

Next in [[dsa/index|DSA]]: [[dsa/topological-sort-kahn-indegree|Topological sort on a DAG (Kahn’s algorithm with indegree)]].
