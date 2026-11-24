---
title: Linux Basics
tags: [phase-1, tools]
aliases: [Linux, Terminal, Command Line, Shell]
---

# Linux Basics

← [[07 Problem-Solving Techniques]] · [[Phase 1 - Start Here|Home]] · **Next:** [[09 Git Basics]] → · [[Software Engineering Roadmap#Linux|Roadmap card]]

> [!abstract] After this note, you can
> Use the terminal with confidence: move around, manage files, search text, and control programs.

## The Big Idea

Most of the world's servers run **Linux**. It's an operating system, like Windows or macOS. Cloud computers, phones running Android and most websites all rely on it.

On servers, there's usually no mouse and no windows. You work by typing commands into a **terminal**. Think of the difference between pointing at a menu in a restaurant and talking to the chef. Typing takes a little more skill, but you can ask for exactly what you want. And commands can be saved and repeated.

Some words you'll hear:

- **Terminal:** the window you type in.
- **Shell:** the program that understands your commands. The most common one is **Bash**.
- **Command:** an instruction you type, such as `ls`.

## Get a Linux Terminal

| Your computer | What to do |
|---|---|
| **Windows** | Install WSL. Open PowerShell **as administrator**, run `wsl --install`, then restart. It gives you a real Ubuntu Linux terminal. |
| **Mac** | Open the Terminal app. macOS is similar to Linux, and most commands here work the same way. |
| **Linux** | You're ready. Open the Terminal app. |

> [!tip] Git Bash is a quick start on Windows
> Git for Windows includes **Git Bash**. Most commands in this note work there, except `apt` and some permission features.

## How a Command Looks

```text
ls   -l   /home
│     │     └── argument: what to act on
│     └──────── option: changes how it behaves
└────────────── command: what to do
```

Spaces separate each part. Capital letters matter: `File.txt` and `file.txt` are different files.

## Moving Around

Files live in a tree of folders. Folders are also called **directories**. The top of the tree is `/`, called the **root**.

```text
/
├── home/
│   └── sam/        ← your home folder, also called ~
│       └── projects/
├── etc/            ← system settings
├── usr/            ← installed programs
├── var/            ← logs and changing data
└── tmp/            ← temporary files
```

| Command | Meaning | Example |
|---|---|---|
| `pwd` | Print the folder you're in | `pwd` |
| `ls` | List the files | `ls`, or `ls -la` for details and hidden files |
| `cd` | Change folder | `cd projects` |
| `cd ..` | Go up one folder | `cd ..` |
| `cd ~` | Go to your home folder | `cd ~` |

> [!tip] Two habits that save hours
> - Press **Tab** to finish a name for you. Type `cd pro`, then press Tab.
> - Press the **Up arrow** to get your last command back.

## Working with Files

Try this whole session yourself. It's safe, and it's all inside a new practice folder.

```bash
mkdir practice
cd practice
touch notes.txt
echo "hello" > notes.txt
echo "world" >> notes.txt
cat notes.txt
cp notes.txt backup.txt
mv backup.txt old.txt
ls
rm old.txt
```

Output of `cat notes.txt` and `ls`:

```text
hello
world
notes.txt  old.txt
```

| Command | What it does |
|---|---|
| `mkdir name` | Make a folder |
| `touch name` | Create an empty file |
| `echo text > file` | Write text to a file, **replacing** what's there |
| `echo text >> file` | **Add** text to the end of a file |
| `cat file` | Show the whole file |
| `less file` | Read a long file, one screen at a time. Press `q` to quit |
| `head file`, `tail file` | Show the first or last 10 lines |
| `cp a b` | Copy `a` to `b` |
| `mv a b` | Move or rename `a` to `b` |
| `rm file` | Delete a file |
| `rm -r folder` | Delete a folder and everything in it |

> [!danger] There is no recycle bin
> `rm` deletes files for good. Before using `rm -r`, run `pwd` and `ls` to check where you are. Don't run commands from the internet with `rm` or `sudo` until you understand them.

Do you notice `>`, the same symbol as in `"w"` mode in [[06 Files, Errors and Modules#File Modes|file modes]]? `>` is like `"w"`, and `>>` is like `"a"`.

## Getting Help

```bash
man ls
ls --help
```

`man` opens the manual for a command. Press `q` to leave. The [tldr pages](https://tldr.sh/) give simpler examples.

## Text Tools and Pipes

This is where the terminal becomes powerful. A **pipe**, written `|`, sends the output of one command into the next. Small tools join together, like pieces of a water pipe.

First, create a practice file:

```bash
printf "apple\nmango\napple\nbanana\napple\nmango\n" > fruits.txt
```

| Command          | What it does                      |
| ---------------- | --------------------------------- |
| `grep word file` | Show lines that contain a word    |
| `wc -l file`     | Count lines                       |
| `sort file`      | Sort lines                        |
| `uniq -c`        | Count repeated neighbouring lines |
| `cut`, `awk`     | Pick out columns                  |
| `sed`            | Find and replace text             |
| `jq`             | Read JSON data (install it first) |

> [!example] Example: which fruit is most common?
> Each step passes its result on to the next step.

```bash
sort fruits.txt | uniq -c | sort -rn
```

Output:

```text
      3 apple
      2 mango
      1 banana
```

Read it from left to right:

1. `sort` puts equal lines next to each other. `uniq` needs this.
2. `uniq -c` counts each group.
3. `sort -rn` sorts by number, biggest first.

More examples:

```bash
grep apple fruits.txt
grep -c apple fruits.txt
sed 's/apple/orange/' fruits.txt | head -n 2
```

Output:

```text
apple
apple
apple
3
orange
mango
```

> [!example] Example: real-life use
> Imagine a web server log with a million lines. This one line shows the 5 most common errors:
> ```bash
> grep "ERROR" app.log | sort | uniq -c | sort -rn | head -n 5
> ```

## Permissions - Who Can Do What

Linux controls who may read, change or run each file. Run `ls -l` to see this.

```text
-rwxr-xr--  1 sam staff  120 Jan 10 09:00 run.sh
```

```text
-   rwx   r-x   r--
│    │     │     └── everyone else: read only
│    │     └──────── the group: read and run
│    └────────────── the owner: read, write and run
└─────────────────── - means a file, d means a folder
```

| Letter | Number | Meaning |
|---|---|---|
| `r` | 4 | Read |
| `w` | 2 | Write (change) |
| `x` | 1 | Execute (run) |

Add the numbers together for each group. So `7 = 4 + 2 + 1 = rwx`, and `5 = 4 + 1 = r-x`.

```bash
chmod +x run.sh
chmod 644 notes.txt
```

- `chmod +x` makes a file runnable.
- `644` means the owner can read and write, and everyone else can only read.

**`sudo`** runs one command as the administrator, who is called **root**. Use it only when you need to, because root can break anything.

## Processes - Running Programs

A **process** is a program that's running. See more in [[Software Engineering Roadmap#Processes and Threads|Processes and Threads]] during Phase 3.

| Command | What it does |
|---|---|
| `ps aux` | List all processes |
| `top` | Live view of what's running. Press `q` to quit |
| `kill 1234` | Politely ask process 1234 to stop |
| `kill -9 1234` | Force it to stop (last resort) |
| Ctrl + C | Stop the program in your terminal |
| `command &` | Run a command in the background |

The number is the **process ID (PID)**. `kill` works by sending a **signal**, which is a small message to the process. Ctrl + C sends one as well.

## Installing Software

On Ubuntu, you install programs with a **package manager** called `apt`.

```bash
sudo apt update
sudo apt install tree
tree
```

`apt update` refreshes the list of available software. `apt install` installs a program.

## A Taste of Scripts and SSH

**A script** is a file of commands. Create `hello.sh`:

```bash
#!/bin/bash
echo "Hello from a script"
```

Then run it:

```bash
chmod +x hello.sh
./hello.sh
```

You'll go deeper with [[Software Engineering Roadmap#Shell Scripting|shell scripting]] later.

**SSH** lets you control another computer securely through a terminal. It's how engineers work on servers:

```bash
ssh sam@203.0.113.10
```

## Common Mistakes

> [!warning] Watch out for these
> - Spaces in file names. `cat my file.txt` looks for two files. Use `cat "my file.txt"`.
> - Using `>` when you meant `>>`, and erasing a file
> - Running `rm` in the wrong folder
> - Using `sudo` to "fix" errors you don't understand
> - Not reading the error message. `No such file or directory` usually means you're in the wrong folder, so run `pwd`.

## Practice

**1. Build a folder tree.** Using only the terminal, create `school/maths` and `school/science`. Put a file named `notes.txt` in each one.

> [!example]- Answer to question 1
> ```bash
> mkdir -p school/maths school/science
> touch school/maths/notes.txt school/science/notes.txt
> ls -R school
> ```
> `-p` creates any missing parent folders. `ls -R` lists everything inside.

**2. Count them.** Using `fruits.txt`, how many lines contain `mango`?

> [!example]- Answer to question 2
> ```bash
> grep -c mango fruits.txt
> ```
> The answer is 2.

**3. Permissions.** What does `chmod 750 run.sh` mean?

> [!example]- Answer to question 3
> - Owner: 7 = read, write and run
> - Group: 5 = read and run
> - Everyone else: 0 = nothing

**4. Game practice.** Play the first 10 levels of [OverTheWire Bandit](https://overthewire.org/wargames/bandit/). It teaches the terminal through a fun game.

## Can You Do This?

- [ ] Move around folders without using a mouse
- [ ] Create, copy, move and delete files safely
- [ ] Use `grep`, `sort`, `uniq` and pipes
- [ ] Read `-rwxr-xr--` and use `chmod`
- [ ] Find and stop a running process
- [ ] Install a package with `apt`
- [ ] Use `man` or `--help` to learn a new command

## Summary

> [!success] Remember
> - `pwd`, `ls`, `cd`: where am I, what's here, go there.
> - `|` joins small tools to do big jobs.
> - `rm` is permanent. Check before you run it.
> - Tab and the Up arrow save huge amounts of typing.

**Next:** [[09 Git Basics]] →
