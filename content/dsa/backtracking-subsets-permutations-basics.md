---
title: Backtracking Basics — Subsets and Permutations
date: 2026-04-24
tags:
  - dsa
  - backtracking
  - recursion
  - java
description: "A simple backtracking starter: decision tree thinking, subsets (include/exclude), and permutations (used array + path)."
---

# Backtracking Basics — Subsets and Permutations

After [[dsa/bit-manipulation-xor-masks-basics]], here is a simple recursion pattern: **backtracking**.

Backtracking explores a decision tree:

1. choose,
2. recurse,
3. undo choice (backtrack).

---

## 1) Subsets (power set)

For each element, you have two choices:

- include it
- skip it

That naturally forms a binary decision tree.

### Java

```java
import java.util.ArrayList;
import java.util.List;

List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> ans = new ArrayList<>();
    dfs(0, nums, new ArrayList<>(), ans);
    return ans;
}

void dfs(int i, int[] nums, List<Integer> path, List<List<Integer>> ans) {
    if (i == nums.length) {
        ans.add(new ArrayList<>(path));
        return;
    }

    // skip nums[i]
    dfs(i + 1, nums, path, ans);

    // take nums[i]
    path.add(nums[i]);
    dfs(i + 1, nums, path, ans);
    path.remove(path.size() - 1); // undo
}
```

---

## 2) Permutations

Now order matters, so at each depth choose any unused element.

### Java

```java
import java.util.ArrayList;
import java.util.List;

List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> ans = new ArrayList<>();
    boolean[] used = new boolean[nums.length];
    backtrack(nums, used, new ArrayList<>(), ans);
    return ans;
}

void backtrack(int[] nums, boolean[] used, List<Integer> path, List<List<Integer>> ans) {
    if (path.size() == nums.length) {
        ans.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, used, path, ans);
        path.remove(path.size() - 1); // undo
        used[i] = false;              // undo
    }
}
```

---

## Complexity intuition

- Subsets: `2^n` results -> O(n * 2^n) including output copy cost.
- Permutations: `n!` results -> O(n * n!) including output copy cost.

Backtracking is often exponential because it enumerates combinations.

---

## Quick pattern

When a problem asks for:

- all subsets / combinations / permutations,
- all valid strings or boards (N-Queens, parentheses),

think “build partial answer, recurse, undo.”

---

Next in [[dsa/index|DSA]]: trie basics (prefix matching, insert/search/startsWith).

