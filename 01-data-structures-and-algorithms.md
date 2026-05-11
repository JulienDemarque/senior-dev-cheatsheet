# Data structures & algorithms

Senior-level reference: what each structure is for, complexity intuition, and tiny usage examples (Python 3 unless noted).

## HashMap

A **hash function** maps keys to bucket indices in an array; each bucket holds entries (or a chain/tree for collisions). Average lookup/insert/delete is **O(1)**; worst case **O(n)** if every key collides.

Java calls it `HashMap`; Python’s `dict` is insertion-ordered as of 3.7+ but still hash-based; JavaScript `Map` preserves insertion order and allows non-string keys.

### Example (Python)

```python
scores = {"ada": 100, "alan": 95}
scores["ada"] += 1
scores.get("grace", 0)  # default if missing — avoids KeyError
```

### Pitfall

Mutable objects must not be dict keys (unhashable). For composite keys, use a tuple of immutables or a stable serialized string.

## Dictionary

Same family as **HashMap**: associative array from keys to values. In Python, `dict` is the built-in; operations mirror **HashMap** complexity.

### Example (Python): grouping with `defaultdict`

```python
from collections import defaultdict
by_lang = defaultdict(list)
by_lang["py"].append("fastapi")
by_lang["py"].append("django")
```

## Set

Unordered collection of unique hashable elements: **O(1)** average membership. Supports union (`|`), intersection (`&`), difference (`-`).

Tree-based sets (e.g. Java `TreeSet`) keep sorted order at **O(log n)** per op.

### Example (Python)

```python
a = {1, 2, 3}
b = {2, 3, 4}
a & b   # {2, 3} intersection
a | b   # {1, 2, 3, 4} union
```

## Hash function

Maps arbitrary input to a fixed-size output (index or digest). For data structures you want **avalanche** (small input change → very different hash) and few collisions. Cryptographic hashes (SHA-256) add preimage/collision resistance for security—not needed for in-memory `dict` buckets.

### Example (Python)

```python
hash("user:42") % 256  # conceptual bucket index (real dicts use C internals)
```

## Binary tree

Each node has ≤2 children (`left`, `right`). Used for BST search (ordered keys), heaps, expression trees. Unbalanced BST degrades to linked-list height → **O(n)**; AVL/red-black trees rebalance to **O(log n)**.

### Example (conceptual node)

```python
class Node:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

## DFS (depth-first search)

Explore as deep as possible before backtracking. Stack-based or recursive; **O(V + E)** on graphs with adjacency lists.

### Example (adjacency list, recursive)

```python
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    for nbr in graph.get(start, []):
        if nbr not in visited:
            dfs(graph, nbr, visited)
    return visited
```

## BFS (breadth-first search)

Layer-by-layer using a **queue**; shortest path on **unweighted** graphs.

### Example

```python
from collections import deque

def bfs(graph, start, goal):
    q = deque([(start, [start])])
    seen = {start}
    while q:
        node, path = q.popleft()
        if node == goal:
            return path
        for nbr in graph.get(node, []):
            if nbr not in seen:
                seen.add(nbr)
                q.append((nbr, path + [nbr]))
    return None
```

## Recursion

Function calls itself with a smaller input until a **base case**. Natural for trees and **DFS**; each call consumes stack space—Python default recursion limit matters for deep chains.

### Example (factorial — illustrative only)

```python
def fact(n):
    if n <= 1:
        return 1
    return n * fact(n - 1)
```

Prefer loops or explicit **stack** for very deep structures to avoid stack overflow.

## Queue (data structure)

FIFO: enqueue at tail, dequeue from head. **BFS**, schedulers, buffering between stages.

### Example (Python `deque`)

```python
from collections import deque
q = deque()
q.append("job1")
q.append("job2")
q.popleft()  # "job1"
```

## Stack

LIFO: `push` / `pop` from the same end. Call stacks, **DFS**, undo, matching brackets.

### Example (balanced brackets)

```python
def brackets_ok(s):
    stack = []
    pairs = {")": "(", "]": "[", "}": "{"}
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
    return not stack
```

## Heap

Complete binary tree with heap property (min-heap: parent ≤ children). `heapq` in Python is a **min-heap** on a list; push/pop **O(log n)**, peek min **O(1)**.

### Example (Python `heapq`)

```python
import heapq
h = [3, 1, 4]
heapq.heapify(h)
heapq.heappush(h, 2)
heapq.heappop(h)  # 1 — smallest
```

### See also

[Priority queue](./09-messaging-and-background-jobs.md#priority-queue) (messaging sense) vs heap as the classic implementation of a priority queue ADT.
