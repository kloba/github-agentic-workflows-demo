---
concurrency:
  cancel-in-progress: false
  group: ai-tester-${{ github.event.workflow_run.id || github.event.inputs.pr_number || 'manual' }}
description: After a successful backend or frontend deploy, verifies that the merged fix actually works on the deployed instance. Uses Playwright MCP for UI tests, HTTP for API tests. Takes a screenshot, posts a verification comment on the originating PR (NOT the issue — review-style feedback belongs on the PR). If the fix didn't land, files a fresh tester-rejected bug issue to re-fire Issue Fixer.
engine: copilot

# Note on runners: this workflow used `runs-on: ubuntu-latest` to
# reach the WG internal URLs, but the org's gh-runner's install_copilot_cli.sh
# step hardcodes /home/runner and fails on that runner's permission model.
# Until the runner is reconfigured (or a WG-attached runner is provisioned
# via cams-infra), we use ubuntu-latest. Smoke mode uses a public URL so WG
# isn't required; PR-verification mode will need a working WG runner.
name: AI Tester
# Self-hosted runner. The org's `gh-runner` is labeled `azure-self-hosted`
# and is online; it does not yet have WireGuard but the AI Tester still
# exercises Playwright + screenshot + comment mechanics. Reaching the
# internal URLs requires either:
#   1. Adding WG to that runner, OR
#   2. Provisioning a new AWS-VPC runner via cams-infra
# Until then, the tester falls back to public-URL verification (smoke
# mode) for non-frontend PRs and reports a "not reachable" comment for
# PRs that would require the internal URL.
runs-on: ubuntu-latest
"on":
  workflow_dispatch:
    inputs:
      pr_number:
        description: PR number to verify (optional — runs against the latest merged automated-fix PR if omitted)
        required: false
        type: string
      smoke_test:
        description: "Smoke test: navigate to a public URL, take a screenshot, post a comment. Verifies Playwright + safe-outputs mechanics without requiring WG."
        required: false
        type: boolean
        default: false
  workflow_run:
    types:
    - completed
    workflows:
    - Deploy Backend
    - Deploy Frontend
    branches:
    - main
permissions:
  contents: read
  issues: read
  pull-requests: read
safe-outputs:
  add-comment:
    max: 1
    # Comments target the PR via the explicit gh pr comment flow in the
    # prompt below. Review/verification feedback must NOT be posted on
    # issues (see #293) — only on PRs where the diff lives.
  add-labels:
    allowed:
    - tester-verified
    - tester-rejected
    - auto-fix-retry
    - bug
    - security
    - ci-failure
    max: 2
  noop:
    report-as-issue: false
timeout-minutes: 25
tools:
  bash:
  - gh issue *
  - gh pr *
  - gh api *
  - git log *
  - git show *
  - curl *
  - jq *
  - grep *
  - head *
  - tail *
  - wc *
  - cat *
  - ls *
  playwright:
    mode: cli
---
# AI Tester

A deploy has just finished. Your job is to verify that the most recently merged automated fix actually works on the deployed instance, post visual/HTTP evidence to the originating PR, and — if the verification fails — file a fresh `[tester-rejected]` bug issue so Issue Fixer can try again.

## SMOKE TEST MODE (manual)

If `${{ github.event.inputs.smoke_test }}` is `"true"`, **skip all normal verification logic** and run this minimal smoke test:

1. Navigate: `playwright-cli browser_navigate --url https://github.com/${{ github.repository }}`
2. Screenshot: `playwright-cli browser_take_screenshot --path /tmp/smoke.png`
3. Verify: `wc -c /tmp/smoke.png` (capture the byte count); `file /tmp/smoke.png` (should say PNG)
4. **Do NOT try to base64-encode.** The gh-aw firewall sandbox blocks `base64`, `xxd`, and other binary→text tools to prevent data exfiltration. Inline `data:image/png;base64,...` URLs are not possible from this environment — accept it and move on.
5. Find a tracking issue: `gh issue list --label agentic-workflows --state open --limit 5 --json number,title` — pick the one titled `[aw] AI Tester smoke run tracker` (issue #317, opened explicitly for this purpose). If that's gone, pick any open `agentic-workflows` issue.
6. Use `add-comment` targeted at the tracker issue with body:

   ```markdown
   ## 🔬 AI Tester smoke test result

   | Check | Result |
   |---|---|
   | Playwright `browser_navigate` invoked | ✅ |
   | Playwright `browser_take_screenshot` invoked | ✅ |
   | Screenshot file exists | ✅ |
   | Screenshot bytes (`wc -c /tmp/smoke.png`) | <paste byte count> |
   | `file` output | <paste output of `file /tmp/smoke.png`> |
   | Runner | $RUNNER_NAME (echo $RUNNER_NAME to grab) |
   | Run URL | ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }} |

   The screenshot binary is preserved in this run's artifacts (visible at the Run URL above → Artifacts panel). The gh-aw firewall sandbox blocks inline base64 data URLs, so an artifact-link is the substitute proof.

   This run was a smoke test only. No PR was verified, no labels applied.
   ```

7. Exit. Do NOT label PRs, do NOT open issues, do NOT call `create-issue`.

The success criteria for smoke mode:
- Playwright tool invokable (browser_navigate exit code 0)
- Screenshot file produced (`wc -c` > 1000)
- Comment lands on the tracker issue with the byte count populated
- Screenshot accessible via the workflow run's artifacts

Inline image rendering in the comment is NOT a success criterion — sandbox blocks it. The artifact link is the equivalent evidence.

## Eligibility

Skip immediately (call `noop`) if **any** of:

- The triggering `workflow_run.conclusion` is not `success`.
- The triggering workflow_run was not on the default branch (`refs/heads/main`).

## Step 1 — Find the PR to verify

```
# If invoked manually with pr_number, use that.
PR_NUM="${{ github.event.inputs.pr_number }}"

# Otherwise, find the most recent merged PR on main with an `automated-fix`
# or `automated-feature` label, that hasn't already been tested by us.
if [ -z "$PR_NUM" ]; then
  PR_NUM=$(gh pr list \
    --repo ${{ github.repository }} \
    --state merged \
    --base main \
    --label automated-fix,automated-feature \
    --limit 5 \
    --json number,labels,mergedAt \
    --jq '.[] | select(([.labels[].name] | index("tester-verified") | not) and ([.labels[].name] | index("tester-rejected") | not)) | .number' \
    | head -1)
fi
```

If `$PR_NUM` is empty, call `noop` — there's nothing fresh to verify.

## Step 2 — Read the PR + issue for context

```
gh pr view "$PR_NUM" --repo ${{ github.repository }} --json number,title,body,labels,closingIssuesReferences,changedFiles,files
```

Parse `closingIssuesReferences` for the issue this PR fixed. If multiple, pick the first. If none, the PR body's `Closes #N` text is the fallback — extract N from there.

```
gh issue view <N> --repo ${{ github.repository }} --json number,title,body,labels,author,state
```

You now have:
- The original issue text (what was broken / what feature was requested)
- The PR diff context (what was changed)
- Files touched (helps decide verification type)

## Step 3 — Decide the verification approach

Based on the files touched in the PR:

- **Frontend (`billing-dashboard/frontend/src/**`)** → Playwright UI test against `https://heart.int.valkyrie.org.ua` (admin login: see test plan in the PR; fallback admin creds from CLAUDE.md).
- **Backend router (`billing-dashboard/backend/routers/**`)** → HTTP test via `curl` against `https://heart-api.int.valkyrie.org.ua/<route>`.
- **Backend tests or pure DB/model** → no UI verification possible; skip with a comment.
- **Both** → run both and combine.

If the original issue specifies an exact reproduction (e.g., "Click X then Y, observe Z"), follow that. If not, derive a verification scenario from the PR title and the closing-issue body.

## Step 4 — Run the verification

### For UI verification (Playwright CLI)

Use `playwright-cli` in bash. Example:

```
playwright-cli navigate "https://heart.int.valkyrie.org.ua/login"
playwright-cli fill "input[name=username]" "admin"
playwright-cli fill "input[name=password]" "<from-CLAUDE.md>"
playwright-cli click "button[type=submit]"
playwright-cli wait-for-selector "[data-testid=dashboard]"
playwright-cli navigate "https://heart.int.valkyrie.org.ua/<affected-page>"
playwright-cli screenshot /tmp/verification.png
```

Assert the bug is gone (page content does NOT contain the original error message, target element IS visible, etc.) by reading the page text via `playwright-cli get-text <selector>` or DOM via `playwright-cli evaluate "<js>"`.

Do **NOT** base64-encode the screenshot — the gh-aw firewall sandbox blocks `base64` and other binary→text tools. Instead, reference the run's artifacts panel in the comment: the screenshot is accessible at the Run URL → Artifacts → agent.

### For API verification (HTTP)

```
# Example: verify a backend endpoint returns expected shape
curl -sS -o /tmp/response.json -w "HTTP %{http_code}\n" \
  "https://heart-api.int.valkyrie.org.ua/<route>"
jq . /tmp/response.json
```

Assert the response matches what the fix promised (status code, shape, absence of the error).

## Step 5 — Post the verdict

### If verification PASSED

1. `add-labels` `tester-verified` on the PR.
2. `add-comment` on the PR (one comment only — do NOT mirror this onto the linked issue; review-style feedback belongs on the PR per #293):

   ```markdown
   ## ✅ AI Tester verified

   Verification scenario for issue #<N> succeeded on the deployed instance.

   **Approach:** <one-line summary of what you tested>

   **Evidence:**
   <inline screenshot or curl output excerpt>

   Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
   ```

### If verification FAILED

Strict-mode gh-aw forbids `issues: write` so we can't reopen the original issue directly. Instead:

1. `add-labels` `tester-rejected` on the PR (so we don't re-test this same PR).
2. `add-comment` on the PR with the failure details.
3. `create-issue` — file a FRESH bug that re-enters the pipeline:

   ```markdown
   ## ❌ AI Tester rejected fix attempt for issue #<N>

   PR #<PR_NUM> claimed to fix issue #<N> but verification on the deployed instance failed.

   **What I tested:** <scenario>
   **What I expected:** <expected outcome>
   **What I observed:** <observed outcome>

   <inline screenshot or response excerpt showing the failure>

   Original issue: #<N>
   Failed PR: #<PR_NUM>
   Tester run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

   The Issue Fixer will pick this new issue up (label `bug` already applied) and try a different approach.
   ```

   The safe-output's auto-applied labels (`bug`, `tester-rejected`) ensure the Triager confirms it and Issue Fixer fires.

## Honest expectations

This agent will sometimes:

- Misidentify the verification scenario — file might be backend but the user-facing impact is on the frontend → posts a non-confirming comment.
- Fail to authenticate against the production UI — leave a comment explaining the auth issue rather than silently noop.
- Time out on slow page loads — extend timeout once, then report degraded performance as part of the result.
- Test against a stale deployed version if the deploy hasn't propagated — the `workflow_run` trigger waits for the deploy workflow to finish, but ECS rollout takes another 30-60s. If `curl` returns the old version, sleep 30s and retry once.

The point is **closing the loop**: every autonomous fix gets a real verification against prod, with visual/HTTP evidence, before the issue is considered closed.