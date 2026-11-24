---
title: Pagebound - The Running Story
tags: [phase-3, story, setup]
aliases: [Pagebound, Running Story, Pagebound Starter, Season Map]
---

# Pagebound - The Running Story

← [[Phase 3 - Start Here|Home]] · **Next:** [[35 Computer Organization and Architecture]] →

> [!abstract] What this page is
> One small app that you grow through all of Phase 3. Set it up once here, in about 10 minutes. After that, every reworked note starts with a **real problem from this app**, and you fix it with real tools.

## The Story

You've just been hired as the **first engineer at Pagebound**, a tiny startup that lends books to a neighbourhood. The app was built in a hurry, so it works... mostly. It has books, members and loans. Real members arrive next week.

Over the next months, things go wrong, the way they do at a real job. A member finds a bug. A ticket arrives. A page gets slow. Each time, your job is to **find out why, fix it properly, and prove that it's fixed**.

![[p3-note-loop.svg]]

Notes that follow this pattern say **Incident Card** near the top. You'll find them in [[37 Concurrency and Parallelism]], [[39 Software Development Lifecycle]] and [[45 Security Fundamentals]].

> [!info] Still in the older style
> The other Phase 3 notes don't have an Incident Card yet. They still teach well, and you can read them in the same order.

## Set Up Your Repo

Do this once. It takes about 10 minutes.

> [!warning] The starter has bugs on purpose
> It's written the way code often is when people are in a hurry. The labs make you find the problems and fix them. **Please don't fix them early.** That's the fun part.

> [!tip] Already have your Phase 2 project?
> Use the starter below anyway. The labs print exact results, and they expect these exact files. When you've finished a lab, you can copy the idea back into your own project.

**1. Make the folder and start Git.**

```bash
mkdir pagebound
cd pagebound
git init -b main
```

**2. Create these three files.** Open each fold, and copy the code into a file with that name.

> [!example]- store.py (the database code)
> ```python
> import sqlite3
> import time
> from contextlib import contextmanager
>
>
> class Store:
>     def __init__(self, path="pagebound.db"):
>         self.path = path
>
>     @contextmanager
>     def connect(self):
>         db = sqlite3.connect(self.path, timeout=30)
>         try:
>             with db:  # Save on success, undo on error
>                 yield db
>         finally:
>             db.close()
>
>     def reset_demo(self):
>         """Wipe everything and add fresh demo data."""
>         with self.connect() as db:
>             db.executescript("""
>                 DROP TABLE IF EXISTS books;
>                 DROP TABLE IF EXISTS loans;
>                 DROP TABLE IF EXISTS users;
>                 CREATE TABLE books (id INTEGER PRIMARY KEY, title TEXT, copies INTEGER, available INTEGER);
>                 CREATE TABLE loans (id INTEGER PRIMARY KEY, book_id INTEGER, member TEXT);
>                 CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, password TEXT, is_admin INTEGER);
>                 INSERT INTO books VALUES (1, 'Dune', 3, 3), (2, 'Emma', 5, 5);
>                 INSERT INTO users VALUES (1, 'admin', 'Tr0ub4dor&3', 1), (2, 'sam', 'pass123', 0);
>             """)
>
>     def get_book(self, book_id):
>         with self.connect() as db:
>             row = db.execute(
>                 "SELECT id, title, copies, available FROM books WHERE id = ?", (book_id,)
>             ).fetchone()
>             if row is None:
>                 return None
>             loans = db.execute(
>                 "SELECT COUNT(*) FROM loans WHERE book_id = ?", (book_id,)
>             ).fetchone()[0]
>         return {"id": row[0], "title": row[1], "copies": row[2], "available": row[3], "loans": loans}
>
>     def borrow(self, book_id, member):
>         with self.connect() as db:
>             row = db.execute("SELECT available FROM books WHERE id = ?", (book_id,)).fetchone()
>             if row is None:
>                 return "no_such_book"
>             if row[0] < 1:
>                 return "none_left"
>             time.sleep(0.01)  # Pretend: check the member's fines, and so on
>             db.execute("UPDATE books SET available = ? WHERE id = ?", (row[0] - 1, book_id))
>             db.execute("INSERT INTO loans (book_id, member) VALUES (?, ?)", (book_id, member))
>             return "ok"
>
>     def search(self, text):
>         with self.connect() as db:
>             sql = f"SELECT id, title, available FROM books WHERE title LIKE '%{text}%'"
>             return [list(row) for row in db.execute(sql).fetchall()]
>
>     def login(self, name, password):
>         with self.connect() as db:
>             sql = f"SELECT id, name, is_admin FROM users WHERE name = '{name}' AND password = '{password}'"
>             row = db.execute(sql).fetchone()
>         if row is None:
>             return None
>         return {"id": row[0], "name": row[1], "is_admin": bool(row[2])}
> ```

> [!example]- server.py (the web server)
> ```python
> import json
> from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
> from urllib.parse import parse_qs, urlparse
>
> from store import Store
>
> store = Store()
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
>         url = urlparse(self.path)
>         if url.path == "/search":
>             text = parse_qs(url.query).get("q", [""])[0]
>             self.reply(200, store.search(text))
>         elif url.path.startswith("/books/"):
>             book = store.get_book(int(url.path.split("/")[-1]))
>             self.reply(200 if book else 404, book or {"error": "not found"})
>         else:
>             self.reply(404, {"error": "not found"})
>
>     def do_POST(self):
>         size = int(self.headers.get("Content-Length", 0))
>         data = json.loads(self.rfile.read(size) or b"{}")
>         if self.path == "/borrow":
>             result = store.borrow(data["book_id"], data["member"])
>             status = {"ok": 201, "none_left": 409, "no_such_book": 404}[result]
>             self.reply(status, {"result": result})
>         elif self.path == "/login":
>             user = store.login(data.get("name", ""), data.get("password", ""))
>             self.reply(200 if user else 401, user or {"error": "wrong name or password"})
>         else:
>             self.reply(404, {"error": "not found"})
>
>     def log_message(self, *args):
>         pass  # Keep the terminal quiet
>
>
> if __name__ == "__main__":
>     store.reset_demo()
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

`tryit.py` does what `curl` does, but it works the same in PowerShell, Git Bash and Linux. Notice the paths have **no leading slash**: Git Bash on Windows would rewrite `/books/1` into a file path.

**3. Run it.** Open two terminals in the `pagebound` folder.

```bash
python server.py
```

```text
Pagebound is running at http://127.0.0.1:8000  (Ctrl+C to stop)
```

In the second terminal:

```bash
python tryit.py GET books/1
```

```text
200 {"id": 1, "title": "Dune", "copies": 3, "available": 3, "loans": 0}
```

Each time you start `server.py`, the demo data **resets**. That makes the labs repeatable.

**4. Make your first commit.** Create the file `.gitignore` with these lines, so that the database file and caches stay out of Git:

```text
pagebound.db
__pycache__/
.pytest_cache/
```

```bash
git add .
git commit -m "Pagebound starter"
```

**5. Put it on GitHub.** On [github.com](https://github.com/), create a new **empty private** repository called `pagebound`. Don't add a README. Then copy the two commands that GitHub shows you, which look like this:

```bash
git remote add origin https://github.com/YOUR-NAME/pagebound.git
git push -u origin main
```

## Your Season Map

![[p3-season-map.svg]]

Phase 3 is long, so you don't wait until the end to ship something. **After each Part, you pass a boss check and tag a release.**

| Release | After | Boss check |
|---|---|---|
| **v0.1** | [[38 Memory Management]] | 20 members press Borrow on 3 copies, and exactly 3 loans happen |
| **v0.2** | [[41 Refactoring]] | A pull request with a review is merged, and the tests are still green |
| **v0.3** | [[44 API Design]] | You add a new kind of item without editing the old code, and the API has a version and pagination |
| **v0.4** | [[49 Continuous Integration and Delivery]] | The attack fails, tests run on a real database, it runs in Docker, and CI is green |
| **v0.5** | [[52 Transactions and ACID]] | Search is fast on 100,000 books, and nobody can borrow the same copy twice |
| **v1.0** | [[53 Phase 3 Final Project]] | Everything together, with a threat model |

To mark a release:

```bash
git tag v0.1
git push --tags
```

## Your Incident Log

Create a file called `LOG.md` in your repo. After each incident, add three lines:

```markdown
## 37 - Two members borrowed the same book
- **What broke:** 20 members borrowed a book that had 3 copies.
- **Why:** Reading the count and changing it were two separate steps.
- **Fix:** One lock around the whole borrow. Tested with 20 parallel requests.
```

When you finish Phase 3, this is a list of real stories to use in interviews. It's also a record of what you've learned. Interviewers love "tell me about a bug you fixed".

**Next:** [[35 Computer Organization and Architecture]] →
