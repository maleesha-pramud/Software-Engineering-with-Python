---
title: Mathematical Logic
tags: [phase-1, math]
aliases: [Logic, Proofs, Truth Tables]
---

# Mathematical Logic

← [[11 Unit Testing Basics]] · [[Phase 1 - Start Here|Home]] · **Next:** [[13 Boolean Algebra]] → · [[Software Engineering Roadmap#Mathematical Logic|Roadmap card]]

> [!abstract] After this note, you can
> Reason exactly about whether something is true, write correct conditions, and argue that your code works.

## The Big Idea

**Logic** is the science of correct reasoning. Every `if` you write is logic, and so is every database search. Learning it helps you stop guessing about your conditions and start knowing.

Don't worry: this isn't hard math. It's careful thinking with a few simple rules.

## Propositions

A **proposition** is a statement that's either true or false. It can't be both.

| Statement | Proposition? | Why |
|---|---|---|
| "2 + 2 = 4" | Yes | True |
| "Paris is in Japan" | Yes | False |
| "x > 5" | Not yet | It depends on x |
| "Close the door" | No | It's a command |
| "Is it raining?" | No | It's a question |

We often use letters like `p` and `q` for propositions.

- `p` = "it is raining"
- `q` = "I have an umbrella"

## Truth Tables

A **truth table** lists every possible case, so you never have to guess.

### AND, OR, NOT

| p | q | p AND q | p OR q | NOT p |
|---|---|---|---|---|
| T | T | **T** | **T** | F |
| T | F | F | **T** | F |
| F | T | F | **T** | **T** |
| F | F | F | F | **T** |

- **AND** is true only when **both** are true.
- **OR** is true when **at least one** is true.
- **NOT** flips the value.

> [!warning] "Or" in logic is different from everyday English
> If a menu says "soup or salad", it usually means one or the other. In logic, `p OR q` is also true when **both** are true. The "one or the other, but not both" version is called **XOR**. See [[13 Boolean Algebra#XOR - One or the Other, Not Both]].

You can make a truth table with code:

```python
print("p      q      p and q  p or q")
for p in [True, False]:
    for q in [True, False]:
        print(f"{p!s:6} {q!s:6} {(p and q)!s:8} {(p or q)!s:6}")
```

Output:

```text
p      q      p and q  p or q
True   True   True     True  
True   False  False    True  
False  True   False    True  
False  False  False    False 
```

## Implication - If p, Then q

**Implication** is a promise. It's written `p → q`, and it means "if p is true, then q must be true".

> [!example] Think of a promise
> A parent says: **"If you finish your homework, then you get ice cream."**
>
> Has the parent broken the promise?

| Homework done (p) | Ice cream (q) | Promise kept? | Why |
|---|---|---|---|
| Yes | Yes | ✅ Kept | They did as they said |
| Yes | No | ❌ **Broken** | The only way to break it |
| No | Yes | ✅ Kept | The promise said nothing about this case |
| No | No | ✅ Kept | The promise said nothing about this case |

The only way `p → q` is false is when `p` is true and `q` is false. So `p → q` means the same as `(NOT p) OR q`.

This trips up many people. If `p` is false, the promise can't be broken, so the implication counts as true.

### The Contrapositive

Here are three related statements. Take "If it's raining (p), the ground is wet (q)."

| Name | Form | Example | Same meaning as the original? |
|---|---|---|---|
| **Original** | p → q | If it's raining, the ground is wet | – |
| **Contrapositive** | NOT q → NOT p | If the ground isn't wet, it isn't raining | ✅ **Always** |
| **Converse** | q → p | If the ground is wet, it's raining | ❌ Maybe someone used a hose! |
| **Inverse** | NOT p → NOT q | If it isn't raining, the ground isn't wet | ❌ Same problem |

The contrapositive is very useful. It lets you prove something from the other direction. Let's check all cases in code:

```python
def implies(p, q):
    return (not p) or q

for p in [True, False]:
    for q in [True, False]:
        original = implies(p, q)
        contrapositive = implies(not q, not p)
        print(original == contrapositive)
```

Output:

```text
True
True
True
True
```

They match in every case, so they mean the same thing.

> [!example] Where it helps in code
> "If the user is an admin, they can delete posts" has the contrapositive "If someone can't delete posts, they aren't an admin". When a tester finds an admin who can't delete, the rule has been broken.

## Predicates and Quantifiers

A **predicate** is a statement with a blank in it, such as "x is even". It becomes true or false once you fill in the blank.

```python
def is_even(x):
    return x % 2 == 0

print(is_even(4))
print(is_even(7))
```

Output:

```text
True
False
```

**Quantifiers** say **how many** things must meet the predicate.

| Quantifier | Symbol | Meaning | Python |
|---|---|---|---|
| For all | ∀ | Every item meets it | `all(...)` |
| There exists | ∃ | At least one item meets it | `any(...)` |

```python
numbers = [2, 4, 6, 7]

print(all(n % 2 == 0 for n in numbers))
print(any(n % 2 == 0 for n in numbers))
print(any(n > 100 for n in numbers))
```

Output:

```text
False
True
False
```

### Negating Quantifiers

To say that "for all" is false, you need just **one** counterexample.

- "All swans are white" is false if you find **one** black swan.
- "Some student passed" is false only if **every** student failed.

| Statement | Its opposite |
|---|---|
| All numbers are even | **There is** a number that isn't even |
| There is a student who passed | **All** students did not pass |

This is why testing can show that code has a bug, but can't prove it has none. **One** failing test disproves "works for all inputs". But a thousand passing tests can't check every input. For that, you need proof.

## Proofs

A **proof** is an argument that shows something is always true, not just in the examples you tried.

### Proof by Induction

Imagine a long line of dominoes. How can you be sure that all of them will fall?

1. **Base case:** the first domino falls.
2. **Inductive step:** if any domino falls, it knocks over the next one.

Then all of them fall, no matter how many there are. Notice that this looks just like [[04 Functions and Scope#Recursion|recursion]]!

> [!example] Prove: 1 + 2 + ... + n = n(n + 1) / 2
> **Base case (n = 1):** the left side is 1. The right side is 1(2)/2 = 1. They match. ✓
>
> **Inductive step:** assume that it's true for some number k. So we assume:
>
> 1 + 2 + ... + k = k(k + 1) / 2
>
> Now add the next number, (k + 1), to both sides:
>
> 1 + 2 + ... + k + (k + 1) = k(k + 1)/2 + (k + 1)
>
> = (k + 1)(k/2 + 1)
>
> = (k + 1)(k + 2) / 2
>
> This is exactly the formula for n = k + 1. So if it's true for k, it's true for k + 1. ✓
>
> Since it's true for 1, it's true for 2, then 3, and so on, forever.

We can test the formula in code. Remember, this is only checking some cases. The proof above covers them all.

```python
def sum_by_loop(n):
    total = 0
    for number in range(1, n + 1):
        total = total + number
    return total

def sum_by_formula(n):
    return n * (n + 1) // 2

print(sum_by_loop(100), sum_by_formula(100))
print(all(sum_by_loop(n) == sum_by_formula(n) for n in range(1, 500)))
```

Output:

```text
5050 5050
True
```

### Proof by Contradiction

Assume the opposite of what you want to prove. Then show that this leads to something impossible. So the opposite must be false.

> [!example] Prove: there's no largest whole number
> 1. Assume the opposite: there **is** a largest whole number. Call it N.
> 2. But N + 1 is also a whole number, and it's bigger than N.
> 3. That contradicts our claim that N is the largest.
> 4. So our assumption was wrong. There's no largest whole number. ✓

## Loop Invariants

A **loop invariant** is a fact that stays true every time the loop goes around. It's like a small proof for your code, and it works the same way as induction:

- **Start:** it's true before the loop begins. (Base case)
- **Each round:** if it's true before a round, it's still true after. (Inductive step)
- **End:** when the loop stops, it tells you that your answer is right.

> [!example] Finding the largest number
> **Invariant:** `largest` is the biggest of the numbers seen so far.

```python
def find_largest(numbers):
    largest = numbers[0]
    for i in range(1, len(numbers)):
        # Invariant: largest is the biggest of numbers[0..i-1]
        assert largest == max(numbers[:i])
        if numbers[i] > largest:
            largest = numbers[i]
    return largest

print(find_largest([4, 9, 2, 7]))
```

Output:

```text
9
```

- **Start:** we've seen one number, and `largest` is that number. ✓
- **Each round:** if the new number is bigger, we update. If not, we keep the old one. Either way, it's still the biggest so far. ✓
- **End:** we've seen all the numbers, so `largest` is the biggest of them all. ✓

The `assert` line checks the invariant while the code runs. If it were ever false, Python would stop with an `AssertionError`. You wouldn't normally keep this line, since it slows the loop, but it's a good learning tool.

## Common Mistakes

> [!warning] Watch out for these
> - Confusing "p → q" with "q → p". Rain makes the ground wet, but a wet ground doesn't prove rain.
> - Treating examples as proof. Passing tests isn't the same as being correct for all inputs.
> - Forgetting the base case in induction, as with recursion.
> - Thinking that "or" means "only one of them".

## Practice

**1. Propositions.** Which of these are propositions? "7 is prime". "Open the file". "x + 1 = 3".

> [!example]- Answer to question 1
> Only "7 is prime" is a proposition (and it's true). "Open the file" is a command. "x + 1 = 3" depends on x, so it's a predicate.

**2. Truth table.** Make a truth table for `p AND (NOT q)`.

> [!example]- Answer to question 2
> | p | q | NOT q | p AND (NOT q) |
> |---|---|---|---|
> | T | T | F | F |
> | T | F | T | **T** |
> | F | T | F | F |
> | F | F | T | F |

**3. Contrapositive.** What's the contrapositive of "If the password is correct, then the login succeeds"?

> [!example]- Answer to question 3
> "If the login doesn't succeed, then the password isn't correct." Notice both parts are flipped **and** swapped.

**4. Counterexample.** Someone claims: "all numbers of the form n² + n + 41 are prime". Find a number n that disproves this. Hint: use code.

> [!example]- Answer to question 4
> ```python
> def is_prime(n):
>     if n < 2:
>         return False
>     for divisor in range(2, n):
>         if n % divisor == 0:
>             return False
>     return True
>
> for n in range(100):
>     value = n * n + n + 41
>     if not is_prime(value):
>         print(n, value)
>         break
> ```
>
> Output:
>
> ```text
> 40 1681
> ```
>
> The first 40 values are all prime, and then it fails! 1681 = 41 × 41. So many examples can look right, and the claim is still false. This is why we need proofs.

**5. Induction.** Using the method above, prove that 2 + 4 + 6 + ... + 2n = n(n + 1).

> [!example]- Answer to question 5
> **Base case (n = 1):** the left side is 2. The right side is 1 × 2 = 2. ✓
>
> **Inductive step:** assume 2 + 4 + ... + 2k = k(k + 1). Add the next even number, 2(k + 1):
>
> k(k + 1) + 2(k + 1) = (k + 1)(k + 2)
>
> This is the formula for n = k + 1. ✓

## Can You Do This?

- [ ] Say whether a statement is a proposition
- [ ] Write truth tables for AND, OR, NOT and implication
- [ ] Explain when "if p then q" is false
- [ ] Write the contrapositive of a statement
- [ ] Use `all` and `any` for "for all" and "there exists"
- [ ] Prove a simple formula by induction
- [ ] Do a simple proof by contradiction
- [ ] Name the invariant of a simple loop

## Summary

> [!success] Remember
> - A **proposition** is true or false.
> - **Truth tables** check every case.
> - `p → q` is false only when p is true and q is false.
> - One **counterexample** disproves "for all". Examples alone can't prove it.
> - **Induction:** a base case, and a step from each case to the next.

**Next:** [[13 Boolean Algebra]] →
