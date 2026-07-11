# GitHub Actions — The `on:` Trigger System

---

## 1. What is the `on:` Trigger?

> The **`on:`** key in a GitHub Actions workflow file is your **event subscription**. It tells GitHub: *"Run this workflow when these specific events happen in this repository."*

A workflow without a correct `on:` block is a workflow that either never runs (silent blind spot) or runs too often (runaway automation).

```
   ┌──────────────────────┐
   │  Event in GitHub      │  (push, pull_request, release,
   │                       │   schedule, manual click, API call)
   └──────────┬───────────┘
              │
              ▼
   ┌──────────────────────┐
   │  on:  in workflow.yml │  matches event + filters?
   └──────────┬───────────┘
              │ yes
              ▼
   ┌──────────────────────┐
   │  Workflow run starts  │
   └──────────────────────┘
```

---

## 2. Why `on:` — What Challenges Does It Solve?

### Challenge 1: Triggers That Are Too Broad

A misconfigured trigger (e.g., `push` on every branch) causes a deploy workflow to fire on every feature branch push. Production gets hit by 40 deploy attempts a day. A staging database can be wiped by someone testing a migration on their personal branch.

**`on:` solves this** by letting you filter precisely — by branch, by path, by tag, by PR event type.

---

### Challenge 2: Triggers That Are Too Narrow

The opposite mistake — writing the trigger so tightly that CI only fires on pushes to `main` and never on pull requests. PRs merge with zero CI coverage, bugs reach `main` untested.

**`on:` solves this** by giving you a rich set of event types — including `pull_request` — so you can subscribe to exactly the moments you care about.

---

### Challenge 3: Triggers Were Not Version-Controlled in Older Systems

In Jenkins, triggers were configured in the UI or in `Jenkinsfile`. Polling SCM wasted resources; webhooks required public IPs and manual maintenance; trigger configs were often not in source control and could drift or disappear when Jenkins was restored from backup.

**`on:` solves this** by being **declarative** and **stored in your repository**. The trigger config lives next to the code it triggers — version-controlled, reviewable, reproducible.

---

## 3. Plain-English Definition

> The `on:` key is the **"when to run"** declaration of a workflow. You list one or more events, and optionally add filters (branches, paths, types, tags) to make the subscription precise.

If `on:` matches the event that just happened in your repo, GitHub queues a workflow run. If it doesn't match, nothing happens.

---

## 4. How `on:` Works — The 7 Important Triggers

### Trigger 1: `push`

Fires when commits are pushed to the repository.

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    branches-ignore:
      - 'dependabot/**'
    paths:
      - 'src/**'
      - 'package.json'
    paths-ignore:
      - '**.md'
    tags:
      - 'v*'
```

| Filter | What it does |
|---|---|
| `branches` | Only run on pushes to these branches. Glob patterns supported (`release/**`). |
| `branches-ignore` | Run on all branches EXCEPT these. **Cannot be used together with `branches`** — pick one. |
| `paths` | Only run when the push includes changed files matching these patterns. Critical for monorepos. |
| `paths-ignore` | Skip the run if ONLY ignored paths changed. A push that changes `README.md` and `src/index.ts` still runs (because `src/index.ts` is outside the ignore list). |
| `tags` | Run when a tag matching the pattern is pushed (`v*` → `v1.0.0`, `v2.1.3-beta`). |

> **Critical subtlety:** `branches:` and `paths:` are **AND-ed** together. Both must match.
> - Push to `main` + `src/` changed → fires
> - Push to `main` + only `docs/` changed → does **not** fire
> - Push to `feature/xyz` + `src/` changed → does **not** fire

---

### Trigger 2: `pull_request`

Fires when a pull request changes state.

```yaml
on:
  pull_request:
    branches:
      - main
    types:
      - opened
      - synchronize
      - reopened
```

| Field | What it does |
|---|---|
| `branches` | The PR's **target** (destination) branch — not the source branch. |
| `types` | Which PR events fire the workflow. Default (when omitted): `[opened, synchronize, reopened]`. |

Common PR event types:

| Type | Meaning |
|---|---|
| `opened` | PR was just created. |
| `synchronize` | New commits were pushed to the PR branch (fires when you push fixes to make CI pass). |
| `reopened` | PR was closed and reopened. |
| `closed` | PR was merged or closed without merging. Use for cleanup jobs. |
| `labeled` | A label was added. Use to trigger special jobs (e.g., `deploy-preview`). |
| `ready_for_review` | Converted from draft → ready. Use to run full CI only on non-draft PRs. |

**Production pattern — skip CI on draft PRs:**

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  test:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
```

---

### Trigger 3: `workflow_dispatch`

Adds a manual **"Run workflow"** button in the GitHub Actions UI. The user can pass typed inputs.

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]
        default: staging
      dry_run:
        description: 'Run without making changes'
        required: false
        type: boolean
        default: false
      version:
        description: 'Image tag to deploy'
        required: true
        type: string
```

Why it matters:
- Emergency rollbacks: *"Deploy v1.2.3 to production now."*
- One-off operational tasks: *"Run the migration on staging."*
- On-demand teardowns: *"Destroy preview env for PR #123."*

Without `workflow_dispatch`, you cannot run a workflow without making a commit.

Using the inputs:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying to: ${{ inputs.environment }}"
          echo "Dry run: ${{ inputs.dry_run }}"
          echo "Version: ${{ inputs.version }}"
```

---

### Trigger 4: `schedule`

Runs the workflow on a cron schedule.

```yaml
on:
  schedule:
    - cron: '0 2 * * 1-5'
    # Runs at 02:00 UTC, Monday through Friday.
    # Format: minute hour day-of-month month day-of-week
```

**Three things every senior engineer knows:**

| Pitfall | Detail |
|---|---|
| **Always UTC** | If your team is in IST (UTC+5:30) and you want 08:00 IST, set cron to `30 2 * * *`. Forgetting this causes wrong-time runs, especially around daylight saving. |
| **Delays on busy infra** | A `0 * * * *` (top of each hour) cron may actually run at, say, `02:07`. Do not depend on sub-minute precision. |
| **Disabled on inactive repos** | If no one has pushed for 60 days, GitHub silently stops scheduled workflows. It sends an email but engineers miss it — your nightly security scan can stop running unnoticed. |

> Analogy: this is a cron job on a Linux server. If the server hibernates, the cron does not run. Monitor whether the cron is **actually running**, not just that it is configured.

---

### Trigger 5: `workflow_call`

Makes a workflow callable by other workflows — the foundation of **reusable workflows**.

```yaml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy_token:
        required: true
```

Another workflow then calls this one via `uses:` at the job level. The called workflow runs in the caller's context.

---

### Trigger 6: `release`

Fires when a **GitHub Release** is published. Note: this is **not** the same as pushing a Git tag. A GitHub Release is a UI concept — a tag + release notes + optionally attached binaries.

```yaml
on:
  release:
    types:
      - published
```

Typical use: trigger a **production deploy** when a release is published. The release itself doubles as documentation of what was deployed and when.

---

### Trigger 7: `repository_dispatch`

Fires when an **external system** calls the GitHub API with a custom event. Lets external tools (Slack bots, monitoring, incident-response platforms) trigger workflows.

```yaml
on:
  repository_dispatch:
    types:
      - deploy-requested
      - rollback-requested
```

External call:

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO/dispatches \
  -d '{"event_type": "deploy-requested", "client_payload": {"version": "v1.2.3"}}'
```

Production use case: monitoring detects a latency spike → calls the API → triggers an automated rollback workflow. The entire incident response is logged in GitHub Actions like any other run.

---

## 5. Syntax — Quick Reference

### Single Event

```yaml
on: push
```

### Multiple Events (List Form)

```yaml
on: [push, pull_request]
```

### Multiple Events with Filters

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

### Filters Available per Trigger

| Trigger | Common filters |
|---|---|
| `push` | `branches`, `branches-ignore`, `paths`, `paths-ignore`, `tags`, `tags-ignore` |
| `pull_request` | `branches`, `types`, `paths`, `paths-ignore` |
| `schedule` | `cron` |
| `release` | `types` |
| `workflow_dispatch` | `inputs` |
| `workflow_call` | `inputs`, `secrets`, `outputs` |
| `repository_dispatch` | `types` |

> Rule: when multiple filters are specified on the **same trigger**, they are **AND-ed** — every filter must match.

---

## 6. Examples

### Example 1 — Production Trigger Architecture (Mature Team)

```yaml
on:
  # CI on PRs
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, ready_for_review]

  # Integration gate when code lands on main
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'package*.json'

  # Release pipeline — triggered by publishing a GitHub Release
  release:
    types: [published]

  # Manual escape hatch
  workflow_dispatch:
    inputs:
      reason:
        description: 'Reason for manual trigger'
        required: true
        type: string
```

### Example 2 — Monorepo Path Filtering

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'services/service-a/**'

  pull_request:
    branches: [main]
    paths:
      - 'services/service-a/**'
```

Only `service-a` changes will trigger this workflow — `service-b`, `service-c`, and unrelated docs are ignored.

### Example 3 — Skip CI on Documentation-Only Changes

```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/CODEOWNERS'
```

GitHub never queues a run if only ignored paths changed. The runner is never allocated, and your workflow history stays clean.

### Example 4 — Release Pipeline Triggered by a Git Tag

```yaml
on:
  push:
    tags:
      - 'v*'
```

Pushing `v1.0.0` or `v2.1.3-beta` triggers the pipeline.

### Example 5 — Scheduled Nightly Security Scan

```yaml
on:
  schedule:
    - cron: '0 3 * * *'   # 03:00 UTC every day

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./run-security-scan.sh
```

---

## 7. Common Mistakes Senior Engineers Catch

### Mistake 1 — Deploying on Every Push to `main`

Every merge to `main` triggers a production deploy. Fine for CD, but means every typo fix, every dependency bump, every docs commit deploys to production. Separate **integration** (push to `main`) from **release** (explicit release event) when production deploys must be intentional.

### Mistake 2 — No `paths:` in a Monorepo

12 services, every push to any file triggers CI for all 12. A change in `service-a/` should only trigger CI for `service-a`. Without `paths:` filters, you waste runner minutes and slow feedback loops.

### Mistake 3 — `push` on Every Branch

```yaml
# Wrong for most teams:
on:
  push:
    branches: ['**']
```

10 engineers pushing WIP commits = 50+ unnecessary runs/day. CI should fire on **PRs** (`pull_request`), not every push to every branch.

---

## 8. Interview Angle — `paths-ignore` vs `if`

**Question:** *"How do you prevent a workflow from running when only documentation files change?"*

**Beginner answer:** *"Add an `if` condition to skip the job."*

**Senior answer:**

`paths-ignore` on the trigger is cleaner than a runtime `if`. A runtime `if` still **starts the workflow**, **creates a run**, and **consumes a runner allocation** — it just skips the steps. With `paths-ignore`, GitHub never queues the workflow at all if only ignored paths changed. No runner is allocated, and the run history stays clean.

```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/CODEOWNERS'
```

**Caveat:** if your branch protection rule has **required status checks**, a `paths-ignore` that skips the workflow can leave the PR stuck — GitHub sees the required check as "pending" forever because the workflow never ran. In that case, use the `if` approach combined with a job that always succeeds with a neutral exit when skipped.

---

## 9. Summary

| Concept | Detail |
|---|---|
| **What it is** | The `on:` key declares **when** a workflow runs |
| **Why use it** | Prevents both runaway automation and silent CI blind spots |
| **Version-controlled** | Lives in the workflow file, not a UI |
| **Main trigger types** | `push`, `pull_request`, `workflow_dispatch`, `schedule`, `workflow_call`, `release`, `repository_dispatch` |
| **Filter combination** | All filters on a trigger are **AND-ed** — every one must match |
| **`branches` vs `branches-ignore`** | Pick one, not both |
| **`pull_request.branches`** | Targets the **destination** branch (not source) |
| **`workflow_dispatch`** | Adds a manual "Run workflow" button; supports typed inputs |
| **`schedule` rules** | Always UTC; not precise to the minute; disabled on inactive repos after 60 days |
| **Monorepo rule** | Always use `paths:` to scope CI to the changed service |
| **Deploy rule** | Prefer `release` over `push: main` for intentional production deploys |
| **`paths-ignore` advantage** | More efficient than runtime `if` — workflow never queues, runner never allocated |
| **`paths-ignore` caveat** | Required status checks may get stuck pending — use `if` + neutral-exit job in that case |
