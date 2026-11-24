---
title: Software Development Lifecycle
tags: [phase-3, process]
aliases: [SDLC, DORA, Environments, Release Management]
---

# Software Development Lifecycle

← [[38 Memory Management]] · [[Phase 3 - Start Here|Home]] · **Next:** [[40 Code Reviews]] → · [[Software Engineering Roadmap#Software Development Lifecycle|Roadmap card]]

> [!abstract] After this note, you can
> Follow a change from idea to production and beyond, explain why teams work in stages, and measure how well a team delivers. You'll also have taken a real ticket through every stage in your own [[Pagebound - The Running Story|Pagebound]] repo.

> [!bug] Incident Card · Pagebound
> **Wednesday, 14:00.** Support has a problem. Members keep forgetting their passwords, and Pagebound has no way to reset one. **About 50 tickets arrive every week**, and each one takes a person 10 minutes to fix by hand.
>
> Your manager stops by your desk: *"Can you fix this by Friday?"*
>
> **Your job:** don't just start typing. Take the ticket through the stages a real team uses. It's less work than it sounds.

## Lab - Walk One Ticket Through Every Stage

*About 40 minutes.* You need your [[Pagebound - The Running Story#Set Up Your Repo|Pagebound repo on GitHub]], and `pytest` (run `python -m pip install pytest`).

### Step 1 - Plan and Requirements: write the Issue

On GitHub, open your repo, go to **Issues**, and click **New issue**. Paste this in:

```markdown
Title: Members can reset a forgotten password

## Why
Support gets about 50 "forgot my password" tickets a week.

## Acceptance criteria
- [ ] A reset token is random and impossible to guess
- [ ] A token stops working after 1 hour
- [ ] A token can be used only once
- [ ] The database never stores the token itself
```

**Acceptance criteria** say when the work is *done*, in a way a test can check.

### Step 2 - Design: one paragraph

Add a comment on the Issue:

```markdown
Design: When a member asks for a reset, we create a long random token and email it.
We store only a hash of the token, plus an expiry time and a "used" flag.
Today I'll build just the token part. The email and web page come in later tickets.
```

Cutting a feature into small **slices** like this is how teams ship safely.

### Step 3 - Build: tests first, on a branch

```bash
git switch -c feature/password-reset
```

Create `test_reset_token.py`. These tests *are* your acceptance criteria, written as code:

```python
from datetime import datetime, timedelta

from reset_token import check_token, new_token, use_token

NOW = datetime(2026, 11, 24, 10, 0)


def test_fresh_token_works():
    token, record = new_token(NOW)
    assert check_token(token, record, NOW + timedelta(minutes=5))


def test_token_expires_after_one_hour():
    token, record = new_token(NOW)
    assert not check_token(token, record, NOW + timedelta(hours=1, minutes=1))


def test_token_cannot_be_used_twice():
    token, record = new_token(NOW)
    use_token(record)
    assert not check_token(token, record, NOW)


def test_wrong_token_is_rejected():
    _, record = new_token(NOW)
    assert not check_token("not-the-real-token", record, NOW)
```

Run them. They **should fail**, because `reset_token.py` doesn't exist yet:

```bash
python -m pytest -q
```

```text
E   ModuleNotFoundError: No module named 'reset_token'
ERROR test_reset_token.py
!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!
```

Now create `reset_token.py`:

```python
import hashlib
import secrets
from datetime import timedelta

LIFETIME = timedelta(hours=1)


def _hash(token):
    return hashlib.sha256(token.encode()).hexdigest()


def new_token(now):
    token = secrets.token_urlsafe(32)  # Long, random, impossible to guess
    record = {"hash": _hash(token), "expires": now + LIFETIME, "used": False}
    return token, record  # Email the token. Store only the record.


def check_token(token, record, now):
    if record["used"] or now > record["expires"]:
        return False
    return secrets.compare_digest(_hash(token), record["hash"])


def use_token(record):
    record["used"] = True
```

Run the tests again:

```bash
python -m pytest -q
```

```text
....                                                                     [100%]
4 passed in 0.02s
```

(Your time will be a little different.) Commit it:

```bash
git add .
git commit -m "Add password reset tokens (#1)"
git push -u origin feature/password-reset
```

### Step 4 - Test and Release: open the pull request

On GitHub, click **Compare & pull request**. Describe it with the template from [[40 Code Reviews#Write a Helpful Description]], and add the line `Closes #1` (your Issue's number). Read your own diff, then click **Squash and merge**. GitHub closes the Issue for you.

Back in your terminal:

```bash
git switch main
git pull
git branch -d feature/password-reset
```

### Step 5 - Operate and Learn: decide how you'll know

Shipping isn't the end. Add two lines to your `LOG.md`:

```markdown
## 39 - Password reset ticket
- **Shipped:** the token part (email and page still to do).
- **How I'll know it worked:** tickets about passwords fall from 50 a week to under 10 within a month.
```

> [!success] You walked the whole loop
> Look at your GitHub Issue, branch and pull request. That trail is what a real team's work looks like.

## The Big Idea

Writing code is only a small part of building software. Someone had to decide what to build. Afterwards, someone must test it, release it, keep it running, fix it and eventually retire it. Together, these stages are called the **Software Development Lifecycle (SDLC)**.

Think about building a house. Pouring the concrete is important, but so are the plans before it and the repairs for decades after it.

![[sdlc-loop.svg]]

> [!important] Most of the cost comes later
> A system may be built in months, and then maintained for ten years. Over its life, far more effort goes into changing and running it than into writing it the first time. So **write code for the person who has to change it.**

## The Stages

You just did every one of these.

| Stage | Question it answers | What you produce | In your lab |
|---|---|---|---|
| **Plan** | Why build this? Is it worth it? | A goal and a rough priority | 50 tickets a week |
| **Requirements** | What must it do? | User stories, acceptance criteria | The Issue |
| **Design** | How will it work? | A design note, diagrams | Your comment |
| **Build** | Write it | Code and tests | The branch |
| **Test** | Does it work? | Passing tests, bug reports | `pytest` |
| **Release** | How does it reach users safely? | A deployed version | The merged PR |
| **Operate** | Is it healthy? | Monitoring, incident fixes | Still to come |
| **Learn** | Did it help? What's next? | Metrics, a new plan | Your `LOG.md` line |

## Why Stages Matter

![[defect-cost.svg]]

A mistake gets more costly the later you find it.

| Found while | Fixing it means |
|---|---|
| Designing | Changing a sentence |
| Coding | Changing a few lines |
| Testing | Debugging, changing, re-testing |
| In production | All of that, plus upset users, an urgent fix, maybe lost data |

> [!tip] This isn't an argument for a long design phase
> Exact ratios vary, and old claims of "100 times the cost" are disputed. The practical lesson is simple: **get fast feedback at every stage**, using small changes, automated tests and early conversations.

## Environments

![[environments.svg]]

| Environment | Purpose | Data |
|---|---|---|
| **Local** | Write and try code | Fake |
| **CI** | Automated checks on each change | Fake |
| **Staging** | Final check, as close to production as possible | Fake, or properly anonymized |
| **Production** | Real users | Real |

> [!danger] Rules for production
> - Never test with real customer data outside production without properly anonymizing it.
> - Keep configuration, such as passwords, separate from code, so the same build runs everywhere.
> - Make environments as similar as you can, to avoid "it worked on staging". [[48 Docker]] helps here.

## Working Methods

The stages don't have to run in one long line. Your lab was a tiny **iterative** cycle: all the stages, in one small slice.

> [!note]- Waterfall, Agile and continuous delivery compared
> | Method | How it works | Good when |
> |---|---|---|
> | **Waterfall** | Finish each stage fully before the next | Requirements are fixed and mistakes are very costly, such as in some medical or aerospace work |
> | **Iterative, Agile** | Go through all the stages in small cycles | Requirements change, and you can release often |
> | **Continuous delivery** | Each small change can flow to production | Mature teams with strong automation |
>
> ```text
> Waterfall:   Requirements → Design → Build → Test → Release (one big release)
>
> Iterative:   [Plan Build Test Release] → [Plan Build Test Release] → ...
> ```
>
> There's more on the [[Software Engineering Roadmap#Development Methodologies|Development Methodologies]] and [[Software Engineering Roadmap#Agile and Scrum|Agile and Scrum]] cards.

## Releasing

**Deploying** puts code on servers. **Releasing** makes a feature available to users. Separating them is a superpower, as you'll see with [[49 Continuous Integration and Delivery#Feature Flags|feature flags]].

### Version Numbers

**Semantic versioning** tells users what to expect from an update. It's the same numbering as your Pagebound release tags, such as `v0.1`.

```text
     2 . 4 . 1
     │   │   └── PATCH: bug fixes only
     │   └────── MINOR: new features, nothing broken
     └────────── MAJOR: breaking changes
```

```python
def bump(version, part):
    major, minor, patch = map(int, version.split("."))
    if part == "major":
        return f"{major + 1}.0.0"
    if part == "minor":
        return f"{major}.{minor + 1}.0"
    return f"{major}.{minor}.{patch + 1}"

print(bump("2.4.1", "patch"))
print(bump("2.4.1", "minor"))
print(bump("2.4.1", "major"))
```

Output:

```text
2.4.2
2.5.0
3.0.0
```

### A Good Release Checklist

- [ ] All tests pass
- [ ] Database changes are [[50 Database Design#Safe Schema Changes|backward compatible]]
- [ ] You know how to roll back
- [ ] Dashboards are ready to watch
- [ ] Release notes are written

## Maintenance and Retirement

Software needs upkeep, even if no features are added. **Deprecation** means announcing that something will go away, while it still works. Do it kindly: announce a date, offer a replacement, warn in the logs, and watch usage until it's near zero.

> [!note]- Four kinds of maintenance
> | Kind of maintenance | Example |
> |---|---|
> | **Corrective** | Fixing bugs |
> | **Adaptive** | Upgrading to a new operating system or library |
> | **Perfective** | Improving speed or usability |
> | **Preventive** | [[41 Refactoring]], paying down [[Software Engineering Roadmap#Technical Debt\|technical debt]] |

## Incidents and Postmortems

Things will break in production. A healthy team asks **"what made this possible?"**, and not "who did this?". That's a **blameless postmortem**.

```text
Incident:   Checkout down for 25 minutes
Timeline:   14:02 deploy, 14:05 errors rise, 14:11 alert, 14:27 rollback
Impact:     About 300 failed orders
Root cause: A migration locked the orders table
What went well:  Rollback was quick
What went badly: The alert took 6 minutes
Actions:
  1. Add a check for locking migrations in CI  (owner: Sam, due: Friday)
  2. Tighten the alert to 2 minutes            (owner: Priya, due: Friday)
```

Your `LOG.md` entries are tiny postmortems. Keep going, and they become a habit.

## Measuring Delivery - DORA

![[dora.svg]]

Years of research from the [DORA](https://dora.dev/) program found that four numbers predict how well a team delivers. **The surprise:** fast teams are also the most stable. Small, frequent releases are less risky than big, rare ones, because less changes at once.

> [!note]- The four numbers, and how to measure them
> | Metric | Elite teams, roughly | Struggling teams, roughly |
> |---|---|---|
> | **Deployment frequency** | Many times a day | Monthly or less |
> | **Lead time for changes** | Under a day | Months |
> | **Change failure rate** | Low | High |
> | **Time to restore** | Under an hour | Days or weeks |
>
> ```python
> from statistics import median
>
> deploys = [
>     {"lead_hours": 4, "failed": False},
>     {"lead_hours": 30, "failed": True},
>     {"lead_hours": 6, "failed": False},
>     {"lead_hours": 8, "failed": False},
> ]
>
> print("Deploys:", len(deploys))
> print("Median lead time:", median(d["lead_hours"] for d in deploys), "hours")
> print("Change failure rate:", sum(d["failed"] for d in deploys) / len(deploys))
> ```
>
> Output:
>
> ```text
> Deploys: 4
> Median lead time: 7.0 hours
> Change failure rate: 0.25
> ```

> [!warning] Metrics can be gamed
> Use these to help the team improve, not to rank people. Once a number becomes a target, people find ways to move it without improving anything.

### Finding Your Bottleneck

The roadmap's mastery test asks you to name what most limits delivery, with evidence. Measure where a change spends its time.

```text
Write code      ████ 1 day
Wait for review ████████████ 3 days     ← the bottleneck
Review          ██ half a day
Wait for test   ████████ 2 days         ← next
Deploy          █ an hour
```

Writing code faster wouldn't help this team at all. Quicker reviews would.

## Common Mistakes

> [!warning] Watch out for these
> - Thinking the job ends when the code merges
> - Testing with real customer data
> - Releasing big batches every few months
> - Skipping the "learn" stage, so nobody knows if the feature helped
> - Blaming people for incidents, so that problems get hidden

## Practice

**1. Which stage?** A user reports that a page is slow. You check dashboards and open a ticket. Which stage is this, and which stage does it feed?

> [!example]- Answer to question 1
> It's **Operate**. It feeds **Learn** and **Plan**, which is why the lifecycle is a loop.

**2. Version numbers.** Your library is at 1.3.2. Which version comes next after each change?
- You fix a small bug
- You add a new optional function
- You remove an old function

> [!example]- Answer to question 2
> 1.3.3, 1.4.0, 2.0.0.

**3. Find the bottleneck.** Coding takes 1 day, waiting for QA takes 5 days, and deploying takes 10 minutes. What would you improve first?

> [!example]- Answer to question 3
> The wait for QA. For example, automate more of the tests so that changes don't queue up.

**4. Write a postmortem.** Think of a time your program lost your work, or a bug caught you out. Write a short blameless postmortem with at least two actions.

**5. Next slice.** Your lab shipped only the token. Write the next two Issues (with acceptance criteria) that would finish the password reset feature.

> [!example]- Answer to question 5
> For example: **"Email the reset link"** (the screen says the same thing whether or not the account exists, so attackers can't check who has one) and **"Reset page"** (it accepts a new password only with a valid token, then marks the token used). Each is small enough to build, review and merge in a day.

**6. Roadmap exercise.** Take a small app through every stage, and write a sentence or two on each stage, including one real fix after release. Your lab already covers most of it. Add the "fix after release" part.

## Can You Do This?

- [ ] Take a ticket from Issue to merged pull request
- [ ] Write acceptance criteria that a test can check
- [ ] Draw the lifecycle as a loop
- [ ] Explain why late bugs cost more
- [ ] Explain each environment
- [ ] Explain semantic versioning
- [ ] Write a blameless postmortem
- [ ] Name the four DORA metrics
- [ ] Find the bottleneck in a team's delivery

## Summary

> [!success] Remember
> - Software lives in a **loop**, and most of its cost comes after release.
> - **Fast feedback** at every stage beats long plans.
> - **Small, frequent releases** are safer. Cut big features into slices.
> - Measure with **DORA**, and fix the biggest bottleneck first.

**Next:** [[40 Code Reviews]] →
