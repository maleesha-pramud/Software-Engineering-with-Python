---
title: Pagebound - Season 2
tags: [phase-4, story, setup]
aliases: [Season 2, Pagebound Season 2, Season 2 Starter, Season 2 Map]
---

# Pagebound - Season 2

← [[Phase 4 - Start Here|Home]] · **Next:** [[54 Layered Architecture]] →

> [!abstract] What this page is
> The story continues. In Phase 3 you made Pagebound safe to ship. In Phase 4 it becomes **popular**, and the problems change. Set up the Season 2 repo once here, in about 10 minutes. After that, every note starts with a real incident, and you fix it with real tools.

## The Story

A year has passed. Pagebound now has **5,000 members** and **200,000 past loans**. Three more libraries want to join. Your first boss hired two more engineers, and they look to you.

Different problems arrive now. Not "a bug in one function", but:

- The code is hard to change safely, because everything is in one place.
- The home page is slow, and the email provider makes every Borrow click wait.
- Something is slow *sometimes*, and nobody can say where.
- A phone on bad Wi-Fi borrows the same book twice.
- A new engineer asks you how things work, and you realise you never wrote it down.

In Phase 3 you fixed code. In Phase 4 you **own a system**: you shape it, watch it, grow it, explain it, and keep it healthy.

![[p4-season-map.svg]]

Notes that follow the pattern say **Incident Card** near the top. Every note in Phase 4 does. The pattern is the same as in Phase 3: *break it, understand it, fix it, prove it*.

> [!tip] Did you finish Phase 3?
> Good. You'll recognise the shape of the app. But please **use the starter below**, even if your own project is further along. The labs print exact numbers, and they expect these files.

## Set Up Your Season 2 Repo

Do this once. It takes about 10 minutes.

> [!warning] The starter works, but it is a tangle on purpose
> It is one file, written the way real code often is: it grew a feature at a time. Phase 4 is about **how to shape it**. Please don't tidy it up early. That's the first lab.

**1. Make the folder and start Git.**

```bash
mkdir pagebound-s2
cd pagebound-s2
git init -b main
```

**2. Create these three files.** Open each fold, and copy the code into a file with that name.

> [!example]- seed.py (builds a database with 200,000 old loans)
> ```python
> """Create pagebound.db with a realistic amount of history."""
> import random
> import sqlite3
> from datetime import date, timedelta
>
> TITLES = [
>     "Dune", "Emma", "Ulysses", "Beloved", "Middlemarch", "Persuasion", "Dracula", "Kim",
>     "Ivanhoe", "Walden", "Rebecca", "Matilda", "Hamlet", "Candide", "Neuromancer", "Solaris",
> ]
>
>
> def build(path="pagebound.db", loans=200_000):
>     random.seed(1)
>     db = sqlite3.connect(path)
>     db.executescript("""
>         DROP TABLE IF EXISTS books;
>         DROP TABLE IF EXISTS loans;
>         CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, copies INTEGER, available INTEGER);
>         CREATE TABLE loans (
>             id INTEGER PRIMARY KEY, book_id INTEGER, member TEXT,
>             borrowed_on TEXT, due_on TEXT, returned_on TEXT
>         );
>     """)
>     for number, title in enumerate(TITLES, start=1):
>         db.execute("INSERT INTO books VALUES (?, ?, 5, 5)", (number, title))
>     start = date(2025, 1, 1)
>     rows = []
>     for _ in range(loans):
>         book = random.choice([1, 1, 1, 2, 2, 3] + list(range(1, len(TITLES) + 1)))
>         borrowed = start + timedelta(days=random.randint(0, 300))
>         rows.append((book, f"member{random.randint(1, 5000)}", borrowed.isoformat(),
>                      (borrowed + timedelta(days=14)).isoformat(), (borrowed + timedelta(days=10)).isoformat()))
>     db.executemany(
>         "INSERT INTO loans (book_id, member, borrowed_on, due_on, returned_on) VALUES (?, ?, ?, ?, ?)", rows
>     )
>     db.commit()
>     db.close()
>
>
> if __name__ == "__main__":
>     build()
>     print("pagebound.db is ready: 16 books, 200000 old loans")
> ```

> [!example]- app.py (the whole app, as shipped at the end of Phase 3)
> ```python
> """Pagebound v1.0, as shipped. It works. It is also one big tangle."""
> import json
> import sqlite3
> import time
> from datetime import date, datetime, timedelta
> from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
> from urllib.parse import urlparse
>
> DB_FILE = "pagebound.db"
>
>
> class Handler(BaseHTTPRequestHandler):
>     def reply(self, status, body):
>         data = json.dumps(body).encode()
>         self.send_response(status)
>         self.send_header("Content-Type", "application/json")
>         self.send_header("Content-Length", str(len(data)))
>         self.end_headers()
>         self.wfile.write(data)
>
>     def do_GET(self):
>         path = urlparse(self.path).path
>         db = sqlite3.connect(DB_FILE, timeout=30)
>         if path == "/popular":
>             rows = db.execute(
>                 "SELECT b.title, COUNT(*) AS n, COUNT(DISTINCT l.member) AS readers "
>                 "FROM loans l JOIN books b ON b.id = l.book_id GROUP BY b.id ORDER BY readers DESC LIMIT 5"
>             ).fetchall()
>             self.reply(200, [{"title": title, "loans": n, "readers": readers} for title, n, readers in rows])
>         elif path.startswith("/books/"):
>             row = db.execute(
>                 "SELECT id, title, copies, available FROM books WHERE id = ?", (int(path.split("/")[-1]),)
>             ).fetchone()
>             if row is None:
>                 self.reply(404, {"error": "not found"})
>             else:
>                 self.reply(200, {"id": row[0], "title": row[1], "copies": row[2], "available": row[3]})
>         else:
>             self.reply(404, {"error": "not found"})
>         db.close()
>
>     def do_POST(self):
>         size = int(self.headers.get("Content-Length", 0))
>         data = json.loads(self.rfile.read(size) or b"{}")
>         db = sqlite3.connect(DB_FILE, timeout=30)
>         if self.path == "/borrow":
>             member, book_id = data["member"], data["book_id"]
>             # Rule: a member may hold at most 3 books at once
>             active = db.execute(
>                 "SELECT COUNT(*) FROM loans WHERE member = ? AND returned_on IS NULL", (member,)
>             ).fetchone()[0]
>             if active >= 3:
>                 self.reply(409, {"result": "limit_reached"})
>             else:
>                 # Taking a copy is one atomic step, so two members can't take the last one
>                 taken = db.execute(
>                     "UPDATE books SET available = available - 1 WHERE id = ? AND available > 0", (book_id,)
>                 ).rowcount
>                 if taken == 0:
>                     self.reply(409, {"result": "none_left"})
>                 else:
>                     due = date.today() + timedelta(days=14)
>                     cursor = db.execute(
>                         "INSERT INTO loans (book_id, member, borrowed_on, due_on) VALUES (?, ?, ?, ?)",
>                         (book_id, member, date.today().isoformat(), due.isoformat()),
>                     )
>                     db.commit()
>                     time.sleep(0.3)  # Sending the confirmation email. The email provider is slow.
>                     with open("sent_emails.log", "a") as log:
>                         log.write(f"loan {cursor.lastrowid}: confirmation sent to {member}\n")
>                     self.reply(201, {"result": "ok", "loan_id": cursor.lastrowid, "due_on": due.isoformat()})
>         elif self.path == "/return":
>             loan = db.execute(
>                 "SELECT book_id, due_on FROM loans WHERE id = ? AND returned_on IS NULL", (data["loan_id"],)
>             ).fetchone()
>             if loan is None:
>                 self.reply(404, {"error": "no such open loan"})
>             else:
>                 today = datetime.fromisoformat(data.get("today", date.today().isoformat())).date()
>                 days_late = max(0, (today - date.fromisoformat(loan[1])).days)
>                 fee_cents = min(days_late * 25, 1000)  # 25 cents a day, at most $10
>                 db.execute("UPDATE loans SET returned_on = ? WHERE id = ?", (today.isoformat(), data["loan_id"]))
>                 db.execute("UPDATE books SET available = available + 1 WHERE id = ?", (loan[0],))
>                 db.commit()
>                 self.reply(200, {"result": "returned", "days_late": days_late, "fee_cents": fee_cents})
>         else:
>             self.reply(404, {"error": "not found"})
>         db.close()
>
>     def log_message(self, *args):
>         pass
>
>
> if __name__ == "__main__":
>     print("Pagebound is running at http://127.0.0.1:8000  (Ctrl+C to stop)")
>     ThreadingHTTPServer(("127.0.0.1", 8000), Handler).serve_forever()
> ```

> [!example]- tryit.py (a tiny tool to send requests)
> ```python
> import json
> import sys
> import urllib.error
> import urllib.parse
> import urllib.request
>
> # Usage: python tryit.py GET search q=Dune
> #        python tryit.py POST login name=sam password=pass123
> # (No leading slash on the path: Git Bash on Windows would rewrite it.)
> method, path, *pairs = sys.argv[1:]
> fields = dict(pair.split("=", 1) for pair in pairs)
>
> url = "http://127.0.0.1:8000/" + path.lstrip("/")
> data = None
> if method.upper() == "GET":
>     if fields:
>         url = url + "?" + urllib.parse.urlencode(fields)
> else:
>     data = json.dumps(fields).encode()
>
> request = urllib.request.Request(
>     url, data=data, method=method.upper(), headers={"Content-Type": "application/json"}
> )
> try:
>     with urllib.request.urlopen(request) as response:
>         status, body = response.status, response.read().decode()
> except urllib.error.HTTPError as error:
>     status, body = error.code, error.read().decode()
>
> print(status, body)
> ```

`tryit.py` does what `curl` does, but it works the same in PowerShell, Git Bash and Linux. The path has **no leading slash**: Git Bash on Windows would rewrite `/books/1` into a file path.

**3. Build the database and run the app.**

```bash
python seed.py
python app.py
```

```text
pagebound.db is ready: 16 books, 200000 old loans
Pagebound is running at http://127.0.0.1:8000  (Ctrl+C to stop)
```

In a second terminal, look around:

```bash
python tryit.py GET books/1
python tryit.py GET popular
python tryit.py POST borrow book_id=1 member=sam
```

```text
200 {"id": 1, "title": "Dune", "copies": 5, "available": 5}
200 [{"title": "Dune", "loans": 36270, "readers": 4999}, {"title": "Emma", "loans": 27391, "readers": 4983}, ...]
201 {"result": "ok", "loan_id": 200001, "due_on": "2026-12-08"}
```

Your `due_on` is 14 days after today. The rules of the app are small:

| Rule | Where it lives today |
|---|---|
| A member may hold at most **3** books | Inside `do_POST` |
| A loan lasts **14 days** | Inside `do_POST` |
| A late return costs **25 cents a day**, at most **$10** | Inside `do_POST` |
| After a borrow, a confirmation email is sent (slow!) | Inside `do_POST` |

Look at that last column. That is the first problem.

**4. Make your first commit.** Create `.gitignore`:

```text
pagebound.db
pagebound.db-*
queue.db*
sent_emails.log
server.log
__pycache__/
.pytest_cache/
```

```bash
git add .
git commit -m "Season 2 starter"
```

**5. Put it on GitHub** (optional but good). Create an empty private repository called `pagebound-s2`, then run the two commands that GitHub shows you.

## Your Season Map

![[p4-season-map.svg]]

| Release | After | Boss check |
|---|---|---|
| **v1.1** | [[56 Domain-Driven Design]] | You swap the database for memory by changing **one file**, and the lending rules are tested in milliseconds with no database. |
| **v1.2** | [[58 Message Queues]] | `GET /popular` is served from a cache, a Borrow takes about 30 ms, and you kill the email worker mid-job without losing or doubling an email. |
| **v1.3** | [[61 Infrastructure as Code]] | You find a hidden slow step using only logs and metrics, and you start the app on any port and database from settings alone. |
| **v1.4** | [[64 System Design]] | You run three load-test rounds and record each change, a retried Borrow is safe, and a design for a new feature sits on paper. |
| **v1.5** | [[67 System Ownership]] | A design doc that people can act on, an SLO with an error budget, and a blameless postmortem. |
| **v2.0** | [[68 Phase 4 Final Project]] | One command, `release_check.py`, prints PASS: tests, load test and promises. |

To mark a release:

```bash
git tag v1.1
git push --tags
```

## Your Incident Log

Phase 3 gave you a `LOG.md` of bugs. Keep it going, and add a column of **numbers**. In Phase 4 the proof is a measurement.

```markdown
## 57 - The home page was slow
- **What broke:** /popular took about 215 ms, and a few members at once made it worse.
- **Why:** Every request ran the same heavy database query.
- **Fix:** A 30 second cache. About 45 requests per second became over 700.
```

When you finish Phase 4, this is a list of real stories about systems, not only code. That is exactly what interviewers for mid-level and senior roles want.

**Next:** [[54 Layered Architecture]] →
