---
title: Mentoring
tags: [phase-4, leadership, mentoring]
aliases: [Mentoring, Coaching, Code Review Feedback, Onboarding, Delegation, Nudge Ladder]
---

# Mentoring

← [[65 Technical Communication]] · [[Phase 4 - Start Here|Home]] · **Next:** [[67 System Ownership]] → · [[Software Engineering Roadmap#Mentoring|Roadmap card]]

> [!abstract] After this note, you can
> Give feedback that helps someone grow, teach by asking questions, delegate a task that stretches without breaking, and onboard a new person so they're useful in week one. You'll have practised each on a real pull request for [[Pagebound - Season 2|Pagebound]].

> [!bug] Incident Card · Pagebound Season 2
> **Wednesday, 16:00.** A new engineer joined last week. After three quiet days they open a pull request: *"List members with overdue books."* It's large, it runs, and it has several problems. You also know that you could rewrite it in ten minutes.
>
> You write a quick comment: *"This is wrong. Use a set and one query."* An hour later the new engineer replies: *"ok, sorry"*, and changes everything, without understanding why. Two days later they make the *same* kind of mistake again. And they've stopped asking questions.
>
> **Your job:** find out what your comment taught (almost nothing), and learn to give feedback that grows the person.

## Lab 1 - Break It

*About 15 minutes.* Here is the pull request. Create `overdue_pr.py` and run it.

```python
"""A teammate's first pull request: 'list the members who have overdue books'. It works. Mostly."""
from datetime import date


class FakeRepo:
    """Counts how many times the database would be asked."""

    def __init__(self, loans_by_member):
        self.loans_by_member = loans_by_member
        self.calls = 0

    def loans_of(self, member):
        self.calls += 1
        return self.loans_by_member[member]


def overdue_members(repo, members, today):
    result = []
    for member in members:
        loans = repo.loans_of(member)
        for loan in loans:
            if loan["due_on"] < today:
                result.append(member)
    return result


repo = FakeRepo({
    "ann": [{"due_on": date(2026, 3, 1)}, {"due_on": date(2026, 3, 2)}],  # Two overdue books
    "ben": [{"due_on": date(2026, 4, 1)}],                                  # Not overdue
    "cat": [],
})
names = overdue_members(repo, ["ann", "ben", "cat"], today=date(2026, 3, 20))
print("Overdue members:", names)
print("Database calls:", repo.calls)
```

```bash
python overdue_pr.py
```

```text
Overdue members: ['ann', 'ann']
Database calls: 3
```

Two real problems are hiding in that output:

1. **`ann` is listed twice.** She has two overdue books, and the loop adds her once per book. A librarian would send her two letters.
2. **3 database calls for 3 members.** With 5,000 members that's **5,000 calls**. (It's the "N+1" pattern: one call per item instead of one call in total. You saw in [[62 Scalability]] how much each database call costs.)

Now compare **your first comment** with what the new engineer needed:

| Your comment | What the other person heard | What was missing |
|---|---|---|
| "This is wrong." | "I'm bad at this." | **What** is wrong, specifically? |
| "Use a set and one query." | A command, to obey | **Why**? Nothing they can reuse next time |
| (nothing about what is good) | "Nothing here was good" | The things that *work* (it runs, the names are clear) |

> [!bug] Fast for you, slow for them
> The comment saved you ten minutes. It cost the other person confidence, taught them nothing, and guaranteed that you'll write the same comment again next month. **Mentoring is an investment:** a slower start for a faster, more independent teammate later.

## The Big Idea

A mentor's job isn't to give answers. It's to help the other person **become able to find them**. Four habits do most of the work.

### 1. Teach by asking questions

A question lets the person *discover* the problem, and what they discover, they keep.

| Instead of telling... | Try asking... |
|---|---|
| "Use a set." | "Run it with `ann` having two overdue books. What do you see in the output?" |
| "That's slow, do one query." | "If we had 5,000 members, how many times would this ask the database?" |
| "Rename this." | "When you read `x` again in a month, what would you hope to see?" |

### 2. The nudge ladder

If a question isn't enough, go **one step** at a time, and stop as soon as they've got it:

1. **Ask** a question.
2. Give a **hint** ("look at what happens inside the inner loop").
3. Show a **similar example** (not the answer).
4. **Pair**: do the next step together, with *them* typing.
5. Only then, **tell** them. And explain why.

### 3. Feedback that's actionable

Good feedback is **specific**, about **behaviour or code** (not the person), **explains why**, and says what **good looks like**. Say what is good, too. People repeat what you notice.

> [!example] A review comment, rewritten
> **Before:** "This is wrong. Use a set and one query."
>
> **After:** "Nice start: it runs, and the names are clear. 👍 Two things to look at. **1.** Try this with a member who has *two* overdue books. What does the list show? **2.** `loans_of` is called once per member. With 5,000 members, how many database calls is that? Have a go, and ping me if you want to talk it through. I'm free at 3."

### 4. Delegate stretch tasks, with a safety net

Growth happens just beyond someone's comfort. A good **stretch task** is a bit **harder than what they've done**, **important but not urgent**, and has a clear goal, a **check-in rhythm** ("show me your plan on Tuesday"), and **a way out** ("if you're stuck for more than an hour, ask"). Don't take it back when they struggle. That teaches them to wait for rescue.

### Onboarding: the first week

A new person's first week sets the tone. Give them a **path**: a running app on day one, a first small win by day three, and a first real merge by the end of the week. Write it down, so it doesn't depend on your memory (see [[65 Technical Communication]]).

> [!note]- Sponsorship (advanced)
> A **mentor** advises. A **sponsor** speaks up for you when you're not in the room: *"She should lead that project."* Once you're senior, you can sponsor people: put their names forward, give them a visible task, and credit them out loud.

## Lab 2 - Fix It

*About 45 minutes.* Do the four things you'd do for real.

### Step 1: write the review comment again

Use the shape *"what's good, a question, a question, an invitation"*. Check it against the list: Is it **specific**? Does it say **why**? Does it **ask**? Does it say what's **good**? Does it **invite** a talk?

### Step 2: see where the conversation should go

If the new engineer answers your questions and they get there, this is what they might arrive at. It's the same feature, after one good conversation:

```python
"""The same feature after a good review conversation: one question to the database, and no duplicates."""
from datetime import date


class FakeRepo:
    def __init__(self, loans):
        self.loans = loans
        self.calls = 0

    def members_with_loans_due_before(self, day):
        """In real life this is ONE SQL query: SELECT member FROM loans WHERE due_on < ? AND returned_on IS NULL"""
        self.calls += 1
        return [member for member, due_on in self.loans if due_on < day]


def overdue_members(repo, today):
    return sorted(set(repo.members_with_loans_due_before(today)))  # set() removes repeats


repo = FakeRepo([("ann", date(2026, 3, 1)), ("ann", date(2026, 3, 2)), ("ben", date(2026, 4, 1))])
print("Overdue members:", overdue_members(repo, today=date(2026, 3, 20)))
print("Database calls:", repo.calls)
```

```bash
python overdue_fixed.py
```

```text
Overdue members: ['ann']
Database calls: 1
```

`ann` once, and one call instead of one per member. And notice what they *learned*: not "use a set", but **"ask the database once, and check for repeats"**. That idea will serve them for years. (The fake repository only stands in for one SQL query. They'd write that query in `repo.py`, as in [[54 Layered Architecture]].)

> [!tip] Praise in public, criticise in private
> Say "nice work on the tests" in the team channel. Talk about problems in a one-to-one chat, or in a PR comment written kindly. And ask the person *how they want feedback*: some like it right away, some like a note first.

### Step 3: write an onboarding page

Create `docs/ONBOARDING.md`. It's short, and it saves every future newcomer a week of confusion:

```markdown
# Welcome to Pagebound

## Day 1: get it running (about 1 hour)
1. Clone the repo, and run the setup in "Pagebound - Season 2".
2. Run the tests: `python -m pytest -q`. They should all pass.
3. Start the app, and borrow a book with `tryit.py`.

## Day 2-3: learn the shape
- Read GLOSSARY.md (the words we use) and LOG.md (the incidents we've had).
- Read the layers: web.py, service.py, repo.py. Which file would you change to add a new rule?

## First ticket (good first issue)
Add the "overdue members" list. Ask me anything, any time. If you're stuck for more than an hour, tell me.

## How we work
Small pull requests. Tests with every change. Reviews within one working day.
```

### Step 4: plan a three-month mentoring arc

The roadmap exercise is to mentor someone for three months with goals. Start with **a conversation about their goals**, not yours. Then write a plan with a **growth target each month**, for example:

| Month | Goal | A stretch task | How we'll know |
|---|---|---|---|
| 1 | Be comfortable in the codebase | Add the "overdue members" feature | A merged PR with tests |
| 2 | Work independently on a medium task | Add a small endpoint end to end | They run the design talk |
| 3 | Review others, and spot risks | Review two PRs from the team | Their comments are specific and kind |

Check in every week, for 20 minutes: *"What went well? Where are you stuck? What do you want to learn next?"* And ask *them* for feedback on **your** mentoring, once a month.

(You can practise on a real person: a classmate, a newcomer to an open-source project, a friend learning to code. If nobody is around, write the plan for an imaginary teammate and swap with a friend.)

```bash
git add docs
git commit -m "Add the onboarding page"
```

Add to `LOG.md`: *"66: my first review comment was 'this is wrong'. It taught nothing. Fix: what's good, two questions, an invitation. And an onboarding page."*

## Practice

**1. Rewrite it.** "Why didn't you write tests?? Add them." Make it helpful.

> [!example]- Answer to question 1
> For example: "The feature works nicely. 👍 Could you add a test for the case where a member has two overdue books? It would have caught the repeated name, and it'd protect this code when we change it later. Want to pair for 15 minutes on writing it? It's a good pattern to know."

**2. Which rung?** A mentee has been stuck for an hour on a bug. You see the cause. Which rung of the nudge ladder do you start on?

> [!example]- Answer to question 2
> The first: **ask**, such as "What have you tried, and what did you expect to happen?" If they're still stuck, give a hint (the file, or the line), then a similar example, and so on. Stop as soon as they take over. And since they've already been stuck for an hour, don't wait much longer: that's the "way out" you set.

**3. Too much, too soon.** You hand a new engineer the whole "migrate to a new queue" project in their first month, with no check-ins. What might go wrong?

> [!example]- Answer to question 3
> It's too big, and too lonely. They may flounder in silence, or hide their problem. A stretch task should be **a step**, not a leap, with a plan, check-ins and a safety net. Cut it into slices: first "read the current queue code and draw it", then "write the new adapter behind a flag".

**4. Roadmap exercise.** Mentor a junior or an open-source newcomer for three months, with goals. Write the plan from Step 4 for a real person, and ask them what **they** want to be able to do in three months.

> [!example]- Answer to question 4
> Their goals come first. A newcomer may want "make my first open-source contribution", so the monthly goals become: find an issue and read the code, send a small fix, handle review feedback. The measure of success is what *they* can do, not what *you* did.

## Can You Do This?

- [ ] Rewrite a blunt review comment so it teaches
- [ ] Teach by asking questions
- [ ] Use the nudge ladder
- [ ] Give feedback that's specific, kind and actionable
- [ ] Choose and delegate a stretch task with a safety net
- [ ] Write an onboarding page
- [ ] Plan a three-month mentoring arc

## Summary

> [!success] Remember
> - A mentor helps people **find** answers. Don't just hand them over.
> - **Ask first.** Then hint, show an example, pair. Tell them last.
> - Feedback is **specific, kind and explains why**. Notice what's good.
> - **Stretch** people, with a plan, check-ins and a way out.
> - Write down the onboarding. The slow start pays back.

**Next:** [[67 System Ownership]] →
