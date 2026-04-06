---
title: Stacks and the Monotonic Stack Pattern
date: 2026-04-06
tags:
  - dsa
  - stack
  - monotonic-stack
  - java
description: LIFO stacks, “next greater element,” and monotonic stacks—keeping indices in increasing or decreasing order to answer range queries in one pass.
---

# Stacks and the Monotonic Stack Pattern

After [[dsa/sliding-window-fixed-and-variable]] walked contiguous windows on a line, **stacks** handle problems where the **last thing pushed** is the **first** popped (LIFO)—undo, parentheses, DFS, and a family of “next greater / smaller element” problems.

---

## Stack basics

A **stack** supports roughly:

- `push(x)` — add on top  
- `pop()` — remove from top  
- `peek()` — look at top without removing  

In Java, `Deque<Integer> st = new ArrayDeque<>()` is a common choice: use `push`, `pop`, `peek` (or `addLast` / `removeLast` for the same end).

**Typical complexity:** push/pop/peek are **O(1)** amortized.

---

## “Next greater element”

**Problem (one common form):** For each index `i` in an array, find the **first** index `j > i` with `nums[j] > nums[i]`, or report “none.”

A naive scan from each `i` is **O(n²)**. A **monotonic stack** does it in **O(n)** total.

**Idea:** Walk `i` from `0` to `n-1`. Keep a stack of **indices** whose values are in **decreasing** order (from bottom to top). When `nums[i]` is **greater** than the value at the stack’s top index, you’ve found the **next greater** for that index—record it and pop. Then push `i`.

```java
int[] nextGreaterIndex(int[] nums) {
    int n = nums.length;
    int[] ans = new int[n];
    Arrays.fill(ans, -1);
    Deque<Integer> st = new ArrayDeque<>();

    for (int i = 0; i < n; i++) {
        while (!st.isEmpty() && nums[i] > nums[st.peek()]) {
            int j = st.pop();
            ans[j] = i; // next greater for j is at index i
        }
        st.push(i);
    }
    return ans;
}
```

(Variants return **values** instead of indices, or use `-1` for “no greater to the right.”)

---

## Why “monotonic”?

After each step, stack indices point to values that are **strictly decreasing** from bottom to top. That invariant lets you **discard** candidates that can never be the “next greater” for future positions once a taller bar arrives.

---

## Related ideas

- **Monotonic queue** — similar spirit for sliding window min/max in O(n).
- **Histogram / largest rectangle** — often combines stack with careful popping.

---

## When to think “stack”

- Nested structure: brackets, tags, file paths.
- “Undo” or **last unmatched** item matters.
- For each position, an answer depends on the **nearest** smaller/greater on the left or right—monotonic stack is a standard tool.

---

Next in [[dsa/index|DSA]]: binary search (not only on sorted arrays—on an answer space).
