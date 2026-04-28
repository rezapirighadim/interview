# Trie (Prefix Tree)

## What is it?

A **Trie** (pronounced "try", from re**trie**val) is a tree where each node represents a **character**, and paths from root to a marked node spell out a complete word. It's optimized for prefix-based operations.

```
Words: ["apple", "app", "apt", "bat"]

        root
       /    \
      a      b
      |      |
      p      a
     / \     |
    p   t    t
    |   |
    l   (end)
    |
    e
  (end)
(end at "app" too)
```

## When to use it?

- **Autocomplete** / prefix search
- **Word search** — does a word or prefix exist?
- **Spell checker**
- Problems involving many words where you need prefix lookups
- **Word Break**, **Replace Words**, **Search Suggestions**

## When is it better than a hash set?

- When you need **prefix operations** (starts with, count words with prefix)
- When you need to **share common prefixes** to save space
- When input is a large dictionary and you do repeated prefix queries

## Implementation

```python
class TrieNode:
    def __init__(self):
        self.children = {}   # char → TrieNode
        self.is_end = False  # marks end of a word

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word: str) -> bool:
        node = self.root
        for char in word:
            if char not in node.children:
                return False
            node = node.children[char]
        return node.is_end

    def startsWith(self, prefix: str) -> bool:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return False
            node = node.children[char]
        return True
```

---

## Problem 1 — Implement Trie (LeetCode #208)

Implement a Trie with `insert`, `search`, and `startsWith`.

### Solution

The full implementation is above. Key methods:
- `insert` — O(m) where m = word length
- `search` — O(m), must reach end of word AND `is_end = True`
- `startsWith` — O(m), only requires reaching the end of the prefix

### Explanation

- Each path from root to a node with `is_end = True` represents a complete word.
- `startsWith` is the same as `search` but without checking `is_end`.
- Using a dict for `children` is more flexible than a fixed array of 26.
- **Time:** O(m) per operation | **Space:** O(total characters inserted)

---

## Problem 2 — Design Add and Search Words (LeetCode #211)

Design a data structure that supports `addWord(word)` and `search(word)` where `search` supports `.` as a wildcard matching any letter.

### Solution

```python
class WordDictionary:
    def __init__(self):
        self.root = TrieNode()

    def addWord(self, word: str) -> None:
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_end = True

    def search(self, word: str) -> bool:
        return self._search(word, 0, self.root)

    def _search(self, word: str, idx: int, node: TrieNode) -> bool:
        if idx == len(word):
            return node.is_end

        char = word[idx]

        if char == '.':
            # try all possible children
            for child in node.children.values():
                if self._search(word, idx + 1, child):
                    return True
            return False
        else:
            if char not in node.children:
                return False
            return self._search(word, idx + 1, node.children[char])
```

### Explanation

- For regular characters, proceed as a normal trie search.
- For `.`, branch into every child and recursively search from there.
- This is DFS on the trie, branching only at wildcard positions.
- **Time:** O(m) for normal words, O(26^k · m) worst case for k wildcards | **Space:** O(m) recursion

---

## Problem 3 — Word Search II (LeetCode #212)

**Given** a 2D board and a list of words, find all words that exist in the board (adjacent cells, no reuse).

### Solution

```python
def findWords(board: list[list[str]], words: list[str]) -> list[str]:
    trie = Trie()
    for word in words:
        trie.insert(word)

    rows, cols = len(board), len(board[0])
    result = set()

    def dfs(node, r, c, path):
        if node.is_end:
            result.add(path)
            node.is_end = False  # avoid duplicates

        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] not in node.children:
            return

        char = board[r][c]
        board[r][c] = '#'  # mark visited

        for dr, dc in [(1,0),(-1,0),(0,1),(0,-1)]:
            dfs(node.children[char], r+dr, c+dc, path+char)

        board[r][c] = char  # restore

    for r in range(rows):
        for c in range(cols):
            if board[r][c] in trie.root.children:
                dfs(trie.root, r, c, '')

    return list(result)
```

### Explanation

- Build a trie from all target words — this lets us prune the DFS early (no matching prefix → stop).
- At each cell, DFS while following the trie. If we reach `is_end`, we found a word.
- This is vastly more efficient than running a separate DFS for each word.
- **Time:** O(rows·cols·4^L) where L = max word length | **Space:** O(total chars in words)

---

## Problem 4 — Replace Words (LeetCode #648)

**Given** a dictionary of roots and a sentence, replace each word with its shortest root prefix.

### Solution

```python
def replaceWords(dictionary: list[str], sentence: str) -> str:
    trie = Trie()
    for root in dictionary:
        trie.insert(root)

    def replace(word):
        node = trie.root
        for i, char in enumerate(word):
            if char not in node.children:
                break
            node = node.children[char]
            if node.is_end:
                return word[:i+1]  # return the root prefix
        return word  # no root found, keep original

    return ' '.join(replace(word) for word in sentence.split())
```

### Explanation

- Insert all roots into the trie.
- For each word in the sentence, traverse the trie — the first `is_end` encountered is the shortest matching root.
- If no root is found, return the word unchanged.
- **Time:** O(total chars) | **Space:** O(total chars in dictionary)

---

## Key Takeaways

| Operation | Trie | Hash Set |
|---|---|---|
| Exact word search | O(m) | O(m) |
| Prefix search | O(m) | O(n·m) — must check all |
| Shared prefixes | Yes — space efficient | No |
| Wildcard search | Possible with DFS | Very hard |

**The main reason to use a Trie over a set:** prefix operations. The moment a problem involves "find all words starting with...", "autocomplete", or pruning a word search by prefix — reach for a Trie.
