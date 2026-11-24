---
title: Database Design
tags: [phase-3, databases]
aliases: [Schema Design, Primary Keys, UUIDs, Migrations]
---

# Database Design

← [[49 Continuous Integration and Delivery]] · [[Phase 3 - Start Here|Home]] · **Next:** [[51 Indexing]] → · [[Software Engineering Roadmap#Database Design|Roadmap card]]

> [!abstract] After this note, you can
> Design schemas that protect your data, choose the right keys and types, and change a live database without downtime.

## The Big Idea

Code can be rewritten in a weekend. **Data lives for years**, and mistakes in its design are painful to fix. You learned the basics of tables, keys and [[29 Relational Databases#Normalization - Store Each Fact Once|normalization]] in Phase 2. This note covers the decisions that professionals make every day.

> [!tip] A useful way to think
> The database is your **last line of defense**. Apps have bugs, and several programs may share one database. If a rule really matters, make the database enforce it.

## Choosing Primary Keys

### Natural or Surrogate?

| | Natural key | Surrogate key |
|---|---|---|
| **What it is** | A real-world value, like an email | A meaningless generated ID |
| **Example** | `sam@example.com` | `4821` |
| **Problem** | Real values change, and aren't always unique | Needs a separate `UNIQUE` rule to prevent duplicates |

**Usually, use a surrogate key, and put a `UNIQUE` constraint on the natural value.**

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("""
    CREATE TABLE users (
        id    INTEGER PRIMARY KEY,
        email TEXT NOT NULL UNIQUE
    )
""")
db.execute("INSERT INTO users (email) VALUES ('sam@example.com')")

# The email can change, and every foreign key stays valid
db.execute("UPDATE users SET email = 'sam@new.example' WHERE id = 1")

try:
    db.execute("INSERT INTO users (email) VALUES ('sam@new.example')")
except sqlite3.IntegrityError as error:
    print(error)
```

Output:

```text
UNIQUE constraint failed: users.email
```

### Numbers or UUIDs?

| | Auto-incrementing number | Random UUID | Time-ordered UUID (UUIDv7) |
|---|---|---|---|
| **Example** | `1, 2, 3` | `9b1d...` | `0190...` |
| **Size** | 4 to 8 bytes | 16 bytes | 16 bytes |
| **Index speed** | Great | Worse, since new keys land everywhere | Good |
| **Reveals how many you have?** | Yes | No | No, but it does reveal creation time |
| **Create without the database?** | No | Yes | Yes |

**A common choice:** numbers internally, and a separate UUID in public URLs, so nobody can guess other records.

```python
import uuid

public_id = uuid.uuid4()
print(len(str(public_id)))
print(public_id.version)
```

Output:

```text
36
4
```

> [!warning] An ID that can't be guessed isn't security
> A hard-to-guess URL doesn't replace [[46 OWASP Top 10#A01 Broken Access Control|checking permissions]].

## Foreign Keys

Foreign keys prevent orphan rows. You also decide what happens when a parent row is deleted.

| Rule | When the parent is deleted | Good for |
|---|---|---|
| `RESTRICT` | Blocks the delete | The safe default: orders and customers |
| `CASCADE` | Also deletes the children | Parts of a whole: an order's items |
| `SET NULL` | Blanks the reference | Optional links: an author who left |

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("PRAGMA foreign_keys = ON")
db.executescript("""
    CREATE TABLE orders (id INTEGER PRIMARY KEY);
    CREATE TABLE items (
        id INTEGER PRIMARY KEY,
        order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE
    );
    INSERT INTO orders VALUES (1);
    INSERT INTO items VALUES (1, 1), (2, 1);
""")

db.execute("DELETE FROM orders WHERE id = 1")
print(db.execute("SELECT COUNT(*) FROM items").fetchone()[0])
```

Output:

```text
0
```

> [!danger] Be careful with CASCADE
> One delete can wipe out thousands of rows. For important data, consider a **soft delete** instead: mark it with a `deleted_at` time, and keep the row.

## Choosing Data Types

### Money

![[float-money.svg]]

```python
price = 0.1
print(sum([price] * 3))
print(sum([price] * 3) == 0.3)

cents = 10
print(sum([cents] * 3) == 30)
```

Output:

```text
0.30000000000000004
False
True
```

**Store whole cents in an integer, or use a `DECIMAL` type. Always store the currency too.**

### Time

```python
from datetime import datetime, timezone, timedelta

now = datetime(2026, 1, 15, 10, 30, tzinfo=timezone.utc)
print(now.isoformat())

colombo = timezone(timedelta(hours=5, minutes=30))
print(now.astimezone(colombo).isoformat())
```

Output:

```text
2026-01-15T10:30:00+00:00
2026-01-15T16:00:00+05:30
```

| Rule | Why |
|---|---|
| **Store events in UTC** | Everyone agrees on one moment, and it isn't affected by daylight saving |
| **Convert only for display** | Each user sees their own local time |
| **Use real date and time types** | You get correct sorting and date math |
| **A future local event is different** | For "9 AM on 5 March in Colombo", also store the time zone, since rules can change |

### Text and Other Types

| Data | Use | Avoid |
|---|---|---|
| **Name, title** | `TEXT`, with a sensible length check | Very short limits that reject real names |
| **Phone, postal code** | `TEXT`, because of leading zeros | Numbers |
| **Yes or no** | `BOOLEAN` | Text like `'Y'` |
| **Status** | `TEXT` with `CHECK`, or an enum | Unexplained numbers like `3` |
| **Email** | Unique, and compared in lowercase | Allowing `A@x.com` and `a@x.com` as different accounts |

## Constraints - Make Bad Data Impossible

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("""
    CREATE TABLE products (
        id          INTEGER PRIMARY KEY,
        sku         TEXT NOT NULL UNIQUE,
        price_cents INTEGER NOT NULL CHECK (price_cents >= 0),
        currency    TEXT NOT NULL CHECK (length(currency) = 3),
        status      TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'retired'))
    )
""")

db.execute("INSERT INTO products (sku, price_cents, currency) VALUES ('T1', 500, 'LKR')")
print(db.execute("SELECT status FROM products").fetchone()[0])

for values in [("T2", -1, "LKR"), ("T3", 100, "RUPEES"), ("T1", 100, "LKR")]:
    try:
        db.execute("INSERT INTO products (sku, price_cents, currency) VALUES (?, ?, ?)", values)
    except sqlite3.IntegrityError as error:
        print(error)
```

Output:

```text
draft
CHECK constraint failed: price_cents >= 0
CHECK constraint failed: length(currency) = 3
UNIQUE constraint failed: products.sku
```

### Rules Across Rows

A **partial unique index** handles rules such as "a user can have only one active subscription".

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.executescript("""
    CREATE TABLE subscriptions (user_id INTEGER, active INTEGER);
    CREATE UNIQUE INDEX one_active ON subscriptions(user_id) WHERE active = 1;
""")

db.execute("INSERT INTO subscriptions VALUES (7, 0)")
db.execute("INSERT INTO subscriptions VALUES (7, 1)")
print("Old inactive plus one active is fine")

try:
    db.execute("INSERT INTO subscriptions VALUES (7, 1)")
except sqlite3.IntegrityError as error:
    print(error)
```

Output:

```text
Old inactive plus one active is fine
UNIQUE constraint failed: subscriptions.user_id
```

## Many-to-Many Relationships

You met [[29 Relational Databases#Many-to-Many|join tables]] already. They can also hold facts about the relationship itself.

```mermaid
erDiagram
    STUDENT ||--o{ ENROLLMENT : has
    COURSE ||--o{ ENROLLMENT : has
    ENROLLMENT {
        int student_id FK
        int course_id FK
        text enrolled_at
        text grade
    }
```

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("""
    CREATE TABLE enrollments (
        student_id  INTEGER NOT NULL,
        course_id   INTEGER NOT NULL,
        enrolled_at TEXT NOT NULL,
        grade       TEXT,
        PRIMARY KEY (student_id, course_id)
    )
""")

db.execute("INSERT INTO enrollments VALUES (1, 7, '2026-01-15', NULL)")

try:
    db.execute("INSERT INTO enrollments VALUES (1, 7, '2026-02-01', NULL)")
except sqlite3.IntegrityError as error:
    print(error)
```

Output:

```text
UNIQUE constraint failed: enrollments.student_id, enrollments.course_id
```

## JSON Columns

Modern databases can store JSON. It's useful, but easy to overuse.

| Good use | Bad use |
|---|---|
| Raw webhook payloads | Fields you filter on all the time |
| Settings that vary per user | Relationships that need foreign keys |
| Attributes that differ per product | Avoiding schema design |

```python
import json
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE events (id INTEGER PRIMARY KEY, payload TEXT)")
db.execute("INSERT INTO events (payload) VALUES (?)", (json.dumps({"type": "paid", "amount": 500}),))

print(db.execute("SELECT json_extract(payload, '$.amount') FROM events").fetchone()[0])
```

Output:

```text
500
```

**The rule of thumb:** if you search it, join on it or need rules on it, give it a real column.

## Safe Schema Changes

Your code and data must change together, while the site stays up. A **migration** is a versioned script that changes the schema. Keep migrations in Git, and run each one only once.

```python
import sqlite3

MIGRATIONS = [
    "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL)",
    "ALTER TABLE users ADD COLUMN email TEXT",
    "CREATE UNIQUE INDEX idx_users_email ON users(email)",
]

def migrate(db):
    version = db.execute("PRAGMA user_version").fetchone()[0]
    for number, sql in enumerate(MIGRATIONS[version:], start=version + 1):
        with db:
            db.execute(sql)
            db.execute(f"PRAGMA user_version = {number}")
        print(f"Applied migration {number}")

db = sqlite3.connect(":memory:")
migrate(db)
migrate(db)
print("Version:", db.execute("PRAGMA user_version").fetchone()[0])
```

Output:

```text
Applied migration 1
Applied migration 2
Applied migration 3
Version: 3
```

Running it a second time does nothing. Real tools include [Alembic](https://alembic.sqlalchemy.org/) and [Flyway](https://documentation.red-gate.com/flyway).

### Expand and Contract

![[expand-contract.svg]]

During a deploy, **old and new code run at the same time**. If you simply rename a column, the old code breaks instantly. So go in safe stages.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, fullname TEXT)")
db.execute("INSERT INTO users (fullname) VALUES ('Sam Perera')")

# Step 1: Expand. Old code still works.
db.execute("ALTER TABLE users ADD COLUMN display_name TEXT")

# Step 2: New code writes to both columns.
db.execute("INSERT INTO users (fullname, display_name) VALUES ('Mei Tan', 'Mei Tan')")

# Step 3: Backfill the old rows.
db.execute("UPDATE users SET display_name = fullname WHERE display_name IS NULL")

# Step 4: New code reads only the new column.
print(db.execute("SELECT display_name FROM users ORDER BY id").fetchall())

# Step 5: Contract, once nothing uses the old column.
db.execute("ALTER TABLE users DROP COLUMN fullname")
print([row[1] for row in db.execute("PRAGMA table_info(users)")])
```

Output:

```text
[('Sam Perera',), ('Mei Tan',)]
['id', 'display_name']
```

### Other Dangers on Large Tables

| Risky change | Why | Safer way |
|---|---|---|
| Adding a `NOT NULL` column with no default | Old rows and old code fail | Add it as nullable, backfill, then add the rule |
| Building an index normally | May block writes | PostgreSQL's `CREATE INDEX CONCURRENTLY` |
| One huge `UPDATE` | Long locks | Batches of a few thousand rows |
| Dropping a column right after a deploy | Old servers still use it | Wait a full release |

## Worked Example - Reviewing a Schema

The roadmap's exercise is to review a schema for integrity problems. How many can you find?

```sql
CREATE TABLE orders (
    customer  TEXT,
    email     TEXT,
    product   TEXT,
    price     REAL,
    qty       INTEGER,
    status    INTEGER,
    date      TEXT
);
```

> [!example]- The review
> 1. No primary key.
> 2. Customer details repeat on every order. Move them to `customers`.
> 3. Product name and price repeat, and there's no link. Add `products`, and an `order_items` table.
> 4. `price REAL` will have rounding errors. Use whole cents.
> 5. An order can have just one product. Use `order_items`.
> 6. `qty` has no check, so -5 is allowed.
> 7. `status` is an unexplained number. Use named values with `CHECK`.
> 8. `date` could be in any format, or empty. Use a timestamp in UTC, and `NOT NULL`.
> 9. Nearly every column allows `NULL`.

```python
import sqlite3

db = sqlite3.connect(":memory:")
db.execute("PRAGMA foreign_keys = ON")
db.executescript("""
    CREATE TABLE customers (
        id    INTEGER PRIMARY KEY,
        email TEXT NOT NULL UNIQUE,
        name  TEXT NOT NULL
    );
    CREATE TABLE products (
        id          INTEGER PRIMARY KEY,
        name        TEXT NOT NULL,
        price_cents INTEGER NOT NULL CHECK (price_cents >= 0)
    );
    CREATE TABLE orders (
        id          INTEGER PRIMARY KEY,
        customer_id INTEGER NOT NULL REFERENCES customers(id),
        status      TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
        created_at  TEXT NOT NULL
    );
    CREATE TABLE order_items (
        order_id         INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
        product_id       INTEGER NOT NULL REFERENCES products(id),
        quantity         INTEGER NOT NULL CHECK (quantity > 0),
        unit_price_cents INTEGER NOT NULL CHECK (unit_price_cents >= 0),
        PRIMARY KEY (order_id, product_id)
    );
""")
print(len(db.execute("SELECT name FROM sqlite_master WHERE type = 'table'").fetchall()))
```

Output:

```text
4
```

Notice `unit_price_cents`. Products' prices change over time, but an old order should keep the price that was actually paid. That's a deliberate copy, not a normalization mistake.

## Common Mistakes

> [!warning] Watch out for these
> - Using `FLOAT` for money
> - Storing local times with no zone
> - Using an email as a primary key
> - Having no constraints, and trusting the app
> - Renaming a column in one risky step
> - Editing a migration that has already run
> - Dumping everything in a JSON column

## Practice

**1. Choose a type.** What would you use for each?
- A product price
- A phone number
- When an order was placed

> [!example]- Answer to question 1
> Integer cents with a currency, or `DECIMAL`. `TEXT`. A timestamp in UTC.

**2. Add the rules.** Write a `reviews` table: each user can review a product once, and stars must be from 1 to 5.

> [!example]- Answer to question 2
> ```python
> import sqlite3
>
> db = sqlite3.connect(":memory:")
> db.execute("""
>     CREATE TABLE reviews (
>         user_id    INTEGER NOT NULL,
>         product_id INTEGER NOT NULL,
>         stars      INTEGER NOT NULL CHECK (stars BETWEEN 1 AND 5),
>         PRIMARY KEY (user_id, product_id)
>     )
> """)
> db.execute("INSERT INTO reviews VALUES (1, 1, 5)")
> try:
>     db.execute("INSERT INTO reviews VALUES (1, 2, 6)")
> except sqlite3.IntegrityError as error:
>     print(error)
> ```
>
> Output:
>
> ```text
> CHECK constraint failed: stars BETWEEN 1 AND 5
> ```

**3. Deletion rules.** Which rule fits best: `RESTRICT`, `CASCADE` or `SET NULL`?
- A user deletes their account, and they have posts you want to keep
- An order is deleted, along with its items
- Someone tries to delete a customer who has unpaid invoices

> [!example]- Answer to question 3
> `SET NULL`, `CASCADE`, `RESTRICT`.

**4. Plan a migration.** You need to make `phone` required, on a table with 50 million rows. List the steps.

> [!example]- Answer to question 4
> 1. Make the app write `phone` on all new rows, and validate it there.
> 2. Backfill old rows in small batches.
> 3. Add the `NOT NULL` rule once every row has a value.

## Can You Do This?

- [ ] Choose between natural keys, numbers and UUIDs
- [ ] Choose `RESTRICT`, `CASCADE` or `SET NULL`
- [ ] Store money and time correctly
- [ ] Use `CHECK`, `UNIQUE` and partial indexes
- [ ] Decide when JSON is acceptable
- [ ] Write versioned migrations
- [ ] Rename a column with no downtime
- [ ] Review a schema, and write a fix-up migration

## Summary

> [!success] Remember
> - **Surrogate key, plus `UNIQUE` on the natural value.**
> - **Money:** whole cents. **Time:** UTC.
> - Let the **database enforce** what really matters.
> - **Expand and contract** for changes with no downtime.

**Next:** [[51 Indexing]] →
