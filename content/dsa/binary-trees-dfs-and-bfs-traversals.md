---
title: Binary Trees — DFS and BFS Traversals
date: 2026-04-12
tags:
  - dsa
  - tree
  - dfs
  - bfs
  - java
description: A minimal binary tree node, preorder/inorder/postorder DFS on paper, and level-order BFS with a queue—enough to read most interview tree problems.
---

# Binary Trees — DFS and BFS Traversals

After [[dsa/heaps-and-priority-queue-basics]], **trees** are the next common shape. This note stays small: one binary tree definition, **DFS** order names, and **BFS** (level order) with a queue.

---

## Node shape (Java)

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode(int v) { val = v; }
}
```

Many problems are “given the root, return …” — recursion or an explicit stack/queue is the usual tool.

---

## DFS: when you visit the root

For each subtree, you visit **root**, **left**, **right** in some order:

| Order | When you process `root` |
|--------|-------------------------|
| **Preorder** | First (root → left → right) |
| **Inorder** | Middle (left → root → right) — for BST, this visits keys in sorted order |
| **Postorder** | Last (left → right → root) — good when children must be solved before the parent |

**Recursive** preorder is the shortest mental model:

```java
void preorder(TreeNode n, List<Integer> out) {
    if (n == null) return;
    out.add(n.val);
    preorder(n.left, out);
    preorder(n.right, out);
}
```

Swap the three lines to get inorder or postorder. **Time:** O(n) nodes, **space:** O(h) for the recursion stack (`h` = height).

---

## BFS: level order (breadth-first)

Visit nodes **row by row**, left to right. Use a **queue**: dequeue a node, enqueue its children.

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Deque<TreeNode> q = new ArrayDeque<>();
    q.add(root);
    while (!q.isEmpty()) {
        int size = q.size(); // nodes in this level
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            TreeNode n = q.removeFirst();
            level.add(n.val);
            if (n.left != null) q.add(n.left);
            if (n.right != null) q.add(n.right);
        }
        res.add(level);
    }
    return res;
}
```

**Time:** O(n). **Extra space:** O(w) where `w` is the maximum width (last level can be ~n/2 nodes in a complete tree).

---

## Quick pick

- Need **sorted order in a BST** or “flatten” with order constraints → often **inorder** DFS.
- Need bottom-up info from subtrees (e.g. height, sum) → often **postorder** DFS.
- Need **depth**, **level**, or “view by row” → **BFS**.

---

Next in [[dsa/index|DSA]]: binary search trees (BST property, search, insert, delete sketch).
