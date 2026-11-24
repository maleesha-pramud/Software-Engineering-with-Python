---
title: Unit Testing Basics
tags: [phase-1, tools]
aliases: [Unit Testing, Tests, pytest]
---

# Unit Testing Basics

← [[10 Debugging]] · [[Phase 1 - Start Here|Home]] · **Next:** [[12 Mathematical Logic]] → · [[Software Engineering Roadmap#Unit Testing|Roadmap card]]

> [!abstract] After this note, you can
> Write small programs that automatically check that your code works, so that you can change it without fear.

## The Big Idea

Imagine a factory making light bulbs. Before each bulb leaves, a machine switches it on to check it works. A **unit test** is that machine for your code. It's a small piece of code that runs a function and checks the result.

A **unit** is a small piece of code, usually one function.

Without tests, you'd run your program by hand after each change and hope that you didn't break anything old. With tests, one command checks everything in seconds.

**Why bother?**

- **You catch bugs early,** when they are cheap to fix.
- **You can change code safely.** If a test fails, you know right away.
- **Tests show how your code should be used.** They're a kind of documentation.
- **They make you think** about strange cases before they happen.

## Your First Test

**Install pytest,** the most popular Python testing tool:

```bash
python -m pip install pytest
```

**Step 1.** Create a file called `calculator.py`:

```python
# file: calculator.py
def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def is_adult(age):
    return age >= 18
```

**Step 2.** Create a file called `test_calculator.py` next to it. The name must start with `test_`.

```python
# file: test_calculator.py
from calculator import add

def test_add_two_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-1, -1) == -2
```

`assert` means "this must be true". If it's false, the test fails.

**Step 3.** Run the tests:

```bash
python -m pytest
```

Example output:

```text
===================== test session starts =====================
collected 2 items

test_calculator.py ..                                    [100%]

====================== 2 passed in 0.01s ======================
```

Each `.` is a passing test.

### What a Failing Test Looks Like

Change `add` to `return a - b`, and run the tests again:

```text
    def test_add_two_numbers():
>       assert add(2, 3) == 5
E       assert -1 == 5
E        +  where -1 = add(2, 3)
```

The test tells you what it expected and what it got. Now you can use your [[10 Debugging|debugging]] skills. Change it back afterward.

> [!tip] Make sure your test can fail
> A test that can never fail is useless. When you write a new test, break the code on purpose for a moment, and check that the test goes red.

## How to Write a Good Test - Arrange, Act, Assert

Most tests have three parts:

1. **Arrange:** set up the data.
2. **Act:** run the code you're testing.
3. **Assert:** check the result.

```python
def apply_discount(price, percent):
    return price - price * percent / 100

def test_apply_discount_takes_percent_off_price():
    # Arrange
    price = 200
    percent = 10

    # Act
    result = apply_discount(price, percent)

    # Assert
    assert result == 180
```

### Clear Test Names

A test's name should say what it checks. When it fails, the name alone should tell you what's broken.

| Bad name | Good name |
|---|---|
| `test_1` | `test_add_two_numbers` |
| `test_divide` | `test_divide_by_zero_raises_error` |
| `test_login_works` | `test_login_fails_with_wrong_password` |

### What Should You Test?

Most bugs hide at the edges, so test more than the normal case.

| Kind of case | For a list function | For `is_adult(age)` |
|---|---|---|
| **Normal** | A list of a few items | `30` |
| **Boundary** (at the limit) | A list with 1 item | `17` and `18` |
| **Empty or zero** | An empty list | `0` |
| **Invalid** | Wrong type of data | `-5` |

> [!example] Why boundaries matter
> Imagine someone wrote `age > 18` by mistake. Testing `30` passes, and testing `10` passes. Only testing exactly `18` catches the bug.

```python
from calculator import is_adult

def test_age_18_is_adult():
    assert is_adult(18) is True

def test_age_17_is_not_adult():
    assert is_adult(17) is False
```

### Testing That Errors Happen

Sometimes the correct behaviour is an error. Use `pytest.raises` to check for it.

```python
import pytest
from calculator import divide

def test_divide_normal():
    assert divide(10, 2) == 5

def test_divide_by_zero_raises_error():
    with pytest.raises(ValueError):
        divide(10, 0)
```

### Testing Many Cases at Once

If you find yourself writing the same test with different values, use **parametrize**. Each row becomes its own test.

```python
import pytest
from calculator import is_adult

@pytest.mark.parametrize("age, expected", [
    (0, False),
    (17, False),
    (18, True),
    (65, True),
])
def test_is_adult(age, expected):
    assert is_adult(age) == expected
```

> [!warning] Decimal numbers in tests
> Because of [[02 Variables, Types and Expressions#Expressions - Calculations|how decimals work]], `0.1 + 0.2 == 0.3` is false! Use `pytest.approx` instead:

```python
import pytest

def test_decimal_addition():
    assert 0.1 + 0.2 == pytest.approx(0.3)
```

## Write Code That's Easy to Test

Some code is hard to test. Compare these two:

```python
def bad_grade():
    mark = int(input("Mark: "))
    if mark >= 50:
        print("Pass")
    else:
        print("Fail")
```

To test `bad_grade`, a test would have to fake typing and read the screen. Now split the **logic** from the **input and output**:

```python
def get_result(mark):
    if mark >= 50:
        return "Pass"
    return "Fail"

def test_mark_of_50_passes():
    assert get_result(50) == "Pass"

def test_mark_of_49_fails():
    assert get_result(49) == "Fail"
```

> [!important] The golden rule
> Put your logic in functions that take inputs and `return` results. Keep `input()`, `print()` and file work in a thin outer layer. These functions are simple to test. This one habit will improve all your code.

## Test Your Bug Fixes

When you find a bug, follow this order:

1. Write a test that **fails** because of the bug.
2. Fix the bug.
3. The test now **passes**, and it will catch the bug if it ever returns.

```python
def count_passing(marks):
    count = 0
    for mark in marks:
        if mark >= 50:
            count = count + 1
    return count

def test_first_mark_is_counted():
    assert count_passing([80, 40, 60]) == 2
```

This is the bug we found in [[10 Debugging#Step 3 and 4 - Guess, Then Test|the Debugging note]].

## Test Behaviour, Not Details

Test **what** a function gives back, not **how** it works inside. Then you can rewrite the inside, and your tests still pass, which means you can [[Software Engineering Roadmap#Refactoring|refactor]] safely.

## What's Coming Later

You don't need these yet, but it's good to know the names:

- [[Software Engineering Roadmap#Integration Testing|Integration tests]] check several parts working together.
- [[Software Engineering Roadmap#End-to-End Testing|End-to-end tests]] check the full app, as a user would use it.
- [[Software Engineering Roadmap#Test-Driven Development|Test-driven development]] means writing the test before the code.
- [[Software Engineering Roadmap#Mocking and Test Doubles|Mocks]] stand in for things like databases during tests.
- [[Software Engineering Roadmap#Code Coverage|Code coverage]] measures which lines your tests run.

## Common Mistakes

> [!warning] Watch out for these
> - File or function names that don't start with `test_`, so pytest doesn't find them
> - Testing only the normal case
> - Testing many unrelated things in a single test
> - Tests that depend on each other, or on the order they run in
> - Writing a test that can't fail
> - Deleting a failing test instead of understanding it

## Practice

**1. Test a function.** Here's a function. Write at least 5 tests, including edge cases.

```python
def shout(text):
    return text.upper() + "!"
```

> [!example]- Answer to question 1
> ```python
> def shout(text):
>     return text.upper() + "!"
>
> def test_normal_word():
>     assert shout("hello") == "HELLO!"
>
> def test_already_capitals():
>     assert shout("HI") == "HI!"
>
> def test_empty_string():
>     assert shout("") == "!"
>
> def test_with_spaces():
>     assert shout("a b") == "A B!"
>
> def test_with_numbers():
>     assert shout("abc1") == "ABC1!"
> ```

**2. Find the case.** This function has a bug. Write a test that fails, then fix the function.

```python
def largest(numbers):
    biggest = 0
    for number in numbers:
        if number > biggest:
            biggest = number
    return biggest
```

> [!example]- Answer to question 2
> If all numbers are negative, it returns 0, which isn't in the list! Testing `[-5, -2]` exposes this. The fix is to begin with the first number.
>
> ```python
> def largest(numbers):
>     biggest = numbers[0]
>     for number in numbers:
>         if number > biggest:
>             biggest = number
>     return biggest
>
> def test_all_negative_numbers():
>     assert largest([-5, -2]) == -2
> ```

**3. Boundaries.** A shop gives free delivery for orders of 1000 or more. What are the most important values to test?

> [!example]- Answer to question 3
> **999** (just below), **1000** (exactly at the limit) and **1001** (just above). Also test 0, and a negative amount if the function could ever get one.

**4. Test driven.** Write the tests first for `is_leap_year(year)`, then write the function. A year is a leap year if it can be divided by 4, except for years divisible by 100, unless they're also divisible by 400.

> [!example]- Answer to question 4
> ```python
> import pytest
>
> def is_leap_year(year):
>     if year % 400 == 0:
>         return True
>     if year % 100 == 0:
>         return False
>     return year % 4 == 0
>
> @pytest.mark.parametrize("year, expected", [
>     (2024, True),
>     (2023, False),
>     (1900, False),
>     (2000, True),
> ])
> def test_is_leap_year(year, expected):
>     assert is_leap_year(year) == expected
> ```

## Can You Do This?

- [ ] Write and run a test with pytest
- [ ] Use Arrange, Act, Assert
- [ ] Give tests clear names
- [ ] Test normal, boundary, empty and invalid cases
- [ ] Test that an error is raised
- [ ] Use `parametrize` for many cases
- [ ] Split logic from input and output, so it's easy to test
- [ ] Write a failing test for a bug before fixing it

## Summary

> [!success] Remember
> - A **unit test** runs a small piece of code and checks the result.
> - **Arrange, Act, Assert.** Clear names. One idea per test.
> - Bugs hide at **boundaries**, so test them.
> - Keep logic separate from `input` and `print`.

**Next:** [[12 Mathematical Logic]] →
