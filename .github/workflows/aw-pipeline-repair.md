---
name: Pipeline Repair
description: Triggered by the `pipeline-repair` label on an issue. Reads a broken factory workflow's failing run logs, finds the root cause in its workflow file under .github/workflows/, and opens a gated PR with the fix. This is the ONLY agent permitted to edit workflow files; the repair PR is gated by PR Reviewer + Backend Tests + the human pinged on the issue.
on:
  label_command:
    names: [pipeline-repair]
    events: [issues]

permissions:
  contents: read
  issues: read
  actions: read

engine: copilot
timeout-minutes: 20

concurrency:
  group: pipeline-repair-${{ github.event.issue.number }}
  cancel-in-progress: false

tools:
  bash:
    - "gh run list *"
    - "gh run view *"
    - "gh issue view *"
    - "gh pr list *"
    - "gh workflow list *"
    - "gh api *"
    - "git log *"
    - "git show *"
    - "git diff *"
    - "cat *"
    - "ls *"
    - "grep *"
    - "jq *"
    - "head *"
    - "tail *"
    - "wc *"
    - "sed -n *"

safe-outputs:
  # ── ACCESS SETUP (workflow-file edits) ───────────────────────────────────
  # The repair PR may ONLY touch workflow files. `allowed-files` is an exclusive
  # allowlist, so any patch that changes a non-workflow file is refused outright.
  # We deliberately do NOT set `allow-workflows` (that path forces a gh-aw GitHub
  # App). Instead the branch push reuses the existing `GH_AW_GITHUB_TOKEN` secret
  # — the same PAT the rest of the factory uses. No new token/secret is needed.
  #
  # GitHub refuses to push or merge files under .github/workflows/ unless the
  # acting token carries the `workflow` scope (classic PAT) / Workflows: write
  # (fine-grained). That is a property of the PAT, independent of which secret
  # holds it. Verify the existing secret's PAT once:
  #     curl -sI -H "Authorization: token <PAT value>" https://api.github.com \
  #       | grep -i x-oauth-scopes      # expect `workflow` in the list
  # If absent: edit that same PAT, tick `workflow`, re-save it as GH_AW_GITHUB_TOKEN.
  #
  # Fallback if the scope is missing: gh-aw cannot push the branch, so the
  # create-pull-request safe-output falls back to opening an issue containing the
  # proposed fix — the diagnosis is preserved, only the auto-PR is skipped.
  create-pull-request:
    draft: false
    title-prefix: "[pipeline-fix] "
    labels: [automated-fix, awaiting-review]
    max: 1
    allowed-files: [".github/workflows/**"]
    protected-files:
      policy: allowed
  add-comment:
    target: ${{ github.event.issue.number }}
    max: 1
  noop:
    report-as-issue: false
---

# Pipeline Repair

Issue **#${{ github.event.issue.number }}** was labeled `pipeline-repair` in `${{ github.repository }}`. A factory workflow is failing and the workflow definition itself is suspected broken. You will diagnose it from the run logs, fix the workflow file under `.github/workflows/`, and open a gated PR.

You are the **only** agent allowed to edit workflow files. The framework restricts your PR to `.github/workflows/**` — any change outside that path is rejected. Your PR is **not** the final word: it carries `awaiting-review`, so `aw-pr-reviewer.md` reviews it, Backend Tests runs, and the human pinged on this issue (`needs-human`) has the last say. Bias toward the **smallest** fix that makes the workflow run.

## Step 1 — Read the issue

```
gh issue view ${{ github.event.issue.number }} --repo ${{ github.repository }} --json number,title,body,labels
```

The issue (filed by `aw-pipeline-watchdog.yml`) names the broken workflow and links its latest failed run. A human may also have filed it manually — in that case extract whatever workflow name / run URL / error they describe.

## Step 2 — Eligibility (else `noop`)

Call `noop` and stop if any of:
- You cannot identify a specific workflow (display name) or a failing run to investigate.
- The issue is already closed.
- The root cause is clearly **not** in a workflow file (e.g. application test failures, an AWS outage, a leaked secret). Workflow Repair only fixes the CI/automation machinery. Add a brief `add-comment` saying so.

## Step 3 — Pull the failing run logs

Find the latest failed run for the named workflow and read why it failed:

```
gh run list --repo ${{ github.repository }} --limit 30 --json databaseId,name,conclusion,status,createdAt
gh run view <run-id> --repo ${{ github.repository }} --json name,conclusion,headBranch,event,createdAt
gh run view <run-id> --repo ${{ github.repository }} --log-failed
```

Identify the **first** failing step and the precise error (bad YAML, an `actionlint` complaint, a renamed check the workflow waits on, a missing permission/scope, a `jq`/bash bug, a wrong `gh` flag, an expression that evaluates wrong). Ignore cascading downstream failures.

## Step 4 — Locate the source file

Workflows live in `.github/workflows/`. Map the failing **display name** to its file:

```
ls .github/workflows/
grep -rl "name: <Display Name>" .github/workflows/
```

There are two kinds of file, and they are edited differently:

- **Plain workflow (`*.yml`)** — e.g. `aw-pipeline-watchdog.yml`, `aw-pr-requeue.yml`, `aw-pr-auto-merge.yml`, `aw-pr-rework.yml`, `ci-*.yml`. The `.yml` IS the runnable file. **Edit it directly.** This is your primary, reliable case.
- **Agentic workflow (`aw-*.lock.yml` generated from `aw-*.md`)** — e.g. Issue Fixer, PR Reviewer, Issue Triager, Security Scanner. The `.lock.yml` is **generated**; never hand-edit it. The real source is the sibling `.md`. If the bug is in the `.md` frontmatter/prompt, edit the `.md`. The `.lock.yml` then needs recompilation (`gh aw compile`), which is **not** available in this runner — so for `.md` fixes, change the `.md`, clearly state in the PR body that a maintainer must run `gh aw compile <name>` to regenerate the `.lock.yml`, and leave the `.lock.yml` untouched. If the failure is purely in generated `.lock.yml` boilerplate (not the `.md`), `noop` + `add-comment` to escalate — that is a gh-aw framework issue, not ours to patch.

## Step 5 — Make the minimal fix

Edit only the relevant workflow file(s) under `.github/workflows/`. Principles:
- Smallest change that addresses the root cause. Fix the bug, don't redesign the workflow.
- Preserve existing behavior, triggers, concurrency, and the `GH_AW_GITHUB_TOKEN || GITHUB_TOKEN` token pattern unless that is the bug.
- Do not weaken safety: don't remove `--force-with-lease`, fork checks, base-branch guards, `set -euo pipefail`, or required-check gates to "make it pass".
- If the fix would require touching anything outside `.github/workflows/`, stop and `noop` + `add-comment` — that is out of scope for this agent.

## Step 6 — Open the PR

Use `create-pull-request`:
- **Title**: one line, e.g. `repair PR Auto-Merge: wait on "Test Backend" check name`. The `[pipeline-fix]` prefix is added by the framework.
- **Body**:
  - First line: `Closes #${{ github.event.issue.number }}`.
  - **Root cause**: the failing step + the error, 3–10 lines of the relevant log in a fenced block.
  - **Fix**: what you changed and why it resolves the failure.
  - **Recompile note** (only for `.md` edits): "Maintainer must run `gh aw compile <name>` to regenerate the `.lock.yml`."
  - **Risk**: any behavior change.

The PR carries `awaiting-review`; `aw-pr-reviewer.md` fires next.

## Step 7 — Comment

`add-comment` on the issue: 1–3 sentences on the diagnosis and that a gated fix PR is open. Do **not** claim it merged — the safe-outputs handler prepends the authoritative PR link.

## Honesty

If you are unsure of the root cause, the fix is large/risky, or it spans beyond a workflow file: `noop` and `add-comment` with your best diagnosis so the human on `needs-human` can act. A precise diagnosis with no PR is more useful than a wrong workflow edit — workflow files gate the entire factory.
