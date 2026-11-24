---
title: Discrete Mathematics
tags: [phase-2, math]
aliases: [Discrete Math, Sets, Combinatorics]
---

# Discrete Mathematics

[[Phase 2 - Start Here|Home]] · **Next:** [[16 Complexity Analysis]] → · [[Software Engineering Roadmap#Discrete Mathematics|Roadmap card]]

> [!abstract] After this note, you can
> Work with sets, count possibilities, use remainder math, and estimate the chance of a collision in a real system.

## The Big Idea

**Discrete** means separate and countable, like whole people or whole files. You can have 3 files, but not 3.7 files. Computers work with separate things, so this is the math of computing.

Don't worry. It's mostly careful counting, with very little algebra.

## Sets

A **set** is a group of different things. Order doesn't matter, and nothing appears twice.

![[sets-venn.svg]]

| Picture | Name | Plain words | Python |
|---|---|---|---|
| Both circles filled | **Union** | In A or B | `a \| b` |
| Middle filled | **Intersection** | In both | `a & b` |
| Left part filled | **Difference** | In A, not in B | `a - b` |

> [!example] Real use: who should get this email?
> Group A is people who bought a course. Group B is people who finished it.
> - **A − B** is who bought it but hasn't finished. Send them a reminder.
> - **A ∩ B** is who finished it. Send them a certificate.

```python
bought = {"Sam", "Priya", "Ali", "Mei"}
finished = {"Priya", "Mei", "Tom"}

print(sorted(bought & finished))
print(sorted(bought - finished))
print(sorted(bought | finished))
```

Output:

```text
['Mei', 'Priya']
['Ali', 'Sam']
['Ali', 'Mei', 'Priya', 'Sam', 'Tom']
```

You'll see the same pictures again with [[30 SQL#Joins in One Picture|SQL joins]].

## Relations and Functions

A **relation** is any connection between things, such as "Sam is friends with Priya". A **function** is a special relation: **each input has exactly one output**.

![[function-vs-relation.svg]]

This is why a Python function, or a dictionary key, gives you one answer. It's also why a database can't store two different values in one cell.

## Counting

Counting tells you how many cases your code could face, how big a search is, and how safe a password is.

### The Multiply Rule

If you make several choices one after another, **multiply** the options.

> [!example] How many outfits?
> 3 shirts and 4 pairs of trousers give 3 × 4 = **12** outfits.

> [!example] How strong is a PIN?
> A 4-digit PIN has 10 options in each place: 10 × 10 × 10 × 10 = **10,000** PINs.
>
> An 8-letter lowercase password has 26⁸, or about **209 billion**.

### Permutations - Order Matters

A **permutation** is an arrangement. For 3 items, there are 3 choices for first place, 2 left for second, then 1.

![[permutation-tree.svg]]

```python
import itertools
import math

print(math.factorial(3))
for arrangement in itertools.permutations("ABC"):
    print("".join(arrangement), end=" ")
```

Output:

```text
6
ABC ACB BAC BCA CAB CBA 
```

Written as `n!` ("n factorial"), this grows extremely fast. 20! is over 2 quintillion. That's why trying every order fails for big problems.

### Combinations - Order Doesn't Matter

A **combination** is a choice of a group. Choosing Sam and Ali is the same as choosing Ali and Sam.

```python
import itertools
import math

print(math.comb(4, 2))
print(list(itertools.combinations("ABCD", 2)))
```

Output:

```text
6
[('A', 'B'), ('A', 'C'), ('A', 'D'), ('B', 'C'), ('B', 'D'), ('C', 'D')]
```

| Question to ask | Use | Example |
|---|---|---|
| Does the order matter? **Yes** | Permutation | Race positions, passwords |
| Does the order matter? **No** | Combination | Picking a team |

### The Pigeonhole Principle

If you have more items than boxes, at least one box must hold 2 or more. It sounds obvious, but it's powerful.

![[pigeonhole.svg]]

> [!example] Why collisions can't be avoided
> A hash function squeezes unlimited inputs into a fixed number of outputs. There are more possible inputs than outputs, so two inputs must share an output. That's a **collision**. See [[19 Hash Tables]].

## Clock Math - Modular Arithmetic

**Modular arithmetic** is math that wraps around, like a clock. In Python, it's the `%` operator from [[02 Variables, Types and Expressions#Expressions - Calculations|Phase 1]].

![[mod-clock.svg]]

Think of a 12-hour clock with 0 at the top. Go 14 hours forward from 0, and you land on 2. So `14 % 12` is 2.

```python
print(14 % 12)

# Which day of the week is it, 10 days after Monday?
days = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
print(days[(0 + 10) % 7])

# Going around a list in a loop
players = ["Sam", "Priya", "Ali"]
for turn in range(5):
    print(players[turn % 3], end=" ")
```

Output:

```text
2
Thu
Sam Priya Ali Sam Priya 
```

Hash tables, cryptography and round-robin load balancers all rely on this.

## Recurrence Relations

A **recurrence** defines each step using earlier steps. You saw this idea in [[04 Functions and Scope#Recursion|recursion]].

> [!example] A savings account
> Start with 100, and add 10 each month.
> - `S(0) = 100`
> - `S(n) = S(n-1) + 10`

| Month | Worked out | Value |
|---|---|---|
| 0 | start | 100 |
| 1 | 100 + 10 | 110 |
| 2 | 110 + 10 | 120 |
| 3 | 120 + 10 | 130 |

Recurrences describe the cost of recursive algorithms, as you'll see in [[16 Complexity Analysis#The Cost of Recursion]].

## Probability for Discrete Events

**Probability** = (ways you want) ÷ (all possible ways).

- A fair coin landing heads: 1 ÷ 2 = 0.5
- A die showing 6: 1 ÷ 6 ≈ 0.17

**A helpful trick:** "at least one" is often easier as 1 − (none at all).

> [!example] The birthday surprise
> With just 23 people in a room, there's about a 50% chance that two share a birthday. It feels wrong, because you're comparing every **pair**, not just yourself against the others.

```python
def chance_of_shared_value(people, options):
    all_different = 1.0
    for i in range(people):
        all_different = all_different * (options - i) / options
    return 1 - all_different

print(round(chance_of_shared_value(23, 365), 3))
print(round(chance_of_shared_value(50, 365), 3))
```

Output:

```text
0.507
0.97
```

**Why engineers care:** say you create random 6-character IDs using lowercase letters, which gives about 309 million options. Collisions arrive far earlier than you might guess.

```python
def chance_of_shared_value(people, options):
    all_different = 1.0
    for i in range(people):
        all_different = all_different * (options - i) / options
    return 1 - all_different

print(26 ** 6)
print(round(chance_of_shared_value(30000, 26 ** 6), 2))
```

Output:

```text
308915776
0.77
```

With only 30,000 IDs out of 309 million, there's already a 77% chance of a duplicate. This is the roadmap's mastery test for this topic, and you can now do it.

## Common Mistakes

> [!warning] Watch out for these
> - Using a permutation when order doesn't matter, or the reverse
> - Forgetting that sets have no order and no duplicates
> - Trusting your gut about collisions. Do the calculation.
> - Treating `%` as a percentage. In code, it means remainder.

## Practice

**1. Sets.** `A = {1, 2, 3, 4}` and `B = {3, 4, 5}`. What are `A ∪ B`, `A ∩ B` and `B − A`?

> [!example]- Answer to question 1
> - A ∪ B = {1, 2, 3, 4, 5}
> - A ∩ B = {3, 4}
> - B − A = {5}

**2. Count.** A car plate has 2 letters, then 4 digits. How many plates are possible?

> [!example]- Answer to question 2
> 26 × 26 × 10 × 10 × 10 × 10 = **6,760,000**

**3. Choose.** How many ways can you pick a team of 3 from 5 people? Does order matter?

> [!example]- Answer to question 3
> Order doesn't matter, so it's a combination. `math.comb(5, 3)` gives **10**.

**4. Clock.** A list has 8 items. You start at index 6, and move forward 5 steps, wrapping around. Which index do you land on?

> [!example]- Answer to question 4
> `(6 + 5) % 8` is **3**.

**5. Estimate.** Using `chance_of_shared_value`, what's the smallest number of people that gives over a 99% chance of a shared birthday?

> [!example]- Answer to question 5
> ```python
> def chance_of_shared_value(people, options):
>     all_different = 1.0
>     for i in range(people):
>         all_different = all_different * (options - i) / options
>     return 1 - all_different
>
> people = 1
> while chance_of_shared_value(people, 365) <= 0.99:
>     people = people + 1
> print(people)
> ```
>
> Output:
>
> ```text
> 57
> ```

**6. Go deeper.** Try the first lectures of [MIT 6.042J](https://ocw.mit.edu/courses/6-042j-mathematics-for-computer-science-spring-2015/).

## Can You Do This?

- [ ] Draw union, intersection and difference from memory
- [ ] Explain what makes a relation a function
- [ ] Use the multiply rule to count possibilities
- [ ] Choose between a permutation and a combination
- [ ] Explain why hash collisions can't be avoided
- [ ] Use `%` to wrap around a list
- [ ] Calculate the collision chance for a set of random IDs

## Summary

> [!success] Remember
> - **Sets** have no order and no duplicates.
> - **Multiply** the options for choices made in a row.
> - **Permutation:** order matters. **Combination:** it doesn't.
> - **Pigeonhole:** too many items means a shared box.
> - **`%`** wraps numbers around, like a clock.

**Next:** [[16 Complexity Analysis]] →
