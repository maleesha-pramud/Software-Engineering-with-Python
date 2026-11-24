---
title: Transactions and ACID
tags: [phase-3, databases]
aliases: [Transactions, ACID, Commit, Rollback, Write-Ahead Log]
---

# Transactions and ACID

← [[51 Indexing]] · [[Phase 3 - Start Here|Home]] · **Next:** [[53 Phase 3 Final Project]] → · [[Software Engineering Roadmap#Transactions and ACID|Roadmap card]]

> [!abstract] After this note, you can
> Group database changes so that they're safe under crashes and heavy traffic, avoid lost updates, and handle deadlocks.

## The Big Idea

Moving money takes two steps: take it from one account, and add it to another. What if the server dies between them? The money vanishes. What if two requests run at once? It could be spent twice.

A **transaction** makes a group of changes act as **one step**. You met this idea in [[29 Relational Databases#Transactions|Phase 2]]. Now let's make it work when things go badly.

## ACID

![[acid.svg]]

| Letter | Promise | Without it |
|---|---|---|
| **Atomicity** | All or nothing | Half a transfer |
| **Consistency** | Rules are never broken | A negative balance, despite a rule against it |
| **Isolation** | Transactions don't interfere | Two people buy one ticket |
| **Durability** | Committed data survives a crash | "Order confirmed", then it's gone |

## Commit and Rollback

| Command | Meaning |
|---|---|
| `BEGIN` | Start |
| `COMMIT` | Make it all permanent |
| `ROLLBACK` | Undo it all |

```python
import sqlite3

db = sqlite3.connect(":memory:", isolation_level=None)
db.execute("CREATE TABLE accounts (name TEXT PRIMARY KEY, balance INTEGER NOT NULL)")
db.execute("INSERT INTO accounts VALUES ('Sam', 100), ('Priya', 50)")

def balances():
    return db.execute("SELECT balance FROM accounts ORDER BY name").fetchall()

db.execute("BEGIN")
db.execute("UPDATE accounts SET balance = balance - 30 WHERE name = 'Sam'")
db.execute("ROLLBACK")
print(balances())

db.execute("BEGIN")
db.execute("UPDATE accounts SET balance = balance - 30 WHERE name = 'Sam'")
db.execute("UPDATE accounts SET balance = balance + 30 WHERE name = 'Priya'")
db.execute("COMMIT")
print(balances())
```

Output:

```text
[(50,), (100,)]
[(80,), (70,)]
```

### In Application Code

Python's `with db:` commits on success, and rolls back on any error. Most languages have something similar.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE accounts (name TEXT PRIMARY KEY, balance INTEGER NOT NULL CHECK (balance >= 0))")
db.execute("INSERT INTO accounts VALUES ('Sam', 100), ('Priya', 50)")
db.commit()

def transfer(sender, receiver, amount):
    with db:
        db.execute("UPDATE accounts SET balance = balance + ? WHERE name = ?", (amount, receiver))
        db.execute("UPDATE accounts SET balance = balance - ? WHERE name = ?", (amount, sender))

transfer("Sam", "Priya", 30)
print(db.execute("SELECT * FROM accounts ORDER BY name").fetchall())

try:
    transfer("Sam", "Priya", 1000)
except sqlite3.IntegrityError as error:
    print("Failed:", error)

print(db.execute("SELECT * FROM accounts ORDER BY name").fetchall())
```

Output:

```text
[('Priya', 80), ('Sam', 70)]
Failed: CHECK constraint failed: balance >= 0
[('Priya', 80), ('Sam', 70)]
```

Priya's addition happened first, and it was undone automatically.

## Where Should a Transaction Begin and End?

![[tx-boundaries.svg]]

A transaction holds locks, and others may have to wait.

> [!important] Rules for boundaries
> - **Keep them short.** Only database work belongs inside.
> - **No slow calls inside:** no web requests, emails or user input.
> - **Include everything that must succeed together.**
> - **Let the service layer decide,** because it knows what counts as one business action.

> [!warning] The classic mistake: emails and payments
> A database can't undo an email. If you send one inside a transaction and then it rolls back, the customer got a false confirmation. Commit first, and then send. To be reliable, save a "to send" row **in the same transaction**, and let a background worker send it. This is called the **outbox pattern**.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.executescript("""
    CREATE TABLE orders (id INTEGER PRIMARY KEY, total INTEGER);
    CREATE TABLE outbox (id INTEGER PRIMARY KEY, message TEXT, sent INTEGER DEFAULT 0);
""")

def place_order(total):
    with db:
        cursor = db.execute("INSERT INTO orders (total) VALUES (?)", (total,))
        db.execute("INSERT INTO outbox (message) VALUES (?)", (f"Receipt for order {cursor.lastrowid}",))

def send_pending():
    rows = db.execute("SELECT id, message FROM outbox WHERE sent = 0").fetchall()
    for row_id, message in rows:
        print("Sending:", message)
        with db:
            db.execute("UPDATE outbox SET sent = 1 WHERE id = ?", (row_id,))

place_order(500)
send_pending()
send_pending()
```

Output:

```text
Sending: Receipt for order 1
```

The second call sends nothing. Either both the order and its message exist, or neither does.

## Concurrency - Lost Updates

![[lost-update.svg]]

The [[37 Concurrency and Parallelism#Race Conditions|race conditions]] you met with threads also happen with database rows, even across different servers. Python locks can't help you here.

```python
import sqlite3
import threading

db_file = "tickets.db"
setup = sqlite3.connect(db_file)
setup.execute("DROP TABLE IF EXISTS tickets")
setup.execute("CREATE TABLE tickets (id INTEGER PRIMARY KEY, available INTEGER)")
setup.execute("INSERT INTO tickets VALUES (1, 1)")
setup.commit()
setup.close()

barrier = threading.Barrier(2)
sold = []

def buy_badly():
    db = sqlite3.connect(db_file, timeout=5)
    available = db.execute("SELECT available FROM tickets WHERE id = 1").fetchone()[0]
    barrier.wait()                # Both have read "1 available"
    if available > 0:
        db.execute("UPDATE tickets SET available = ? WHERE id = 1", (available - 1,))
        db.commit()
        sold.append(True)
    db.close()

threads = [threading.Thread(target=buy_badly) for _ in range(2)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join()

print("Tickets sold:", len(sold))
```

Output:

```text
Tickets sold: 2
```

There was only one ticket!

### Fix 1 - Make the Update Atomic

Let the database do the check and the change in **one statement**. This is often the simplest and best fix.

```python
import sqlite3
import threading

db_file = "tickets2.db"
setup = sqlite3.connect(db_file)
setup.execute("DROP TABLE IF EXISTS tickets")
setup.execute("CREATE TABLE tickets (id INTEGER PRIMARY KEY, available INTEGER)")
setup.execute("INSERT INTO tickets VALUES (1, 1)")
setup.commit()
setup.close()

results = []

def buy():
    db = sqlite3.connect(db_file, timeout=5)
    with db:
        cursor = db.execute("UPDATE tickets SET available = available - 1 WHERE id = 1 AND available > 0")
        results.append(cursor.rowcount == 1)
    db.close()

threads = [threading.Thread(target=buy) for _ in range(5)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join()

print("Tickets sold:", sum(results))
```

Output:

```text
Tickets sold: 1
```

`rowcount` tells you whether your update won.

### Fix 2 - Optimistic Locking

Add a version number. When you save, check that nobody changed it since you read. It's like editing a shared document: if it changed under you, you find out instead of overwriting. It's great when conflicts are rare.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE docs (id INTEGER PRIMARY KEY, body TEXT, version INTEGER)")
db.execute("INSERT INTO docs VALUES (1, 'draft', 1)")

def save(new_body, version_read):
    cursor = db.execute(
        "UPDATE docs SET body = ?, version = version + 1 WHERE id = 1 AND version = ?",
        (new_body, version_read),
    )
    return cursor.rowcount == 1

print(save("Sam's edit", 1))
print(save("Priya's edit", 1))
print(db.execute("SELECT body, version FROM docs").fetchone())
```

Output:

```text
True
False
("Sam's edit", 2)
```

Priya's save failed safely, so the app can show her the newer version.

### Fix 3 - Pessimistic Locking

Lock the row when you read it, so that others must wait. It's good when conflicts are common. PostgreSQL and MySQL have this; SQLite locks the whole database instead.

```sql
BEGIN;
SELECT available FROM tickets WHERE id = 1 FOR UPDATE;
UPDATE tickets SET available = available - 1 WHERE id = 1;
COMMIT;
```

| Approach | Good when | Cost |
|---|---|---|
| **Atomic update** | One simple change | None. Use it if you can |
| **Optimistic** | Conflicts are rare | Must handle retries |
| **Pessimistic** | Conflicts are common | Others wait |

### Unique Constraints Fix Races Too

"Check whether the username is free, then insert" is a race. Let a `UNIQUE` rule decide, and handle the error.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE users (name TEXT UNIQUE)")

def sign_up(name):
    try:
        with db:
            db.execute("INSERT INTO users VALUES (?)", (name,))
        return "Welcome!"
    except sqlite3.IntegrityError:
        return "That name is taken"

print(sign_up("sam"))
print(sign_up("sam"))
```

Output:

```text
Welcome!
That name is taken
```

## Isolation Levels

Stronger isolation is safer, but slower. Databases let you choose.

```mermaid
flowchart LR
    A[Read uncommitted] --> B[Read committed] --> C[Repeatable read] --> D[Serializable]
```

| Level | Sees | Can still go wrong |
|---|---|---|
| **Read uncommitted** | Others' unfinished changes | Dirty reads |
| **Read committed** | Only committed data | Different results if you read twice |
| **Repeatable read** | A stable view of rows | Phantom rows, in some databases |
| **Serializable** | As if transactions ran one at a time | Nothing, but expect retries |

| Problem | What happens |
|---|---|
| **Dirty read** | You read a change that later rolls back |
| **Non-repeatable read** | You read a row twice and get two values |
| **Phantom** | A repeated query finds new rows |
| **Write skew** | Two transactions check a rule, each writes something different, and together they break it |

> [!example] Write skew
> A hospital needs at least one doctor on call. Two are on call. Each checks, and sees "2 on call, so I can leave". Both leave at once, and nobody is left. Neither changed the same row, so a simple lock doesn't catch it. **Serializable** does.

PostgreSQL defaults to read committed. **Check your own database's default.** More is on the [[Software Engineering Roadmap#Isolation Levels|Isolation Levels card]].

## Deadlocks and Retries

The [[37 Concurrency and Parallelism#Deadlocks|same deadlocks as with threads]] happen with rows. Transaction A locks row 1 and wants row 2, while B locks row 2 and wants row 1. Databases detect this, and cancel one of them, so **your code must be ready to retry**.

```python
import random
import sqlite3
import time

class SerializationError(Exception):
    pass

def with_retries(action, attempts=3, base_delay=0.001):
    for attempt in range(attempts):
        try:
            return action()
        except (SerializationError, sqlite3.OperationalError):
            if attempt == attempts - 1:
                raise
            time.sleep(base_delay * 2 ** attempt * random.random())

calls = 0

def flaky_transaction():
    global calls
    calls = calls + 1
    if calls < 3:
        raise SerializationError("Deadlock detected")
    return "Committed"

print(with_retries(flaky_transaction))
print("Attempts:", calls)
```

Output:

```text
Committed
Attempts: 3
```

**The random delay** stops the two from colliding again and again.

**To make deadlocks rarer:**

- Keep transactions short
- Update rows in a consistent order, such as by id

```python
def transfer_order(sender_id, receiver_id):
    return sorted([sender_id, receiver_id])

print(transfer_order(9, 2))
print(transfer_order(2, 9))
```

Output:

```text
[2, 9]
[2, 9]
```

Both directions lock in the same order, so no circle can form.

## The Write-Ahead Log

![[wal.svg]]

How can a database promise durability, when writing to disk is slow? It writes down its plan first, in a **write-ahead log (WAL)**. Appending to the end of a log is fast. Once that's safely on disk, using [[36 Operating Systems#Files and Storage|fsync]], the change counts as committed, even if the tables haven't been updated yet.

After a crash, the database replays the log. It's like a chef's order tickets. If the kitchen stops, the tickets show what's still to do.

```python
import sqlite3

db = sqlite3.connect("wal_demo.db")
print(db.execute("PRAGMA journal_mode = WAL").fetchone()[0])
db.close()
```

Output:

```text
wal
```

**Other things the WAL makes possible:** replication to other servers, and restoring to a point in time.

## Worked Example - A Safe Transfer

The roadmap's exercise is to build a transfer, and test it under concurrency. A good test checks an **invariant**: the total money never changes.

```python
import sqlite3
import threading
import random

db_file = "bank.db"
setup = sqlite3.connect(db_file)
setup.execute("PRAGMA journal_mode = WAL")
setup.execute("DROP TABLE IF EXISTS accounts")
setup.execute("CREATE TABLE accounts (id INTEGER PRIMARY KEY, balance INTEGER NOT NULL CHECK (balance >= 0))")
setup.executemany("INSERT INTO accounts VALUES (?, 100)", [(i,) for i in range(5)])
setup.commit()
setup.close()

def worker(seed):
    generator = random.Random(seed)
    db = sqlite3.connect(db_file, timeout=30, isolation_level=None)
    for _ in range(50):
        sender, receiver = generator.sample(range(5), 2)
        try:
            db.execute("BEGIN IMMEDIATE")
            cursor = db.execute(
                "UPDATE accounts SET balance = balance - 10 WHERE id = ? AND balance >= 10", (sender,)
            )
            if cursor.rowcount == 1:
                db.execute("UPDATE accounts SET balance = balance + 10 WHERE id = ?", (receiver,))
            db.execute("COMMIT")
        except sqlite3.OperationalError:
            if db.in_transaction:
                db.execute("ROLLBACK")
    db.close()

threads = [threading.Thread(target=worker, args=(seed,)) for seed in range(4)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join()

check = sqlite3.connect(db_file)
total, lowest = check.execute("SELECT SUM(balance), MIN(balance) FROM accounts").fetchone()
print("Total money:", total)
print("No negative balances:", lowest >= 0)
```

Output:

```text
Total money: 500
No negative balances: True
```

Four threads made hundreds of transfers, and not one cent was lost. `BEGIN IMMEDIATE` takes SQLite's write lock at the start.

## Common Mistakes

> [!warning] Watch out for these
> - Read, change in code, then write back, without any protection
> - Calling slow services inside a transaction
> - Sending emails before committing
> - Not retrying after a deadlock
> - Assuming the default isolation level is the strongest
> - Using app-level locks when you run several servers

## Practice

**1. Which ACID letter?**
- Power fails right after "payment successful", and the record survives
- A transfer fails halfway, and nothing changes
- Two transactions can't see each other's half-finished work

> [!example]- Answer to question 1
> Durability, atomicity, isolation.

**2. Find the bug.** How do two buyers break this? How would you fix it?

```python
def buy(db, item_id):
    stock = db.execute("SELECT stock FROM items WHERE id = ?", (item_id,)).fetchone()[0]
    if stock > 0:
        db.execute("UPDATE items SET stock = ? WHERE id = ?", (stock - 1, item_id))
```

> [!example]- Answer to question 2
> Both read `stock = 1`, and both write 0, so two items are sold. Use one atomic statement:
> ```sql
> UPDATE items SET stock = stock - 1 WHERE id = ? AND stock > 0
> ```
> Then check `rowcount`.

**3. Boundaries.** What's wrong here?

```text
BEGIN
  Save the order
  Call the payment provider (takes up to 30 seconds)
  Send a confirmation email
COMMIT
```

> [!example]- Answer to question 3
> Locks are held for up to 30 seconds, blocking others. If the commit then fails, the customer has been charged and emailed for an order that doesn't exist. Save the order as `pending`, and commit. Call the provider outside. Then update the status in a new transaction, and use an outbox for the email.

**4. Which approach?** Optimistic or pessimistic?
- Editing a wiki page, with rare clashes
- Booking the last seats on a very popular flight

> [!example]- Answer to question 4
> Optimistic. Pessimistic, or an atomic update, since conflicts are common.

**5. Break it.** Remove `BEGIN IMMEDIATE` and the atomic check from the transfer example, so it reads the balance and writes it back. Does the total stay at 500? Then fix it.

## Can You Do This?

- [ ] Explain ACID with examples
- [ ] Use commit and rollback in code
- [ ] Choose good transaction boundaries
- [ ] Fix a lost update with an atomic statement
- [ ] Use optimistic locking
- [ ] Explain the isolation levels and write skew
- [ ] Retry after a deadlock
- [ ] Explain how a write-ahead log gives durability
- [ ] Test an invariant under concurrent load

## Summary

> [!success] Remember
> - **Transaction:** all or nothing.
> - **Keep them short.** No slow calls inside.
> - **Prefer one atomic statement** over read-then-write.
> - **Expect failures**, and retry deadlocks.
> - The **write-ahead log** makes durability possible.

**Next:** [[53 Phase 3 Final Project]] →
