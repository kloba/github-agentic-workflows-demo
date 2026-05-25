---
name: Issue Triager
description: Auto-labels newly opened or edited issues with one classification label and a one-line clarifying comment if needed.
on:
  issues:
    types: [opened, edited]

permissions:
  contents: read
  issues: read

engine: copilot
timeout-minutes: 10

concurrency:
  group: issue-triager-${{ github.event.issue.number }}
  cancel-in-progress: true

tools:
  bash:
    - "gh issue view *"
    - "gh issue list *"
    - "gh label list *"
    - "gh api *"

safe-outputs:
  add-labels:
    target: ${{ github.event.issue.number }}
    max: 3
  add-comment:
    target: ${{ github.event.issue.number }}
    max: 1
    hide-older-comments: false
  noop:
    report-as-issue: false
---

# Issue Triager

Issue **#${{ github.event.issue.number }}** has been opened or edited in `${{ github.repository }}`.

## Step 1 — Fetch issue metadata

```
gh issue view ${{ github.event.issue.number }} --repo ${{ github.repository }} --json number,title,body,author,labels
```

## Skip conditions

Call `noop` if **any** of the following hold:
- Issue already has a label from this list (already triaged): `bug`, `feature-request`, `question`, `chore`, `documentation`, `security`, `ci-failure`, `needs-triage` already removed.
- Author is a bot (`*[bot]`).
- Issue body is empty or fewer than 20 characters (insufficient context to classify).

## Classification

Pick exactly **one** primary label, by reading the issue title and body:

- `bug` — Something is broken or behaving incorrectly.
- `feature-request` — A new capability or enhancement.
- `question` — Author is asking how something works.
- `chore` — Maintenance, dependency bumps, refactors.
- `documentation` — Docs are missing or wrong.
- `security` — Vulnerability or concern. **If you choose this, also add label `needs-triage`.**

Optionally add ONE area label if obviously applicable:
- `area:backend`, `area:frontend`, `area:billing`, `area:infra`, `area:entra-id`, `area:webhooks`.

Discover which labels actually exist in the repo first:
```
gh label list --limit 100 --json name
```
Only apply labels that exist. Do NOT invent new ones.

## Optional clarifying comment

If the issue is clearly missing something essential to act on (no repro steps for a bug, no use case for a feature request), post a single short comment listing **at most 3** specific questions. If the issue is well-formed, skip the comment.

Be brief. The triager's job is to route, not to investigate.
