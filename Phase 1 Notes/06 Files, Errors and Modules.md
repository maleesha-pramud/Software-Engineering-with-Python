---
title: Files, Errors and Modules
tags: [phase-1, programming]
aliases: [Files, Error Handling, Exceptions, Modules, Packages]
---

# Files, Errors and Modules

← [[05 Lists, Strings and Dictionaries]] · [[Phase 1 - Start Here|Home]] · **Next:** [[07 Problem-Solving Techniques]] → · [[Software Engineering Roadmap#Programming Fundamentals|Roadmap card]]

> [!abstract] After this note, you can
> Save data so that it stays after your program ends, handle problems without crashing, and use code written by other people.

## Files - Memory That Lasts

Variables live in memory. When your program ends, they are gone. Think of memory as a whiteboard that gets wiped clean each night. A **file** is a notebook: what you write there stays.

### Writing a File

```python
with open("notes.txt", "w") as file:
    file.write("First line\n")
    file.write("Second line\n")

print("Saved")
```

Output:

```text
Saved
```

This creates `notes.txt` next to your program. Some key points:

- `"w"` means **write**. Be careful, because it **erases** anything already in the file.
- `\n` means "start a new line".
- `with` closes the file for you when the block ends, even if something goes wrong. Always use it.

### Reading a File

```python
with open("notes.txt", "r") as file:
    for line in file:
        print(line.strip())
```

Output:

```text
First line
Second line
```

`.strip()` removes the `\n` at the end of each line.

### File Modes

| Mode | Meaning | If the file exists |
|---|---|---|
| `"r"` | Read | Reads it. If the file is missing, you get an error |
| `"w"` | Write | **Erases it** and starts again |
| `"a"` | Append | Adds to the end, and keeps what's there |

```python
with open("notes.txt", "a") as file:
    file.write("Third line\n")

with open("notes.txt") as file:
    print(file.read())
```

Output:

```text
First line
Second line
Third line
```

> [!example] Worked example: save and load a list with JSON
> **JSON** is a popular text format that looks a lot like Python's lists and dictionaries. Almost every language can read it, and websites use it constantly. It's the easiest way to save structured data.

```python
import json

tasks = [
    {"title": "Buy milk", "done": False},
    {"title": "Call mom", "done": True},
]

with open("tasks.json", "w") as file:
    json.dump(tasks, file, indent=2)

with open("tasks.json") as file:
    loaded = json.load(file)

print(loaded[0]["title"])
print(loaded[1]["done"])
```

Output:

```text
Buy milk
True
```

You'll use exactly this in the [[14 Phase 1 Final Project|final project]].

## Errors and Exceptions

Things go wrong in real programs. Users type nonsense. Files go missing. Networks fail. When Python hits a problem it can't handle, it creates an **exception**. If nothing deals with it, your program crashes.

```python
number = int("hello")
```

Output:

```text
Traceback (most recent call last):
  File "main.py", line 1, in <module>
    number = int("hello")
ValueError: invalid literal for int() with base 10: 'hello'
```

### Handling Errors with try and except

`try` means "attempt this". `except` means "if this specific error happens, do this instead".

```python
text = "hello"

try:
    number = int(text)
    print(f"You typed {number}")
except ValueError:
    print("That is not a number")

print("The program continues")
```

Output:

```text
That is not a number
The program continues
```

> [!example] Worked example: keep asking until the input is valid
> This combines a loop, `try` and `break`. It's a very common pattern.

```python
def ask_for_number():
    while True:
        text = input("Enter a number: ")
        try:
            return int(text)
        except ValueError:
            print("Please type a whole number.")

age = ask_for_number()
print(f"Got {age}")
```

Example run:

```text
Enter a number: abc
Please type a whole number.
Enter a number: 20
Got 20
```

### Handling a Missing File

```python
import json

def load_tasks():
    try:
        with open("missing_file.json") as file:
            return json.load(file)
    except FileNotFoundError:
        return []

print(load_tasks())
```

Output:

```text
[]
```

A missing file isn't really a problem here. It only means the user has no tasks yet, so we return an empty list.

### Raising Your Own Errors

You can create an error on purpose when your function gets a bad value. This is much better than continuing with wrong data.

```python
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("Not enough money")
    return balance - amount

print(withdraw(100, 30))

try:
    withdraw(100, 500)
except ValueError as error:
    print(f"Problem: {error}")
```

Output:

```text
70
Problem: Not enough money
```

> [!warning] Rules for handling errors
> - Catch **specific** errors like `except ValueError`. Don't write a bare `except:`, because it also hides bugs in your code.
> - Never silently ignore an error. Do something useful, or at least tell the user.
> - Use `try` only around code that can really fail, such as user input, files or networks. Don't use it to hide your own mistakes.

## Modules and Packages

A **module** is a Python file you can reuse. A **package** is a folder of modules. This is how you use code written by others, so you don't have to write everything yourself.

### The Standard Library

Python comes with a huge collection of ready-made modules, called the **standard library**. Use `import` to get one.

```python
import math
from datetime import date

print(math.sqrt(16))
print(math.floor(3.7))
print(date(2026, 1, 15).year)
```

Output:

```text
4.0
3
2026
```

There's also the `random` module. For example, `random.randint(1, 6)` gives a random whole number from 1 to 6, like rolling a dice.

| Module | Useful for |
|---|---|
| `math` | Square roots, rounding, pi |
| `random` | Random numbers and choices |
| `datetime` | Dates and times |
| `json` | Saving and loading JSON |
| `os`, `pathlib` | Files and folders |
| `sys` | Command-line arguments |

### Your Own Modules

Any `.py` file is a module. Say you create a file called `tools.py`:

```python
# file: tools.py
def add_tax(price):
    return price + price * 0.18
```

Then another file in the same folder can import it:

```python
# file: shop.py
import tools

print(tools.add_tax(100))
```

Output:

```text
118.0
```

Don't give your files the same names as standard modules. A file called `random.py` will stop `import random` from working.

### The `__name__` Check

You will often see this at the bottom of Python files:

```python
def main():
    print("Program started")

if __name__ == "__main__":
    main()
```

Output:

```text
Program started
```

It means: run `main()` only if this file was started directly, and not when another file imports it. This lets one file work as both a program and a module. You'll need it for testing in [[11 Unit Testing Basics]].

### Installing Packages with pip

Beyond the standard library, thousands of free packages exist. Install them with `pip`:

```bash
python -m pip install requests
```

> [!tip] Virtual environments
> A **virtual environment** gives each project its own private set of packages, so projects don't conflict.
> ```bash
> python -m venv .venv
> ```
> Then turn it on:
> - **Windows:** `.venv\Scripts\activate`
> - **Mac and Linux:** `source .venv/bin/activate`

## Reading Documentation

Nobody remembers everything. Good programmers look things up all day. Learning to read docs is a real skill.

The official guide is at [docs.python.org](https://docs.python.org/3/). The [standard library page](https://docs.python.org/3/library/index.html) lists every module.

You can also ask Python itself. Type `python` in the terminal to open the Python prompt, then try `help(str.split)`. Press `q` to close the help page, and type `exit()` to leave the prompt.

**How to read a doc page:**

1. Find the function's name and its parameters.
2. Read what it returns.
3. Read the example, and type it yourself.
4. Look at what errors it can raise.

## Common Mistakes

> [!warning] Watch out for these
> - Using `"w"` when you meant `"a"`, and erasing your file
> - Opening a file without `with`, and forgetting to close it
> - A file isn't found, because your terminal is in a different folder than you think. See [[08 Linux Basics#Moving Around]].
> - Catching every error with a bare `except:`
> - Forgetting `.strip()` when reading lines

## Practice

**1. Line counter.** Write a program that counts the lines in a file.

> [!example]- Answer to question 1
> ```python
> with open("sample.txt", "w") as file:
>     file.write("one\ntwo\nthree\n")
>
> count = 0
> with open("sample.txt") as file:
>     for line in file:
>         count = count + 1
>
> print(count)
> ```
>
> Output:
>
> ```text
> 3
> ```

**2. Safe divide.** Write `safe_divide(a, b)`, which returns `a / b`. If `b` is 0, return `None` by catching `ZeroDivisionError`.

> [!example]- Answer to question 2
> ```python
> def safe_divide(a, b):
>     try:
>         return a / b
>     except ZeroDivisionError:
>         return None
>
> print(safe_divide(10, 2))
> print(safe_divide(10, 0))
> ```
>
> Output:
>
> ```text
> 5.0
> None
> ```

**3. Guess my number.** Use `random.randint` to pick a number from 1 to 20, and make a guessing game that handles non-number input without crashing.

> [!example]- Answer to question 3
> ```python
> import random
>
> secret = random.randint(1, 20)
>
> while True:
>     try:
>         guess = int(input("Guess: "))
>     except ValueError:
>         print("Numbers only, please.")
>         continue
>
>     if guess > secret:
>         print("Too high")
>     elif guess < secret:
>         print("Too low")
>     else:
>         print("You got it!")
>         break
> ```

**4. Word counter.** Read a text file and print the five most common words. This uses [[05 Lists, Strings and Dictionaries#Dictionaries|dictionaries]] and `sorted`.

> [!example]- Answer to question 4
> ```python
> with open("story.txt", "w") as file:
>     file.write("the cat saw the dog and the dog saw the cat")
>
> counts = {}
> with open("story.txt") as file:
>     for word in file.read().lower().split():
>         counts[word] = counts.get(word, 0) + 1
>
> top = sorted(counts.items(), key=lambda pair: pair[1], reverse=True)
> print(top[:2])
> ```
>
> Output:
>
> ```text
> [('the', 4), ('cat', 2)]
> ```

## Can You Do This?

- [ ] Read and write a text file using `with`
- [ ] Explain the difference between modes `r`, `w` and `a`
- [ ] Save and load data as JSON
- [ ] Handle bad input with `try` and `except`
- [ ] Raise your own error with `raise`
- [ ] Import a module, and create your own
- [ ] Install a package with `pip`
- [ ] Find a function in the official docs and use it

## Summary

> [!success] Remember
> - **Files** keep data after your program ends. Always use `with`.
> - `"w"` erases the file. `"a"` adds to the end.
> - `try` and `except` handle problems you expect. Catch specific errors.
> - **Modules** let you reuse code. Check the standard library before writing your own.
> - Reading docs is a normal part of the job.

You've now finished the core of programming. Next, you'll learn how to use these tools to solve problems.

**Next:** [[07 Problem-Solving Techniques]] →
