---
title: Phase 1 - Start Here
tags: [software-engineering, phase-1, index]
aliases: [Phase 1, Phase 1 Home]
---

# Phase 1 - Learn to Program

> [!abstract] Your goal
> By the end of Phase 1, you can write small programs without following a tutorial. You can save your work with Git, find your own bugs, test your code, and use the terminal. That is enough to be ready for an internship.

This is Phase 1 of the [[Software Engineering Roadmap#Recommended Study Order|Software Engineering Roadmap]]. You do not need any experience. If you can use a computer and you are willing to practice, you can do this.

## The Notes, in Order

Study them from top to bottom. Each note uses ideas from the notes before it.

### Part 1 - Learn to think like a programmer
- [ ] [[01 Computational Thinking]]: break big problems into small steps

### Part 2 - Learn to write code
- [ ] [[02 Variables, Types and Expressions]]: store and calculate data
- [ ] [[03 Conditions and Loops]]: make choices and repeat work
- [ ] [[04 Functions and Scope]]: build reusable pieces of code
- [ ] [[05 Lists, Strings and Dictionaries]]: work with groups of data
- [ ] [[06 Files, Errors and Modules]]: save data, handle problems, use other people's code

### Part 3 - Learn to solve problems
- [ ] [[07 Problem-Solving Techniques]]: what to do when you are stuck

### Part 4 - Learn the tools of the job
- [ ] [[08 Linux Basics]]: use the terminal
- [ ] [[09 Git Basics]]: save and share your work
- [ ] [[10 Debugging]]: find and fix bugs
- [ ] [[11 Unit Testing Basics]]: prove that your code works

### Part 5 - Learn the logic underneath
- [ ] [[12 Mathematical Logic]]: reason correctly about code
- [ ] [[13 Boolean Algebra]]: understand true, false and bits

### Part 6 - Put it all together
- [ ] [[14 Phase 1 Final Project]]: build a real to-do app

**Need a word explained?** See the [[Phase 1 Glossary]].

## How to Study These Notes

> [!important] The most important rule
> **Type the code yourself. Do not copy and paste.** You learn programming with your hands, the way you learn to ride a bike. Reading about it is not enough.

Follow this loop for each note:

1. **Read** one section slowly.
2. **Type** the example and run it.
3. **Change** something and guess what will happen. Then run it and see. This is where most of the learning happens.
4. **Try** the practice questions. Open the hidden answers only after you have really tried.
5. **Check** yourself with the "Can you do this?" list at the end of the note.

Getting stuck is normal. It does not mean you are bad at this. Every programmer in the world gets stuck every day. Stuck is what learning feels like.

## Set Up Your Computer

You need four free tools. Install them once.

| Tool | What it does | Where to get it |
|---|---|---|
| **Python** | Runs your code | [python.org/downloads](https://www.python.org/downloads/) |
| **VS Code** | The app you write code in | [code.visualstudio.com](https://code.visualstudio.com/) |
| **Git** | Saves versions of your work | [git-scm.com/downloads](https://git-scm.com/downloads) |
| **WSL** (Windows only) | Gives you a real Linux terminal | See [[08 Linux Basics#Get a Linux Terminal]] |

> [!tip] Installing Python on Windows
> On the first screen of the installer, tick **Add python.exe to PATH**. This lets you run Python from any folder.

**Check that it worked.** Open a terminal and type:

```bash
python --version
git --version
```

You should see a version number for each one. On Mac and Linux, you may need to type `python3` instead of `python`.

### Your First Program

1. Make a folder called `learning`, and open it in VS Code.
2. Create a file called `hello.py`.
3. Type this line:

```python
print("Hello, world!")
```

4. Open the terminal in VS Code (**Terminal → New Terminal**) and run:

```bash
python hello.py
```

You should see:

```text
Hello, world!
```

You just wrote a program. 

> [!note] Why Python?
> Python reads almost like English, so you can focus on ideas instead of symbols. The ideas in these notes work in nearly every language. Later, you will learn a second language much faster than the first one.

## A Simple Study Plan

This plan assumes about 8 to 10 hours each week. Go slower if you need to. Understanding matters more than speed.

| Week | Study |
|---|---|
| 1 | [[01 Computational Thinking]], setup, [[02 Variables, Types and Expressions]] |
| 2 to 3 | [[03 Conditions and Loops]] |
| 4 to 5 | [[04 Functions and Scope]] |
| 6 to 7 | [[05 Lists, Strings and Dictionaries]] |
| 8 | [[06 Files, Errors and Modules]] |
| 9 | [[07 Problem-Solving Techniques]] |
| 10 | [[08 Linux Basics]] |
| 11 | [[09 Git Basics]] |
| 12 | [[10 Debugging]] |
| 13 | [[11 Unit Testing Basics]] |
| 14 | [[12 Mathematical Logic]], [[13 Boolean Algebra]] |
| 15 to 16 | [[14 Phase 1 Final Project]] |

## How to Mark Your Progress

These notes use the same four boxes as the roadmap. Type the letter or symbol between the brackets.

| You type | It means |
|---|---|
| `- [ ]` | Not started |
| `- [/]` | Currently learning |
| `- [p]` | Practiced |
| `- [x]` | Mastered: you pass the "Can you do this?" list |

## When You Finish Phase 1

Check yourself with the roadmap's [[Software Engineering Roadmap#Ready for an Internship|Ready for an Internship]] list. Then move on to [[Phase 2 - Start Here|Phase 2]].

**Start here:** [[01 Computational Thinking]] →
