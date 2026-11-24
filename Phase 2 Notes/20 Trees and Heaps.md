---
title: Trees and Heaps
tags: [phase-2, data-structures]
aliases: [Trees, Binary Search Tree, BST, Heaps, Priority Queue, Tries]
---

# Trees and Heaps

← [[19 Hash Tables]] · [[Phase 2 - Start Here|Home]] · **Next:** [[21 Graphs]] → · [[Software Engineering Roadmap#Data Structures|Roadmap card]]

> [!abstract] After this note, you can
> Understand tree words, search a binary search tree, walk a tree in several ways, and use a heap to always get the smallest item.

## The Big Idea

Some data naturally branches: folders inside folders, a family tree, the tags in a web page. A **tree** stores this kind of data. Trees are also how databases find a row among billions of rows in just a few steps.

## The Parts of a Tree

![[tree-terms.svg]]

| Word | Meaning |
|---|---|
| **Node** | One item in the tree |
| **Edge** | A line connecting two nodes |
| **Root** | The top node, which has no parent |
| **Parent, child** | A node and the nodes directly below it |
| **Leaf** | A node with no children |
| **Height** | The number of edges on the longest path from root to leaf |
| **Binary tree** | Each node has at most 2 children |

### Build One

```python
class TreeNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

root = TreeNode("A",
    TreeNode("B", TreeNode("D"), TreeNode("E")),
    TreeNode("C", None, TreeNode("F")))

print(root.value, root.left.value, root.right.right.value)
```

Output:

```text
A B F
```

> [!tip] Trees and recursion are best friends
> A tree is a root with two smaller trees below it. So most tree code follows one simple pattern: handle this node, then call yourself on the left and right. The base case is an empty tree.

```python
class TreeNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

def count_nodes(node):
    if node is None:
        return 0
    return 1 + count_nodes(node.left) + count_nodes(node.right)

def height(node):
    if node is None:
        return -1
    return 1 + max(height(node.left), height(node.right))

root = TreeNode("A", TreeNode("B", TreeNode("D")), TreeNode("C"))
print(count_nodes(root))
print(height(root))
```

Output:

```text
4
2
```

## Walking Through a Tree

![[tree-traversal.svg]]

The three orders differ only in **when you visit the node itself**.

| Order | Steps | Useful for |
|---|---|---|
| **Pre-order** | **Node**, left, right | Copying a tree |
| **In-order** | Left, **node**, right | Sorted output from a BST |
| **Post-order** | Left, right, **node** | Deleting a tree, folder sizes |

```python
class TreeNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

def in_order(node):
    if node is None:
        return []
    return in_order(node.left) + [node.value] + in_order(node.right)

def pre_order(node):
    if node is None:
        return []
    return [node.value] + pre_order(node.left) + pre_order(node.right)

def post_order(node):
    if node is None:
        return []
    return post_order(node.left) + post_order(node.right) + [node.value]

root = TreeNode(4,
    TreeNode(2, TreeNode(1), TreeNode(3)),
    TreeNode(6, TreeNode(5), TreeNode(7)))

print(pre_order(root))
print(in_order(root))
print(post_order(root))
```

Output:

```text
[4, 2, 1, 3, 6, 5, 7]
[1, 2, 3, 4, 5, 6, 7]
[1, 3, 2, 5, 7, 6, 4]
```

> [!example] Real use: how big is a folder?
> You must know the size of everything inside before you know the folder's total. That is post-order: children first, then the parent.

## Binary Search Trees

A **binary search tree (BST)** follows one rule at every node:

> **Everything on the left is smaller. Everything on the right is bigger.**

![[bst-search.svg]]

Each comparison throws away a whole side of the tree, just like the phone book. In a well-shaped tree, that's **O(log n)**.

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None

def insert(node, value):
    if node is None:
        return Node(value)
    if value < node.value:
        node.left = insert(node.left, value)
    elif value > node.value:
        node.right = insert(node.right, value)
    return node

def search(node, value):
    steps = 0
    while node is not None:
        steps = steps + 1
        if value == node.value:
            return f"found in {steps} steps"
        if value < node.value:
            node = node.left
        else:
            node = node.right
    return f"not found after {steps} steps"

root = None
for value in [8, 3, 10, 1, 6, 14, 4, 7]:
    root = insert(root, value)

print(search(root, 6))
print(search(root, 5))
```

Output:

```text
found in 3 steps
not found after 4 steps
```

**Watch it run** for `search(root, 6)`:

| Step | At node | Compare | Go |
|---|---|---|---|
| 1 | 8 | 6 < 8 | Left |
| 2 | 3 | 6 > 3 | Right |
| 3 | 6 | 6 = 6 | Found! |

### Balance Matters

![[balanced-vs-unbalanced.svg]]

Insert 0, 1, 2, 3, 4 in sorted order, and you get the right-hand shape. It's really just a linked list, so search becomes **O(n)**.

**Balanced trees** fix this. They rearrange themselves automatically as you insert, keeping the height near log n. You'll hear these names: **AVL tree** and **red-black tree**. You'll almost never write one, but you should know they exist.

### Why Databases Love Trees

Databases use a relative called the **B-tree**. Each node holds many keys and has many children, so the tree is very short and wide. Finding one row among a billion takes only about 4 disk reads. See [[29 Relational Databases#Indexes in One Picture]].

## Heaps

Sometimes you don't need everything sorted. You just keep asking: **what's the smallest item right now?** A **heap** answers this in O(1).

![[heap.svg]]

A **min-heap** follows one rule: **every parent is smaller than or equal to its children.** The smallest is always at the root. It's stored in a plain list, and simple math finds the relatives.

> [!note] A heap isn't a sorted list
> In the picture's list, 7 comes before 4, even though it's bigger. That's fine! The rule compares only parents with their own children.

### Adding - Bubble Up

Add the new item at the end, then swap it upward while it's smaller than its parent.

```text
Add 0 to [1, 3, 2]:

        1               1               0
       / \             / \             / \
      3   2    →      0   2    →      1   2
     /               /               /
    0               3               3
  (added)        (0 < 3: swap)    (0 < 1: swap)
```

### Removing the Smallest - Sink Down

Move the last item to the root, then swap it downward with its smaller child.

Both actions follow one path through the tree, so they're **O(log n)**.

### Using a Heap in Python

Python's `heapq` module turns a plain list into a min-heap.

```python
import heapq

numbers = []
for value in [5, 1, 8, 3]:
    heapq.heappush(numbers, value)

print(numbers[0])
print(heapq.heappop(numbers))
print(heapq.heappop(numbers))
print(numbers[0])
```

Output:

```text
1
1
3
5
```

### Build One Yourself

```python
class MinHeap:
    def __init__(self):
        self.items = []

    def push(self, value):
        self.items.append(value)
        i = len(self.items) - 1
        while i > 0:
            parent = (i - 1) // 2
            if self.items[i] >= self.items[parent]:
                break
            self.items[i], self.items[parent] = self.items[parent], self.items[i]
            i = parent

    def pop(self):
        top = self.items[0]
        last = self.items.pop()
        if self.items:
            self.items[0] = last
            i = 0
            while True:
                left, right = 2 * i + 1, 2 * i + 2
                smallest = i
                if left < len(self.items) and self.items[left] < self.items[smallest]:
                    smallest = left
                if right < len(self.items) and self.items[right] < self.items[smallest]:
                    smallest = right
                if smallest == i:
                    break
                self.items[i], self.items[smallest] = self.items[smallest], self.items[i]
                i = smallest
        return top

heap = MinHeap()
for value in [5, 1, 8, 3, 2]:
    heap.push(value)

print([heap.pop() for _ in range(5)])
```

Output:

```text
[1, 2, 3, 5, 8]
```

### Priority Queues

A **priority queue** serves the most important item first, not the oldest. Think of a hospital emergency room. A heap is the usual way to build one.

```python
import heapq

patients = []
heapq.heappush(patients, (3, "sore throat"))
heapq.heappush(patients, (1, "heart attack"))
heapq.heappush(patients, (2, "broken arm"))

while patients:
    priority, problem = heapq.heappop(patients)
    print(priority, problem)
```

Output:

```text
1 heart attack
2 broken arm
3 sore throat
```

> [!example] Worked example: the top 3 scores from a million
> Sorting everything would be wasteful. `nlargest` uses a small heap instead.

```python
import heapq

scores = [50, 90, 20, 70, 85, 10]
print(heapq.nlargest(3, scores))
```

Output:

```text
[90, 85, 70]
```

## Tries

![[trie.svg]]

A **trie** (say "try") is a tree for words. Words with the same start share the same path. It's ideal for **autocomplete**: type "ca", and you can quickly list all the words below that point.

```python
class Trie:
    def __init__(self):
        self.root = {}

    def add(self, word):
        node = self.root
        for letter in word:
            node = node.setdefault(letter, {})
        node["end"] = True

    def starts_with(self, prefix):
        node = self.root
        for letter in prefix:
            if letter not in node:
                return []
            node = node[letter]
        return self._collect(node, prefix)

    def _collect(self, node, word):
        words = [word] if "end" in node else []
        for letter, child in node.items():
            if letter != "end":
                words = words + self._collect(child, word + letter)
        return words

trie = Trie()
for word in ["cat", "car", "dog"]:
    trie.add(word)

print(trie.starts_with("ca"))
print(trie.starts_with("x"))
```

Output:

```text
['cat', 'car']
[]
```

## Which Tree Do I Need?

| You need | Use | Key cost |
|---|---|---|
| Sorted data and range searches | Balanced BST or B-tree | O(log n) search |
| Always the smallest or largest | Heap | O(1) peek, O(log n) pop |
| Search by word start | Trie | O(length of word) |
| Just find by exact key | Not a tree: use a [[19 Hash Tables\|hash table]] | O(1) |

## Common Mistakes

> [!warning] Watch out for these
> - Forgetting the base case, `if node is None`
> - Thinking a heap is fully sorted. Only the top is guaranteed.
> - Assuming a BST is always O(log n). It can become O(n) if unbalanced.
> - Mixing up height and number of nodes

## Practice

**1. Draw it.** Insert 5, 3, 8, 1, 4 into an empty BST. Draw it, then write its in-order walk.

> [!example]- Answer to question 1
> ```text
>       5
>      / \
>     3   8
>    / \
>   1   4
> ```
> In-order gives 1, 3, 4, 5, 8. It's sorted!

**2. Sum the tree.** Write a function that adds all the values in a tree.

> [!example]- Answer to question 2
> ```python
> class Node:
>     def __init__(self, value, left=None, right=None):
>         self.value = value
>         self.left = left
>         self.right = right
>
> def total(node):
>     if node is None:
>         return 0
>     return node.value + total(node.left) + total(node.right)
>
> print(total(Node(1, Node(2), Node(3))))
> ```
>
> Output:
>
> ```text
> 6
> ```

**3. Is it a BST?** Why is this not a BST, even though each parent is between its own children?

```text
      5
     / \
    3   8
     \
      6
```

> [!example]- Answer to question 3
> 6 is in the left part of 5, but it's bigger than 5. The rule applies to the **whole** side, not just the direct child.

**4. Kth smallest.** Use a heap to find the 3rd smallest in `[7, 2, 9, 4, 1]`.

> [!example]- Answer to question 4
> ```python
> import heapq
>
> numbers = [7, 2, 9, 4, 1]
> heapq.heapify(numbers)
> heapq.heappop(numbers)
> heapq.heappop(numbers)
> print(heapq.heappop(numbers))
> ```
>
> Output:
>
> ```text
> 4
> ```

## Can You Do This?

- [ ] Name the parts of a tree on a drawing
- [ ] Write pre-, in- and post-order walks
- [ ] Insert into and search a BST
- [ ] Explain why balance matters
- [ ] Explain the heap rule, and draw bubble-up
- [ ] Use `heapq` for a priority queue
- [ ] Build a small trie

## Summary

> [!success] Remember
> - **Tree:** data that branches. Think in recursion.
> - **BST:** left is smaller, right is bigger. O(log n) if balanced.
> - **Heap:** smallest on top, stored in a list.
> - **Trie:** shared paths for shared word starts.

**Next:** [[21 Graphs]] →
