---
title: Caching
tags: [phase-4, performance, caching]
aliases: [Caching, Cache, Cache-Aside, TTL, LRU, Cache Stampede, Invalidation]
---

# Caching

← [[56 Domain-Driven Design]] · [[Phase 4 - Start Here|Home]] · **Next:** [[58 Message Queues]] → · [[Software Engineering Roadmap#Caching|Roadmap card]]

> [!abstract] After this note, you can
> Add a cache with a sensible lifetime, say exactly how stale it may get, avoid a cache stampede, and know what must never be cached. You'll have made [[Pagebound - Season 2|Pagebound's]] busiest page about 50 times faster.

> [!bug] Incident Card · Pagebound Season 2
> **Monday, 08:00.** Marketing sends the newsletter to 5,000 members: *"See what's popular this week!"* The home page calls `GET /popular`. By 08:02 the page takes several seconds, and the support inbox fills.
>
> You open the code. `/popular` runs one big query over **200,000 loans** every time anyone asks. And the answer barely changes in a day.
>
> **Your job:** measure the problem, then stop answering the same question 5,000 times.

## Lab 1 - Break It

*About 10 minutes.* You need the code from [[56 Domain-Driven Design|the last notes]], the database seeded (`python seed.py`), and the server running (`python main.py`).

Create `hammer.py`, a tiny load tester. It sends many requests with several threads at once, and reports the speed.

> [!example]- hammer.py (new)
> ```python
> """A tiny load tester. Usage: python hammer.py URL_PATH [REQUESTS] [THREADS]"""
> import sys
> import threading
> import time
> import urllib.request
>
> path = sys.argv[1]
> total = int(sys.argv[2]) if len(sys.argv) > 2 else 40
> workers = int(sys.argv[3]) if len(sys.argv) > 3 else 10
> url = "http://127.0.0.1:8000/" + path.lstrip("/")
>
> times = []
> errors = []
> lock = threading.Lock()
> next_job = [0]
>
>
> def worker():
>     while True:
>         with lock:
>             if next_job[0] >= total:
>                 return
>             next_job[0] += 1
>         began = time.perf_counter()
>         try:
>             urllib.request.urlopen(url).read()
>             elapsed = time.perf_counter() - began
>             with lock:
>                 times.append(elapsed)
>         except Exception as error:
>             with lock:
>                 errors.append(error)
>
>
> began = time.perf_counter()
> threads = [threading.Thread(target=worker) for _ in range(workers)]
> for thread in threads:
>     thread.start()
> for thread in threads:
>     thread.join()
> wall = time.perf_counter() - began
>
> times.sort()
> p95 = times[int(len(times) * 0.95) - 1] if times else 0
> print(f"{len(times)} ok, {len(errors)} errors in {wall:.1f}s  "
>       f"{len(times) / wall:.0f} req/s  average {sum(times) / max(1, len(times)) * 1000:.0f} ms  p95 {p95 * 1000:.0f} ms")
> ```

Run it twice against `/popular`. Send 40 requests, 10 at a time:

```bash
python hammer.py popular 40 10
python hammer.py popular 40 10
```

```text
40 ok, 0 errors in 0.9s  44 req/s  average 220 ms  p95 289 ms
40 ok, 0 errors in 0.9s  44 req/s  average 218 ms  p95 270 ms
```

(Your numbers will differ. The *shape* will be the same. **p95** means 95 out of 100 requests were faster than this.)

> [!bug] You just measured the problem
> About **44 requests per second**, and each one takes over 200 ms. Five thousand members at once means a queue of seconds. Every request repeats the *same* heavy work.

Stop the server with `Ctrl+C`.

## The Big Idea

A **cache** is a faster place where you keep a *copy* of something that is slow to get. Caches are everywhere:

| Level | What it caches | Example |
|---|---|---|
| CPU | Memory contents | L1/L2 cache ([[35 Computer Organization and Architecture]]) |
| Browser | Files and pages | Images you've seen before |
| CDN | Pages and files near the user | Static files served from a nearby city |
| **Application** | Answers to questions | **What we build today** |
| Database | Recent pages and query plans | Built in, mostly invisible |

![[p4-cache-aside.svg]]

### The most common pattern: cache-aside

1. Ask the cache.
2. **Hit:** return the answer. Done.
3. **Miss:** ask the database, **save the answer in the cache**, return it.

The application is in charge, and the cache is just a helper on the side ("aside"). That's why it's called cache-aside.

| Pattern | How it works | Good for | Watch out |
|---|---|---|---|
| **Cache-aside** | App checks cache, then loads on a miss | Most things; simple | Stale data until it expires |
| **Read-through** | The cache itself loads on a miss | Cleaner app code | Needs a smart cache library |
| **Write-through** | Every write goes to cache **and** database | Reads must always be fresh | Writes are slower |
| **Write-behind** | Write to cache, save to the database later | Very fast writes | You can lose data in a crash |

> [!quote] A famous joke
> "There are only two hard things in computer science: cache invalidation and naming things." Knowing **when a copy is no longer true** is the hard part.

## Lab 2 - Fix It

*About 25 minutes.* You'll build a small cache, plug it in, and measure again.

**Step 1: the cache.** `cache.py` does four jobs: it remembers answers (**TTL**, time to live), forgets old ones (**LRU**, least recently used, when full), counts hits, and is safe when many threads ask at once. Read the comments.

```python
"""A small in-memory cache: expires old entries, forgets the least recently used, and avoids stampedes."""
import threading
import time
from collections import OrderedDict

MISSING = object()


class TTLCache:
    def __init__(self, ttl_seconds, max_items=1000, clock=time.monotonic):
        self.ttl = ttl_seconds
        self.max_items = max_items
        self.clock = clock
        self.items = OrderedDict()   # key -> (expires_at, value), least recently used first
        self.lock = threading.Lock()
        self.key_locks = {}          # One lock per key, so only one thread reloads a missing key
        self.hits = 0
        self.loads = 0

    def _get(self, key):
        with self.lock:
            entry = self.items.get(key)
            if entry is None or entry[0] <= self.clock():
                self.items.pop(key, None)
                return MISSING
            self.items.move_to_end(key)  # Used just now
            return entry[1]

    def _set(self, key, value):
        with self.lock:
            self.items[key] = (self.clock() + self.ttl, value)
            self.items.move_to_end(key)
            while len(self.items) > self.max_items:
                self.items.popitem(last=False)  # Drop the least recently used

    def get_or_load(self, key, loader):
        value = self._get(key)
        if value is not MISSING:
            self.hits += 1
            return value
        with self.lock:
            key_lock = self.key_locks.setdefault(key, threading.Lock())
        with key_lock:  # Only one thread loads. The others wait here, then find it cached.
            value = self._get(key)
            if value is not MISSING:
                self.hits += 1
                return value
            self.loads += 1
            value = loader()
            self._set(key, value)
            return value

    def hit_rate(self):
        total = self.hits + self.loads
        return self.hits / total if total else 0.0

    def clear(self, key):
        with self.lock:
            self.items.pop(key, None)
```

**Step 2: plug it into the service.** Only the `popular` method changes, plus one extra argument. The rules are untouched, and `Lending` still doesn't know *what kind* of cache it has.

> [!example]- service.py (changed: cache argument and popular)
> ```python
> """Use cases: the steps of "borrow" and "return". They follow the rules in domain.py."""
> from datetime import date
>
> import domain
> from ports import LoanRepository, Notifier
>
>
> class Lending:
>     def __init__(self, repo: LoanRepository, send_email: Notifier, today=date.today, cache=None):
>         self.repo = repo
>         self.cache = cache
>         self.send_email = send_email
>         self.today = today
>
>     def borrow(self, book_id, member):
>         if self.repo.get_book(book_id) is None:
>             return {"result": "no_such_book"}
>         if not domain.may_borrow(self.repo.count_active_loans(member)):
>             return {"result": "limit_reached"}
>         if not self.repo.take_copy(book_id):
>             return {"result": "none_left"}
>         due = domain.due_date(self.today())
>         loan_id = self.repo.add_loan(book_id, member, self.today().isoformat(), due.isoformat())
>         self.send_email(member, f"loan {loan_id}: confirmation")
>         return {"result": "ok", "loan_id": loan_id, "due_on": due.isoformat()}
>
>     def give_back(self, loan_id, today=None):
>         loan = self.repo.get_open_loan(loan_id)
>         if loan is None:
>             return {"result": "no_such_loan"}
>         today = today or self.today()
>         due = date.fromisoformat(loan["due_on"])
>         self.repo.close_loan(loan_id, loan["book_id"], today.isoformat())
>         return {
>             "result": "returned",
>             "days_late": domain.days_late(due, today),
>             "fee_cents": domain.late_fee_cents(due, today),
>         }
>
>     def popular(self):
>         if self.cache is None:
>             return self.repo.popular(5)
>         # Cache-aside: look in the cache first. On a miss, ask the database, then remember the answer.
>         return self.cache.get_or_load("popular", lambda: self.repo.popular(5))
>
>     def get_book(self, book_id):
>         return self.repo.get_book(book_id)
> ```

**Step 3: give the cache a 30 second life** when you build the service in `main.py`:

```python
"""Wires the layers together, then starts the server."""
from http.server import ThreadingHTTPServer

from cache import TTLCache
from emailer import send_email
from repo import SqliteRepo
from service import Lending
from web import make_handler

if __name__ == "__main__":
    service = Lending(SqliteRepo("pagebound.db"), send_email, cache=TTLCache(ttl_seconds=30))
    print("Pagebound is running at http://127.0.0.1:8000  (Ctrl+C to stop)")
    ThreadingHTTPServer(("127.0.0.1", 8000), make_handler(service)).serve_forever()
```

**Step 4: run, and measure again.** Seed again if you like, start the server, and run:

```bash
python hammer.py popular 40 10
python hammer.py popular 400 10
```

```text
40 ok, 0 errors in 0.2s  226 req/s  average 43 ms  p95 165 ms
400 ok, 0 errors in 0.2s  2523 req/s  average 4 ms  p95 3 ms
```

> [!success] About 50 times faster
> From **44** to over **2,000 requests per second**, and from 220 ms to 4 ms. The first batch still paid for one real query (some of the 40 requests were waiting for it). After that, almost everything is a **hit**.

**Step 5: meet the price of caching: stale data.** Look at the numbers, then borrow *Dune* as a new member, then look again:

```bash
python tryit.py POST borrow book_id=1 member=newbie1
python tryit.py GET popular
python -c "import sqlite3; print('The database says Dune has', sqlite3.connect('pagebound.db').execute('select count(*) from loans where book_id=1').fetchone()[0], 'loans')"
```

```text
201 {"result": "ok", "loan_id": 200001, "due_on": "2026-12-08"}
200 [{"title": "Dune", "loans": 36270, "readers": 4999}, ...]
The database says Dune has 36271 loans
```

The API still says 36270. The cache is **up to 30 seconds old**. That's a **choice**, and you must be able to say it out loud:

> [!important] State your staleness
> *"The Popular list may be up to 30 seconds out of date. If the cache is empty or lost, the next request takes about 200 ms and refills it."* If you can't say a sentence like that, you don't understand your cache yet. For a "popular this week" list, 30 seconds is fine. For other data it isn't.

### What you must NOT cache

Look at what we cached: the *popular list*. Not `GET /books/1`. That shows how many copies are **available**. If you cached it, two members could see "1 copy left", and both press Borrow. Data that must be **correct right now** (money, stock, permissions) should not sit in a cache. Cache what is **expensive and tolerant of being a little old**.

## Cache Problems You'll Meet

**Invalidation.** Besides waiting for the TTL, you can **delete** the entry when data changes (`cache.clear("popular")`). Or you can add a version to the key. Short TTLs are the simplest safe choice.

**Eviction.** Memory is limited. When the cache is full, **LRU** removes whatever was used least recently. Our `TTLCache(max_items=...)` does that. Other policies exist (least frequently used, random), and LRU is a good default.

**Cache stampede.** A popular entry expires. In the same moment, **twenty requests** all see "missing", and all twenty run the heavy query. The database that the cache was protecting gets hit by 20 at once. Try it. Create `stampede_demo.py`:

> [!example]- stampede_demo.py (new)
> ```python
> """20 requests arrive in the same moment that a cache entry has just expired."""
> import threading
> import time
>
> from cache import TTLCache
>
> loads = []
>
>
> def slow_database_query():
>     loads.append(1)
>     time.sleep(0.2)
>     return ["Dune", "Emma"]
>
>
> def run(read):
>     loads.clear()
>     start = threading.Barrier(20)
>
>     def request():
>         start.wait()
>         read()
>
>     threads = [threading.Thread(target=request) for _ in range(20)]
>     for thread in threads:
>         thread.start()
>     for thread in threads:
>         thread.join()
>     return len(loads)
>
>
> # The simple way: look, and if it is missing, load it. Twenty threads all see "missing".
> naive = {}
>
>
> def naive_read():
>     if "popular" not in naive:
>         naive["popular"] = slow_database_query()
>
>
> print("Without protection, the database was asked", run(naive_read), "times")
>
> # The protected way: one thread loads, the other nineteen wait and then reuse the answer.
> protected = TTLCache(ttl_seconds=30)
> print("With get_or_load, the database was asked", run(lambda: protected.get_or_load("popular", slow_database_query)), "time")
> ```

```bash
python stampede_demo.py
```

```text
Without protection, the database was asked 20 times
With get_or_load, the database was asked 1 time
```

Our `get_or_load` uses **one lock per key**. The first thread loads, the other nineteen wait, and then find the answer in the cache. This is called *single flight*. (The lock only protects one process. With several servers you'd use something shared, like Redis, plus other tricks.)

**Many servers.** An in-memory cache lives inside one process. Run three copies of the app, and you have three separate caches, each with its own staleness. A shared cache such as **Redis** solves that, at the cost of a network call and one more thing to run. Start with the simplest cache that works, and add Redis when you really have several servers.

> [!note]- HTTP caching: let the browser and the CDN do some work
> A response can carry headers: `Cache-Control: max-age=30` means "anyone may reuse this for 30 seconds", and an `ETag` lets a client ask "has it changed?" and get a tiny "no" back. This is the cheapest cache of all, because the request never even reaches your server. Use it for public data like the popular list.

**Step 6: tests that prove it.** `test_cache.py` uses a fake clock, so time can be "moved" without waiting:

> [!example]- test_cache.py (new)
> ```python
> from cache import TTLCache
>
>
> class FakeClock:
>     def __init__(self):
>         self.now = 0.0
>
>     def __call__(self):
>         return self.now
>
>
> def test_second_read_is_a_hit_and_the_loader_runs_once():
>     cache = TTLCache(ttl_seconds=30)
>     calls = []
>     for _ in range(3):
>         cache.get_or_load("popular", lambda: calls.append(1) or ["Dune"])
>     assert len(calls) == 1
>     assert cache.hit_rate() == 2 / 3
>
>
> def test_entries_expire_after_the_ttl():
>     clock = FakeClock()
>     cache = TTLCache(ttl_seconds=30, clock=clock)
>     cache.get_or_load("k", lambda: "old")
>     clock.now = 31
>     assert cache.get_or_load("k", lambda: "new") == "new"
>
>
> def test_least_recently_used_entry_is_dropped_first():
>     cache = TTLCache(ttl_seconds=30, max_items=2)
>     cache.get_or_load("a", lambda: 1)
>     cache.get_or_load("b", lambda: 2)
>     cache.get_or_load("a", lambda: 1)   # Touch "a", so "b" is now the oldest
>     cache.get_or_load("c", lambda: 3)   # Over the limit: "b" must go
>     assert "b" not in cache.items
>     assert "a" in cache.items and "c" in cache.items
> ```

```bash
python -m pytest -q
```

```text
................                                                          [100%]
16 passed in 0.09s
```

Commit it, and add to `LOG.md`:

```bash
git add .
git commit -m "Cache the popular list for 30 seconds"
```

```markdown
## 57 - The home page was slow
- **What broke:** /popular took about 220 ms, and 5,000 members arriving at once made it worse.
- **Why:** Every request ran the same heavy query.
- **Fix:** Cache-aside with a 30 s TTL. About 44 requests per second became over 2,000.
- **Staleness:** up to 30 s. We do NOT cache availability.
```

## Practice

**1. Cache or not?** For each, say yes or no, and how long: (a) the list of all titles, (b) how many copies of *Dune* are available, (c) a member's fee total on the checkout page, (d) the weather widget on the home page.

> [!example]- Answer to question 1
> (a) **Yes**, for minutes or hours: it changes rarely. (b) **No**: it must be right now. (c) **No**, or at most a few seconds: money must be correct. (d) **Yes**, for 10 to 30 minutes.

**2. Stale on purpose.** With a 30 second TTL, a member borrows the last copy of *Dune*. The popular list still shows the old number. Is that a bug?

> [!example]- Answer to question 2
> No, it's the **agreed trade-off**, as long as nothing important depends on that number. Availability is *not* cached, so nobody gets a wrong "you may borrow".

**3. Choose the TTL.** A TTL that's too short and one that's too long each cause a different problem. What are they?

> [!example]- Answer to question 3
> **Too short:** the cache is rarely hit, so the database stays busy and you gain little. **Too long:** people see old data. Pick the longest time that the business can accept.

**4. What if the cache dies?** If the cache is emptied, what happens to the database in the next second?

> [!example]- Answer to question 4
> Every request misses, so all the traffic goes to the database at once. That's a stampede on a big scale. Protect against it with single flight, a warm-up, and by making sure the database can survive the cold moment.

**5. Roadmap exercise.** Measure the hit rate: add a `GET /cache` route that returns `cache.hit_rate()`. Then run `hammer.py` and see what it says.

> [!example]- Answer to question 5
> After the first batch of 40 plus 400 requests, `hit_rate()` is close to **1.0** (for example, 0.99), since only a few requests had to load. Note 59 shows how to expose numbers like this properly in `/metrics`.

## Can You Do This?

- [ ] Explain cache-aside, step by step
- [ ] Choose a TTL, and state the staleness out loud
- [ ] Explain LRU eviction
- [ ] Explain a cache stampede and single flight
- [ ] Say what must never be cached
- [ ] Explain why an in-memory cache is not shared between servers

## Summary

> [!success] Remember
> - A cache is a **faster copy**. It trades **freshness** for **speed**.
> - **Cache-aside:** check the cache, load on a miss, save the result.
> - Always **state how stale** it may get, and what happens if it's lost.
> - **Stampedes:** let one request reload while the others wait.
> - Never cache what must be **right now**.

**Next:** [[58 Message Queues]] →
