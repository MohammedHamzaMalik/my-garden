---
title: Binary Search Trees — Property, Search, Insert, Delete
date: 2026-04-13
tags:
  - dsa
  - tree
  - bst
  - java
description: The BST invariant, iterative search, insert, and a short sketch of delete (leaf, one child, two children)—building on binary tree traversals.
---

# Binary Search Trees — Property, Search, Insert, Delete

[[dsa/binary-trees-dfs-and-bfs-traversals]] introduced a generic binary tree. A **binary search tree (BST)** adds an ordering rule so **search** and **insert** behave like binary search on a sorted array. Use the same `TreeNode` shape (`val`, `left`, `right`) as in that note.

---

## The invariant

For every node `n`:

- All values in the **left** subtree are **strictly less than** `n.val` (or equal if you put duplicates on the left—pick one convention and stay consistent).
- All values in the **right** subtree are **strictly greater than** `n.val`.

**Inorder traversal** (left → root → right) visits keys in **sorted order** — that is the usual way to check or print a BST in order.

---

## Search

Start at the root. Compare `target` to `node.val`: go **left** if smaller, **right** if larger, **done** if equal, **null** if you fall off the tree.

```java
TreeNode search(TreeNode root, int target) {
    TreeNode cur = root;
    while (cur != null) {
        if (target == cur.val) return cur;
        if (target < cur.val) cur = cur.left;
        else cur = cur.right;
    }
    return null;
}
```

**Time:** O(h) where `h` is height — O(log n) for a balanced tree, O(n) if the tree is a thin chain.

---

## Insert

Same walk as search. When you reach a **null** child, attach a new node there.

```java
TreeNode insert(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    if (val < root.val) root.left = insert(root.left, val);
    else if (val > root.val) root.right = insert(root.right, val);
    // if duplicates not allowed and val == root.val, do nothing or handle by policy
    return root;
}
```

(Iterative insert mirrors the search loop and keeps a `parent` reference—same idea.)

---

## Delete (sketch)

Find the node, then three cases:

1. **Leaf** — remove it (parent’s pointer to `null`).
2. **One child** — replace the node with its only child.
3. **Two children** — replace `node.val` with the **inorder successor** (smallest value in the **right** subtree: go right once, then left as far as possible), then **delete that successor node** (which has at most one child).

Implementations often use a helper `minNode(root)` for case 3. The full recursive delete is a bit long for a “simple” note; in interviews, explaining the three cases is often enough before coding.

---

## Why BSTs matter

- **Sorted order** via inorder; **range queries** are natural (with extra structure for production).
- **Unbalanced** BSTs degenerate to linked lists — real libraries often use **balanced** trees (AVL, red-black) or skip BSTs for pure in-memory maps and use **hash tables** instead when only key→value lookup matters.

---

Next in [[dsa/index|DSA]]: [[dsa/graphs-adjacency-list-bfs-dfs|Graphs — adjacency list, BFS, and DFS]].
