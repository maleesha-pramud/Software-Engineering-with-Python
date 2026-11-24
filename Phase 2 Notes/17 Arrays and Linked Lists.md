---
title: Arrays and Linked Lists
tags: [phase-2, data-structures]
aliases: [Arrays, Linked Lists, Dynamic Arrays]
---

# Arrays and Linked Lists

← [[16 Complexity Analysis]] · [[Phase 2 - Start Here|Home]] · **Next:** [[18 Stacks, Queues and Deques]] → · [[Software Engineering Roadmap#Data Structures|Roadmap card]]

> [!abstract] After this note, you can
> Explain how arrays and linked lists sit in memory, predict their speed, and build a linked list yourself.

## The Big Idea

A **data structure** is a way of organizing data so that certain jobs become easy. Think of a kitchen. Spices in a labeled rack are easy to find. Thrown in a big bag, they aren't. Same spices, but a different structure.

There are two basic ways to keep items in order:

| | Array | Linked list |
|---|---|---|
| **Like** | Seats in a cinema row, numbered | A treasure hunt, where each clue points to the next |
| **Memory** | Side by side | Anywhere |
| **Good at** | Jumping to a position | Adding and removing in place |

## Arrays

![[array-memory.svg]]

An **array** stores its items next to each other in memory. To find an item, the computer uses simple math:

`address = start + index × item size`

No searching is needed. That's why `items[1000000]` is just as fast as `items[0]`. This is **O(1)**.

### Inserting Is the Weak Spot

![[array-insert.svg]]

There are no gaps in an array. To insert at the front, every other item must first move over to make room. That's **O(n)**.

```python
items = ["B", "C", "D"]
items.insert(0, "A")
print(items)

items.pop(0)
print(items)

items.append("E")
print(items)
```

Output:

```text
['A', 'B', 'C', 'D']
['B', 'C', 'D']
['B', 'C', 'D', 'E']
```

Here, the first two actions shift everything, while `append` is cheap.

### Dynamic Arrays

A basic array has a fixed size. A **dynamic array** grows when it's full. Python's `list` is one. Here's how it grows:

```text
Capacity 4:  [A][B][C][D]         Full! Next add needs more room.

Step 1. Get a bigger space:   [ ][ ][ ][ ][ ][ ][ ][ ]
Step 2. Copy everything over: [A][B][C][D][ ][ ][ ][ ]
Step 3. Add the new item:     [A][B][C][D][E][ ][ ][ ]
```

Copying is O(n), but it happens rarely, because the space grows by a proportion each time. Averaged over all appends, each one is **amortized O(1)**. See [[16 Complexity Analysis#Amortized Cost]].

**Watch it grow.** `getsizeof` shows the list's memory in bytes. Notice that it jumps only now and then:

```python
import sys

items = []
last_size = sys.getsizeof(items)
growths = 0
for i in range(1000):
    items.append(i)
    size = sys.getsizeof(items)
    if size != last_size:
        growths = growths + 1
        last_size = size

print(f"1000 appends, but memory grew only a few times: {growths < 50}")
```

Output:

```text
1000 appends, but memory grew only a few times: True
```

## Linked Lists

![[linked-list.svg]]

A **linked list** is a chain of **nodes**. Each node holds a value and a **pointer**, which is the location of the next node. The nodes can be anywhere in memory. The start of the chain is called the **head**.

### Build One

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

head = Node(10)
head.next = Node(20)
head.next.next = Node(30)

current = head
while current is not None:
    print(current.value, end=" → ")
    current = current.next
print("None")
```

Output:

```text
10 → 20 → 30 → None
```

(`class` makes your own type of object. You'll study it properly in [[26 Object-Oriented Programming]]. For now, think of a `Node` as a small box with two compartments.)

### Watch the Walk

There's no math to jump to a position. You must follow the arrows, one at a time.

| Step | `current` is on | Printed so far |
|---|---|---|
| 1 | 10 | 10 |
| 2 | 20 | 10 → 20 |
| 3 | 30 | 10 → 20 → 30 |
| 4 | None | Loop stops |

So reaching position `k` takes `k` steps: **O(n)**.

### Inserting Is the Strong Spot

![[linked-insert.svg]]

Nothing moves! You only change two arrows, which is **O(1)**. The order matters: connect the new node **first**, or you'll lose the rest of the chain.

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

a = Node("A")
c = Node("C")
a.next = c

b = Node("B")
b.next = a.next     # 1. New node points to C
a.next = b          # 2. A now points to the new node

current = a
while current:
    print(current.value, end=" ")
    current = current.next
```

Output:

```text
A B C 
```

### A Complete Linked List

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.next = None

class LinkedList:
    def __init__(self):
        self.head = None

    def add_front(self, value):
        node = Node(value)
        node.next = self.head
        self.head = node

    def remove(self, value):
        previous = None
        current = self.head
        while current is not None:
            if current.value == value:
                if previous is None:
                    self.head = current.next
                else:
                    previous.next = current.next
                return True
            previous = current
            current = current.next
        return False

    def to_list(self):
        result = []
        current = self.head
        while current is not None:
            result.append(current.value)
            current = current.next
        return result

items = LinkedList()
items.add_front(3)
items.add_front(2)
items.add_front(1)
print(items.to_list())

items.remove(2)
print(items.to_list())
```

Output:

```text
[1, 2, 3]
[1, 3]
```

> [!example] A classic trick: reverse a linked list
> Walk along the chain, and flip each arrow backward. You need three names, to avoid losing your place.

```python
class Node:
    def __init__(self, value, next=None):
        self.value = value
        self.next = next

def reverse(head):
    previous = None
    current = head
    while current is not None:
        next_node = current.next   # Save the rest of the chain
        current.next = previous    # Flip the arrow
        previous = current         # Move both names forward
        current = next_node
    return previous

head = Node(1, Node(2, Node(3)))
new_head = reverse(head)
print(new_head.value, new_head.next.value, new_head.next.next.value)
```

Output:

```text
3 2 1
```

**Watch it run:**

```text
Start:   None    1 → 2 → 3
Step 1:  None ← 1    2 → 3
Step 2:  None ← 1 ← 2    3
Step 3:  None ← 1 ← 2 ← 3
```

### Other Kinds of Linked List

```text
Singly linked:   A → B → C → None
Doubly linked:   None ← A ⇄ B ⇄ C → None      (can walk both ways)
Circular:        A → B → C → back to A        (a loop)
```

## Side by Side

| Action | Array | Linked list |
|---|---|---|
| Get item at position | **O(1)** | O(n) |
| Search for a value | O(n) | O(n) |
| Add at the end | **O(1)** amortized | O(n), or O(1) if you keep a tail pointer |
| Add at the front | O(n) | **O(1)** |
| Add in the middle, once you're there | O(n) | **O(1)** |
| Extra memory per item | Low | Higher, for the pointer |

> [!tip] Which should you choose?
> **Use an array (Python `list`) almost every time.** Modern CPUs read side-by-side memory extremely fast. Linked lists are mostly useful as a building block, such as inside [[18 Stacks, Queues and Deques|queues]] and the LRU cache in [[19 Hash Tables#Build an LRU Cache]]. They also appear often in interviews, and they teach you to think in pointers.

## Common Mistakes

> [!warning] Watch out for these
> - Losing the rest of the chain by changing `next` too early
> - Not handling an empty list, where `head` is `None`
> - Removing the first node without updating `head`
> - Using `insert(0, x)` in a loop. It's O(n²) overall. Use `deque` from [[18 Stacks, Queues and Deques#Queue Code]].

## Practice

**1. Draw it.** Draw what memory looks like for the array `[7, 8, 9]`, and for a linked list with the same values.

**2. Length.** Write `length(head)`, which counts a linked list's nodes.

> [!example]- Answer to question 2
> ```python
> class Node:
>     def __init__(self, value, next=None):
>         self.value = value
>         self.next = next
>
> def length(head):
>     count = 0
>     current = head
>     while current is not None:
>         count = count + 1
>         current = current.next
>     return count
>
> print(length(Node(1, Node(2))))
> print(length(None))
> ```
>
> Output:
>
> ```text
> 2
> 0
> ```

**3. Find the middle.** Find the middle node in one pass. **Hint:** use a slow pointer that moves 1 step and a fast pointer that moves 2 steps.

> [!example]- Answer to question 3
> When the fast pointer reaches the end, the slow one is halfway.
> ```python
> class Node:
>     def __init__(self, value, next=None):
>         self.value = value
>         self.next = next
>
> def middle(head):
>     slow = head
>     fast = head
>     while fast is not None and fast.next is not None:
>         slow = slow.next
>         fast = fast.next.next
>     return slow.value
>
> print(middle(Node(1, Node(2, Node(3, Node(4, Node(5)))))))
> ```
>
> Output:
>
> ```text
> 3
> ```

**4. Think.** You're building a music app's "next song" list. You mostly add songs at the end, and jump to song numbers. Array or linked list?

> [!example]- Answer to question 4
> **Array.** Adding at the end and jumping to a position are both O(1). A linked list would be slow at jumping.

## Can You Do This?

- [ ] Draw an array and a linked list in memory
- [ ] Explain why array lookup is O(1)
- [ ] Explain how a dynamic array grows
- [ ] Build a linked list with add and remove
- [ ] Reverse a linked list, and explain each step
- [ ] Choose between the two for a given problem

## Summary

> [!success] Remember
> - **Array:** side by side. Jumping is fast. Shifting is slow.
> - **Linked list:** chain of arrows. Inserting is fast. Walking is slow.
> - In real code, use an array unless you have a clear reason not to.

**Next:** [[18 Stacks, Queues and Deques]] →
