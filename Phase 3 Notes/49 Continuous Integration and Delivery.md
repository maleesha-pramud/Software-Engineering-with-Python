---
title: Continuous Integration and Delivery
tags: [phase-3, devops]
aliases: [CI/CD, CI, CD, GitHub Actions, Feature Flags, Trunk-Based Development]
---

# Continuous Integration and Delivery

← [[48 Docker]] · [[Phase 3 - Start Here|Home]] · **Next:** [[50 Database Design]] → · [[Software Engineering Roadmap#Continuous Integration and Delivery|Roadmap card]]

> [!abstract] After this note, you can
> Build an automatic pipeline that checks each change and ships it safely, so that deploying becomes boring.

## The Big Idea

Imagine a team that merges all its work once every three months. Everything conflicts, and testing takes weeks. Release day is terrifying.

Now imagine that every small change is automatically tested within minutes, and it can go live the same day. Each release is tiny, so it's easy to check and easy to undo. **Frequent, small and automated is the safest way to ship.** [DORA's research](https://dora.dev/) backs this up.

![[ci-pipeline.svg]]

## The Three Terms

![[delivery-vs-deployment.svg]]

| Term | Meaning | Question it answers |
|---|---|---|
| **Continuous integration (CI)** | Everyone merges often, and each merge is automatically built and tested | Does it still work? |
| **Continuous delivery** | Every passing change is ready to release, at the press of a button | Could we ship right now? |
| **Continuous deployment** | Every passing change goes live automatically | Is it shipped yet? |

## Trunk-Based Development

CI only works if people really do merge often. **Trunk-based development** means short-lived branches that merge into `main` within a day or two.

```mermaid
gitGraph
    commit id: "A"
    branch small-fix
    commit id: "fix"
    checkout main
    merge small-fix
    commit id: "B"
    branch small-feature
    commit id: "part 1"
    checkout main
    merge small-feature
    commit id: "C"
```

| | Long-lived branches | Trunk-based |
|---|---|---|
| **Branch lifetime** | Weeks | Hours or a day or two |
| **Merge conflicts** | Painful | Small |
| **Unfinished work** | Hidden on a branch | Merged, but behind a [[#Feature Flags\|feature flag]] |

## Your First Pipeline - GitHub Actions

A pipeline is a file in your repo. Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Get the code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"
          cache: pip

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Check style
        run: ruff check .

      - name: Run tests
        run: python -m pytest
```

Push it to GitHub, and open the **Actions** tab. Each push and pull request now runs your checks automatically. See the [GitHub Actions docs](https://docs.github.com/en/actions).

| Word | Meaning |
|---|---|
| **Workflow** | The whole automated process |
| **Trigger** (`on`) | What starts it |
| **Job** | A group of steps on one machine |
| **Step** | One command or action |
| **Runner** | The machine that does the work |

> [!tip] Turn on branch protection
> In your GitHub settings, require the CI check to pass before merging. Otherwise, it's only a suggestion.

## A Complete Pipeline

```mermaid
flowchart LR
    A[Lint] --> B[Unit tests]
    B --> C[Integration tests]
    C --> D[Security scan]
    D --> E[Build image]
    E --> F[Deploy to staging]
    F --> G[Smoke test]
    G --> H[Deploy to production]
    H --> I[Watch metrics]
```

**Put the fastest checks first,** so that simple problems fail in seconds.

```yaml
name: Full pipeline

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_PASSWORD: test-only-password
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.13"
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: python -m pytest tests/unit
      - run: python -m pytest tests/integration
        env:
          DATABASE_URL: postgresql://postgres:test-only-password@localhost:5432/postgres
      - run: pip install pip-audit && pip-audit

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t my-app:${{ github.sha }} .

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploy my-app:${{ github.sha }} here"
```

Notice these points:

- `services` starts a real database for [[47 Integration Testing|integration tests]].
- `needs` makes the jobs run in order.
- The image is tagged with the **commit hash**, so you always know exactly what's running.
- `environment: production` can require a person's approval.

## Build Once, Deploy Everywhere

```mermaid
flowchart LR
    A[Commit] --> B[Build image abc123 once]
    B --> C[Staging runs abc123]
    C --> D[Production runs abc123]
```

**Don't rebuild for each environment.** Otherwise, what you tested isn't what you ship. Build one **artifact**, and change only the configuration, using environment variables.

## Secrets in Pipelines

Pipelines often need passwords and keys to deploy. Those must never be in the workflow file.

```yaml
name: Deploy

on: workflow_dispatch

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

Add secrets under **Settings → Secrets and variables → Actions**. GitHub hides them in logs, but you should still never print them.

## Hardening the Pipeline

A pipeline can reach production, so it's a prime target for attackers. See [[46 OWASP Top 10#A03 Software Supply Chain Failures]].

| Risk | Defense |
|---|---|
| Too much access | Set `permissions:` to the minimum |
| A hijacked third-party action | Pin actions to a full commit hash |
| Pull requests from strangers reading secrets | GitHub withholds secrets from fork pull requests by default. Don't weaken it |
| Long-lived cloud keys | Use short-lived credentials with [OIDC](https://docs.github.com/en/actions/concepts/security/openid-connect) |
| Vulnerable dependencies | Scan in CI, and use Dependabot |

## Feature Flags

![[feature-flag.svg]]

A **feature flag** separates **deploying** code from **releasing** a feature. This has big benefits:

- Merge unfinished work safely, since it's switched off
- Try a feature on 1% of users first
- If something breaks, switch it off in seconds, with no deploy

```python
import hashlib

FLAGS = {"new_checkout": 10}   # Percent of users who get it

def is_enabled(flag, user_id):
    percent = FLAGS.get(flag, 0)
    digest = hashlib.sha256(f"{flag}:{user_id}".encode()).hexdigest()
    bucket = int(digest, 16) % 100
    return bucket < percent

enabled = sum(is_enabled("new_checkout", user) for user in range(10000))
print(f"About 10%: {900 < enabled < 1100}")

print(is_enabled("new_checkout", 7) == is_enabled("new_checkout", 7))
print(is_enabled("unknown_flag", 7))
```

Output:

```text
About 10%: True
True
False
```

Hashing keeps each user in the same group on every visit. Unknown flags default to off, which fails safely.

> [!warning] Remove old flags
> Each flag doubles the paths through your code. Once a feature is fully live, delete its flag.

## Safe Deployments

| Strategy | How it works | Picture |
|---|---|---|
| **Rolling** | Replace servers a few at a time | Old old old → new old old → new new new |
| **Blue-green** | Run two copies, then switch traffic | Blue live, green ready, then flip |
| **Canary** | Send a little traffic to the new version | 1% → 10% → 100%, watching errors |

More is on the [[Software Engineering Roadmap#Deployment Strategies|Deployment Strategies card]].

### Rolling Back

Know how to undo a release **before** you need to. Usually, you redeploy the previous image.

```bash
kubectl rollout undo deployment/my-app
```

Or, in a simple setup:

```bash
docker compose pull && IMAGE_TAG=abc123 docker compose up -d
```

> [!important] Database changes can't simply be undone
> Code rolls back easily, but data doesn't. Make schema changes backward compatible, using [[50 Database Design#Safe Schema Changes|expand and contract]].

## A Peek at GitOps

**GitOps** means Git holds the desired state of your production system. A tool like Argo CD watches the repo, and makes production match. To deploy or roll back, you just change a file and merge. Every change is then reviewed and recorded.

## What Makes a Good Pipeline?

| Quality | Target |
|---|---|
| **Fast** | Under 10 minutes for the main checks |
| **Reliable** | No [[47 Integration Testing#Flaky Tests\|flaky tests]] |
| **Trusted** | A green build means safe to ship |
| **Repeatable** | You can run the same steps locally |

> [!tip] Keep the logic in scripts
> Put real commands in a script or `Makefile`, and have CI call that. Then you can run and debug the same steps on your own machine.

```bash
make test
```

## Common Mistakes

> [!warning] Watch out for these
> - Letting the main branch stay broken. Fixing it comes first
> - Slow pipelines that people stop waiting for
> - Secrets in workflow files
> - Rebuilding separately for each environment
> - Giving pipelines full access to everything
> - Long-lived branches, which defeat the whole idea

## Practice

**1. Which term?**
- Every merge is tested automatically, but releases happen monthly
- Each passing change goes live with no human step
- Everything's always ready, and a person clicks release

> [!example]- Answer to question 1
> Continuous integration, continuous deployment, continuous delivery.

**2. Order these.** Build image, deploy to production, unit tests, lint, smoke test on staging.

> [!example]- Answer to question 2
> Lint, unit tests, build image, smoke test on staging, deploy to production. The fastest checks go first.

**3. Spot the problems.**

```yaml
name: Bad

on: push

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh --token sk_live_abc123
```

> [!example]- Answer to question 3
> A real secret is in the file, so anyone who can read the repo has it. It deploys on **every** push from any branch, with no tests at all. And it has no `permissions` block.

**4. Why flags?** Your new checkout is half finished, but your team merges daily. What do you do?

> [!example]- Answer to question 4
> Merge it behind a feature flag that's switched off. It goes to production, but no user sees it until it's ready.

**5. Roadmap exercise.** Add a pipeline to your [[34 Phase 2 Final Project|library API]] that runs lint, tests, a security scan and an image build on each change. Aim to deploy on any day with confidence.

## Can You Do This?

- [ ] Explain CI, continuous delivery and continuous deployment
- [ ] Explain trunk-based development
- [ ] Write a GitHub Actions workflow
- [ ] Use a real database in CI
- [ ] Store and use secrets safely
- [ ] Build once, and promote one artifact
- [ ] Add a feature flag
- [ ] Explain rolling, blue-green and canary releases
- [ ] Roll back quickly

## Summary

> [!success] Remember
> - **Small and frequent beats big and rare.**
> - **Automate everything**, and put the fastest checks first.
> - **Build once. Deploy the same thing everywhere.**
> - **Feature flags** separate deploying from releasing.
> - Know your **rollback** before you need it.

**Next:** [[50 Database Design]] →
