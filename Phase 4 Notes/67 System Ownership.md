---
title: System Ownership
tags: [phase-4, operations, reliability]
aliases: [System Ownership, On-Call, Incident Response, Postmortem, SLO, SLI, SLA, Error Budget, Runbook]
---

# System Ownership

← [[66 Mentoring]] · [[Phase 4 - Start Here|Home]] · **Next:** [[68 Phase 4 Final Project]] → · [[Software Engineering Roadmap#System Ownership|Roadmap card]]

> [!abstract] After this note, you can
> Set a service level objective (SLO) and watch its error budget, respond to an incident calmly, write a blameless postmortem, and keep a runbook. You'll have run a fire drill on [[Pagebound - Season 2|Pagebound]], and tagged release **v1.5**.

> [!bug] Incident Card · Pagebound Season 2
> **02:07.** Your phone buzzes. The alert says *"Borrow is slow"*. A teammate shipped a change last evening. You're the one on call this week.
>
> Half asleep, you notice something uncomfortable. You can't say **how bad** it is: "slow" for whom, how often, compared with what? And if you fix it, you can't say **how bad it was**, so nobody will learn from it.
>
> **Your job:** define what "good enough" means in numbers, run an incident from page to postmortem, and leave the system better than you found it.

## Lab 1 - Break It

*About 20 minutes.* This is a **fire drill**. First you'll see what a *healthy* system measures, then you'll cause an incident, and then you'll respond to it.

**Step 1: the promise, written as code.** `slo.py` reads the server log and checks two promises, called **SLOs**:

- **SLO 1:** at least **99.5%** of requests are not server errors.
- **SLO 2:** at least **95%** of Borrow requests finish in **under 300 ms**.

> [!example]- slo.py (new)
> ```python
> """Are we keeping our promises? Reads server.log and checks two Service Level Objectives (SLOs)."""
> import json
> import sys
>
> path = sys.argv[1] if len(sys.argv) > 1 else "server.log"
> requests = [e for e in (json.loads(l) for l in open(path, encoding="utf-8") if l.startswith("{"))
>             if e["event"] == "request" and e["path"] not in ("/metrics", "/healthz")]
> borrows = [r for r in requests if r["method"] == "POST" and r["path"] == "/borrow"]
>
> # SLO 1: at least 99.5% of requests must not be server errors (5xx)
> good = sum(1 for r in requests if r["status"] < 500)
> availability = good / len(requests)
> budget_total = len(requests) * (1 - 0.995)          # How many bad requests we may "spend"
> budget_used = len(requests) - good
>
> # SLO 2: at least 95% of Borrow requests must finish in under 300 ms
> fast = sum(1 for r in borrows if r["ms"] < 300)
> fast_share = fast / len(borrows)
>
> print(f"SLO 1  availability     target 99.5%   actual {availability * 100:.1f}%   "
>       f"{'OK' if availability >= 0.995 else 'MISSED'}   (error budget used: {budget_used} of {budget_total:.0f})")
> print(f"SLO 2  Borrow < 300 ms  target 95.0%   actual {fast_share * 100:.1f}%   "
>       f"{'OK' if fast_share >= 0.95 else 'MISSED'}")
> ```

**Step 2: measure a healthy system.** Seed, start the server with its log saved (as in [[62 Scalability]]), and play 100 members, 5 at a time:

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

In another terminal, then stop the server and check:

```bash
python loadtest.py 100 5
python slo.py
```

```text
SLO 1  availability     target 99.5%   actual 100.0%   OK   (error budget used: 0 of 2)
SLO 2  Borrow < 300 ms  target 95.0%   actual 100.0%   OK
```

**Step 3: the "bad deploy".** Someone ships a change at midnight. We'll switch on the hidden fault from [[59 Observability]]. (It's the same `faults.py`, which you already have. If not, go and make it first.) Seed again, start the server **with the fault on**, and repeat the test:

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

```bash
python loadtest.py 100 5
python slo.py
```

```text
SLO 1  availability     target 99.5%   actual 100.0%   OK   (error budget used: 0 of 2)
SLO 2  Borrow < 300 ms  target 95.0%   actual 75.0%   MISSED
```

> [!bug] Nothing is "down", yet the promise is broken
> Availability is still 100%: no request failed. But a quarter of Borrow requests were slow, and the promise was **95% under 300 ms**. This is why you set SLOs on *what users feel*, and not only on "is the server up?". Without the number you'd argue about whether "slow" is "bad".

Stop the server. (Remove the environment variable in PowerShell: `Remove-Item Env:MYSTERY`.)

## The Big Idea

**System ownership** means taking **long-term responsibility** for a system's health. Not "I shipped it", but "I keep it good, and my successor can take it over."

### SLI, SLO, SLA

| Term | Meaning | Pagebound example |
|---|---|---|
| **SLI** (indicator) | A **measurement** of service quality | The share of Borrow requests under 300 ms |
| **SLO** (objective) | Your **target** for the SLI | 95% under 300 ms |
| **SLA** (agreement) | A **promise to customers**, often with a penalty | "99.5% uptime or a refund" (we don't have one) |

Choose SLOs that match **what users care about**: availability, speed, and correctness. Not too strict (perfect costs a fortune), and not too loose.

### The error budget

A 99.5% target means you may fail **0.5%** of the time: that's your **error budget**. Over a 30-day month that's about **3.6 hours** (30 × 24 × 0.005). While there's budget left, you can ship fast and take risks. If it's spent, you **slow down and fix reliability** first. It turns "reliability vs new features" from an argument into a measurement.

### On-call and incident response

An **incident** is something that harms users now. A calm routine beats heroics:

1. **Acknowledge** the page. Say you're on it.
2. **Assess:** how bad? Who is affected? (Check the SLO. You did.)
3. **Mitigate first**, fix later. **Roll back** the bad change, turn off the feature, or add capacity. Stopping the pain comes before understanding the cause.
4. **Communicate:** a status update that follows [[65 Technical Communication]].
5. **Find the cause**, using your logs and metrics ([[59 Observability]]).
6. **Write the postmortem.**

### Blameless postmortems

After an incident, the team writes down what happened and **how to prevent it**. It's **blameless**: it asks *"what in our system allowed this?"* and not *"whose fault is it?"*. People make mistakes in all systems. If you blame, people hide problems and the next failure arrives in secret. If you look for the **weak spot** (no alert, no test, an unsafe default), you fix it for everyone.

### Runbooks, health reviews and roadmaps

- A **runbook** is a short, tested set of steps for a known problem, written so that a tired person at 2 a.m. can follow it.
- A **service health review** (monthly) looks at the SLOs, the incidents, the alerts that woke people, and the error budget.
- A **roadmap for your system** lists the reliability and maintenance work you owe it, next to features.

> [!note]- Decommissioning (advanced)
> Owning a system also means **retiring** it well: announce a date, move users, archive data you must keep, turn it off, and delete what you needn't keep. Old systems nobody owns become security holes. Write "how to turn this off" at the start, not the end.

## Lab 2 - Fix It

*About 45 minutes.* Respond to the incident you caused.

**Step 1: mitigate first.** The quickest way to stop the pain is to **roll back**. Restart the server without the fault (the "previous version"), and check the promise:

```bash
python seed.py
python main.py > rollback.log
```

Then, in another terminal, `python loadtest.py 100 5`, stop the server, and:

```bash
python slo.py rollback.log
```

```text
SLO 1  availability     target 99.5%   actual 100.0%   OK   (error budget used: 0 of 2)
SLO 2  Borrow < 300 ms  target 95.0%   actual 100.0%   OK
```

The page is quiet. The users are fine. **Now** you have time to understand it.

**Step 2: find the cause**, from the log of the bad run. It's still in `server.log` (the rollback wrote to `rollback.log`). Use your tools:

```bash
python logsummary.py server.log
python trace.py
```

Everything you need is in [[59 Observability#Lab 2 - Fix It]]: `repo.add_loan` is the slow step, and only some of its calls are slow.

**Step 3: write the postmortem.** Create `docs/postmortem-borrow-slow.md`. Here is a complete one for the drill:

```markdown
# Postmortem: Borrow was slow (blameless)

**Date:** last night, 00:10 to 02:20 · **Duration:** 2 h 10 min · **Severity:** medium

## Summary
For about two hours, about 1 in 4 Borrow requests took more than 300 ms (as slow as 440 ms).
No requests failed and no data was lost. The Borrow SLO (95% under 300 ms) was missed, at 75%.

## Impact
Members saw a "hanging" Borrow button some of the time. About a quarter of Borrow clicks were slow.

## Timeline (all times local)
- 00:10  Change deployed.
- 02:07  An alert fired: "Borrow is slow". On-call acknowledged.
- 02:12  SLO check: availability 100%, Borrow SLO at 75%. Decision: roll back.
- 02:20  Rolled back. SLO back to 100%.
- 02:40  Cause found in the logs: `repo.add_loan` stalled on about every 4th call.

## Root cause
A change made `add_loan` stall for 0.4 s on every 4th call.

## Contributing factors (the weak spots, not people)
- No test or load check ran before the deploy, so the slowdown wasn't caught.
- The alert fired **two hours** after the deploy, because it looked at a 2-hour window.
- There was no easy rollback note in the runbook.

## What went well
- Logs and metrics found the slow step in minutes, with no new code.
- Rolling back first stopped the pain before the cause was known.

## Action items
| Action | Owner | Due |
|---|---|---|
| Run a load test and the SLO check before every deploy (you'll build `release_check.py` in the final project) | you | Friday |
| Shorten the alert window to 10 minutes | you | Friday |
| Add "how to roll back" to the runbook | teammate | Monday |
```

Notice the language: **"a change made"**, never "Sam broke it". Every action item has an **owner and a date**, since an action with no owner never happens.

**Step 4: write a runbook**, so the next person at 2 a.m. doesn't start from zero. Create `docs/RUNBOOK.md`:

```markdown
# Runbook: Borrow is slow

**Alert:** "Borrow is slow" (SLO 2).

1. **Check the promise:** `python slo.py server.log`. Is it availability, speed, or both?
2. **Did we deploy recently?** If yes, **roll back first**, and investigate afterwards.
3. **Find the slow step:** `python logsummary.py`, then `python trace.py`.
4. **Check the queue:** `python -c "from jobs import JobQueue; print(JobQueue().counts())"`.
   Many `ready` or any `dead` jobs? Is the worker running?
5. **Tell people** using the five-part update (status, impact, cause, action, next update).
6. **After it's over:** write the postmortem within 2 working days.
```

**Step 5: set your SLOs for real.** Write down, in `docs/SLOs.md`, the two SLOs, the error budget in plain words ("about 3.6 hours a month of failed requests"), and *what you'll do when the budget is spent* ("stop new features, fix reliability").

```bash
git add docs
git commit -m "Add SLOs, a runbook and the first postmortem"
```

Add to `LOG.md`: *"67: Borrow SLO missed at 75% after a bad deploy. Mitigated by rolling back, found the cause in the logs, wrote a blameless postmortem and a runbook."*

## Boss Check - Release v1.5

- [ ] Your design doc from [[65 Technical Communication]] has had **at least one review comment** answered
- [ ] `docs/ONBOARDING.md` exists (note 66)
- [ ] `docs/SLOs.md`, `docs/RUNBOOK.md` and a **blameless postmortem** exist (this note)
- [ ] `python slo.py` reports OK on a healthy run

```bash
git tag v1.5
git push --tags
```

## Practice

**1. Which one?** (a) "99.9% of requests succeed", (b) the measurement "98.7% of requests succeeded this week", (c) "we'll refund you if the service is under 99.9%".

> [!example]- Answer to question 1
> (a) **SLO**. (b) **SLI** (a measurement). (c) **SLA** (a promise to customers).

**2. Budget maths.** The SLO is 99.9% availability over 30 days. How much downtime is the error budget?

> [!example]- Answer to question 2
> 0.1% of 30 days = 30 × 24 × 60 × 0.001 = **43.2 minutes**. Strict SLOs leave very little room: this is why 99.99% is so costly.

**3. Mitigate or fix?** A new release doubles the error rate. You think you know the bug. Do you fix it now?

> [!example]- Answer to question 3
> **Mitigate first.** Roll back, and let users recover. Then fix the bug properly, with tests and review, in daylight. Fixing in a hurry under pressure often causes the next incident.

**4. Blameless or not?** Rewrite: "Sam forgot to run the tests, and that broke production."

> [!example]- Answer to question 4
> For example: "The change was deployed without running the tests, because nothing in the pipeline required it." The weak spot is the missing guard rail. The fix is an action item: *make the pipeline run the tests and block the deploy if they fail*.

**5. Roadmap exercise.** Take over a neglected system, set SLOs, and improve them over a quarter. What would the first week look like for Pagebound if *you* had just inherited it?

> [!example]- Answer to question 5
> Week 1: **learn and measure**. Find out what it does, who uses it, and how it fails. Add logs, metrics and a health check if missing. Measure current speed and error rate. Write a first draft of SLOs, and a list of known risks. Don't rewrite anything yet. A baseline comes before an improvement.

## Can You Do This?

- [ ] Explain SLI, SLO and SLA
- [ ] Work out an error budget, and say how it guides decisions
- [ ] Run an incident: acknowledge, assess, mitigate, communicate, find the cause
- [ ] Write a blameless postmortem with owned action items
- [ ] Write a runbook that a tired person could follow
- [ ] Say what a service health review checks

## Summary

> [!success] Remember
> - **Own the whole life** of the system, not just the launch.
> - Set **SLOs** on what users feel. The **error budget** guides risk.
> - **Mitigate first**, understand second, and write it down.
> - **Blameless** postmortems fix the system, not the person.
> - A **runbook** is a gift to the next tired person.

**Next:** [[68 Phase 4 Final Project]] →
