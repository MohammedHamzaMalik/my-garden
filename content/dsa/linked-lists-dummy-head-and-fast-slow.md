---
title: Linked Lists — Dummy Head and Fast/Slow Pointers
date: 2026-04-08
tags:
  - dsa
  - linked-list
  - two-pointers
  - java
description: Linked list fundamentals, why dummy heads simplify edge cases, and the fast/slow pointer pattern for middle, cycle detection, and related problems.
---

# Linked Lists — Dummy Head and Fast/Slow Pointers

After [[dsa/binary-search-sorted-array-and-answer-space]], the next “core” structure to get comfortable with is the **linked list**. It shows up in interviews because it forces you to reason about **pointers**, **edges**, and **invariants** (instead of indexing).

---

## What makes linked lists different?

In an array, you can jump to `arr[i]` in O(1).  
In a singly linked list, you can only move forward node-by-node:

- **Access by position:** O(n)
- **Insert/delete at head (or after a known node):** O(1)
- **Insert/delete at tail:** O(1) *only if* you keep a tail pointer; otherwise O(n) to walk there

Most interview problems focus on **pointer manipulation**, not on building a fancy list implementation.

---

## A minimal `ListNode` (Java)

```java
static class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}
```

---

## Pattern 1: Dummy head (sentinel node)

Many bugs come from deleting or inserting at the **head**, because the “previous” node doesn’t exist. A **dummy head** solves this by giving you a guaranteed node *before* the real head.

### Example: remove all nodes with a target value

```java
static ListNode removeAll(ListNode head, int target) {
    ListNode dummy = new ListNode(0, head);
    ListNode prev = dummy;
    ListNode cur = head;

    while (cur != null) {
        if (cur.val == target) {
            prev.next = cur.next; // delete cur
        } else {
            prev = cur;           // keep cur
        }
        cur = cur.next;
    }
    return dummy.next;
}
```

**Why dummy helps:** removing the first real node becomes the same operation as removing any other node—just redirect `prev.next`.

---

## Pattern 2: Reversing a linked list (3 pointers)

This is the “hello world” of pointer rewiring. It’s also a subroutine for many harder tasks (reverse in k-group, palindrome list, etc.).

```java
static ListNode reverse(ListNode head) {
    ListNode prev = null;
    ListNode cur = head;
    while (cur != null) {
        ListNode next = cur.next;
        cur.next = prev;
        prev = cur;
        cur = next;
    }
    return prev;
}
```

Invariant: `prev` is the head of the already-reversed prefix, and `cur` is the next node to reverse.

---

## Pattern 3: Fast/slow pointers (tortoise/hare)

Fast/slow pointers are “two pointers” again, but instead of bounding a range (sliding window), they move at different speeds:

- `slow` moves 1 step
- `fast` moves 2 steps

This finds midpoints and detects cycles with O(1) extra space.

### A) Find the middle node

For even length lists, you should decide whether you want the **left-middle** or **right-middle**. The loop below returns the **right-middle**.

```java
static ListNode middleRight(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow;
}
```

### B) Detect a cycle (Floyd’s algorithm)

If there’s a cycle, `fast` will eventually “lap” `slow` and they meet.

```java
static boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### C) Find the cycle entry

Once they meet, reset one pointer to head. Move both 1 step; they meet at the entry.

```java
static ListNode cycleEntry(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) {
            ListNode p1 = head;
            ListNode p2 = slow;
            while (p1 != p2) {
                p1 = p1.next;
                p2 = p2.next;
            }
            return p1;
        }
    }
    return null;
}
```

---

## When to think “linked list patterns”

- You see **head**, **next**, **node**, **cycle**, **remove nth**, **reverse**, **merge**, **palindrome**.
- The problem says **O(1) extra space** and the input is a list.
- Edge cases around the head feel annoying → reach for a **dummy head**.
- You need a midpoint or cycle detection → reach for **fast/slow**.

---

Next in [[dsa/index|DSA]]: monotonic queues for sliding window min/max (a close cousin of the monotonic stack idea).

