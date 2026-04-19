---
title: Grid DP — Unique Paths and Min Path Sum
date: 2026-04-19
tags:
  - dsa
  - dynamic-programming
  - grid
  - java
description: "A simple 2D dynamic programming intro with two classic problems: counting unique paths and minimizing path sum in a grid."
---

# Grid DP — Unique Paths and Min Path Sum

After [[dsa/recursion-memoization-and-dp-intro]], this is the first 2D DP pattern.

In many grid problems, each cell depends on the cell **above** and **left**, so a table-based DP is natural.

---

## Problem 1: Unique Paths

You start at top-left and can move only **right** or **down**.  
How many ways to reach bottom-right?

### DP state

`dp[r][c]` = number of ways to reach cell `(r, c)`.

### Transition

`dp[r][c] = dp[r - 1][c] + dp[r][c - 1]`

Why: last move into `(r, c)` comes from top or left.

### Java

```java
int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int r = 0; r < m; r++) dp[r][0] = 1;
    for (int c = 0; c < n; c++) dp[0][c] = 1;

    for (int r = 1; r < m; r++) {
        for (int c = 1; c < n; c++) {
            dp[r][c] = dp[r - 1][c] + dp[r][c - 1];
        }
    }
    return dp[m - 1][n - 1];
}
```

---

## Problem 2: Minimum Path Sum

Same moves (right/down), but each cell has a cost.  
Find the minimum sum from top-left to bottom-right.

### DP state

`dp[r][c]` = minimum path sum to reach `(r, c)`.

### Transition

`dp[r][c] = grid[r][c] + min(dp[r - 1][c], dp[r][c - 1])`

### Java

```java
int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];

    for (int r = 1; r < m; r++) {
        dp[r][0] = dp[r - 1][0] + grid[r][0];
    }
    for (int c = 1; c < n; c++) {
        dp[0][c] = dp[0][c - 1] + grid[0][c];
    }

    for (int r = 1; r < m; r++) {
        for (int c = 1; c < n; c++) {
            dp[r][c] = grid[r][c] + Math.min(dp[r - 1][c], dp[r][c - 1]);
        }
    }
    return dp[m - 1][n - 1];
}
```

---

## Complexity

For both:

- **Time:** O(m * n)
- **Space:** O(m * n) with a full table

You can often reduce space to O(n) using one row.

---

## Pattern takeaway

For grid DP, always define:

1. what `dp[r][c]` means,
2. base row/column,
3. transition from smaller cells.

Once this is clear, coding is straightforward.

---

Next in [[dsa/index|DSA]]: 1D DP on arrays (house robber and maximum subarray).

