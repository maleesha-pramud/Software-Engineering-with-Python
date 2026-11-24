---
title: Scalability
tags: [phase-4, performance, scalability]
aliases: [Scalability, Scaling, Load Testing, Bottleneck, Horizontal Scaling, Vertical Scaling, Amdahl's Law, Capacity Planning]
---

# Scalability

← [[61 Infrastructure as Code]] · [[Phase 4 - Start Here|Home]] · **Next:** [[63 Distributed Computing]] → · [[Software Engineering Roadmap#Scalability|Roadmap card]]

> [!abstract] After this note, you can
> Load-test an app, find the top bottleneck from real numbers, fix one thing at a time, and say what will break first at ten times today's load. You'll have run three measured rounds on [[Pagebound - Season 2|Pagebound]] and recorded each change.

> [!bug] Incident Card · Pagebound Season 2
> **Monday, 09:00.** Three new libraries start next month. That means about **10 times** today's members. Your boss asks one question: *"Will it cope?"*
>
> You could guess "yes", or "buy bigger servers". Both are guesses. Last Monday's guess about the popular list cost a morning of angry email.
>
> **Your job:** answer with **numbers**. Measure now, find what breaks first, fix it, and measure again.

## Lab 1 - Break It

*About 15 minutes.* You need the code from [[60 Cloud Computing]]: layered, cached, with a queue, logs and metrics.

Create `loadtest.py`. It plays many members at once. Each member does what a real one does: looks at a book, looks at what's popular, borrows, and returns.

> [!example]- loadtest.py (new)
> ```python
> """Pretend to be many members. Each one looks at a book, sees what is popular, borrows, and returns.
> Usage: python loadtest.py [MEMBERS] [AT_THE_SAME_TIME]"""
> import json
> import os
> import sys
> import threading
> import time
> import urllib.request
>
> PORTS = os.environ.get("PORTS", "8000").split(",")  # Several ports means several copies of the app
> members = int(sys.argv[1]) if len(sys.argv) > 1 else 200
> at_once = int(sys.argv[2]) if len(sys.argv) > 2 else 20
>
> lock = threading.Lock()
> next_member = [0]
> session_times = []
> failures = []
>
>
> def call(port, method, path, body=None):
>     data = json.dumps(body).encode() if body is not None else None
>     request = urllib.request.Request(
>         f"http://127.0.0.1:{port}{path}", data=data, method=method, headers={"Content-Type": "application/json"}
>     )
>     with urllib.request.urlopen(request) as response:
>         return json.loads(response.read())
>
>
> def one_member(number):
>     port = PORTS[number % len(PORTS)]  # Each member talks to one copy of the app
>     began = time.perf_counter()
>     call(port, "GET", f"/books/{number % 16 + 1}")
>     call(port, "GET", "/popular")
>     loan = call(port, "POST", "/borrow", {"book_id": number % 16 + 1, "member": f"load{number}"})
>     call(port, "POST", "/return", {"loan_id": loan["loan_id"]})
>     return time.perf_counter() - began
>
>
> def worker():
>     while True:
>         with lock:
>             number = next_member[0]
>             if number >= members:
>                 return
>             next_member[0] += 1
>         try:
>             took = one_member(number)
>             with lock:
>                 session_times.append(took)
>         except Exception as error:
>             with lock:
>                 failures.append(repr(error))
>
>
> began = time.perf_counter()
> threads = [threading.Thread(target=worker) for _ in range(at_once)]
> for thread in threads:
>     thread.start()
> for thread in threads:
>     thread.join()
> wall = time.perf_counter() - began
>
> session_times.sort()
> p95 = session_times[int(len(session_times) * 0.95) - 1] if session_times else 0
> print(f"{len(session_times)} members finished, {len(failures)} failed, in {wall:.1f}s")
> print(f"{len(session_times) / wall:.1f} members per second, 95% finished within {p95 * 1000:.0f} ms")
> ```

Seed the database, start the server with its log saved, and run **200 members, 20 at the same moment**. (Make sure no `MYSTERY` variable is set.)

Git Bash or Linux:

```bash
python seed.py
python main.py > server.log
```

PowerShell:

```powershell
python seed.py
python main.py | Out-File -Encoding utf8 server.log
```

In another terminal:

```bash
python loadtest.py 200 20
```

```text
200 members finished, 0 failed, in 8.0s
24.8 members per second, 95% finished within 3143 ms
```

Stop the server, and ask the log what was slow, using the tool from [[59 Observability]]:

```bash
python logsummary.py
```

```text
step                          calls  average ms  slowest ms
POST /borrow                    200         650        7323
repo.take_copy                  200         597        7237
POST /return                    200          69        3164
repo.close_loan                 200          65        3085
repo.get_book                   400          13         201
...
```

> [!bug] You have a number, and a suspect
> **About 25 members per second.** One in twenty members waited more than 3 seconds. And the top suspect is clear: `repo.take_copy`, the step that *writes*, averages almost **600 ms** and once took 7 seconds. Not the cache. Not the query you'd have guessed. (Your numbers will differ. The *order* should match.)

Write the baseline in `LOG.md` before you change anything. Every later number needs something to compare with.

## The Big Idea

**Scalability** is the ability to handle **more load** without a matching jump in cost or a collapse in speed. It is **not** the same as "fast". A fast app can still fall over at 10 times the traffic.

### Two ways to grow

| | Vertical ("scale up") | Horizontal ("scale out") |
|---|---|---|
| **Means** | A bigger machine | More machines |
| **Easy?** | Very. Nothing changes in the code | Needs a design: statelessness, shared data |
| **Limit** | The biggest machine you can buy | Far higher |
| **Risk** | One machine is a single point of failure | Coordination, consistency |

**Statelessness** is what makes horizontal scaling easy: any server can answer any request, because nothing important is kept in one server's memory. Pagebound's cache is only a *copy*, so that's fine. The truth lives in the database.

![[p4-scale-loop.svg]]

### The loop

1. **Measure** with a load test, and write the numbers down.
2. **Find** the top bottleneck: the slowest step *in your measurements*, not the one you suspect.
3. **Fix one thing.** One change at a time, so you know what helped.
4. **Measure again.** Did it help? What's the slowest step now?

Stop when you meet your goal. Speed beyond what you need is wasted effort.

### Amdahl's Law: the part you can't speed up

If **10%** of the work can't be done in parallel (for example, every request must write to the same database file), then even with a thousand servers, you can't be more than **10 times** faster. The shared, serial part sets the ceiling. That is **Amdahl's Law**, and it explains Round 3 below.

### Back-of-the-envelope capacity planning

You don't need a perfect model, just a good guess with the right *size*. For example:

- 5,000 members. In the busiest hour, say 10% are active: 500 people.
- Each does about one session (4 requests) every 2 minutes: 500 × 4 / 120 ≈ **17 requests per second**.
- Times 10 (the three new libraries): about **170 requests per second**, and a peak maybe 3 times that: **500 per second**.

Now compare with measurements: reads ran at over 1,000 requests per second, but *mixed* sessions with writes ran at only 25 members (100 requests) per second. The **writes** are the gap. That tells you where to look.

> [!note]- Other tools for scaling
> **Caching** ([[57 Caching]]) removes repeated reads. **Replication** copies data to read-only followers, so reads spread out. **Partitioning (sharding)** splits data across databases by a key. **Queues** ([[58 Message Queues]]) smooth bursts. **Backpressure** and **load shedding** protect a system under overload by slowing callers or refusing some requests, so that the rest succeed. And **cost** is a dimension too: the cheapest way to handle 10 times is sometimes a cache, not 10 servers.

## Lab 2 - Fix It

*About 40 minutes.* Three rounds. After **each** one: `python seed.py`, start the server with `> server.log`, run `python loadtest.py 200 20`, stop the server, run `python logsummary.py`, and note the numbers. (Run each round twice, since the first run has a cold start.)

### Round 1 - The suspect: writes wait in line

SQLite's default mode locks the *whole file* for every write, and readers wait for writers. **WAL mode** (write-ahead logging) lets readers keep reading while one writer writes. In `repo.py`, change `__init__` and `_connect`:

```python
    def __init__(self, path="pagebound.db"):
        self.path = path
        db = sqlite3.connect(self.path)
        db.execute("PRAGMA journal_mode=WAL")  # Readers stop blocking the writer
        db.close()

    def _connect(self):
        db = sqlite3.connect(self.path, timeout=30)
        db.execute("PRAGMA synchronous=NORMAL")  # Fewer slow waits for the disk
        return db
```

```text
200 members finished, 0 failed, in 4.5s
44.8 members per second, 95% finished within 664 ms
```

From **25 to about 45 members per second**, and the slow tail from 3.1 seconds to well under one. One change, and you know exactly what bought it.

> [!warning] A mistake worth remembering
> My first try put `PRAGMA journal_mode=WAL` inside `_connect`, so every connection tried to switch modes, and under load some requests failed with *database is locked*. The mode is saved in the file, so set it **once**. A load test finds bugs like this. A single request never would.

### Round 2 - Stop opening a new connection for every query

Look at `logsummary.py` again. Each request now makes several repository calls, and **each call opens and closes a database connection**. Opening one isn't free. Also, `count_active_loans` searches the 200,000 old loans for a member, with no index. Replace the whole `SqliteRepo` class in `repo.py`:

> [!example]- repo.py (changed: reuse one connection per thread, and add an index)
> ```python
> """Data layer: the only place that knows about SQLite."""
> import sqlite3
> import threading
>
>
> class SqliteRepo:
>     def __init__(self, path="pagebound.db"):
>         self.path = path
>         self.local = threading.local()  # Each thread keeps its own connection
>         db = sqlite3.connect(self.path)
>         db.execute("PRAGMA journal_mode=WAL")  # Readers stop blocking the writer
>         db.execute("CREATE INDEX IF NOT EXISTS loans_by_member ON loans (member)")  # A phone book for members
>         db.close()
>
>     def _db(self):
>         """Open a connection once per thread, and reuse it. Opening a connection is not free."""
>         if not hasattr(self.local, "db"):
>             self.local.db = sqlite3.connect(self.path, timeout=30)
>             self.local.db.execute("PRAGMA synchronous=NORMAL")  # Fewer slow waits for the disk
>         return self.local.db
>
>     def get_book(self, book_id):
>         row = self._db().execute(
>             "SELECT id, title, copies, available FROM books WHERE id = ?", (book_id,)
>         ).fetchone()
>         if row is None:
>             return None
>         return {"id": row[0], "title": row[1], "copies": row[2], "available": row[3]}
>
>     def count_active_loans(self, member):
>         return self._db().execute(
>             "SELECT COUNT(*) FROM loans WHERE member = ? AND returned_on IS NULL", (member,)
>         ).fetchone()[0]
>
>     def take_copy(self, book_id):
>         """Take one copy off the shelf. Returns False if none are left."""
>         db = self._db()
>         with db:
>             taken = db.execute(
>                 "UPDATE books SET available = available - 1 WHERE id = ? AND available > 0", (book_id,)
>             ).rowcount
>         return taken == 1
>
>     def add_loan(self, book_id, member, borrowed_on, due_on):
>         db = self._db()
>         with db:
>             cursor = db.execute(
>                 "INSERT INTO loans (book_id, member, borrowed_on, due_on) VALUES (?, ?, ?, ?)",
>                 (book_id, member, borrowed_on, due_on),
>             )
>         return cursor.lastrowid
>
>     def get_open_loan(self, loan_id):
>         row = self._db().execute(
>             "SELECT book_id, due_on FROM loans WHERE id = ? AND returned_on IS NULL", (loan_id,)
>         ).fetchone()
>         return None if row is None else {"book_id": row[0], "due_on": row[1]}
>
>     def close_loan(self, loan_id, book_id, returned_on):
>         db = self._db()
>         with db:
>             db.execute("UPDATE loans SET returned_on = ? WHERE id = ?", (returned_on, loan_id))
>             db.execute("UPDATE books SET available = available + 1 WHERE id = ?", (book_id,))
>
>     def popular(self, limit=5):
>         rows = self._db().execute(
>             "SELECT b.title, COUNT(*) AS n, COUNT(DISTINCT l.member) AS readers "
>             "FROM loans l JOIN books b ON b.id = l.book_id GROUP BY b.id ORDER BY readers DESC LIMIT ?",
>             (limit,),
>         ).fetchall()
>         return [{"title": title, "loans": n, "readers": readers} for title, n, readers in rows]
>
>
> class MemoryRepo:
>     """The same promises, kept in plain Python dicts. Perfect for fast tests."""
>
>     def __init__(self):
>         self.books = {}
>         self.loans = {}
>         self.next_loan_id = 1
>
>     def add_book(self, book_id, title, copies):
>         self.books[book_id] = {"id": book_id, "title": title, "copies": copies, "available": copies}
>
>     def get_book(self, book_id):
>         book = self.books.get(book_id)
>         return None if book is None else dict(book)
>
>     def count_active_loans(self, member):
>         return sum(1 for loan in self.loans.values() if loan["member"] == member and loan["returned_on"] is None)
>
>     def take_copy(self, book_id):
>         if self.books[book_id]["available"] < 1:
>             return False
>         self.books[book_id]["available"] -= 1
>         return True
>
>     def add_loan(self, book_id, member, borrowed_on, due_on):
>         loan_id = self.next_loan_id
>         self.next_loan_id += 1
>         self.loans[loan_id] = {
>             "book_id": book_id, "member": member, "due_on": due_on, "returned_on": None,
>         }
>         return loan_id
>
>     def get_open_loan(self, loan_id):
>         loan = self.loans.get(loan_id)
>         if loan is None or loan["returned_on"] is not None:
>             return None
>         return {"book_id": loan["book_id"], "due_on": loan["due_on"]}
>
>     def close_loan(self, loan_id, book_id, returned_on):
>         self.loans[loan_id]["returned_on"] = returned_on
>         self.books[book_id]["available"] += 1
>
>     def popular(self, limit=5):
>         loans, readers = {}, {}
>         for loan in self.loans.values():
>             title = self.books[loan["book_id"]]["title"]
>             loans[title] = loans.get(title, 0) + 1
>             readers.setdefault(title, set()).add(loan["member"])
>         ranked = sorted(loans, key=lambda title: -len(readers[title]))[:limit]
>         return [{"title": t, "loans": loans[t], "readers": len(readers[t])} for t in ranked]
> ```

Two ideas: `_db()` opens a connection once **per thread** and reuses it, and the index `loans_by_member` is a phone book for finding a member's loans without reading everything. (Indexes are the topic of [[51 Indexing]].)

```text
200 members finished, 0 failed, in 2.7s
74.5 members per second, 95% finished within 545 ms
```

Another gain: **about 70 members per second**, nearly three times the start.

### Round 3 - Add a second copy of the app

Now try the "obvious" fix: more servers. Terminal 1: `python main.py > server.log`. Terminal 2 (a second copy, on another port):

Git Bash or Linux:

```bash
PORT=8001 python main.py > server2.log
```

PowerShell:

```powershell
$env:PORT = "8001"; python main.py | Out-File -Encoding utf8 server2.log
```

Tell the load test to share its members between the two ports:

```bash
PORTS=8000,8001 python loadtest.py 200 20
```

(In PowerShell: `$env:PORTS = "8000,8001"; python loadtest.py 200 20`.)

```text
200 members finished, 0 failed, in 2.5s
80.4 members per second, 95% finished within 446 ms
```

Better, but **not double**: about 70 became about 80. Try a heavier test (`400 40`) and you'll see roughly 100. Why isn't it twice as fast? Both copies write into **the same database file**, and writes take turns. That's Amdahl's Law, and it's a real finding. For reads, two copies *do* scale well (a pure read test went from about 1,200 to about 2,200 requests per second in my tests). To scale *writes* further, you would move to a database server such as PostgreSQL, and use [[Software Engineering Roadmap#Database Replication|replication]] and partitioning. That's the next big step, and you now have the numbers to justify it.

### Write it down

Fill a table in `LOG.md`. This is the proof you show your boss:

| Round | Change | Members per second | p95 |
|---|---|---|---|
| 0 | Baseline | about 25 | about 3,100 ms |
| 1 | WAL mode | about 45 | about 670 ms |
| 2 | Reuse connections, add an index | about 70 | about 520 ms |
| 3 | Second copy of the app | about 80 | about 450 ms |

```markdown
## 62 - Will it cope with 10 times the members?
- **What I measured:** 25 members/s at the start, with slow writes.
- **What I changed:** one thing per round (WAL, connection reuse + index, a second server).
- **Result:** about 3 times faster. A second server helps reads, but writes share one database file. Next: a database server.
- **Answer for the boss:** at 10x we expect about 170 requests/s on average. We handle about 300 mixed requests/s on one machine today.
```

```bash
git add .
git commit -m "Load test, WAL mode, connection reuse, index"
```

## What Breaks First at 10 Times?

Predict, then test. For Pagebound, from your measurements:

| Part | At 10x | Why |
|---|---|---|
| Reads of one book | Fine | 1,000+ per second on one machine |
| `/popular` | Fine | Cached |
| Borrow and return (writes) | **First to hurt** | They share one database file |
| The email queue | Fine for now | A worker can't fall behind unless emails are slower than borrows |
| Memory | Fine | The cache is small |

A prediction that you checked with a test is worth ten opinions.

## Practice

**1. Which kind?** Your server is at 90% CPU. (a) You move to a machine with twice the cores. (b) You run two more copies behind a load balancer. Name each.

> [!example]- Answer to question 1
> (a) **Vertical** scaling. (b) **Horizontal** scaling. (Horizontal needs the app to be stateless, so any copy can serve any request.)

**2. Amdahl.** One step takes 20% of the time and can't be run in parallel. What's the most you can ever speed the whole job up?

> [!example]- Answer to question 2
> At most **5 times** (1 ÷ 0.2). Even with unlimited machines for the other 80%, the serial 20% remains. So find and shrink the serial part first.

**3. One change only.** You make three changes at once and the test gets 30% faster. What went wrong with the method?

> [!example]- Answer to question 3
> You can't tell which change helped, which did nothing, and which might even have hurt. Change **one thing per round**, so each number has a cause.

**4. Estimate.** A new library adds 2,000 members, who do the same as the others. By what share does the load grow?

> [!example]- Answer to question 4
> From 5,000 to 7,000 members: **40% more**, as long as they behave alike. Always check that assumption: new groups often differ.

**5. Roadmap exercise.** Load-test an app, fix the top bottleneck, repeat three times, and record each change. You just did. Now run a **fourth** round of your own. What is the top suspect now, and what would you try?

> [!example]- Answer to question 5
> Run `logsummary.py` after Round 3. The top rows will likely still be `take_copy` and `close_loan` (writes). Ideas to try: batch writes, make the loan write and the copy update **one** transaction, cut work inside the request, or move to a client-server database. Predict the result first, then measure.

## Can You Do This?

- [ ] Explain scalability, and how it differs from speed
- [ ] Compare vertical and horizontal scaling
- [ ] Explain statelessness, and why it helps
- [ ] Run a load test, and find the top bottleneck from numbers
- [ ] Change one thing at a time, and record the result
- [ ] Explain Amdahl's Law with an example
- [ ] Make a back-of-the-envelope capacity estimate

## Summary

> [!success] Remember
> - **Measure, find the top bottleneck, fix one thing, measure again.**
> - The slowest step in your **numbers** is the target, not your hunch.
> - **Stateless** apps scale out. A **shared, serial** part limits the gain (Amdahl).
> - Predict what breaks first, and **test the prediction**.

**Next:** [[63 Distributed Computing]] →
