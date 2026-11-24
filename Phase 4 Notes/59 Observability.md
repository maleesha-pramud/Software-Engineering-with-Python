---
title: Observability
tags: [phase-4, operations, observability]
aliases: [Observability, Logging, Metrics, Tracing, Structured Logs, Request ID, OpenTelemetry]
---

# Observability

← [[58 Message Queues]] · [[Phase 4 - Start Here|Home]] · **Next:** [[60 Cloud Computing]] → · [[Software Engineering Roadmap#Observability|Roadmap card]]

> [!abstract] After this note, you can
> Add structured logs, metrics and a request id to an app, and find a hidden slow step using only what the app tells you. You'll have solved a mystery slowdown in [[Pagebound - Season 2|Pagebound]] without adding any new code to the problem.

> [!bug] Incident Card · Pagebound Season 2
> **Tuesday, 17:40.** Three members write in: *"Borrow is **sometimes** really slow."* Support can't reproduce it. Your boss asks the question that matters:
>
> *"Which step is slow, and how often?"*
>
> You open the terminal where the server runs. It's empty. The app says nothing about itself.
>
> **Your job:** make the app explain itself, then use that to find the slow step.

## Lab 1 - Break It

*About 10 minutes.* You'll start the app with a **hidden problem**. Create `faults.py`, and **don't read it**. Reading it would spoil the game.

> [!example]- faults.py (don't peek!)
> ```python
> """A hidden bug for the lab. Please do NOT read this until you have found the problem from the logs."""
> import time
>
>
> class MysteryRepo:
>     def __init__(self, repo):
>         self._repo = repo
>         self.calls = 0
>
>     def __getattr__(self, name):
>         attribute = getattr(self._repo, name)
>         if name != "add_loan":
>             return attribute
>
>         def sometimes_slow(*args, **kwargs):
>             self.calls += 1
>             if self.calls % 4 == 0:
>                 time.sleep(0.4)
>             return attribute(*args, **kwargs)
>
>         return sometimes_slow
> ```

In `main.py` (the version from [[58 Message Queues]]), change the line that builds the service, so that it uses the hidden fault when you ask for it:

```python
    repo = SqliteRepo("pagebound.db")
    if os.environ.get("MYSTERY"):  # Turns the hidden problem on
        from faults import MysteryRepo
        repo = MysteryRepo(repo)
    service = Lending(repo, notify, cache=TTLCache(ttl_seconds=30))
```

(Add `import os` at the top. You'll rewrite this file in a moment.) Seed the database, start the server with the problem on, and borrow 12 books:

Git Bash or Linux:

```bash
python seed.py
MYSTERY=1 python main.py
```

PowerShell:

```powershell
python seed.py
$env:MYSTERY = "1"; python main.py
```

In another terminal:

```bash
python borrow_speed.py 12
```

```text
Average Borrow time: 132 ms
```

The average went from 30 ms to 132 ms. But **where**? Look at the server terminal: nothing. Try to answer these from what you can see:

- Which of the four database calls in `borrow` is slow?
- Is it slow every time, or only sometimes?
- Which request was the slowest one, and what did it do?

> [!bug] You are blind
> You can't answer a single question. You could *guess* and change code, or add `print` lines and redeploy. That's slow and risky. **Observability** means you can answer such questions from the outside, **without shipping new code**.

Stop the server.

## The Big Idea

**Observability** is how well you can understand what's happening *inside* a system from what it sends *out*, including for failures you didn't predict. Three kinds of signals do most of the work:

![[p4-three-signals.svg]]

| Signal | What it is | Good for | Weak at |
|---|---|---|---|
| **Logs** | Events, one line each | The story of one request; odd errors | Counting and trends, if unstructured |
| **Metrics** | Numbers over time | Dashboards and alerts: rates, errors, latency | The detail of one request |
| **Traces** | One request followed across steps and services | "Where did the time go?" | Cost; usually *sampled* |

> [!tip] Monitoring vs observability
> **Monitoring** watches for problems you already know about ("alert if errors go above 1%"). **Observability** lets you ask *new* questions when something odd happens. You need both.

### Good logs are structured

A bad log: `Borrow took long for sam`. You can't search or count it. A good log is one **JSON object per line**, with named fields:

```json
{"ts": 1795543943.973, "event": "repo_call", "request_id": "dbe87cad", "method": "add_loan", "ms": 413}
```

Now a machine can filter `method == "add_loan"` and compute averages.

### The request id: your thread through the maze

Give every request an id, put it on **every** log line, and send it back in a response header (`X-Request-Id`). To follow one request, search for its id. That's a tiny **trace**. Real systems pass such an id between services (the standard header is `traceparent`), and tools such as **OpenTelemetry** gather it into proper traces.

### Metrics: counts, and times

- A **counter** only goes up: `requests_total{route="/borrow",status="201"}`.
- A **timing** records how long things took. We store the count, the sum, the maximum, and **buckets** (how many were under 50 ms, 100 ms, and so on). From buckets you can estimate percentiles such as p95.

> [!warning] Watch the cardinality
> Every different label value makes a **new series**. `route="/borrow"` is fine, since there are a few routes. `member="sam"` is **not**: 5,000 members make 5,000 series, and your metrics system slows to a crawl. Put high-detail values (member, request id) in **logs**, and keep metric labels to a small set.

## Lab 2 - Fix It

*About 30 minutes.* Make the app talk, then solve the case.

**Step 1: `obs.py`**, the toolbox. `log()` prints one JSON line with the request id. `Metrics` keeps counters and timings, and prints them in the plain-text format that [Prometheus](https://prometheus.io/docs/introduction/overview/) reads. `TimedRepo` wraps *any* repository and times every call. That means the service and the repository don't change at all. Observability is one more **adapter**.

> [!example]- obs.py (new)
> ```python
> """Observability in one small file: structured logs, metrics, and a timing wrapper."""
> import contextvars
> import json
> import threading
> import time
>
> request_id = contextvars.ContextVar("request_id", default="-")
>
>
> def log(event, **fields):
>     """One JSON object per line, so a machine can search and count them."""
>     print(json.dumps({"ts": round(time.time(), 3), "event": event, "request_id": request_id.get(), **fields}),
>           flush=True)
>
>
> class Metrics:
>     BUCKETS = (0.05, 0.1, 0.5, 1.0)
>
>     def __init__(self):
>         self.lock = threading.Lock()
>         self.counters = {}
>         self.timings = {}   # (name, labels) -> {"count", "sum", "max", "buckets": [..]}
>
>     def count(self, name, **labels):
>         with self.lock:
>             key = (name, tuple(sorted(labels.items())))
>             self.counters[key] = self.counters.get(key, 0) + 1
>
>     def observe(self, name, seconds, **labels):
>         with self.lock:
>             key = (name, tuple(sorted(labels.items())))
>             t = self.timings.setdefault(key, {"count": 0, "sum": 0.0, "max": 0.0, "buckets": [0] * len(self.BUCKETS)})
>             t["count"] += 1
>             t["sum"] += seconds
>             t["max"] = max(t["max"], seconds)
>             for index, limit in enumerate(self.BUCKETS):
>                 if seconds <= limit:
>                     t["buckets"][index] += 1
>
>     def render(self):
>         """The plain-text format that tools like Prometheus read."""
>         def fmt(labels, extra=""):
>             parts = [f'{k}="{v}"' for k, v in labels] + ([extra] if extra else [])
>             return "{" + ",".join(parts) + "}" if parts else ""
>
>         lines = []
>         with self.lock:
>             for (name, labels), value in sorted(self.counters.items()):
>                 lines.append(f"{name}{fmt(labels)} {value}")
>             for (name, labels), t in sorted(self.timings.items()):
>                 for limit, bucket in zip(self.BUCKETS, t["buckets"]):
>                     le = 'le="%s"' % limit
>                     lines.append(f"{name}_bucket{fmt(labels, le)} {bucket}")
>                 lines.append(f"{name}_count{fmt(labels)} {t['count']}")
>                 lines.append(f"{name}_sum{fmt(labels)} {t['sum']:.3f}")
>                 lines.append(f"{name}_max{fmt(labels)} {t['max']:.3f}")
>         return "\n".join(lines) + "\n"
>
>
> metrics = Metrics()
>
>
> class TimedRepo:
>     """Wraps any repo. Every call is timed, counted and logged. The service never knows."""
>
>     def __init__(self, repo):
>         self._repo = repo
>
>     def __getattr__(self, name):
>         attribute = getattr(self._repo, name)
>         if not callable(attribute):
>             return attribute
>
>         def timed(*args, **kwargs):
>             began = time.perf_counter()
>             try:
>                 return attribute(*args, **kwargs)
>             finally:
>                 took = time.perf_counter() - began
>                 metrics.observe("repo_call_seconds", took, method=name)
>                 log("repo_call", method=name, ms=round(took * 1000))
>
>         return timed
> ```

**Step 2: `web.py`.** Every request gets an id, a timer, a log line and a metric. Two routes are new: `/healthz` (is it alive?) and `/metrics`.

> [!example]- web.py (changed)
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
> def make_handler(service):
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

**Step 3: `main.py`.** It wraps the repo in `TimedRepo`, and still has the hidden-problem switch.

```python
"""Wires the layers together, then starts the server."""
import os
from http.server import ThreadingHTTPServer

from cache import TTLCache
from jobs import JobQueue
from obs import TimedRepo
from repo import SqliteRepo
from service import Lending
from web import make_handler

if __name__ == "__main__":
    queue = JobQueue("queue.db")

    def notify(member, text):
        queue.put({"member": member, "text": text})  # Instant. A worker sends the email later.

    repo = SqliteRepo("pagebound.db")
    if os.environ.get("MYSTERY"):  # Used by a lab, later
        from faults import MysteryRepo
        repo = MysteryRepo(repo)
    service = Lending(TimedRepo(repo), notify, cache=TTLCache(ttl_seconds=30))
    print("Pagebound is running at http://127.0.0.1:8000  (Ctrl+C to stop)", flush=True)
    ThreadingHTTPServer(("127.0.0.1", 8000), make_handler(service)).serve_forever()
```

**Step 4: two small tools** that read the log. `logsummary.py` groups timings by step. `trace.py` follows one request.

> [!example]- logsummary.py (new)
> ```python
> """Read server.log and answer: which step is slow? Usage: python logsummary.py [server.log]"""
> import json
> import re
> import sys
> from collections import defaultdict
>
> path = sys.argv[1] if len(sys.argv) > 1 else "server.log"
> steps = defaultdict(list)
> for line in open(path, encoding="utf-8"):
>     if not line.startswith("{"):
>         continue
>     entry = json.loads(line)
>     if entry["event"] == "repo_call":
>         steps["repo." + entry["method"]].append(entry["ms"])
>     elif entry["event"] == "request":
>         steps[f"{entry['method']} " + re.sub(r"/\d+", "/{id}", entry["path"])].append(entry["ms"])
>
> print(f"{'step':28} {'calls':>6} {'average ms':>11} {'slowest ms':>11}")
> for name, times in sorted(steps.items(), key=lambda item: -max(item[1])):
>     print(f"{name:28} {len(times):>6} {sum(times) / len(times):>11.0f} {max(times):>11}")
> ```

> [!example]- trace.py (new)
> ```python
> """Follow ONE request through the log. Usage: python trace.py [REQUEST_ID]   (default: the slowest one)"""
> import json
> import sys
>
> path = "server.log"
> entries = [json.loads(line) for line in open(path, encoding="utf-8") if line.startswith("{")]
> requests = [e for e in entries if e["event"] == "request"]
>
> wanted = sys.argv[1] if len(sys.argv) > 1 else max(requests, key=lambda e: e["ms"])["request_id"]
> print(f"Request {wanted}:")
> for entry in entries:
>     if entry["request_id"] == wanted:
>         what = entry.get("method") if entry["event"] == "repo_call" else f"{entry['method']} {entry['path']} -> {entry['status']}"
>         print(f"  {entry['event']:10} {what:28} {entry['ms']:>5} ms")
> ```

**Step 5: run the mystery again, with the output saved to a file.** Seed first.

Git Bash or Linux:

```bash
python seed.py
MYSTERY=1 python main.py > server.log
```

PowerShell:

```powershell
python seed.py
$env:MYSTERY = "1"; python main.py | Out-File -Encoding utf8 server.log
```

In another terminal, borrow 12 books, then stop the server and ask the log:

```bash
python borrow_speed.py 12
python logsummary.py
```

```text
Average Borrow time: 132 ms
step                          calls  average ms  slowest ms
POST /borrow                     12         130         439
repo.add_loan                    12         108         416
repo.count_active_loans          12          11          12
repo.take_copy                    12           5           8
repo.get_book                    12           1           2
```

> [!success] Case solved from the outside
> The table sorts by the slowest step. `POST /borrow` takes up to 439 ms, and **`repo.add_loan`** accounts for 416 ms of it. The other three database calls are all quick. You didn't read any code or add a `print`.

Now look at the slowest request alone:

```bash
python trace.py
```

```text
Request d5d8d85b:
  repo_call  get_book                         1 ms
  repo_call  count_active_loans              12 ms
  repo_call  take_copy                        5 ms
  repo_call  add_loan                       416 ms
  request    POST /borrow -> 201            439 ms
```

The average for `add_loan` is **108 ms**, but the slowest is **416 ms**. That gap is the clue: it's **slow only sometimes**. Averages hide this, and the *maximum* and the *distribution* show it. (Look at `GET /metrics` too: the `repo_call_seconds_bucket` lines count how many `add_loan` calls were under 50 ms, and how many needed more.)

```bash
python tryit.py GET metrics
```

```text
200 requests_total{route="/borrow",status="201"} 12
repo_call_seconds_bucket{method="add_loan",le="0.05"} 9
repo_call_seconds_bucket{method="add_loan",le="0.5"} 12
repo_call_seconds_count{method="add_loan"} 12
repo_call_seconds_sum{method="add_loan"} 1.293
repo_call_seconds_max{method="add_loan"} 0.417
```

**Step 6: now peek** at `faults.py`. The mystery repo makes every **4th** call to `add_loan` sleep for 0.4 seconds. In real life, the same pattern could be a lock held too long, a disk that stalls now and then, or garbage collection. Your logs found it without knowing the cause.

**Step 7: prove it's gone.** Run the server *without* the problem, borrow again, and compare:

```bash
python seed.py
python main.py > server.log
python borrow_speed.py 12
```

```text
Average Borrow time: 28 ms
```

Back to normal. Add to `LOG.md`, and commit:

```markdown
## 59 - Borrow was sometimes slow, and we were blind
- **What broke:** Some borrows took 400+ ms. Support couldn't reproduce it.
- **Why:** One step (add_loan) stalled on every 4th call, and the app reported nothing.
- **Fix:** Structured logs, a request id, and metrics at /metrics. Found the slow step in a minute, with no new code.
```

```bash
git add .
git commit -m "Add structured logs, request ids and metrics"
```

> [!note]- What real tools add
> **OpenTelemetry** is the standard way to produce logs, metrics and traces in many languages. **Prometheus** collects metrics, and **Grafana** draws dashboards. **Jaeger** and **Tempo** store traces. Cloud providers have their own. All of them rely on the *same* ideas you just built: structured events, an id that follows the request, and numbers with a few labels. **Sampling** keeps only some traces (say 1 in 100), so that the cost stays small. **Continuous profiling** goes further and shows which *line of code* uses the time.

## Practice

**1. Which signal?** (a) "Did error rates rise after the release?" (b) "What exactly happened to member sam's failed borrow at 14:03?" (c) "Where inside one slow request did the time go?"

> [!example]- Answer to question 1
> (a) **Metrics**: a trend over time. (b) **Logs**: search by member in the details. (c) **Traces**: the steps of one request. (A request-id search in logs can stand in for a trace in a small app.)

**2. Fix the label.** Someone adds `member` as a metric label, to see "borrows per member". What goes wrong?

> [!example]- Answer to question 2
> High cardinality: 5,000 members become 5,000 time series per metric. Memory and cost explode. Count borrows by *route and status*. Put the member in the **log line**, and search there.

**3. Why the max?** The average of `add_loan` was 108 ms, and `p95` and `max` told a different story. Why are averages dangerous?

> [!example]- Answer to question 3
> An average mixes fast and slow requests into one comfortable-looking number. The slow ones are the pain that users feel. Look at **percentiles** (p95, p99) and the **maximum**.

**4. Add a field.** You want to know how long the *whole* `Lending.borrow` takes, not just each repo call. Where would you add a timer, and why not inside `service.py`?

> [!example]- Answer to question 4
> In `web.py`: it already times the whole request, and `/borrow` is the borrow. Or wrap the service in the same way `TimedRepo` wraps the repo. Not inside `service.py`: the core must not import `obs` (the guard test from [[55 Clean Architecture]] would fail).

**5. Roadmap exercise.** Hide a different fault: make `take_copy` slow for one book only. Find it using only the logs and metrics. What do you search for?

> [!example]- Answer to question 5
> `logsummary.py` shows `repo.take_copy` with a high maximum. To find *which book*, add `book_id` to the `repo_call` log line (logs can hold detail), then filter. This is **high-cardinality exploration** done in logs, not in metrics.

## Can You Do This?

- [ ] Name the three signals, and what each one is best at
- [ ] Write a structured log line, and explain why it beats plain text
- [ ] Add a request id and follow one request
- [ ] Explain why averages mislead, and what to look at instead
- [ ] Explain metric cardinality
- [ ] Find a slow step from the outside, with no new code

## Summary

> [!success] Remember
> - **Logs** tell stories, **metrics** show shapes, **traces** show paths.
> - Make logs **structured**, and put a **request id** on every line.
> - Keep metric labels **few**. Put the detail in logs.
> - Look at **p95 and max**, not only the average.
> - You can't fix what you can't see. Build the eyes **before** the incident.

**Next:** [[60 Cloud Computing]] →
