---
name: Security Scanner
description: Daily targeted security review of the backend — opens issues for high-confidence findings only.
on:
  schedule: daily
  workflow_dispatch:

permissions:
  contents: read
  issues: read

engine: copilot
timeout-minutes: 25

tools:
  bash:
    - "git *"
    - "grep *"
    - "find *"
    - "cat *"
    - "head *"
    - "wc *"
    - "gh issue list *"
    - "gh api *"

safe-outputs:
  create-issue:
    title-prefix: "[security] "
    labels: [security, automated-analysis, needs-triage]
    max: 3
    close-older-issues: false
    expires: 30d
  noop:
    report-as-issue: false
---

# Security Scanner

Daily targeted scan of `billing-dashboard/backend/`. **Bias hard toward false negatives** — only file an issue when you would defend the finding in code review.

## Scope

Look only under `billing-dashboard/backend/`. Skip everything else, including:
- `billing-dashboard/backend/tests/**` (fixtures may legitimately contain sample-looking secrets)
- `billing-dashboard/backend/alembic/versions/**`
- `__pycache__`, `*.pyc`

## Findings to look for

1. **Hardcoded secrets.** API keys, passwords, tokens, Azure Service Principal client secrets, Storage Account access keys, SAS tokens, connection strings, JWT signing keys, database passwords directly in source. Exclude:
   - Anything matching `os.environ`, `os.getenv`, `Settings()`, or read from Azure Key Vault (`SecretClient.get_secret`).
   - Comments, docstrings, example strings clearly marked as such.

2. **SQL injection.** String-formatted SQL (`f"SELECT ... {user_input}"`, `"... " + var`, `%s`-style with manual interpolation) that is NOT going through SQLAlchemy parameterized binds or `text(":param")` with bindparams. Note: raw `text()` calls with `:named` bindings ARE safe.

3. **Command injection.** `subprocess.run(..., shell=True)` or `os.system(...)` where any input could be non-static.

4. **Unsafe deserialization.** `pickle.loads`, `yaml.load` without `SafeLoader`, `eval(`, `exec(` on non-static input.

5. **Auth-bypass risks.** A FastAPI router endpoint missing `Depends(get_current_user)` / `Depends(require_admin)` / equivalent where the *peer endpoints in the same router file* have it. Skip routers that are intentionally public (e.g., `/health`, `/login`).

## Dedup

Before filing anything:
```
gh issue list --label security --state open --json number,title,body --limit 50
```
If an open `security` issue covers the same `file:line` finding, do NOT create a duplicate. Call `noop` for that finding.

## Issue format

Per finding, file one issue with:
- Title: `[security] <category>: <short description>` (e.g., `[security] hardcoded-secret: Azure Storage account key in storage_client.py:42`)
- Body:
  - File:line
  - 3–8 lines of code context in a fenced block
  - Why it's risky (one paragraph)
  - One concrete remediation
  - Severity guess: `low`, `medium`, `high`, `critical`

If no qualifying findings, call `noop`. The default is no issue, not "find something".
