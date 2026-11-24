---
title: Variables, Types and Expressions
tags: [phase-1, programming]
aliases: [Variables, Data Types, Expressions]
---

# Variables, Types and Expressions

← [[01 Computational Thinking]] · [[Phase 1 - Start Here|Home]] · **Next:** [[03 Conditions and Loops]] → · [[Software Engineering Roadmap#Programming Fundamentals|Roadmap card]]

> [!abstract] After this note, you can
> Store information in your program, do calculations with it, and get input from the user.

## Variables - Labeled Boxes

A **variable** is a name for a piece of data. Think of a box with a label on it. You put something in the box, and you use the label to find it later.

```python
age = 20
name = "Sam"

print(name)
print(age)
```

Output:

```text
Sam
20
```

The `=` sign means "put this value into this name". It does **not** mean "is equal to", as it does in math. So this line makes sense in code, even though it looks wrong in math:

```python
score = 10
score = score + 5
print(score)
```

Output:

```text
15
```

Read it from right to left: take the current `score` (10), add 5, and put the result (15) back into `score`.

### Good Names

Choose names that explain what the data **means**.

| Bad name | Good name | Why |
|---|---|---|
| `x` | `price` | Says what it holds |
| `d` | `days_left` | Clear without a comment |
| `n1` | `first_name` | Nobody has to guess |

Python rules: names can use letters, numbers and `_`. They can't start with a number, and they can't have spaces. Capital letters matter, so `Age` and `age` are different variables. Python programmers usually write `words_like_this`.

## Types - Different Kinds of Data

Every value has a **type**. The type tells Python what the value is, and what you can do with it.

| Type | Name in Python | Examples | Used for |
|---|---|---|---|
| Whole number | `int` | `7`, `-3`, `2024` | Counting, ages |
| Decimal number | `float` | `3.14`, `0.5` | Prices, measurements |
| Text | `str` (string) | `"hello"`, `'Sam'` | Names, messages |
| True or false | `bool` | `True`, `False` | Yes or no answers |
| Nothing | `NoneType` | `None` | "There is no value" |

You can ask Python for the type of any value:

```python
print(type(7))
print(type(3.14))
print(type("hello"))
print(type(True))
```

Output:

```text
<class 'int'>
<class 'float'>
<class 'str'>
<class 'bool'>
```

### Why Types Matter

The same symbol can do different things for different types:

```python
print(2 + 3)
print("2" + "3")
```

Output:

```text
5
23
```

Numbers get added. Strings get joined together. `"2"` is not a number. It is a piece of text that happens to look like one.

If you mix them, Python stops and tells you:

```python
print("Age: " + 20)
```

Output:

```text
Traceback (most recent call last):
  File "main.py", line 1, in <module>
    print("Age: " + 20)
TypeError: can only concatenate str (not "int") to str
```

Don't be scared by error messages. **Read the last line first.** It tells you the problem in fairly plain words: you can't join a string and a number. We study errors properly in [[10 Debugging]].

### Changing Types

You can convert a value from one type to another:

```python
age_text = "20"
age_number = int(age_text)
print(age_number + 1)

print(str(99) + " red balloons")
print(float("3.5"))
print(int(3.9))
```

Output:

```text
21
99 red balloons
3.5
3
```

Notice the last line. `int()` **cuts off** the decimal part. It doesn't round up.

## Expressions - Calculations

An **expression** is anything that Python can work out to get a value. `2 + 3` is an expression, and its value is 5.

| Symbol | Meaning | Example | Result |
|---|---|---|---|
| `+` | Add | `7 + 2` | `9` |
| `-` | Subtract | `7 - 2` | `5` |
| `*` | Multiply | `7 * 2` | `14` |
| `/` | Divide | `7 / 2` | `3.5` |
| `//` | Divide, drop the decimal | `7 // 2` | `3` |
| `%` | Remainder after dividing | `7 % 2` | `1` |
| `**` | Power | `7 ** 2` | `49` |

The remainder symbol `%` is more useful than it looks. A number is **even** if its remainder after dividing by 2 is 0.

Python follows the math order: brackets first, then powers, then multiply and divide, then add and subtract.

```python
print(2 + 3 * 4)
print((2 + 3) * 4)
```

Output:

```text
14
20
```

> [!example] Worked example: a shopping bill
> Three notebooks cost 2.50 each. You get a 10% discount. What do you pay?

```python
price = 2.50
quantity = 3
discount = 0.10

total = price * quantity
amount_off = total * discount
to_pay = total - amount_off

print(to_pay)
```

Output:

```text
6.75
```

See how the good variable names make the code read like a story.

> [!warning] Decimal numbers are not always exact
> Try `print(0.1 + 0.2)`. You'll see `0.30000000000000004`. This isn't a Python bug. Computers store decimals in binary, and many decimals can't be stored exactly, just as 1/3 can't be written exactly as 0.333... For this reason, real banking software doesn't use `float` for money.

## Printing Nicely with f-strings

An **f-string** lets you put values inside text. Put an `f` before the quote mark, and put your variables in `{ }`.

```python
name = "Sam"
age = 20
print(f"{name} is {age} years old.")
print(f"Next year: {age + 1}")

price = 6.756
print(f"Total: {price:.2f}")
```

Output:

```text
Sam is 20 years old.
Next year: 21
Total: 6.76
```

`:.2f` means "show exactly 2 decimal places". You will use it a lot.

## Getting Input from the User

`input()` shows a message and waits for the user to type something.

```python
name = input("What is your name? ")
print(f"Hello, {name}!")
```

Example run:

```text
What is your name? Sam
Hello, Sam!
```

> [!warning] The most common beginner mistake
> `input()` **always gives you a string**, even if the user types a number. You must convert it before doing math.

```python
age = int(input("How old are you? "))
print(f"In 10 years, you will be {age + 10}.")
```

Example run:

```text
How old are you? 20
In 10 years, you will be 30.
```

## Comments - Notes for Humans

Python ignores anything after a `#`. Use comments to explain **why** you did something. The code itself already shows what it does.

```python
# Tax rate changed by the government in 2024
tax_rate = 0.18

price = 100
print(price * (1 + tax_rate))
```

Output:

```text
118.0
```

## Common Mistakes

> [!warning] Watch out for these
> - Using `=` when you want to compare. `=` stores a value. `==` compares two values. See [[03 Conditions and Loops]].
> - Forgetting quote marks: `name = Sam` makes Python look for a variable called `Sam`.
> - Forgetting to convert `input()` to a number.
> - Spelling a variable differently: `Score` and `score` are two different names.

## Practice

**1. Predict.** Before running this, what will it print?

```python
a = 5
b = a
a = 10
print(a, b)
```

> [!example]- Answer to question 1
> It prints `10 5`. When you wrote `b = a`, `b` got the value 5. Changing `a` later doesn't change `b`.

**2. Temperature.** Write a program that asks for a temperature in Celsius and prints it in Fahrenheit. The formula is `F = C × 9 / 5 + 32`.

> [!example]- Answer to question 2
> ```python
> celsius = float(input("Temperature in Celsius: "))
> fahrenheit = celsius * 9 / 5 + 32
> print(f"{celsius} C is {fahrenheit} F")
> ```

**3. Time.** You have 125 minutes. Print it as hours and minutes. Use `//` and `%`.

> [!example]- Answer to question 3
> ```python
> minutes = 125
> hours = minutes // 60
> left_over = minutes % 60
> print(f"{hours} hours and {left_over} minutes")
> ```
>
> Output:
>
> ```text
> 2 hours and 5 minutes
> ```

**4. Fix it.** This program crashes. Why?

```python
age = input("Age: ")
print(age + 1)
```

> [!example]- Answer to question 4
> `input()` gives a string, and you can't add a string and a number. Fix it with `age = int(input("Age: "))`.

## Can You Do This?

- [ ] Create variables with clear names
- [ ] Name the five basic types and give an example of each
- [ ] Explain why `"2" + "3"` gives `23`
- [ ] Use all the math symbols, including `%` and `//`
- [ ] Use an f-string with 2 decimal places
- [ ] Read a number from the user and do math with it

## Summary

> [!success] Remember
> - A **variable** is a name for a value. `=` stores a value.
> - A **type** decides what you can do with a value.
> - An **expression** is something Python works out to get a value.
> - `input()` always gives a string. Convert it before doing math.

**Next:** [[03 Conditions and Loops]] →
