---
title: Memory Management
tags: [phase-3, systems]
aliases: [Stack and Heap, Pointers, Memory Leaks, Garbage Collection]
---

# Memory Management

← [[37 Concurrency and Parallelism]] · [[Phase 3 - Start Here|Home]] · **Next:** [[39 Software Development Lifecycle]] → · [[Software Engineering Roadmap#Memory Management|Roadmap card]]

> [!abstract] After this note, you can
> Explain the stack and the heap, use pointers in C, recognise the classic memory bugs, and find a memory leak in both C and Python.

## The Big Idea

Every value in your program lives somewhere in memory. Someone must give out that space and take it back when it's no longer needed. Languages differ mainly in **who does the cleanup**: you, the compiler, or a garbage collector.

| Language | Who frees memory | Speed | Safety |
|---|---|---|---|
| **C** | You | Fastest | Your responsibility |
| **C++** | You, helped by smart pointers | Fastest | Better |
| **Rust** | The compiler, using ownership rules | Fastest | Very safe |
| **Java, Go, C#, Python, JavaScript** | A garbage collector | Slower, with some pauses | Very safe |

We'll use C to see what's really happening, as it hides nothing. To run C, install a compiler: on Ubuntu or WSL, run `sudo apt install gcc`. Or use [Compiler Explorer](https://godbolt.org/) in your browser.

```bash
gcc -Wall -g program.c -o program
./program
```

## Stack and Heap

![[process-memory.svg]]

| | Stack | Heap |
|---|---|---|
| **Used for** | Local variables, function calls | Data of any size, that must outlive a function |
| **Freed** | Automatically, when the function returns | By you, or by a garbage collector |
| **Speed** | Very fast | Slower |
| **Size** | Small, often a few MB | Large |
| **Like** | A stack of plates | A big warehouse |

The stack is the real [[18 Stacks, Queues and Deques#The Call Stack|call stack]] from Phase 2.

```c
#include <stdio.h>
#include <stdlib.h>

int add(int a, int b) {
    int sum = a + b;      /* sum is on the stack */
    return sum;           /* sum disappears when add returns */
}

int main(void) {
    int total = add(2, 3);                 /* on the stack */
    int *numbers = malloc(3 * sizeof(int)); /* on the heap */
    numbers[0] = total;
    printf("%d\n", numbers[0]);
    free(numbers);                         /* you must give it back */
    return 0;
}
```

Output:

```text
5
```

**Watch the stack during `add(2, 3)`:**

```text
Before the call      During add          After it returns

                     | add           |
                     |  a=2 b=3 sum=5|
| main           |   | main          |   | main          |
|  total=?       |   |  total=?      |   |  total=5      |
```

## Pointers

![[pointer.svg]]

A **pointer** is a variable that holds a memory address. You've been using this idea since [[05 Lists, Strings and Dictionaries#Values and References - The Big Surprise|references in Phase 1]]. C just makes it visible.

| Symbol | Meaning | Read it as |
|---|---|---|
| `&x` | The address of x | "address of" |
| `*p` | The value that p points to | "value at" |
| `int *p` | p is a pointer to an int | |

```c
#include <stdio.h>

void double_it(int *value) {
    *value = *value * 2;
}

int main(void) {
    int age = 20;
    int *p = &age;

    printf("%d\n", *p);
    *p = 21;
    printf("%d\n", age);

    double_it(&age);
    printf("%d\n", age);
    return 0;
}
```

Output:

```text
20
21
42
```

By passing an address, `double_it` can change the original. That's the same reason a Python function can change a list you pass in.

### Arrays and Pointer Math

In C, an array is simply items side by side, and its name acts like a pointer to the first item.

```c
#include <stdio.h>

int main(void) {
    int numbers[3] = {10, 20, 30};
    int *p = numbers;

    printf("%d\n", numbers[1]);
    printf("%d\n", *(p + 1));
    return 0;
}
```

Output:

```text
20
20
```

`p + 1` moves forward by one **int**, not one byte. That's the [[17 Arrays and Linked Lists#Arrays|address math from Phase 2]].

## Manual Memory - malloc and free

`malloc` asks for heap memory. `free` gives it back. Every `malloc` needs exactly one `free`.

> [!example] A linked list in C
> The roadmap's exercise. Compare it with [[17 Arrays and Linked Lists#Build One|your Python version]]. Here you must clean up, too.

```c
#include <stdio.h>
#include <stdlib.h>

struct Node {
    int value;
    struct Node *next;
};

struct Node *push(struct Node *head, int value) {
    struct Node *node = malloc(sizeof(struct Node));
    if (node == NULL) {
        exit(1);
    }
    node->value = value;
    node->next = head;
    return node;
}

void free_list(struct Node *head) {
    while (head != NULL) {
        struct Node *next = head->next;   /* Save this before freeing! */
        free(head);
        head = next;
    }
}

int main(void) {
    struct Node *head = NULL;
    head = push(head, 3);
    head = push(head, 2);
    head = push(head, 1);

    for (struct Node *n = head; n != NULL; n = n->next) {
        printf("%d ", n->value);
    }
    printf("\n");

    free_list(head);
    return 0;
}
```

Output:

```text
1 2 3 
```

## The Classic Memory Bugs

![[memory-bugs.svg]]

### Memory Leak

You allocate memory, and never free it. The program slowly uses more and more, until it crashes.

```c
#include <stdlib.h>

void leaky(void) {
    int *data = malloc(1000 * sizeof(int));
    data[0] = 1;
    /* Forgot free(data). The address is lost when we return. */
}

int main(void) {
    leaky();
    return 0;
}
```

### Dangling Pointer

You keep using a pointer after the memory has been freed. This is called **use after free**.

```c
#include <stdlib.h>
#include <stdio.h>

int main(void) {
    int *data = malloc(sizeof(int));
    *data = 42;
    free(data);
    /* printf("%d\n", *data);   BUG: data is now dangling */
    data = NULL;                /* A good habit */
    printf("%s\n", data == NULL ? "safe" : "unsafe");
    return 0;
}
```

Output:

```text
safe
```

### Buffer Overflow

You write past the end of a block. This overwrites neighbouring data, and it has caused some of history's worst security holes.

```c
#include <stdio.h>
#include <string.h>

int main(void) {
    char name[8];
    /* strcpy(name, "a very long name");   BUG: writes past the end */
    strncpy(name, "a very long name", sizeof(name) - 1);
    name[sizeof(name) - 1] = '\0';
    printf("%s\n", name);
    return 0;
}
```

Output:

```text
a very 
```

Python, Java and Rust check array bounds for you, so this class of bug largely disappears.

### Finding Bugs with Tools

Don't hunt for these by eye. Use a tool.

```bash
gcc -g -fsanitize=address buggy.c -o buggy
./buggy
```

| Tool | Finds |
|---|---|
| **AddressSanitizer** (`-fsanitize=address`) | Overflows, use after free, leaks |
| **Valgrind** (`valgrind ./program`) | Leaks and invalid reads |

Example report from AddressSanitizer:

```text
==1234==ERROR: AddressSanitizer: heap-use-after-free on address 0x602000000010
READ of size 4 at 0x602000000010 thread T0
    #0 0x401234 in main buggy.c:7
```

It names the bug, and the exact line. Now you can apply your [[10 Debugging|debugging method]].

## Automatic Memory

### Reference Counting

Each object counts how many references point to it. At zero, it's freed right away. Python uses this as its main method.

```python
import sys

data = [1, 2, 3]
print(sys.getrefcount(data) >= 2)

other = data
print(sys.getrefcount(data) >= 3)

del other
print(data)
```

Output:

```text
True
True
[1, 2, 3]
```

**Its weakness is cycles.** If A points to B, and B points to A, both counts stay above zero forever, even when nothing else can reach them.

### Garbage Collection

![[gc-reachable.svg]]

A **garbage collector (GC)** starts from the **roots**, such as your variables, and marks everything it can reach. Whatever isn't marked is garbage, even if objects form a cycle. This is **mark and sweep**.

```python
import gc

class Node:
    def __init__(self):
        self.other = None

a = Node()
b = Node()
a.other = b
b.other = a       # A cycle

del a
del b             # Now nothing can reach them

print(gc.collect() >= 2)
```

Output:

```text
True
```

Most programs create many short-lived objects, so modern collectors are **generational**: they check new objects often and old ones rarely.

| Trade-off | What it costs |
|---|---|
| **Pauses** | The program may briefly stop while collecting |
| **Memory** | Needs extra room to work well |
| **CPU** | The collector itself uses time |

There's more on the [[Software Engineering Roadmap#Garbage Collection|Garbage Collection card]].

### You Can Still Leak in Python

Garbage collectors only free what's **unreachable**. If you keep a reference, it stays forever. The classic leak is a cache or list that only ever grows.

```python
import tracemalloc

cache = []

def handle_request(data):
    cache.append(data * 1000)     # Bug: never removed

tracemalloc.start()
before = tracemalloc.get_traced_memory()[0]

for i in range(1000):
    handle_request("x")

after = tracemalloc.get_traced_memory()[0]
top = tracemalloc.take_snapshot().statistics("lineno")[0]
tracemalloc.stop()

print(f"Memory grew: {after > before}")
print(f"Biggest source is line {top.traceback[0].lineno}")
```

Output:

```text
Memory grew: True
Biggest source is line 6
```

`tracemalloc` even finds the line. **The fix:** set a limit, or use an [[19 Hash Tables#Build an LRU Cache|LRU cache]].

**Other common leaks in managed languages:**

- Event listeners that are never removed
- Global dictionaries that keep growing
- Files or connections that aren't closed

## Other Approaches

### RAII - Cleanup Tied to Scope

In C++, an object's cleanup runs automatically when it goes out of scope. **Smart pointers** use this idea, so you rarely need to call `delete` yourself.

```cpp
#include <memory>

void example() {
    auto data = std::make_unique<int>(42);
    // Freed automatically when this function ends.
}
```

Python's `with` follows the same spirit, for files and locks.

### Ownership in Rust

Rust's compiler tracks who **owns** each value. When the owner goes out of scope, the value is freed. There's no GC, and no manual `free`, and many memory bugs simply won't compile.

```rust
fn main() {
    let name = String::from("Sam");
    let other = name;       // Ownership moves to other
    // println!("{}", name);   Compile error: name was moved
    println!("{}", other);
}
```

### Custom Allocators

`malloc` is general, so it can be slow. Programs that need speed, such as games and databases, often use a **memory pool** or **arena**: grab one big block, hand out pieces, and free it all at once.

```python
class Arena:
    def __init__(self, size):
        self.memory = bytearray(size)
        self.used = 0

    def allocate(self, size):
        if self.used + size > len(self.memory):
            raise MemoryError("Arena is full")
        start = self.used
        self.used = self.used + size
        return start

    def reset(self):
        self.used = 0

arena = Arena(100)
print(arena.allocate(40))
print(arena.allocate(40))

try:
    arena.allocate(40)
except MemoryError as error:
    print(error)

arena.reset()
print(arena.allocate(40))
```

Output:

```text
0
40
Arena is full
0
```

## Common Mistakes

> [!warning] Watch out for these
> - Forgetting `free`, or calling it twice
> - Returning a pointer to a local variable, which dies when the function returns
> - Off-by-one errors when writing to arrays
> - Assuming a garbage collector means no leaks
> - Not running a sanitizer on C or C++ code

## Practice

**1. Where does it live?** Stack or heap?
- `int x = 5;` inside a function
- `malloc(100)`
- A Python list

> [!example]- Answer to question 1
> Stack, heap, heap. (In Python, nearly all objects live on the heap.)

**2. Spot the bug.**

```c
int *make_number(void) {
    int x = 5;
    return &x;
}
```

> [!example]- Answer to question 2
> `x` lives on the stack, and it disappears when the function returns. The returned pointer dangles. Use `malloc`, or return the value itself.

**3. Predict.** What does this print?

```c
#include <stdio.h>

int main(void) {
    int a = 1;
    int *p = &a;
    *p = *p + 1;
    a = a + 1;
    printf("%d\n", *p);
    return 0;
}
```

> [!example]- Answer to question 3
> `3`. Both names refer to the same memory, so both additions count.

**4. Fix the leak.** Change the Python cache so it holds at most 100 items.

> [!example]- Answer to question 4
> ```python
> from collections import deque
>
> cache = deque(maxlen=100)
>
> for i in range(1000):
>     cache.append(i)
>
> print(len(cache))
> ```
>
> Output:
>
> ```text
> 100
> ```

**5. Build.** Write a linked list in C with add, print and free. Run it under AddressSanitizer or Valgrind until it's completely clean.

## Can You Do This?

- [ ] Draw a program's memory layout
- [ ] Explain the stack and the heap
- [ ] Use `&` and `*` in C
- [ ] Use `malloc` and `free` correctly
- [ ] Explain leaks, dangling pointers and buffer overflows
- [ ] Use a sanitizer or Valgrind
- [ ] Explain reference counting and mark and sweep
- [ ] Find a leak in Python with `tracemalloc`

## Summary

> [!success] Remember
> - **Stack:** automatic and fast. **Heap:** flexible and long-lived.
> - A **pointer** is an address.
> - Every `malloc` needs one `free`.
> - **A GC frees only what's unreachable**, so leaks are still possible.

**Next:** [[39 Software Development Lifecycle]] →
