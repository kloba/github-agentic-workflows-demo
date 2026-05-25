---
name: CI Doctor
description: When a deploy/verify workflow fails, investigate the run logs and open a single root-cause issue.
on:
  workflow_run:
    workflows: ["Deploy Backend", "Deploy Frontend", "Deployment Verification"]
    types: [completed]
    branches: [main]

permissions:
  actions: read
  contents: read
  issues: read

engine: copilot
timeout-minutes: 15

tools:
  bash:
    - "gh run *"
    - "gh api *"
    - "gh issue list *"
    - "cat *"
    - "jq *"
    - "grep *"
    - "head *"
    - "tail *"
    - "wc *"

safe-outputs:
  create-issue:
    title-prefix: "[ci-doctor] "
    labels: [ci-failure, automated-analysis, needs-triage]
    max: 1
    close-older-issues: false
    expires: 7d
  noop:
    report-as-issue: false
---

# CI Doctor

A CI workflow run has just completed with conclusion `${{ github.event.workflow_run.conclusion }}`.

Run ID: `${{ github.event.workflow_run.id }}`
Run URL: ${{ github.event.workflow_run.html_url }}
Repository: `${{ github.repository }}`

## Your task

1. **Exit early unless it's a genuine failure.** If the conclusion above is **anything other than `failure`** (i.e. `success`, `cancelled`, `skipped`, `neutral`, `stale`), call `noop` immediately. In particular, **do NOT file issues for `cancelled` runs** — on this repo cancellations are almost always a concurrency group superseding an older run when commits land on `main` in quick succession (normal, expected, not actionable). Filing them sends Issue Fixer into a loop it cannot resolve (the only "fix" is a `.github/workflows/` edit, which Issue Fixer is forbidden from).

2. **Fetch run metadata** (workflow name, branch, head SHA, triggering actor):
   ```
   gh run view ${{ github.event.workflow_run.id }} --repo ${{ github.repository }} --json name,displayTitle,headBranch,headSha,event,conclusion,createdAt,url
   ```

3. **Fetch the failure logs.**
   ```
   gh run view ${{ github.event.workflow_run.id }} --repo ${{ github.repository }} --log-failed
   ```

3. **Identify the root failure.** Find the *first* failing step. Ignore cascading failures (a downstream step failing because an earlier step already failed). Extract the most relevant 3–10 lines of error output.

4. **Deduplicate.** Check open issues with label `ci-failure`:
   ```
   gh issue list --label ci-failure --state open --json number,title,body
   ```
   If an open issue clearly describes the same root cause (same error signature, same workflow, same step), call `noop` — do not duplicate.

5. **File one issue.** Title format: `[ci-doctor] <workflow name>: <one-line failure summary>`. Body must include:
   - Run URL and head SHA
   - The failing workflow and the failing step name
   - 3–10 lines of the most relevant error output, in a fenced code block
   - One short paragraph stating the most likely cause
   - One concrete next step for the human on-call

Be concise. The issue should fit on one screen.
