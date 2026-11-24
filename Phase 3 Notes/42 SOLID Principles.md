---
title: SOLID Principles
tags: [phase-3, design]
aliases: [SOLID, Single Responsibility, Dependency Inversion]
---

# SOLID Principles

← [[41 Refactoring]] · [[Phase 3 - Start Here|Home]] · **Next:** [[43 Design Patterns]] → · [[Software Engineering Roadmap#SOLID Principles|Roadmap card]]

> [!abstract] After this note, you can
> Use five design principles to keep code easy to change, and, just as importantly, tell when **not** to use them.

## The Big Idea

Requirements always change. Good design means that a change touches a little code, not everything. **SOLID** is a set of five guidelines for achieving that, collected by Robert C. Martin.

![[solid-overview.svg]]

> [!warning] They are tools, not rules
> Applied everywhere, they create piles of tiny classes that nobody can follow. Use them to fix real pain, or to prepare for a change you can actually see coming. **Simple code you can change later is usually better than clever code built for a future that never comes.**

## S - Single Responsibility

> A class should have **one reason to change**.

Think of a Swiss Army knife. It's handy on a hike, but in a kitchen you want separate tools. If the blade breaks, you don't want to lose the scissors too.

![[srp.svg]]

**The test:** who would ask for this code to change? If the accountants want a new tax rule and the marketing team wants a new email design, those shouldn't be in the same class.

**Before:**

```python
class Invoice:
    def __init__(self, items):
        self.items = items

    def total(self):
        return sum(price for name, price in self.items)

    def to_text(self):                  # Formatting: a different reason to change
        return f"Total: {self.total()}"

    def save(self, path):               # Storage: another reason to change
        with open(path, "w") as file:
            file.write(self.to_text())

print(Invoice([("Tea", 50)]).to_text())
```

**After:**

```python
class Invoice:
    def __init__(self, items):
        self.items = items

    def total(self):
        return sum(price for name, price in self.items)

class InvoiceFormatter:
    def to_text(self, invoice):
        return f"Total: {invoice.total()}"

class InvoiceFile:
    def save(self, text, path):
        with open(path, "w") as file:
            file.write(text)

invoice = Invoice([("Tea", 50), ("Cake", 120)])
text = InvoiceFormatter().to_text(invoice)
InvoiceFile().save(text, "invoice.txt")
print(text)
```

Output:

```text
Total: 170
```

Now the money logic can be tested with no files or formatting. The same goes for functions, too: `calculate_and_save_and_email()` is a warning sign.

**Don't overdo it.** A small script doesn't need three classes. Split when the parts actually change separately.

## O - Open/Closed

> Code should be **open for extension, but closed for modification**.

Think of a power strip. You can plug in a new lamp without rewiring the house.

![[ocp.svg]]

**Before:** each new payment type means editing this function, and risking the ones that already work.

```python
def pay(kind, amount):
    if kind == "card":
        return f"Charged {amount} to card"
    elif kind == "cash":
        return f"Received {amount} in cash"
    raise ValueError(kind)

print(pay("card", 10))
```

**After:** a new payment type is simply a new class. No existing code changes.

```python
class CardPayment:
    def pay(self, amount):
        return f"Charged {amount} to card"

class CashPayment:
    def pay(self, amount):
        return f"Received {amount} in cash"

class CryptoPayment:                # Added later. Nothing else changed.
    def pay(self, amount):
        return f"Sent {amount} in crypto"

def checkout(payment, amount):
    print(payment.pay(amount))

checkout(CardPayment(), 10)
checkout(CryptoPayment(), 10)
```

Output:

```text
Charged 10 to card
Sent 10 in crypto
```

**When not to use it:** you can't predict everything. Wait until the second or third variation appears, then refactor. Two `if` branches are fine.

## L - Liskov Substitution

> A child class must work **anywhere its parent is expected**.

If it looks like a duck and quacks like a duck, but it needs batteries, you have the wrong abstraction. Named after computer scientist Barbara Liskov.

![[lsp.svg]]

**The classic trap:** mathematically, a square is a rectangle. But in code, it breaks a promise.

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def set_width(self, width):
        self.width = width

    def area(self):
        return self.width * self.height

class Square(Rectangle):
    def __init__(self, side):
        super().__init__(side, side)

    def set_width(self, width):       # Must keep the sides equal
        self.width = width
        self.height = width

def stretch(shape):
    shape.set_width(10)
    return shape.area()

print(stretch(Rectangle(2, 3)))
print(stretch(Square(3)))
```

Output:

```text
30
100
```

Code using a `Rectangle` expects that `set_width` leaves the height alone. `Square` breaks that promise, so the caller gets a surprise.

**A fix:** don't use inheritance here. Let them share a small interface.

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

class Square:
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side ** 2

for shape in [Rectangle(2, 3), Square(3)]:
    print(shape.area())
```

Output:

```text
6
9
```

**Warning signs of a violation:**

- A child method raises `NotImplementedError`
- Callers check `isinstance(x, Child)` before calling a method
- A child method does nothing, or does something surprising

## I - Interface Segregation

> Don't force anyone to depend on **methods they don't use**.

A restaurant has separate menus for food, drinks and desserts. You don't need to read the whole book just for a coffee.

![[isp.svg]]

```python
from abc import ABC, abstractmethod

class Printer(ABC):
    @abstractmethod
    def print_page(self, text): ...

class Scanner(ABC):
    @abstractmethod
    def scan(self): ...

class BasicPrinter(Printer):
    def print_page(self, text):
        return f"Printed {text}"

class OfficeMachine(Printer, Scanner):
    def print_page(self, text):
        return f"Printed {text}"

    def scan(self):
        return "Scanned"

print(BasicPrinter().print_page("hi"))
print(OfficeMachine().scan())
```

Output:

```text
Printed hi
Scanned
```

`BasicPrinter` no longer needs a fake `scan()`. And code that only prints can ask for a `Printer`, so it can't call anything else.

> [!tip] Python's lighter version
> Python often doesn't need formal interfaces. Any object with the right method works. This is called **duck typing**. `typing.Protocol` lets you name that expectation without inheritance.

## D - Dependency Inversion

> High-level code shouldn't depend on low-level details. **Both should depend on an abstraction.**

A lamp doesn't care which power station made its electricity. It only knows the wall socket. The socket is the abstraction.

![[dip.svg]]

**Before:** the business logic is tied to one database. You can't test it without that database.

```python
class MySqlDatabase:
    def save(self, order):
        return f"Saved {order} to MySQL"

class OrderService:
    def __init__(self):
        self.db = MySqlDatabase()      # Hard-wired

    def place(self, order):
        return self.db.save(order)

print(OrderService().place("order 1"))
```

**After:** `OrderService` only knows that it needs **something with a `save` method**.

```python
from typing import Protocol

class OrderStore(Protocol):
    def save(self, order): ...

class SqlStore:
    def save(self, order):
        return f"Saved {order} to SQL"

class MemoryStore:
    def __init__(self):
        self.orders = []

    def save(self, order):
        self.orders.append(order)
        return f"Remembered {order}"

class OrderService:
    def __init__(self, store: OrderStore):
        self.store = store

    def place(self, order):
        return self.store.save(order)

print(OrderService(SqlStore()).place("order 1"))

memory = MemoryStore()
print(OrderService(memory).place("order 2"))
print(memory.orders)
```

Output:

```text
Saved order 1 to SQL
Remembered order 2
['order 2']
```

Passing the dependency in like this is called **dependency injection**. See [[43 Design Patterns#Dependency Injection]]. It's what makes code easy to test, because tests can pass a fast fake.

## All Five Together

| Principle | Question to ask | Warning sign |
|---|---|---|
| **S**ingle responsibility | Who would ask for this to change? | The word "and" in a class's description |
| **O**pen/closed | Can I add a case without editing old code? | The same `if` chain growing in many places |
| **L**iskov | Does a child keep its parent's promises? | `isinstance` checks |
| **I**nterface segregation | Does anyone implement fake methods? | `raise NotImplementedError` |
| **D**ependency inversion | Can I test this without a database? | `self.db = RealDb()` inside a constructor |

## When Not to Use Them

| Situation | Better choice |
|---|---|
| A small script or prototype | Keep it simple |
| Only one implementation, and no second in sight | Skip the interface |
| A change you're only guessing about | Wait for real need |
| It takes five files to follow a simple action | You've gone too far |

> [!example] Over-engineering
> `UserFactoryProviderImpl` builds a `UserFactory`, which makes a `User`, which has one field. Each piece is "SOLID", yet the code is much harder to read than `User(name)`.

**The mastery test on the roadmap:** can you argue when **not** to apply one? The best engineers see both the benefit and the cost.

## Common Mistakes

> [!warning] Watch out for these
> - Applying every principle to every class
> - Adding interfaces with only one implementation "just in case"
> - Thinking that SRP means "one method per class"
> - Using inheritance to reuse code, which often breaks Liskov
> - Treating these as laws in code review

## Practice

**1. Which principle?** Which one does each break?
- A `Report` class that builds, saves and emails reports
- `class Penguin(Bird)`, where `fly()` raises an error
- `self.client = StripeClient()` inside `CheckoutService`
- Adding a new shipping type means editing six `if` chains

> [!example]- Answer to question 1
> Single Responsibility, Liskov, Dependency Inversion, Open/Closed.

**2. Refactor.** Make this testable without sending real emails.

```python
class EmailSender:
    def send(self, to, text):
        print(f"Sending a real email to {to}")

class SignupService:
    def signup(self, email):
        EmailSender().send(email, "Welcome!")
        return True

print(SignupService().signup("a@example.com"))
```

> [!example]- Answer to question 2
> ```python
> class FakeEmailSender:
>     def __init__(self):
>         self.sent = []
>
>     def send(self, to, text):
>         self.sent.append(to)
>
> class SignupService:
>     def __init__(self, sender):
>         self.sender = sender
>
>     def signup(self, email):
>         self.sender.send(email, "Welcome!")
>         return True
>
> fake = FakeEmailSender()
> SignupService(fake).signup("a@example.com")
> print(fake.sent)
> ```
>
> Output:
>
> ```text
> ['a@example.com']
> ```

**3. Judgment call.** You have one notification type, email, and no plans for others. Should you add a `Notifier` interface?

> [!example]- Answer to question 3
> Probably not yet, unless you need it for testing. A plain class is simpler, and you can add the interface in minutes when a second type really appears.

**4. Roadmap exercise.** Find a "god class" in your own code, perhaps from [[34 Phase 2 Final Project]]. Split it using these principles. Write down what improved, and whether anything got worse.

## Can You Do This?

- [ ] Explain each letter with your own example
- [ ] Split a class that has many reasons to change
- [ ] Replace a growing `if` chain with new classes
- [ ] Spot a Liskov violation
- [ ] Inject a dependency, and test with a fake
- [ ] Argue when **not** to apply a principle

## Summary

> [!success] Remember
> - **S:** one reason to change.
> - **O:** add new code rather than editing old code.
> - **L:** children keep their parents' promises.
> - **I:** small, focused interfaces.
> - **D:** depend on abstractions, and pass them in.
> - **Use them to reduce pain, not to follow rules.**

**Next:** [[43 Design Patterns]] →
