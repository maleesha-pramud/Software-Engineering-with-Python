---
title: Conditions and Loops
tags: [phase-1, programming]
aliases: [Conditionals, Loops, If Statements]
---

# Conditions and Loops

← [[02 Variables, Types and Expressions]] · [[Phase 1 - Start Here|Home]] · **Next:** [[04 Functions and Scope]] → · [[Software Engineering Roadmap#Programming Fundamentals|Roadmap card]]

> [!abstract] After this note, you can
> Make your program choose between options and repeat work. With these two powers, you can write almost any algorithm.

## Making Choices with if

Every day, you make decisions. **If** it is raining, take an umbrella. **Otherwise**, wear sunglasses. Programs decide in the same way.

```python
is_raining = True

if is_raining:
    print("Take an umbrella")
else:
    print("Wear sunglasses")

print("Go outside")
```

Output:

```text
Take an umbrella
Go outside
```

> [!important] Spaces matter in Python
> The spaces at the start of a line are called **indentation**. They show which lines belong to the `if`. Use 4 spaces. `print("Go outside")` has no spaces, so it always runs. Don't forget the `:` at the end of the `if` and `else` lines.

### Comparing Values

A comparison gives you `True` or `False`.

| Symbol | Meaning | Example | Result |
|---|---|---|---|
| `==` | Equal to | `5 == 5` | `True` |
| `!=` | Not equal to | `5 != 5` | `False` |
| `>` | Greater than | `7 > 3` | `True` |
| `<` | Less than | `7 < 3` | `False` |
| `>=` | Greater than or equal to | `5 >= 5` | `True` |
| `<=` | Less than or equal to | `4 <= 3` | `False` |

### More Than Two Choices with elif

`elif` means "otherwise, if". Python checks each condition from top to bottom, and stops at the **first** true one.

```python
mark = 72

if mark >= 75:
    grade = "A"
elif mark >= 65:
    grade = "B"
elif mark >= 50:
    grade = "C"
else:
    grade = "F"

print(grade)
```

Output:

```text
B
```

72 is not 75 or more, so Python moves on. 72 is 65 or more, so the grade is `B`, and Python skips the rest. The order matters here. If `mark >= 50` came first, almost everyone would get `C`.

### Joining Conditions with and, or, not

| Word | True when | Example |
|---|---|---|
| `and` | **Both** sides are true | `age >= 18 and has_id` |
| `or` | **At least one** side is true | `is_weekend or is_holiday` |
| `not` | The opposite | `not is_raining` |

```python
age = 20
has_ticket = True

if age >= 18 and has_ticket:
    print("You can enter")
else:
    print("Sorry, you can't enter")
```

Output:

```text
You can enter
```

There's a deep and useful science behind these words. You'll meet it in [[13 Boolean Algebra]].

## Repeating Work - Why Loops?

Say you want to print the numbers from 1 to 1000. Writing 1000 `print` lines is silly. A **loop** repeats code for you. Remember [[01 Computational Thinking#2. Pattern Recognition|pattern recognition]]: a loop is how you act on a pattern.

Python has two main loops:

| Loop | Use it when | Real-life example |
|---|---|---|
| `for` | You know what you are going through | "For each student, mark the paper" |
| `while` | You repeat until something changes | "While the water isn't hot, keep heating" |

### The for Loop

```python
for number in range(1, 6):
    print(number)
```

Output:

```text
1
2
3
4
5
```

`range(1, 6)` gives the numbers from 1 up to, **but not including**, 6. This trips up nearly every beginner. Some useful forms:

| Code | Gives you |
|---|---|
| `range(5)` | 0, 1, 2, 3, 4 |
| `range(1, 5)` | 1, 2, 3, 4 |
| `range(0, 10, 2)` | 0, 2, 4, 6, 8 (steps of 2) |

A `for` loop can go through other things too, such as the letters in a word or the items in a list:

```python
for letter in "cat":
    print(letter)

for fruit in ["apple", "mango"]:
    print(f"I like {fruit}")
```

Output:

```text
c
a
t
I like apple
I like mango
```

> [!example] Worked example: add up the numbers from 1 to 5
> This pattern is one of the most useful in programming. Start a total at 0, then add to it each time around the loop.

```python
total = 0

for number in range(1, 6):
    total = total + number
    print(f"Added {number}, total is now {total}")

print(f"Final total: {total}")
```

Output:

```text
Added 1, total is now 1
Added 2, total is now 3
Added 3, total is now 6
Added 4, total is now 10
Added 5, total is now 15
Final total: 15
```

### The while Loop

A `while` loop keeps running as long as its condition is true.

```python
count = 3

while count > 0:
    print(count)
    count = count - 1

print("Go!")
```

Output:

```text
3
2
1
Go!
```

> [!danger] Infinite loops
> If the condition never becomes false, the loop runs forever. Remove `count = count - 1` from the code above, and it prints `3` without ever stopping. If this happens, press **Ctrl + C** in the terminal to stop your program. It happens to everyone.

### break and continue

- `break` leaves the loop at once.
- `continue` skips the rest of this round and goes on to the next one.

```python
for number in range(1, 10):
    if number == 3:
        continue
    if number == 6:
        break
    print(number)
```

Output:

```text
1
2
4
5
```

> [!example] Worked example: keep asking until the password is right
> A `while True` loop runs forever, until a `break` stops it. This is the normal way to keep asking the user for something.

```python
while True:
    word = input("Password: ")
    if word == "open123":
        print("Welcome!")
        break
    print("Wrong, try again.")
```

Example run:

```text
Password: hello
Wrong, try again.
Password: open123
Welcome!
```

### Loops Inside Loops

You can put a loop inside another loop. The inner loop finishes all its rounds for **each** round of the outer loop, like the hands of a clock.

```python
for row in range(1, 4):
    for column in range(1, 4):
        print(row * column, end=" ")
    print()
```

Output:

```text
1 2 3 
2 4 6 
3 6 9 
```

`end=" "` tells `print` to end with a space instead of starting a new line.

## Recursion - A Function That Calls Itself

There's one more way to repeat. It needs functions, so you'll learn it in [[04 Functions and Scope#Recursion]].

## Common Mistakes

> [!warning] Watch out for these
> - Writing `if x = 5:` instead of `if x == 5:`
> - Forgetting the `:` at the end of a line
> - Wrong indentation, which changes what your code means
> - **Off-by-one errors:** the loop runs one time too many or too few. Check where `range` stops!
> - Forgetting to change the variable in a `while` loop

## Practice

**1. Even or odd.** Write a program that reads a number and prints `even` or `odd`. Hint: use `%`.

> [!example]- Answer to question 1
> ```python
> number = int(input("Number: "))
> if number % 2 == 0:
>     print("even")
> else:
>     print("odd")
> ```

**2. FizzBuzz.** This is a famous beginner test. Print the numbers from 1 to 15. For multiples of 3, print `Fizz` instead. For multiples of 5, print `Buzz`. For multiples of both, print `FizzBuzz`.

> [!example]- Answer to question 2
> ```python
> for number in range(1, 16):
>     if number % 3 == 0 and number % 5 == 0:
>         print("FizzBuzz")
>     elif number % 3 == 0:
>         print("Fizz")
>     elif number % 5 == 0:
>         print("Buzz")
>     else:
>         print(number)
> ```
>
> Why check "both" first? Because 15 is also a multiple of 3. If you checked 3 first, you would print `Fizz` and never reach `FizzBuzz`. Order matters!

**3. Guessing game.** The secret number is 7. Keep asking the user to guess. Tell them "too high" or "too low", and stop when they are right. Then count how many guesses they took.

> [!example]- Answer to question 3
> ```python
> secret = 7
> guesses = 0
>
> while True:
>     guess = int(input("Your guess: "))
>     guesses = guesses + 1
>     if guess > secret:
>         print("Too high")
>     elif guess < secret:
>         print("Too low")
>     else:
>         print(f"Correct! You took {guesses} guesses.")
>         break
> ```

**4. Stars.** Use a loop to print this:

```text
*
**
***
****
```

> [!example]- Answer to question 4
> ```python
> for count in range(1, 5):
>     print("*" * count)
> ```
>
> In Python, `"*" * 3` gives `***`.

**5. Predict.** What does this print?

```python
total = 0
for number in range(3):
    total = total + number
print(total)
```

> [!example]- Answer to question 5
> It prints `3`. `range(3)` gives 0, 1, 2. So 0 + 1 + 2 = 3.

## Can You Do This?

- [ ] Write `if`, `elif` and `else` with correct indentation
- [ ] Join conditions using `and`, `or` and `not`
- [ ] Choose between a `for` loop and a `while` loop
- [ ] Say exactly which numbers a `range` gives
- [ ] Use a loop to add up a total
- [ ] Stop an infinite loop with Ctrl + C
- [ ] Write FizzBuzz without looking

## Summary

> [!success] Remember
> - `if`, `elif`, `else` choose what to run. Python takes the first true option.
> - `for` goes through a known group of things.
> - `while` repeats while a condition is true. Make sure it can end!
> - `range(a, b)` stops **before** `b`.

**Next:** [[04 Functions and Scope]] →
