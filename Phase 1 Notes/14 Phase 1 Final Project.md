---
title: Phase 1 Final Project
tags: [phase-1, project]
aliases: [Final Project, To-Do App]
---

# Phase 1 Final Project - A To-Do App

← [[13 Boolean Algebra]] · [[Phase 1 - Start Here|Home]]

> [!abstract] Your goal
> Build a real command-line to-do app, using everything from Phase 1. Save it on GitHub, with tests and a README. This is the first piece of your portfolio.

## What You'll Build

A program that you control from the terminal. It remembers your tasks in a file.

```text
$ python todo.py add "Buy milk"
Added: Buy milk

$ python todo.py add "Call mom"
Added: Call mom

$ python todo.py done 1
Completed task 1

$ python todo.py list
1. [x] Buy milk
2. [ ] Call mom

$ python todo.py delete 1
Deleted task 1
```

> [!important] Try it yourself first
> Do the steps below **without** opening the solution. Struggling here is the most valuable part of Phase 1. Open a hint only when you've been stuck for at least 20 minutes.

## What You'll Use

| Skill | Where you learned it |
|---|---|
| Planning the steps | [[01 Computational Thinking]] |
| Lists and dictionaries | [[05 Lists, Strings and Dictionaries]] |
| Saving to a file with JSON | [[06 Files, Errors and Modules]] |
| Handling bad input | [[06 Files, Errors and Modules]] |
| Solving problems when stuck | [[07 Problem-Solving Techniques]] |
| Using the terminal | [[08 Linux Basics]] |
| Saving your progress | [[09 Git Basics]] |
| Fixing bugs | [[10 Debugging]] |
| Writing tests | [[11 Unit Testing Basics]] |

## Step by Step

### Step 1 - Set Up the Project

- [ ] Make a folder called `todo-app`, and open it in VS Code.
- [ ] Run `git init`.
- [ ] Create `todo.py`, `test_todo.py`, `README.md` and `.gitignore`.
- [ ] Add `tasks.json`, `.venv/` and `__pycache__/` to `.gitignore`.
- [ ] Make your first commit.

### Step 2 - Plan Before You Code

Use [[01 Computational Thinking#3. Abstraction|abstraction]]. What does a task need? Here's a good model:

```python
task = {"title": "Buy milk", "done": False}
tasks = [task]
print(tasks)
```

Output:

```text
[{'title': 'Buy milk', 'done': False}]
```

Next, use [[01 Computational Thinking#1. Decomposition|decomposition]]. Which functions do you need?

- `load_tasks` and `save_tasks`: work with the file
- `add_task`, `complete_task`, `delete_task`: change the list
- `format_tasks`: turn the list into text
- `main`: read the command and call the right function

> [!tip] Follow the golden rule
> Keep your logic away from `print` and files, as you learned in [[11 Unit Testing Basics#Write Code That's Easy to Test]]. `add_task` should only change the list. That makes it simple to test.

### Step 3 - Build the Logic, with Tests

Build one function at a time. For each one:

1. Write the function and its tests.
2. Run `python -m pytest`.
3. Commit when the tests pass.

Suggested order:

- [ ] `add_task`. Test: does the list grow? Does an empty title raise an error?
- [ ] `complete_task`. Test: is the task marked done? Does an invalid number raise an error?
- [ ] `delete_task`. Test: is the right task removed?
- [ ] `format_tasks`. Test: what about an empty list?

> [!example]- Hint: task numbers
> Users count from 1, but list indexes start at 0. So task number 1 is at `tasks[0]`. You'll need `number - 1`. Also check the boundaries: 0, and a number bigger than the list.

### Step 4 - Save and Load

- [ ] Write `load_tasks` and `save_tasks` with `json`.
- [ ] Handle `FileNotFoundError`, because the file doesn't exist the first time.

> [!example]- Hint: testing with files
> pytest gives you a built-in temporary folder, called `tmp_path`, so your tests don't touch your real tasks. Just add `tmp_path` as a parameter to your test function.

### Step 5 - Read Commands from the Terminal

The words you type after `python todo.py` are stored in `sys.argv`, which is a list of strings:

```python
# file: show_args.py
import sys

print(sys.argv)
```

If you ran `python show_args.py add "Buy milk"`, it would print:

```text
['show_args.py', 'add', 'Buy milk']
```

- [ ] Write `main` to read the command, and call the right function.
- [ ] Print a helpful message for an unknown command.
- [ ] Handle bad input such as `python todo.py done abc`. It must **not** crash.

### Step 6 - Write the README

Your `README.md` is what visitors see first. Include these:

- [ ] What the project does, in one sentence
- [ ] How to run it, with example commands
- [ ] How to run the tests
- [ ] What you learned

### Step 7 - Share It

- [ ] Create a GitHub repo, and push your code. See [[09 Git Basics#Remotes and GitHub]].
- [ ] Ask a friend to follow your README. Fix anything that confuses them.

## Check Your Work

Your project is finished when you can tick all of these:

- [ ] All commands work: `add`, `list`, `done`, `delete`
- [ ] Tasks are still there after you restart the program
- [ ] No input can crash it: try `done 99`, `done abc`, and no command at all
- [ ] You have at least 8 tests, and all of them pass
- [ ] You have at least 10 small commits with clear messages
- [ ] Your code is on GitHub, with a README
- [ ] You can explain every line without notes

## One Possible Solution

> [!warning] Finish your own version first
> There's no single right answer, and yours may be better. Compare them only when you're done, or when you're truly stuck. Then close this, and write it again yourself.

> [!example]- Solution: todo.py
> ```python
> # file: todo.py
> """A small to-do app that saves its tasks in a file."""
> import json
> import sys
>
> TASKS_FILE = "tasks.json"
>
>
> def load_tasks(path=TASKS_FILE):
>     try:
>         with open(path) as file:
>             return json.load(file)
>     except FileNotFoundError:
>         return []
>
>
> def save_tasks(tasks, path=TASKS_FILE):
>     with open(path, "w") as file:
>         json.dump(tasks, file, indent=2)
>
>
> def add_task(tasks, title):
>     title = title.strip()
>     if not title:
>         raise ValueError("A task needs a title")
>     tasks.append({"title": title, "done": False})
>
>
> def check_number(tasks, number):
>     if number < 1 or number > len(tasks):
>         raise ValueError(f"There is no task number {number}")
>
>
> def complete_task(tasks, number):
>     check_number(tasks, number)
>     tasks[number - 1]["done"] = True
>
>
> def delete_task(tasks, number):
>     check_number(tasks, number)
>     tasks.pop(number - 1)
>
>
> def format_tasks(tasks):
>     if not tasks:
>         return "No tasks yet."
>     lines = []
>     for number, task in enumerate(tasks, start=1):
>         mark = "x" if task["done"] else " "
>         lines.append(f"{number}. [{mark}] {task['title']}")
>     return "\n".join(lines)
>
>
> def read_number(text):
>     try:
>         return int(text)
>     except ValueError:
>         raise ValueError(f"'{text}' is not a task number")
>
>
> def run_command(args):
>     """Run one command and return the message to show the user."""
>     if not args:
>         return "Usage: todo.py add TITLE | list | done NUMBER | delete NUMBER"
>
>     command = args[0]
>     tasks = load_tasks()
>
>     if command == "list":
>         return format_tasks(tasks)
>
>     if len(args) != 2:
>         raise ValueError(f"'{command}' needs exactly one value")
>
>     if command == "add":
>         add_task(tasks, args[1])
>         message = f"Added: {args[1]}"
>     elif command == "done":
>         number = read_number(args[1])
>         complete_task(tasks, number)
>         message = f"Completed task {number}"
>     elif command == "delete":
>         number = read_number(args[1])
>         delete_task(tasks, number)
>         message = f"Deleted task {number}"
>     else:
>         raise ValueError(f"Unknown command: {command}")
>
>     save_tasks(tasks)
>     return message
>
>
> def main():
>     try:
>         print(run_command(sys.argv[1:]))
>     except ValueError as error:
>         print(f"Error: {error}")
>         sys.exit(1)
>
>
> if __name__ == "__main__":
>     main()
> ```

> [!example]- Solution: test_todo.py
> ```python
> # file: test_todo.py
> import pytest
>
> from todo import (
>     add_task,
>     complete_task,
>     delete_task,
>     format_tasks,
>     load_tasks,
>     save_tasks,
> )
>
>
> def test_add_task_adds_to_list():
>     tasks = []
>     add_task(tasks, "Buy milk")
>     assert tasks == [{"title": "Buy milk", "done": False}]
>
>
> def test_add_task_rejects_empty_title():
>     with pytest.raises(ValueError):
>         add_task([], "   ")
>
>
> def test_complete_task_marks_it_done():
>     tasks = [{"title": "A", "done": False}]
>     complete_task(tasks, 1)
>     assert tasks[0]["done"] is True
>
>
> @pytest.mark.parametrize("number", [0, 2, -1])
> def test_complete_task_rejects_bad_numbers(number):
>     tasks = [{"title": "A", "done": False}]
>     with pytest.raises(ValueError):
>         complete_task(tasks, number)
>
>
> def test_delete_task_removes_the_right_one():
>     tasks = [{"title": "A", "done": False}, {"title": "B", "done": False}]
>     delete_task(tasks, 1)
>     assert tasks == [{"title": "B", "done": False}]
>
>
> def test_format_tasks_with_no_tasks():
>     assert format_tasks([]) == "No tasks yet."
>
>
> def test_format_tasks_shows_status():
>     tasks = [{"title": "A", "done": True}, {"title": "B", "done": False}]
>     assert format_tasks(tasks) == "1. [x] A\n2. [ ] B"
>
>
> def test_save_then_load_gives_same_tasks(tmp_path):
>     path = tmp_path / "tasks.json"
>     tasks = [{"title": "A", "done": False}]
>     save_tasks(tasks, path)
>     assert load_tasks(path) == tasks
>
>
> def test_load_missing_file_gives_empty_list(tmp_path):
>     assert load_tasks(tmp_path / "missing.json") == []
> ```

## Make It Your Own

Finished? Add a feature. Each of these teaches something new:

- [ ] **Priorities:** high, medium or low, with sorted output
- [ ] **Due dates:** use `datetime`, and show overdue tasks
- [ ] **Search:** `python todo.py search milk`
- [ ] **Edit:** change the title of a task
- [ ] **Better commands:** rewrite `main` using Python's [argparse](https://docs.python.org/3/library/argparse.html) module

## Congratulations

> [!success] You've finished Phase 1
> You can think through a problem, write a working program, save it with Git, find your own bugs, and prove your code works with tests. Many people never get this far. Be proud of yourself.

Now check the [[Software Engineering Roadmap#Ready for an Internship|Ready for an Internship]] list. Then move on to [[Phase 2 - Start Here|Phase 2]], where you'll learn data structures and algorithms.

**Back to:** [[Phase 1 - Start Here]]
