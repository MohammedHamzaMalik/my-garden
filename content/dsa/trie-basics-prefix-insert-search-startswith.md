---
title: Trie Basics — Prefix Matching, Insert, Search, startsWith
date: 2026-04-25
tags:
  - dsa
  - trie
  - strings
  - java
description: A simple trie introduction for prefix matching with insert, exact search, and startsWith operations in Java.
---

# Trie Basics — Prefix Matching, Insert, Search, startsWith

After [[dsa/backtracking-subsets-permutations-basics]], here is a string data structure that appears often in interviews: the **trie** (prefix tree).

A trie stores words character by character, sharing common prefixes.

---

## Why use a trie?

If you need many prefix checks, a trie is often cleaner than repeated string scans.

Typical operations for a word of length `L`:

- `insert(word)` -> O(L)
- `search(word)` -> O(L)
- `startsWith(prefix)` -> O(L)

---

## Node idea

Each node holds:

- references to children (next characters),
- a boolean `isWord` marking end of a full word.

This implementation uses lowercase English letters (`a` to `z`).

---

## Java implementation

```java
class Trie {
    static class Node {
        Node[] next = new Node[26];
        boolean isWord;
    }

    private final Node root = new Node();

    public void insert(String word) {
        Node cur = root;
        for (char ch : word.toCharArray()) {
            int i = ch - 'a';
            if (cur.next[i] == null) cur.next[i] = new Node();
            cur = cur.next[i];
        }
        cur.isWord = true;
    }

    public boolean search(String word) {
        Node node = walk(word);
        return node != null && node.isWord;
    }

    public boolean startsWith(String prefix) {
        return walk(prefix) != null;
    }

    private Node walk(String s) {
        Node cur = root;
        for (char ch : s.toCharArray()) {
            int i = ch - 'a';
            if (i < 0 || i >= 26 || cur.next[i] == null) return null;
            cur = cur.next[i];
        }
        return cur;
    }
}
```

---

## Quick example

- Insert `"cat"`, `"car"`, `"dog"`.
- `search("car")` -> true
- `search("ca")` -> false (prefix but not full word)
- `startsWith("ca")` -> true

---

## Trade-offs

- Fast prefix operations.
- Extra memory due to many node references.
- For large alphabets, use a `HashMap<Character, Node>` per node to save space.

---

## Pattern cue

When a problem says:

- “find words by prefix,”
- “autocomplete,”
- “dictionary search with shared prefixes,”

think trie.

---

Next in [[dsa/index|DSA]]: greedy basics (activity selection, jump game intuition).

