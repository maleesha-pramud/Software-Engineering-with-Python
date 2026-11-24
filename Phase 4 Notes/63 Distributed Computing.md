---
title: Distributed Computing
tags: [phase-4, distributed-systems]
aliases: [Distributed Computing, Distributed Systems, Partial Failure, Idempotency Key, Lamport Clock, Timeouts, Fallacies of Distributed Computing]
---

# Distributed Computing

← [[62 Scalability]] · [[Phase 4 - Start Here|Home]] · **Next:** [[64 System Design]] → · [[Software Engineering Roadmap#Distributed Computing|Roadmap card]]

> [!abstract] After this note, you can
> Explain why a network makes everything harder, tell a slow reply from a dead server (you can't!), make retries safe with idempotency keys, and order events without trusting clocks. You'll have fixed a double-borrow bug caused by a lost reply in [[Pagebound - Season 2|Pagebound]].

> [!bug] Incident Card · Pagebound Season 2
> **Friday, 18:05.** A member, Dana, writes in, upset: *"I tapped Borrow ONE time on the train. The app said 'try again'. Now I'm holding **two** copies of* Dune*, and I'm only allowed three books."*
>
> You look at the logs. One tap, two loans. The first request **worked**. The reply never reached the phone, so the app asked again.
>
> **Your job:** reproduce a lost reply on your own machine, and make retries safe.

## Lab 1 - Break It

*About 15 minutes.* A distributed system is one where the parts talk **over a network that can fail**. A real bad network is hard to summon, so we'll build one. `flaky_proxy.py` sits between the phone and Pagebound. It passes every request on, but **throws away the reply** of the 1st, 3rd, 5th... request.

> [!example]- flaky_proxy.py (new)
> ```python
> """A pretend bad phone network. It passes every POST on to Pagebound, but it loses every 1st, 3rd, 5th... reply."""
> import http.client
> import itertools
> import socket
> from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
>
> TARGET = ("127.0.0.1", 8000)
> counter = itertools.count(1)
>
>
> class Proxy(BaseHTTPRequestHandler):
>     def do_POST(self):
>         body = self.rfile.read(int(self.headers.get("Content-Length", 0)))
>         forward = {k: v for k, v in self.headers.items() if k.lower() in ("content-type", "idempotency-key")}
>         connection = http.client.HTTPConnection(*TARGET, timeout=10)
>         connection.request("POST", self.path, body, forward)
>         reply = connection.getresponse()
>         data = reply.read()  # Pagebound has now DONE the work
>         if next(counter) % 2 == 1:
>             print("proxy: the reply got lost on the way back", flush=True)
>             self.connection.shutdown(socket.SHUT_RDWR)  # The phone never hears back
>             return
>         self.send_response(reply.status)
>         self.send_header("Content-Type", "application/json")
>         self.send_header("Content-Length", str(len(data)))
>         self.end_headers()
>         self.wfile.write(data)
>
>     def log_message(self, *args):
>         pass
>
>
> if __name__ == "__main__":
>     print("Flaky proxy on http://127.0.0.1:8001  ->  Pagebound on port 8000", flush=True)
>     ThreadingHTTPServer(("127.0.0.1", 8001), Proxy).serve_forever()
> ```

And `phone.py` is a pretend app. It taps Borrow **once**, waits 3 seconds for an answer, and retries up to 3 times:

> [!example]- phone.py (new)
> ```python
> """A pretend phone app on bad Wi-Fi. It taps Borrow once, and retries if no answer comes back.
> Usage: python phone.py            (no idempotency key)
>        python phone.py --key      (send one Idempotency-Key per tap)"""
> import json
> import sys
> import urllib.error
> import urllib.request
> import uuid
>
> use_key = "--key" in sys.argv
> tap_key = uuid.uuid4().hex  # Made ONCE for this tap, and reused for every retry
>
>
> def post(path, body):
>     headers = {"Content-Type": "application/json"}
>     if use_key:
>         headers["Idempotency-Key"] = tap_key
>     request = urllib.request.Request("http://127.0.0.1:8001" + path, data=json.dumps(body).encode(),
>                                      method="POST", headers=headers)
>     with urllib.request.urlopen(request, timeout=3) as response:
>         return response.status, json.loads(response.read())
>
>
> for attempt in range(1, 4):
>     try:
>         status, body = post("/borrow", {"book_id": 1, "member": "dana"})
>         print(f"attempt {attempt}: got an answer: {status} {body['result']}")
>         break
>     except (urllib.error.URLError, ConnectionError, OSError) as error:
>         print(f"attempt {attempt}: no answer ({type(error).__name__}), trying again")
>
> with urllib.request.urlopen("http://127.0.0.1:8000/books/1") as response:
>     book = json.loads(response.read())
> print(f"Dana tapped Borrow ONCE. Copies of '{book['title']}' left: {book['available']} (it started at 5)")
> ```

Seed the database, and open three terminals:

```bash
python seed.py
python main.py
```

```bash
python flaky_proxy.py
```

```bash
python phone.py
```

```text
attempt 1: no answer (RemoteDisconnected), trying again
attempt 2: got an answer: 201 ok
Dana tapped Borrow ONCE. Copies of 'Dune' left: 3 (it started at 5)
```

(The proxy terminal also prints `proxy: the reply got lost on the way back`.)

> [!bug] One tap, two books
> *Dune* started with 5 copies. Dana tapped once. It should say **4**. It says **3**. Both attempts reached Pagebound, and Pagebound did the work twice. From the phone's side, "my request was lost" and "my request worked but the reply was lost" look **exactly the same**.

![[p4-retry-lost-reply.svg]]

## The Big Idea

A **distributed system** is several computers working together over a network. It sounds like "one big computer", but it breaks one thing that a single machine mostly gives you for free: **either everything works, or everything fails**.

### Partial failure

On one machine, if the program crashes, it all stops. Across a network, **one part can fail while the rest carries on**, and you can't always tell which part. That is **partial failure**, and it's the heart of why distributed systems are hard.

### The fallacies

In the 1990s engineers at Sun wrote down [eight false beliefs](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) that new distributed-system builders hold. The first few say it all:

1. The network is reliable. *(Dana's train tunnel says no.)*
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

### Slow or dead? You can't tell

Your phone waits 3 seconds and hears nothing. What happened?

| Possibility | What the phone sees |
|---|---|
| The request never arrived | Nothing |
| The server is slow (busy) | Nothing, for a while |
| The server crashed while working | Nothing |
| The server finished, **the reply was lost** | Nothing |

All four look the same: **silence**. A **timeout** is only a *guess* that something is wrong. After a timeout you have three choices: give up, ask again, or ask "did it work?". Asking again is the usual choice, and it's dangerous unless the operation is safe to repeat.

### Idempotency: safe to repeat

An operation is **idempotent** if doing it twice has the same effect as doing it once. "Set the price to 5" is idempotent. "Add 5 to the price" is not. "Borrow a book" is not... unless you **make** it so.

The standard trick is an **idempotency key**:

1. The client invents a random key **once per user action** (one tap = one key).
2. It sends the key with the request, and **reuses the same key for every retry**.
3. The server remembers `key → answer`. A repeated key doesn't run the work again. It **replays the saved answer**.

Payment companies and email providers use this everywhere. The header is usually called `Idempotency-Key`.

### Retry wisely

Retrying is good, but with care: wait a bit longer each time (**exponential backoff**), add a little randomness (**jitter**) so thousands of phones don't retry in the same moment, and **stop after a few tries**. Without limits, a retry storm can finish off a server that was merely struggling.

### Time and order: whose clock?

Two machines never agree exactly on the time, and clocks can jump. So "which happened first?" can't be answered by timestamps from different machines. A **Lamport clock** is a simple fix: each machine keeps a counter. It adds 1 for each event, sends the counter with each message, and when it *receives* one, it sets its counter to `max(own, received) + 1`. Then, if A could have caused B, A's number is **smaller** than B's. It orders *cause and effect*, not real time.

> [!note]- Vector clocks and gossip, in a few words
> Lamport clocks can say "A may have caused B", but not "these two were independent". **Vector clocks** keep one counter per machine, and can tell those cases apart. **Gossip protocols** spread information like rumours: each machine tells a few random others, and soon everybody knows. Both are used in real databases and are covered in the roadmap's advanced items.

## Lab 2 - Fix It

*About 25 minutes.* The server will now honour an `Idempotency-Key`.

**Step 1: the store.** `idem.py` saves the answer for each key in the database. `run_once` returns the saved answer if the key has been seen, and otherwise runs the action and saves its answer.

```python
"""Remember the answer to each Idempotency-Key, so a retry gets the SAME answer and does nothing twice."""
import json
import sqlite3
import threading


class IdempotencyStore:
    def __init__(self, path="pagebound.db"):
        self.path = path
        self.lock = threading.Lock()  # Works inside one process. See the warning in the note.
        db = sqlite3.connect(self.path)
        db.execute("CREATE TABLE IF NOT EXISTS idempotency (key TEXT PRIMARY KEY, status INTEGER, body TEXT)")
        db.commit()
        db.close()

    def run_once(self, key, action):
        """action() returns (status, body). With the same key, it runs only the first time."""
        with self.lock:
            db = sqlite3.connect(self.path, timeout=30)
            try:
                row = db.execute("SELECT status, body FROM idempotency WHERE key = ?", (key,)).fetchone()
                if row is not None:
                    return row[0], json.loads(row[1])  # A retry: replay the saved answer
                status, body = action()
                db.execute("INSERT INTO idempotency VALUES (?, ?, ?)", (key, status, json.dumps(body)))
                db.commit()
                return status, body
            finally:
                db.close()
```

**Step 2: use it in `web.py` for Borrow**, and **Step 3: pass it in from `main.py`**. If a request carries a key, the borrow runs inside `run_once`. Without a key, it works as before.

> [!example]- web.py (changed: make_handler takes idem, and /borrow uses it)
> ```python
> """Web layer: turns HTTP into service calls, and results into HTTP. Nothing else."""
> import json
> import time
> import uuid
> from datetime import date
> from http.server import BaseHTTPRequestHandler
> from urllib.parse import urlparse
>
> import obs
>
> STATUS = {"ok": 201, "none_left": 409, "limit_reached": 409, "no_such_book": 404,
>           "returned": 200, "no_such_loan": 404}
>
>
> def make_handler(service, idem=None):
>     class Handler(BaseHTTPRequestHandler):
>         def reply(self, status, body, content_type="application/json"):
>             data = body.encode() if isinstance(body, str) else json.dumps(body).encode()
>             self.status_sent = status
>             self.send_response(status)
>             self.send_header("Content-Type", content_type)
>             self.send_header("Content-Length", str(len(data)))
>             self.send_header("X-Request-Id", obs.request_id.get())
>             self.end_headers()
>             self.wfile.write(data)
>
>         def observed(self, route):
>             """Give every request an id, time it, log it, and count it."""
>             obs.request_id.set(self.headers.get("X-Request-Id") or uuid.uuid4().hex[:8])
>             began = time.perf_counter()
>             self.status_sent = 500
>             try:
>                 route()
>             finally:
>                 took = time.perf_counter() - began
>                 path = urlparse(self.path).path
>                 label = "/books/{id}" if path.startswith("/books/") else path
>                 obs.metrics.count("requests_total", route=label, status=self.status_sent)
>                 obs.metrics.observe("request_seconds", took, route=label)
>                 obs.log("request", method=self.command, path=path, status=self.status_sent, ms=round(took * 1000))
>
>         def do_GET(self):
>             self.observed(self.route_get)
>
>         def do_POST(self):
>             self.observed(self.route_post)
>
>         def route_get(self):
>             path = urlparse(self.path).path
>             if path == "/healthz":
>                 self.reply(200, {"status": "ok"})
>             elif path == "/metrics":
>                 self.reply(200, obs.metrics.render(), "text/plain")
>             elif path == "/popular":
>                 self.reply(200, service.popular())
>             elif path.startswith("/books/"):
>                 book = service.get_book(int(path.split("/")[-1]))
>                 self.reply(200 if book else 404, book or {"error": "not found"})
>             else:
>                 self.reply(404, {"error": "not found"})
>
>         def route_post(self):
>             size = int(self.headers.get("Content-Length", 0))
>             data = json.loads(self.rfile.read(size) or b"{}")
>             if self.path == "/borrow":
>                 key = self.headers.get("Idempotency-Key")
>                 if key and idem:
>                     # Same key again? Then this is a retry. Replay the first answer, do nothing new.
>                     def borrow_once():
>                         result = service.borrow(int(data["book_id"]), data["member"])
>                         return STATUS[result["result"]], result
>
>                     status, result = idem.run_once(key, borrow_once)
>                     return self.reply(status, result)
>                 result = service.borrow(int(data["book_id"]), data["member"])
>             elif self.path == "/return":
>                 today = date.fromisoformat(data["today"]) if "today" in data else None
>                 result = service.give_back(int(data["loan_id"]), today)
>             else:
>                 return self.reply(404, {"error": "not found"})
>             self.reply(STATUS[result["result"]], result)
>
>         def log_message(self, *args):
>             pass
>
>     return Handler
> ```

```python
"""Wires the layers together, then starts the server."""
import os
from http.server import ThreadingHTTPServer

import config
from cache import TTLCache
from idem import IdempotencyStore
from jobs import JobQueue
from obs import TimedRepo
from repo import SqliteRepo
from service import Lending
from web import make_handler

if __name__ == "__main__":
    queue = JobQueue(config.QUEUE_PATH)

    def notify(member, text):
        queue.put({"member": member, "text": text})  # Instant. A worker sends the email later.

    repo = SqliteRepo(config.DB_PATH)
    if os.environ.get("MYSTERY"):  # Used by a lab, later
        from faults import MysteryRepo
        repo = MysteryRepo(repo)
    service = Lending(TimedRepo(repo), notify, cache=TTLCache(ttl_seconds=config.CACHE_TTL_SECONDS))
    print(f"Pagebound is running at http://{config.HOST}:{config.PORT}  (Ctrl+C to stop)", flush=True)
    handler = make_handler(service, IdempotencyStore(config.DB_PATH))
    ThreadingHTTPServer((config.HOST, config.PORT), handler).serve_forever()
```

**Step 4: test it.** `test_idem.py` checks that the same key runs the action once, and different keys are different taps:

> [!example]- test_idem.py (new)
> ```python
> from idem import IdempotencyStore
>
>
> def test_the_same_key_runs_the_action_only_once(tmp_path):
>     store = IdempotencyStore(str(tmp_path / "idem.db"))
>     calls = []
>
>     def action():
>         calls.append(1)
>         return 201, {"result": "ok", "loan_id": len(calls)}
>
>     first = store.run_once("tap-1", action)
>     second = store.run_once("tap-1", action)   # A retry of the same tap
>     assert first == second == (201, {"result": "ok", "loan_id": 1})
>     assert len(calls) == 1
>
>
> def test_a_different_key_is_a_different_tap(tmp_path):
>     store = IdempotencyStore(str(tmp_path / "idem.db"))
>     calls = []
>     action = lambda: calls.append(1) or (201, {"result": "ok"})
>     store.run_once("tap-1", action)
>     store.run_once("tap-2", action)
>     assert len(calls) == 2
> ```

```bash
python -m pytest -q
```

```text
.....................                                                     [100%]
21 passed in 0.23s
```

**Step 5: replay the incident, now with a key.** Stop the old server, and keep the proxy. (If you restart the proxy, remember it loses the 1st reply and delivers the 2nd, which is what you want.) Seed again, start the server, and run the phone with `--key`:

```bash
python seed.py
python main.py
python phone.py --key
```

```text
attempt 1: no answer (RemoteDisconnected), trying again
attempt 2: got an answer: 201 ok
Dana tapped Borrow ONCE. Copies of 'Dune' left: 4 (it started at 5)
```

> [!success] One tap, one book
> The same two attempts reached Pagebound. The second one carried the **same key**, so the server did not borrow again. It replayed the first answer. **4 copies left.** Dana holds one book, and gets her answer.

**Step 6: what the lock does not protect.** `IdempotencyStore` uses a `threading.Lock`, which only works **inside one process**. If two *copies* of the server get the same key at the same instant, both could see "not seen". The real fix is to let the database decide: insert the key **first** with a `PRIMARY KEY`, and only the request whose insert succeeds does the work. Same lesson as [[37 Concurrency and Parallelism#Locks|locks]] and [[52 Transactions and ACID]], one level up. Try it in Practice.

**Step 7: Lamport clocks.** A short script, `lamport.py`, tells the story of one borrow across three machines:

> [!example]- lamport.py (new)
> ```python
> """Lamport clocks: a way to order events when machines cannot trust each other's wall clocks."""
>
>
> class Node:
>     def __init__(self, name):
>         self.name = name
>         self.clock = 0
>
>     def event(self, what):
>         self.clock += 1
>         print(f"{self.name}  t={self.clock}  {what}")
>
>     def send(self, what):
>         self.clock += 1
>         print(f"{self.name}  t={self.clock}  sends: {what}")
>         return self.clock  # The message carries our clock
>
>     def receive(self, what, their_clock):
>         self.clock = max(self.clock, their_clock) + 1  # Jump past anything we have now "heard about"
>         print(f"{self.name}  t={self.clock}  receives: {what}")
>
>
> phone, server, worker = Node("phone "), Node("server"), Node("worker")
>
> phone.event("member taps Borrow")
> stamp = phone.send("POST /borrow")
> server.receive("POST /borrow", stamp)
> server.event("saves the loan")
> stamp = server.send("job: email the member")
> worker.event("starts up")                  # Happened on its own: it knows nothing about the loan yet
> worker.receive("job: email the member", stamp)
> worker.event("email sent")
> ```

```bash
python lamport.py
```

```text
phone   t=1  member taps Borrow
phone   t=2  sends: POST /borrow
server  t=3  receives: POST /borrow
server  t=4  saves the loan
server  t=5  sends: job: email the member
worker  t=1  starts up
worker  t=6  receives: job: email the member
worker  t=7  email sent
```

Look at the `worker`. When it starts up it has no idea about the loan, so its clock is 1. When the job arrives with `t=5`, it jumps to **6**. After that, "email sent" (7) is correctly ordered after "saves the loan" (4), without any real clock. And `worker t=1` has no order with `phone t=1`: they were independent.

Commit and log it:

```markdown
## 63 - One tap, two loans
- **What broke:** A reply was lost. The phone retried, and the server borrowed twice.
- **Why:** Silence can't tell "lost request" from "lost reply", and Borrow wasn't safe to repeat.
- **Fix:** An Idempotency-Key. Retries replay the first answer. Reproduced with a proxy that drops replies.
```

```bash
git add .
git commit -m "Make Borrow safe to retry with an Idempotency-Key"
```

## Practice

**1. Idempotent?** (a) "Set the member's email to a@b.com", (b) "Add 1 to the like count", (c) "Delete loan 7" (twice), (d) "Create a loan" (with a key).

> [!example]- Answer to question 1
> (a) **Yes.** (b) **No.** (c) **Yes in effect**: after the second call it's still deleted (the second may just say "not found"). (d) **Yes**, when the same key replays the same answer.

**2. Choose the key.** Where should the phone create the idempotency key: when the **user taps**, or when each **retry** happens?

> [!example]- Answer to question 2
> When the user **taps**, once. Retries must **reuse** it. A fresh key per retry would make every retry look new, and you'd be back to double borrows.

**3. Backoff.** 10,000 phones lose connection at once, and all retry after exactly 1 second. What happens, and what two tricks help?

> [!example]- Answer to question 3
> The server gets 10,000 requests at the same moment, which can knock it over. Use **exponential backoff** (wait 1 s, 2 s, 4 s...) and **jitter** (a random extra wait), so that the retries spread out.

**4. Make the key store process-safe.** Change `run_once` so that two server *processes* can't both do the work. What goes first, and what makes it safe?

> [!example]- Answer to question 4
> First `INSERT` the key (with the status "in progress") into a table where the key is a `PRIMARY KEY`. If the insert **fails** because the key exists, the work has been (or is being) done: return the saved answer, or ask the client to wait. If it **succeeds**, do the work, then update the row with the answer. The *database's* uniqueness check is the lock.

**5. Roadmap exercise.** Try the [Gossip Glomers](https://fly.io/dist-sys/) challenges. They're a series of small distributed-system puzzles, with a test harness that injects delays and partitions. Start with "Echo" and "Unique ID Generation".

> [!example]- Answer to question 5
> These are practice problems, not a quiz, and the first two take under an hour. They need the free `maelstrom` tool, which runs on Linux or macOS (use WSL on Windows). The "Unique ID" one is a good test of what you learned about not trusting clocks.

## Can You Do This?

- [ ] Explain partial failure, and why a timeout is only a guess
- [ ] Name three of the fallacies of distributed computing
- [ ] Define idempotent, and give an example and a counter-example
- [ ] Implement an idempotency key, step by step
- [ ] Describe backoff and jitter
- [ ] Explain what a Lamport clock orders, and what it doesn't

## Summary

> [!success] Remember
> - A network gives **partial failure**: part of the system fails, and you can't always tell which.
> - **Silence is ambiguous.** A timeout is a guess.
> - **Make operations idempotent**: one key per user action, reused on retry.
> - Retry with **backoff and jitter**, and a limit.
> - Don't trust clocks across machines. Use logical order for cause and effect.

**Next:** [[64 System Design]] →
