---
title: Technical Communication
tags: [phase-4, communication, leadership]
aliases: [Technical Communication, Status Updates, Design Doc, RFC, BLUF, Writing for Engineers]
---

# Technical Communication

← [[64 System Design]] · [[Phase 4 - Start Here|Home]] · **Next:** [[66 Mentoring]] → · [[Software Engineering Roadmap#Technical Communication|Roadmap card]]

> [!abstract] After this note, you can
> Write a status update that people act on without asking questions, tailor a message to its reader, write a design doc that gets useful feedback, and explain a system to a non-engineer in two minutes. You'll have done all of this for [[Pagebound - Season 2|Pagebound]].

> [!bug] Incident Card · Pagebound Season 2
> **Thursday, 22:40.** The email worker was down for three hours this evening. You fixed it, and you're tired. You post this in the team chat, which the three librarians and your boss also read:
>
> > *"hey all, there was some issue w/ the thing earlier, I think the queue maybe? anyway should be mostly fixed, emails might be delayed or whatever. will look more tomorrow"*
>
> Within ten minutes you have **seven replies**, all questions. The head librarian is worried. Your boss wonders if the problem is still happening. Nobody knows what to do.
>
> **Your job:** find out exactly what was wrong with that message, and learn to write one that nobody needs to reply to.

## Lab 1 - Break It

*About 10 minutes.* Read the message again, **as the head librarian**. She doesn't know what a queue is. She has 40 members at the desk who didn't get a confirmation email. Write down every question you'd ask. Don't read on until you have at least five.

> [!example]- Questions people would ask (open after you have your own)
> 1. **What** went wrong? "The thing" and "some issue" tell me nothing.
> 2. Is it **fixed** now, or only "mostly"? Which part is not?
> 3. Which **emails** were delayed, and for how long? Do I need to tell members?
> 4. Did anyone lose **loans or data**? (The message doesn't say. Silence feels like "yes".)
> 5. Do I need to **do** anything tonight? Tomorrow?
> 6. **Why** did it happen? Will it happen again?
> 7. When is the **next** update, and who do I ask?

Count them. **Seven questions** from six sentences. Each question is a small failure of the message: the writer had the answers, and the reader paid for them with time and worry.

> [!bug] The writer knew everything, and the reader got nothing
> The message was written from **your** head ("the thing", "the queue"), in the order **you** lived it, to nobody in particular. A good message does the opposite: it starts from what the **reader** needs.

## The Big Idea

Technical communication is **engineering for human heads**. Your reader is busy, doesn't share your context, and has to *do* something with what you wrote. Four rules do most of the work.

### 1. Conclusion first (the "BLUF")

BLUF means **bottom line up front**. Say the outcome in the first sentence, then the detail, in order of importance. A reader who stops after one line should still act correctly.

| Story order (bad) | Conclusion first (good) |
|---|---|
| "At 19:05 I noticed the worker had stopped, so I looked at the logs, and then..." | "Confirmation emails stopped for 3 hours tonight. They are all sent now. No loans or data were lost." |

### 2. Know your reader

Ask three questions before you write: **Who** is reading? **What do they already know?** **What should they do afterwards?**

| Reader | Needs | Leave out |
|---|---|---|
| **Teammates** | Cause, fix, how to avoid it | Nothing. Give technical detail |
| **Librarians** | What members saw, what to tell them, anything to do | Queue names, code, log lines |
| **Your boss** | Impact, risk, cost, whether it's over | The debugging story |

### 3. A clear status update has five parts

1. **Status** in one line (fixed, in progress, blocked).
2. **Impact**: who was affected, how, and for how long.
3. **Cause**, in plain words (one sentence).
4. **What we are doing**, and what the reader needs to do (even if "nothing").
5. **Next update**: when, and where.

### 4. Pictures that explain

A diagram is for **one idea**. Give it a title that says the idea, label every arrow with what it carries, order things in the direction of reading, and use colour to mean something (or not at all). If you need five minutes to explain a diagram, it isn't finished. The pictures in these notes follow those rules, so use them as models.

## Lab 2 - Fix It

*About 50 minutes.* Four rewrites. Write real text for each, in files in your repo's `docs/` folder.

### Step 1 - Rewrite the update

Rewrite the incident message with the five parts. Try yours first. Here is one possible answer:

> **Status: fixed.** Confirmation emails stopped for about 3 hours this evening (19:00 to 22:00). They have all been sent now.
>
> **Impact:** about 40 members borrowed a book in that time and got their email late. No loans or data were lost. Books were lent normally.
>
> **Cause:** the program that sends emails stopped, and no alarm told us. It was restarted at 22:30.
>
> **What to do:** nothing tonight. If a member asks, you can say: *"Your confirmation email was late, but your loan is safe."*
>
> **Next update:** tomorrow at 10:00 here, with what we'll change so that an alarm goes off next time.

Count its questions: ideally **zero**. That is the test, and the roadmap's mastery goal: *people act correctly on your messages without any follow-up questions.*

### Step 2 - Write it three ways

Take the same event, and write a **two-line** version for each reader: a teammate, a librarian and your boss. Notice what each version keeps, and what it drops.

> [!example]- A sample answer
> **Teammate:** "Email worker died at 19:00 (OOM, I think). No alert, since we only monitor the API. Restarted at 22:30, backlog drained. TODO: alert on queue age and worker heartbeat."
>
> **Librarian:** "Confirmation emails were up to 3 hours late tonight. All sent now. Loans are safe. Nothing for you to do."
>
> **Boss:** "3-hour email delay, no data lost, no member impact beyond late emails. Root cause: unmonitored background worker. Fix: add an alarm, due Friday."

### Step 3 - Write a design doc

In [[64 System Design]] you made a one-page design for share links. Now turn it into a **design doc** (some teams call it an **RFC**, request for comments). Its job is to get **useful feedback before you build**, when changes are cheap. Use this template in `docs/share-links-design.md`:

```markdown
# Share links: design

**Author:** you · **Status:** Draft · **Reviewers:** the other engineers, the marketing lead

## Summary
Two sentences. What we will build, and why. (Conclusion first.)

## Goals and non-goals
- Goal: make a short link, open a short link in under 100 ms.
- Non-goal: custom names, expiry, analytics dashboards (maybe later).

## Proposal
How it works, with one diagram (Context or Containers). The data model and the API.

## Estimates
The numbers from estimate.py, and what they mean.

## Alternatives considered
- Random 4-character codes: rejected. 317 clashes in 100,000 links.
- Hashing the URL: ...
Why each alternative lost. This section earns trust.

## Risks and open questions
- Abuse (links to bad sites). Mitigation: ...
- Question for reviewers: should expired links reuse codes?

## Plan
Three milestones, each a pull request.
```

Then do what real teams do: **open a pull request** that adds only this file, and ask two people to comment. (I haven't run this step for you, because it needs your GitHub repo and your reviewers.) Their questions show you where your writing isn't clear. Answer in the doc, not in the comments, so the next reader benefits.

> [!tip] Why "Alternatives considered" matters
> Reviewers will think of the alternatives anyway. If you've already answered *"why not X?"*, you save a meeting, and you show that your choice was a choice.

### Step 4 - Explain it in two minutes, to a non-engineer

Find someone who isn't an engineer (a friend, a librarian). Explain how the **emails stay safe even when a machine dies**. A good shape:

1. **An everyday comparison** (10 seconds): *"It's like writing a to-do note and putting it in a box. A helper takes notes from the box. If the helper faints, the note is still there."*
2. **The problem** (20 seconds): emails used to vanish when the sender broke.
3. **The fix** (40 seconds): the box, the helper, the note put back.
4. **What it means for them** (20 seconds): *"Members get every email, even if it's a bit late."*

Then ask them to say it back to you. Whatever they get wrong is **your** explanation's fault, not theirs.

> [!note]- Presenting to executives (advanced)
> Executives have little time, and they make decisions. Start with the **decision** you need, or the **result**. Give the **one number** that matters (cost, risk, time). Offer two or three **options** with your recommendation. Keep the detail ready in your pocket, not on the slide. Expect to be interrupted, and be happy to be: it means they're engaged.

Commit your writing:

```bash
git add docs
git commit -m "Write the status update, the share-links design doc"
```

Add to `LOG.md`: *"65: my outage message caused seven questions. Fix: conclusion first, five parts, tailored to the reader. Zero questions the second time."*

## Practice

**1. Fix the first line.** "So I was looking at the database yesterday and I noticed something odd with the loans table, and..." Rewrite the first line, conclusion first.

> [!example]- Answer to question 1
> For example: "**Some loans were saved twice last week (about 14 of them). I've found the cause and I need your approval to fix the data.**" The reader now knows the problem, its size, the status and the ask.

**2. Who is reading?** You must tell the finance manager that the cloud bill will rise by 30%. What do you lead with, and what do you skip?

> [!example]- Answer to question 2
> Lead with the **amount and the reason**: "Next month's cloud cost rises by about $20, from $66 to $86, because we added a second server so the app stays up if one fails." Offer the option of not doing it, and its risk. Skip the technology.

**3. Review a doc.** A teammate's design doc has no "Alternatives considered" and no numbers. What do you write in your review comment?

> [!example]- Answer to question 3
> Be specific and kind: *"Could you add the expected requests per second and data size? Then it's easier to judge the design. Also, what other approaches did you consider, and why did you pick this one?"* Ask questions that make the doc better, and not ones that make the person look bad.

**4. Roadmap exercise.** Write a design doc and get feedback. Explain it to a non-engineer in two minutes. Did anyone need a follow-up question? Record how many in `LOG.md`.

> [!example]- Answer to question 4
> Success is **zero** follow-up questions on the doc *and* the explanation, but a few questions on the first draft are normal, and useful. Each one is a sentence to rewrite. Keep a list of the questions you got. Over time, you'll notice the same ones, and you'll start answering them in advance.

## Can You Do This?

- [ ] Write a conclusion-first message
- [ ] Write a five-part status update
- [ ] Tailor the same news to three different readers
- [ ] Write a design doc with goals, non-goals, alternatives and risks
- [ ] Draw a diagram that explains one idea
- [ ] Explain a system to a non-engineer in two minutes

## Summary

> [!success] Remember
> - **Conclusion first.** One line should be enough to act on.
> - **Write for the reader**, not from your own head.
> - A status update has **status, impact, cause, action, next update**.
> - A design doc earns feedback **while changes are cheap**. Show the alternatives.
> - The test: **no follow-up questions**.

**Next:** [[66 Mentoring]] →
