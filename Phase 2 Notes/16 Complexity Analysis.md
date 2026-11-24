---
title: Complexity Analysis
tags: [phase-2, algorithms]
aliases: [Big O, Time Complexity, Space Complexity]
---

# Complexity Analysis

← [[15 Discrete Mathematics]] · [[Phase 2 - Start Here|Home]] · **Next:** [[17 Arrays and Linked Lists]] → · [[Software Engineering Roadmap#Complexity Analysis|Roadmap card]]

> [!abstract] After this note, you can
> Look at code and say how its work grows as the data grows. This helps you spot slow code before your users do.

## The Big Idea

Imagine two ways to find a name in a phone book with 1,000,000 names:

- **Way A:** read every name from the first page. Up to 1,000,000 checks.
- **Way B:** open the middle, and keep halving. About 20 checks.

Both work on 10 names. Only one works on a million. **Complexity analysis** describes how the work grows as the input grows.

We don't measure seconds, because seconds change from computer to computer. We count **steps**, and we write the answer with **Big-O** notation. Here, `n` means the size of the input.

## The One Picture to Remember

![[big-o-growth.svg]]

| Big-O | Name | Example | Steps for n = 1,000 |
|---|---|---|---|
| O(1) | Constant | Get a list item by index | 1 |
| O(log n) | Logarithmic | Binary search | about 10 |
| O(n) | Linear | Check every item once | 1,000 |
| O(n log n) | Log-linear | Good sorting | about 10,000 |
| O(n²) | Quadratic | Compare every pair | 1,000,000 |
| O(2ⁿ) | Exponential | Try every subset | More than the atoms in the universe |

> [!important] What to take from this
> For small inputs, almost anything works. As `n` grows, the shape of the curve decides everything. A faster computer can't save an O(2ⁿ) algorithm.

## How to Find the Big-O

### Three Simple Rules

1. **Count the loops.** One loop over `n` items is O(n). A loop inside a loop is O(n²).
2. **Keep only the biggest part.** O(n² + n) becomes O(n²), because n² takes over as n grows.
3. **Drop constant numbers.** O(2n) becomes O(n). We care about the shape, not the exact count.

### O(1) - Constant

The work stays the same, whatever the size.

```python
def first_item(items):
    return items[0]

print(first_item([5, 8, 2]))
```

Output:

```text
5
```

### O(n) - Linear

Double the input, and you double the work.

```python
def contains(items, target):
    for item in items:
        if item == target:
            return True
    return False

print(contains([5, 8, 2], 2))
```

Output:

```text
True
```

### O(n²) - Quadratic

![[n-squared-grid.svg]]

A loop inside a loop is a grid of steps. Double the input, and you **quadruple** the work.

```python
def has_duplicate(items):
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[i] == items[j]:
                return True
    return False

print(has_duplicate([1, 2, 3, 2]))
```

Output:

```text
True
```

(This inner loop starts at `i + 1`, so it does about n²/2 steps. We drop the ½, so it's still O(n²).)

### O(log n) - Logarithmic

![[log-halving.svg]]

Each step cuts the problem in half. **log n** answers this question: how many times can I halve `n` before reaching 1?

| n | Halvings (log₂ n) |
|---|---|
| 16 | 4 |
| 1,024 | 10 |
| 1,000,000 | about 20 |
| 1,000,000,000 | about 30 |

Multiplying the input by a thousand adds only 10 more steps. That's why O(log n) feels like magic.

```python
def count_halvings(n):
    steps = 0
    while n > 1:
        n = n // 2
        steps = steps + 1
    return steps

print(count_halvings(16))
print(count_halvings(1000000))
```

Output:

```text
4
19
```

### O(n log n) - Log-Linear

Do O(log n) work for each of n items. Good sorting algorithms take this time. See [[22 Searching and Sorting]]. It's only a little worse than O(n).

### Spot the Big-O

> [!example]- Try it: what's the Big-O of each one? Then open this.
> ```python
> def a(items):
>     total = 0
>     for item in items:
>         total = total + item
>     for item in items:
>         print(item, end=" ")
>     print()
>     return total
>
> print(a([1, 2]))
> ```
>
> Output:
>
> ```text
> 1 2 
> 3
> ```
>
> **O(n).** Two loops **one after the other** add up: n + n = 2n, which is O(n). They only multiply when one is **inside** the other.

## Hidden Costs

A single line can hide a loop. Know what your tools really cost.

| Python code | Cost | Why |
|---|---|---|
| `items[5]` | O(1) | Direct jump |
| `items.append(x)` | O(1) on average | Adds at the end |
| `items.insert(0, x)` | O(n) | Shifts everything |
| `items.pop(0)` | O(n) | Shifts everything |
| `x in my_list` | O(n) | Checks each item |
| `x in my_set` | O(1) on average | Uses a hash |
| `my_dict[key]` | O(1) on average | Uses a hash |
| `sorted(items)` | O(n log n) | Good sort |
| `items[:k]` | O(k) | Copies k items |

> [!warning] A very common hidden O(n²)
> ```python
> seen = []
> for n in [1, 2, 3]:
>     if n not in seen:     # This line is a hidden loop!
>         seen.append(n)
> print(seen)
> ```
>
> Output:
>
> ```text
> [1, 2, 3]
> ```
>
> Change `seen` to a set, and the whole thing becomes O(n).

## Best, Average and Worst Case

The same code can do different amounts of work, depending on the data.

Searching a list of 1,000 names for one name:

| Case | When | Steps |
|---|---|---|
| **Best** | It's the first name | 1 |
| **Average** | It's somewhere in the middle | about 500 |
| **Worst** | It's last, or it's missing | 1,000 |

We usually talk about the **worst case**, since that's what can hurt you.

### Big-O, Big-Omega and Big-Theta

| Symbol | Meaning | Think of it as |
|---|---|---|
| **O** (Big-O) | Grows no faster than this | Upper limit |
| **Ω** (Big-Omega) | Grows at least this fast | Lower limit |
| **Θ** (Big-Theta) | Grows exactly like this | Exact fit |

In everyday work, people say "Big-O" and usually mean the tightest fit.

## Space Complexity

**Space complexity** asks how much extra memory your code needs as the input grows.

```python
def reverse_copy(items):      # O(n) space: builds a new list
    return items[::-1]

def total(items):             # O(1) space: only one number
    answer = 0
    for item in items:
        answer = answer + item
    return answer

print(reverse_copy([1, 2]), total([1, 2]))
```

Output:

```text
[2, 1] 3
```

> [!note] Recursion uses memory too
> Each waiting function call takes up space on the [[18 Stacks, Queues and Deques#The Call Stack|call stack]]. So a recursion that goes `n` levels deep uses O(n) space.

### Trading Space for Time

![[space-time-tradeoff.svg]]

This is the most common trade in programming. You keep some extra data so that you don't have to work it out again.

```python
def has_duplicate(items):
    seen = set()
    for item in items:
        if item in seen:
            return True
        seen.add(item)
    return False

print(has_duplicate([1, 2, 3, 2]))
```

Output:

```text
True
```

## See It for Yourself

The roadmap's exercise is to measure this. Run the code, and compare the numbers.

```python
import time

def slow_duplicate(items):
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[i] == items[j]:
                return True
    return False

def fast_duplicate(items):
    seen = set()
    for item in items:
        if item in seen:
            return True
        seen.add(item)
    return False

for n in [1000, 2000, 4000]:
    items = list(range(n))
    start = time.perf_counter()
    slow_duplicate(items)
    slow = time.perf_counter() - start
    start = time.perf_counter()
    fast_duplicate(items)
    fast = time.perf_counter() - start
    print(f"n={n}  slow={slow:.4f}s  fast={fast:.4f}s")
```

Example output (your times will differ):

```text
n=1000  slow=0.0150s  fast=0.0001s
n=2000  slow=0.0600s  fast=0.0002s
n=4000  slow=0.2400s  fast=0.0004s
```

Each time `n` doubles, the slow one takes about **4 times** as long, while the fast one takes about **2 times** as long. That's O(n²) and O(n), right there in your terminal.

### Will It Be Fast Enough?

Here's a rough guide. Python does about 10 million simple steps each second.

| Input size | Largest Big-O that's usually OK |
|---|---|
| Up to 20 | O(2ⁿ) |
| Up to 5,000 | O(n²) |
| Up to 1,000,000 | O(n log n) |
| Over 10,000,000 | O(n) or O(log n) |

## Going Deeper

### Amortized Cost

Python's `append` is usually instant, but sometimes the list must move to a bigger space, which is O(n). That big cost is rare, so spread across all the appends, it averages out to O(1). This is called **amortized O(1)**. It's like paying yearly for a subscription: one big payment, but cheap per day. See it in [[17 Arrays and Linked Lists#Dynamic Arrays]].

### The Cost of Recursion

To find the cost of a recursive function, count how many calls happen, and how much work each one does.

| Algorithm | Recurrence | Picture | Big-O |
|---|---|---|---|
| Countdown | T(n) = T(n−1) + 1 | One long line of calls | O(n) |
| Binary search | T(n) = T(n/2) + 1 | One path, halving | O(log n) |
| Merge sort | T(n) = 2T(n/2) + n | Tree: log n levels, n work per level | O(n log n) |
| Naive Fibonacci | T(n) = T(n−1) + T(n−2) + 1 | Exploding tree | O(2ⁿ) |

The **Master Theorem** is a formula for solving recurrences like these. You don't need to memorize it now, but you should know it exists.

### When Big-O Isn't the Whole Story

- **Constants matter for small inputs.** An O(n²) sort can beat an O(n log n) sort on 10 items.
- **Memory layout matters.** Arrays are much faster than linked lists in practice, even for the same Big-O, because of [[Software Engineering Roadmap#Memory Hierarchy|CPU caches]].
- **Some problems have no known fast solution.** These are called **NP-hard**, such as finding the shortest route that visits every city. You'll study them later.

## Common Mistakes

> [!warning] Watch out for these
> - Calling two loops in a row O(n²). They add, so they stay O(n).
> - Forgetting hidden loops such as `in` on a list
> - Optimizing before measuring
> - Saying "O(n) is always faster than O(n²)". For tiny inputs, it may not be.

## Practice

For each one, give the time complexity.

**1.**
```python
def f(n):
    count = 0
    for i in range(n):
        for j in range(10):
            count = count + 1
    return count

print(f(3))
```

> [!example]- Answer to question 1
> **O(n).** The inner loop always runs 10 times, whatever `n` is, so it's 10n steps.

**2.**
```python
def g(n):
    count = 0
    i = 1
    while i < n:
        i = i * 2
        count = count + 1
    return count

print(g(8))
```

> [!example]- Answer to question 2
> **O(log n).** `i` doubles each round, which is the mirror of halving.

**3.** A function sorts a list, then loops over it once. What's the total?

> [!example]- Answer to question 3
> O(n log n) + O(n). Keep the biggest: **O(n log n)**.

**4.** How would you make this O(n)?

```python
def common(a, b):
    result = []
    for x in a:
        if x in b:
            result.append(x)
    return result

print(common([1, 2], [2, 3]))
```

> [!example]- Answer to question 4
> `x in b` is a hidden loop, so it's O(n × m). Turn `b` into a set first:
> ```python
> def common(a, b):
>     b_set = set(b)
>     result = []
>     for x in a:
>         if x in b_set:
>             result.append(x)
>     return result
>
> print(common([1, 2], [2, 3]))
> ```
>
> Output:
>
> ```text
> [2]
> ```

## Can You Do This?

- [ ] Draw the growth chart from memory
- [ ] Give the Big-O of simple code just by reading it
- [ ] Say the cost of common Python operations
- [ ] Explain best, average and worst case
- [ ] Explain space complexity, and trading space for time
- [ ] Benchmark two solutions and match them to theory
- [ ] Explain what amortized O(1) means

## Summary

> [!success] Remember
> - Big-O describes **how work grows**, not exact speed.
> - **Nested loops multiply. Loops in a row add.**
> - Drop constants, and keep the biggest part.
> - Halving gives O(log n), which is wonderful.
> - Watch out for hidden loops.

**Next:** [[17 Arrays and Linked Lists]] →
