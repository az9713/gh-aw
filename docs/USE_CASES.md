# Use Cases - GitHub Agentic Workflows

**25 copy-paste-ready workflows** you can drop into any repository. Each example is a
complete `.md` file — save it to `.github/workflows/`, compile with `gh aw compile`, and push.

---

## Table of Contents

**Getting Started**
- [Quick Setup](#quick-setup)

**Issue Management**
1. [Auto-Triage New Issues](#1-auto-triage-new-issues)
2. [Stale Issue Closer](#2-stale-issue-closer)
3. [Bug Report Validator](#3-bug-report-validator)
4. [Duplicate Issue Detector](#4-duplicate-issue-detector)

**Pull Request Automation**
5. [PR Review Bot](#5-pr-review-bot)
6. [PR Welcome Message](#6-pr-welcome-message)
7. [Stale Draft PR Cleanup](#7-stale-draft-pr-cleanup)
8. [Breaking Change Detector](#8-breaking-change-detector)

**CI/CD & DevOps**
9. [CI Failure Doctor](#9-ci-failure-doctor)
10. [Dependency Update Bundler](#10-dependency-update-bundler)
11. [Release Notes Generator](#11-release-notes-generator)

**Security**
12. [Code Scanning Auto-Fixer](#12-code-scanning-auto-fixer)
13. [Secret Leak Scanner](#13-secret-leak-scanner)

**Documentation**
14. [README Freshness Checker](#14-readme-freshness-checker)
15. [API Docs Generator](#15-api-docs-generator)

**Slash Commands**
16. [/deploy Command](#16-deploy-command)
17. [/explain Command](#17-explain-command)
18. [/refactor Command](#18-refactor-command)
19. [/merge-main Command](#19-merge-main-command)

**Scheduled Reports**
20. [Daily Repository Health Report](#20-daily-repository-health-report)
21. [Weekly Contributor Summary](#21-weekly-contributor-summary)

**Advanced Patterns**
22. [Multi-Engine Workflow (Claude)](#22-multi-engine-workflow-claude)
23. [Workflow with Custom MCP Server](#23-workflow-with-custom-mcp-server)
24. [Shared Config via Imports](#24-shared-config-via-imports)
25. [Rate-Limited Scheduled Workflow](#25-rate-limited-scheduled-workflow)

---

## Quick Setup

```bash
# Install (one-time)
gh extension install github/gh-aw

# Initialize your repo
cd your-repo
gh aw init

# Save any example below to .github/workflows/<name>.md
# Then compile and push
gh aw compile .github/workflows/<name>.md
git add .github/workflows/<name>.md .github/workflows/<name>.lock.yml
git commit -m "Add <name> agentic workflow"
git push
```

---

## Issue Management

### 1. Auto-Triage New Issues

**What it does:** When an issue is opened, the agent reads the title and body, then applies
appropriate labels (bug, enhancement, question, etc.) and a priority level. Runs every 6
hours to catch anything missed.

**Save as:** `.github/workflows/auto-triage.md`

```markdown
---
name: Auto-Triage Issues
description: Labels new issues by type and priority based on content analysis
on:
  issues:
    types: [opened, edited]
  schedule: every 6h
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 10
timeout-minutes: 10
---

# Auto-Triage Agent

You are an issue triage agent. When triggered, analyze issues and apply labels.

## On Issue Events (opened/edited)

Analyze the triggering issue from `github.event.issue`.

## On Schedule

Fetch up to 10 unlabeled issues and label each one.

## Classification Rules

Apply one **type** label:
- `bug` — error reports, stack traces, "doesn't work", "broken"
- `enhancement` — feature requests, "add support for", "would be nice"
- `documentation` — docs updates, README changes, "how to" guides
- `question` — usage questions, "how do I", clarification requests

Apply one **priority** label:
- `priority: critical` — "blocking", "urgent", data loss
- `priority: high` — impacts many users, security related
- `priority: medium` — default for bugs
- `priority: low` — cosmetic, nice-to-have

Apply **component** labels when clear:
- `area: frontend`, `area: backend`, `area: api`, `area: ci`

When uncertain, apply `needs-triage` and skip other labels.
```

---

### 2. Stale Issue Closer

**What it does:** Runs daily. Warns issues inactive for 30 days, then closes them at 45
days. Exempt labels prevent closure.

**Save as:** `.github/workflows/stale-issues.md`

```markdown
---
name: Stale Issue Closer
description: Warns and closes issues that have been inactive for extended periods
on:
  schedule: daily at 8am
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 20
  add-comments:
    max: 20
  close-issue:
    max: 10
timeout-minutes: 15
---

# Stale Issue Closer

You manage stale issues to keep the tracker clean.

## Policy

- **Warning (30+ days inactive):** Add `stale` label and a comment:
  "This issue has been inactive for 30 days. It will be closed in 15 days
  unless there is new activity. Add the `keep-open` label to prevent this."
- **Close (45+ days inactive):** Close with comment:
  "Closing due to inactivity. Reopen if this is still relevant."
- **Exempt labels:** Skip issues with `keep-open`, `blocked`, `in-progress`,
  or `priority: critical`.

## Steps

1. Search open issues sorted by `updated` ascending.
2. For each issue older than 30 days since last update:
   - Skip if it has an exempt label.
   - If 45+ days: close with comment.
   - Else if 30-44 days and no `stale` label: add label + warning comment.
3. Report a summary of actions taken.

## Inactivity Calculation

Use the `updated_at` field. A comment, label change, or assignment resets the clock.
```

---

### 3. Bug Report Validator

**What it does:** Checks newly opened bug reports for required information (steps to
reproduce, expected vs actual behavior). Posts a checklist comment if anything is missing.

**Save as:** `.github/workflows/bug-validator.md`

```markdown
---
name: Bug Report Validator
description: Checks bug reports for completeness and requests missing information
on:
  issues:
    types: [opened]
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 3
  add-comments:
    max: 1
timeout-minutes: 5
---

# Bug Report Validator

When a new issue is opened, check if it looks like a bug report. If it does,
validate completeness.

## Required Sections

A complete bug report should contain:
- **Steps to Reproduce** — numbered steps or clear description
- **Expected Behavior** — what should happen
- **Actual Behavior** — what happens instead
- **Environment** — OS, version, browser, or runtime

## Actions

- If the issue is **not** a bug report (feature request, question, etc.): do nothing.
- If the bug report has all required sections: add `validated` label.
- If sections are missing: add `needs-info` label and post a comment listing
  what is missing, using a checklist:
  ```
  Thanks for the report! A few details would help us investigate:
  - [ ] Steps to reproduce
  - [ ] Expected behavior
  - [ ] Actual behavior
  - [ ] Environment info (OS, version, etc.)
  ```
  Only list the missing items.
```

---

### 4. Duplicate Issue Detector

**What it does:** When a new issue is opened, searches for semantically similar existing
issues and posts links if duplicates are likely found.

**Save as:** `.github/workflows/duplicate-detector.md`

```markdown
---
name: Duplicate Issue Detector
description: Searches for similar existing issues when a new one is opened
on:
  issues:
    types: [opened]
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [issues, repos]
safe-outputs:
  add-labels:
    max: 2
  add-comments:
    max: 1
timeout-minutes: 10
---

# Duplicate Issue Detector

When a new issue is opened, search for potential duplicates.

## Process

1. Read the new issue title and body.
2. Extract 3-5 key terms (ignore common words like "the", "is", "not working").
3. Search open AND recently closed issues (last 90 days) using those terms.
4. For each candidate, compare:
   - Title similarity
   - Problem description overlap
   - Error messages or stack traces in common
5. If you find likely duplicates (high confidence):
   - Add `possible-duplicate` label.
   - Comment with links:
     ```
     This may be a duplicate of an existing issue:
     - #123 — [title]
     - #456 — [title]
     Please check if any of these address your problem.
     ```
6. If no duplicates found, do nothing. Don't post "no duplicates found" comments.
```

---

## Pull Request Automation

### 5. PR Review Bot

**What it does:** Activated by `/review` in a PR comment. Analyzes the diff and posts
targeted review comments on specific lines.

**Save as:** `.github/workflows/pr-review.md`

```markdown
---
name: PR Review Bot
description: AI-powered code review triggered by /review command
on:
  slash_command:
    name: review
    events: [pull_request_comment, pull_request_review_comment]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  create-pull-request-review-comment:
    max: 10
  add-comments:
    max: 1
  messages:
    footer: "> Reviewed by [{workflow_name}]({run_url})"
timeout-minutes: 15
---

# PR Review Bot

You are a thorough code reviewer. When triggered by `/review`, analyze the PR diff.

## Review Focus

Check for these categories (in priority order):
1. **Security** — injection, hardcoded secrets, unsafe deserialization
2. **Bugs** — null dereference, off-by-one, race conditions, resource leaks
3. **Performance** — N+1 queries, unnecessary allocations, missing indexes
4. **Error handling** — swallowed errors, missing validation, panic paths
5. **Readability** — unclear names, overly complex logic, missing context

## Output

- Post up to 10 inline review comments on specific lines with issues.
- Each comment should: state the problem, explain why it matters, suggest a fix.
- Post one summary comment with an overall assessment:
  - Number of issues by severity (critical / warning / suggestion)
  - Overall quality impression
  - What was done well (acknowledge good patterns)

## Guidelines

- Focus only on changed lines in the diff, not the entire file.
- Be constructive. Critique code, not people.
- If the code is clean, say so briefly. Don't invent issues.
- Prefix comments with severity: `🔴 Critical:`, `🟡 Warning:`, `💡 Suggestion:`.
```

---

### 6. PR Welcome Message

**What it does:** Greets first-time contributors with a welcome message and checklist when
they open their first PR.

**Save as:** `.github/workflows/pr-welcome.md`

```markdown
---
name: PR Welcome Bot
description: Welcomes first-time contributors with a helpful onboarding message
on:
  pull_request:
    types: [opened]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  add-labels:
    max: 1
  add-comments:
    max: 1
timeout-minutes: 5
---

# First-Time Contributor Welcome Bot

When a PR is opened, check if this is the author's first PR to this repository.

## Steps

1. Get the PR author's username from `github.event.pull_request.user.login`.
2. Search for previous merged or closed PRs by the same author in this repo.
3. If this is their first PR:
   - Add the `first-contribution` label.
   - Post a welcome comment:
     ```
     Welcome to the project, @<author>! Thanks for your first contribution.

     Here is a quick checklist:
     - [ ] Tests pass locally
     - [ ] Code follows the project style guide
     - [ ] Changes are documented (if applicable)

     A maintainer will review your PR soon. Feel free to ask questions!
     ```
4. If they have previous PRs, do nothing.
```

---

### 7. Stale Draft PR Cleanup

**What it does:** Runs daily. Warns draft PRs after 10 days of inactivity, closes them
after 14 days. Respects exemption labels.

**Save as:** `.github/workflows/draft-cleanup.md`

```markdown
---
name: Draft PR Cleanup
description: Warns and closes stale draft pull requests to keep the PR list clean
on:
  schedule: daily
  workflow_dispatch:
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  add-labels:
    max: 20
  add-comments:
    max: 20
  close-pull-request:
    max: 10
timeout-minutes: 15
---

# Draft PR Cleanup Agent

Enforce a cleanup policy for inactive draft pull requests.

## Policy

| Days Inactive | Action |
|--------------|--------|
| 0-9 | No action |
| 10-13 | Add `stale-draft` label + warning comment |
| 14+ | Close PR with explanation comment |

**Exempt labels:** `keep-draft`, `blocked`, `awaiting-review`

## Process

1. Query all open draft PRs: `is:pr is:open is:draft`
2. For each, calculate days since last activity (most recent of: last commit,
   last comment, last label change, `updated_at`).
3. Skip PRs with exempt labels.
4. Apply the appropriate action from the table above.
5. Warning comment text:
   "This draft PR has been inactive for X days and will be auto-closed in
   Y days. Push a commit, add a comment, or add `keep-draft` to prevent this."
6. Closing comment text:
   "Closing this draft due to 14+ days of inactivity. This is not a rejection.
   Feel free to reopen when work continues."
```

---

### 8. Breaking Change Detector

**What it does:** When a PR targets `main`, scans the diff for breaking changes (removed
public APIs, changed function signatures, deleted exports) and posts a warning.

**Save as:** `.github/workflows/breaking-changes.md`

```markdown
---
name: Breaking Change Detector
description: Warns when a PR contains potential breaking changes to public APIs
on:
  pull_request:
    types: [opened, synchronize]
    branches: [main]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  add-labels:
    max: 2
  add-comments:
    max: 1
timeout-minutes: 10
---

# Breaking Change Detector

Analyze PR diffs targeting `main` for potential breaking changes.

## What Counts as Breaking

- **Removed exports** — public functions, classes, types, or constants deleted
- **Changed signatures** — parameters added/removed/reordered in public functions
- **Renamed identifiers** — public symbols renamed without aliases
- **Changed return types** — functions returning different types
- **Deleted files** — files that other packages may import
- **Config format changes** — altered config file schemas

## Process

1. Get the list of changed files in the PR.
2. For each changed file, read the diff.
3. Look for patterns that match breaking changes above.
4. If breaking changes are found:
   - Add `breaking-change` label.
   - Post a comment listing each breaking change with file and line.
   - Format: `**File:** path/to/file.ts:42 — Removed export \`functionName\``
5. If no breaking changes found, do nothing. Don't add noise.

## Exclusions

Ignore changes in:
- Test files (`*_test.go`, `*.test.ts`, `*.spec.js`)
- Internal/private modules (paths containing `/internal/`, `_internal.py`)
- Generated files (`.lock.yml`, `package-lock.json`, `go.sum`)
```

---

## CI/CD & DevOps

### 9. CI Failure Doctor

**What it does:** Triggers when your CI workflow fails. Pulls logs, identifies the root
cause, and opens an issue with diagnosis and fix suggestions.

**Save as:** `.github/workflows/ci-doctor.md`

```markdown
---
name: CI Doctor
description: Investigates CI failures and creates diagnostic issues
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]
if: ${{ github.event.workflow_run.conclusion == 'failure' }}
permissions:
  actions: read
  contents: read
  issues: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [default, actions]
  cache-memory: true
safe-outputs:
  create-issue:
    title-prefix: "[CI Doctor] "
    close-older-issues: true
    max: 1
  noop:
timeout-minutes: 20
---

# CI Failure Doctor

You investigate failed CI runs and create diagnostic issues.

## Investigation Steps

1. **Verify failure:** If `github.event.workflow_run.conclusion` is not `failure`,
   call the `noop` tool and stop.
2. **Get run details:** Use `get_workflow_run` for the failed run ID.
3. **List failed jobs:** Use `list_workflow_jobs` to find which jobs failed.
4. **Pull logs:** Use `get_job_logs` with `failed_only=true`.
5. **Analyze:** Identify the root cause:
   - Test failures (which tests, what assertions)
   - Build errors (compilation, missing deps)
   - Infrastructure (runner, network, timeout)
   - Flaky tests (intermittent, timing-dependent)
6. **Check history:** Search existing issues for the same error pattern.
7. **Create issue** (if not a duplicate of a recent one):

   ```
   ## Summary
   [One-line description]

   ## Failed Jobs
   - Job: [name] — [error summary]

   ## Root Cause
   [Detailed analysis]

   ## Suggested Fix
   - [ ] [Specific actionable step]
   - [ ] [Another step]

   ## Logs
   <details><summary>Relevant log output</summary>
   [Key error lines — not the full log]
   </details>
   ```
```

---

### 10. Dependency Update Bundler

**What it does:** Runs weekly. Finds all open Dependabot PRs and creates a single tracking
issue grouping updates by runtime (npm, Go, pip, etc.).

**Save as:** `.github/workflows/dep-bundler.md`

```markdown
---
name: Dependency Update Bundler
description: Groups open Dependabot PRs into a summary issue per runtime
on:
  schedule: weekly on monday
permissions:
  contents: read
  issues: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, issues]
safe-outputs:
  create-issue:
    title-prefix: "[Deps] "
    close-older-issues: true
    max: 3
timeout-minutes: 10
---

# Dependency Update Bundler

Create summary issues for open Dependabot PRs, grouped by ecosystem.

## Steps

1. Search for open PRs authored by `dependabot[bot]` or `app/dependabot`.
2. Group them by ecosystem:
   - **npm** — `package.json`, `package-lock.json`
   - **Go** — `go.mod`, `go.sum`
   - **pip** — `requirements.txt`, `pyproject.toml`
   - **Actions** — `.github/workflows/*.yml`
   - **Other** — anything else
3. For each ecosystem with open PRs, create one issue:
   ```
   ## Dependency Updates — [Ecosystem]

   | Package | From | To | PR | Severity |
   |---------|------|----|----|----------|
   | lodash | 4.17.20 | 4.17.21 | #123 | patch |
   | express | 4.18.0 | 5.0.0 | #124 | major |

   ### Recommended Action
   - Patch updates: safe to merge
   - Minor updates: review changelog
   - Major updates: test thoroughly before merging
   ```
4. If no open Dependabot PRs exist, do nothing.
```

---

### 11. Release Notes Generator

**What it does:** When a release is published, reads the commits since the last release
and generates categorized release notes.

**Save as:** `.github/workflows/release-notes.md`

```markdown
---
name: Release Notes Generator
description: Generates categorized release notes when a release is published
on:
  release:
    types: [published]
permissions:
  contents: read
engine: copilot
tools:
  github:
    toolsets: [repos, pull_requests]
safe-outputs:
  update-release:
    max: 1
timeout-minutes: 10
---

# Release Notes Generator

Generate structured release notes for the just-published release.

## Steps

1. Get the current release tag from `github.event.release.tag_name`.
2. Find the previous release tag.
3. List all commits between the two tags.
4. For each commit, check if it was part of a PR (extract PR number from message).
5. Categorize by conventional commit prefix or PR labels:
   - **Features** — `feat:`, label `enhancement`
   - **Bug Fixes** — `fix:`, label `bug`
   - **Performance** — `perf:`, label `performance`
   - **Documentation** — `docs:`, label `documentation`
   - **Breaking Changes** — `BREAKING CHANGE:`, label `breaking-change`
   - **Other** — everything else
6. Update the release body using the `update-release` safe output:
   ```
   ## What's Changed

   ### Breaking Changes
   - Description (#PR) by @author

   ### Features
   - Description (#PR) by @author

   ### Bug Fixes
   - Description (#PR) by @author

   ### Other Changes
   - Description (#PR) by @author

   **Full Changelog**: compare/v1.0.0...v1.1.0
   ```
```

---

## Security

### 12. Code Scanning Auto-Fixer

**What it does:** Manually triggered. Picks the highest-severity open code scanning alert,
writes a fix, and opens a PR.

**Save as:** `.github/workflows/security-fixer.md`

```markdown
---
name: Security Alert Fixer
description: Automatically fixes code scanning alerts by creating pull requests
on:
  workflow_dispatch:
  skip-if-match: 'is:pr is:open in:title "[security-fix]"'
permissions:
  contents: read
  pull-requests: read
  security-events: read
engine: copilot
tools:
  github:
    toolsets: [context, repos, code_security, pull_requests]
  edit:
  bash:
    - "npm test *"
    - "go test *"
    - "python -m pytest *"
safe-outputs:
  create-pull-request:
    title-prefix: "[security-fix] "
    labels: [security, automated-fix]
    reviewers: [copilot]
    max: 1
timeout-minutes: 20
---

# Security Alert Fixer

Fix the highest-severity open code scanning alert.

## Process

1. List all open code scanning alerts, sorted by severity
   (critical > high > medium > low).
2. Select the top alert that doesn't already have an open fix PR
   (check `skip-if-match` to avoid duplicates).
3. Read the affected file and understand the vulnerability.
4. Write a minimal, surgical fix:
   - Address the root cause (not just the symptom).
   - Follow security best practices for the vulnerability type.
   - Don't change unrelated code.
5. Run the project's test suite to verify nothing breaks.
6. Create a PR with:
   ```
   ## Security Fix: [Rule ID]

   **Alert:** #[number] | **Severity:** [level] | **CWE:** [id]

   ### Vulnerability
   [What the issue is and why it matters]

   ### Fix
   [What was changed and why this approach was chosen]

   ### Testing
   - [ ] Existing tests pass
   - [ ] Fix addresses the root cause (not just the flagged line)
   ```

## Rules

- One alert per run. Quality over quantity.
- Never introduce new vulnerabilities while fixing one.
- If a fix requires architectural changes, document it in the PR instead of
  making a risky automated change.
```

---

### 13. Secret Leak Scanner

**What it does:** Scans PR diffs for patterns that look like leaked secrets (API keys,
tokens, passwords) and posts a warning before the PR can be merged.

**Save as:** `.github/workflows/secret-scanner.md`

```markdown
---
name: Secret Leak Scanner
description: Scans PR diffs for accidentally committed secrets
on:
  pull_request:
    types: [opened, synchronize]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  add-labels:
    max: 1
  add-comments:
    max: 1
timeout-minutes: 10
---

# Secret Leak Scanner

Scan PR diffs for patterns that look like accidentally committed secrets.

## Patterns to Detect

- API keys: `AKIA[0-9A-Z]{16}`, `sk-[a-zA-Z0-9]{32,}`, `ghp_[a-zA-Z0-9]{36}`
- Tokens: `token`, `bearer`, `authorization` followed by a long string
- Passwords: `password`, `passwd`, `secret` assigned a literal string value
- Private keys: `-----BEGIN (RSA |EC |DSA )?PRIVATE KEY-----`
- Connection strings: `mysql://`, `postgres://`, `mongodb://` with credentials
- Environment files: `.env` files being committed

## Process

1. Get the list of files changed in the PR.
2. For each file, read the diff (added lines only — `+` lines).
3. Check each added line against the patterns above.
4. Ignore:
   - Test fixtures and mock data (files in `test/`, `__tests__/`, `fixtures/`)
   - Example/placeholder values (`xxx`, `your-key-here`, `REPLACE_ME`)
   - Lines in `.gitignore` or documentation explaining secrets setup
5. If secrets are found:
   - Add `security-review` label.
   - Post a comment:
     ```
     **Potential secret detected in this PR.**

     | File | Line | Pattern |
     |------|------|---------|
     | path/to/file | 42 | AWS Access Key (AKIA...) |

     Please remove these before merging. Use environment variables
     or a secrets manager instead.
     ```
6. If no secrets found, do nothing.
```

---

## Documentation

### 14. README Freshness Checker

**What it does:** Runs weekly. Compares the README against the actual codebase to find
outdated instructions, dead links, or missing sections.

**Save as:** `.github/workflows/readme-checker.md`

```markdown
---
name: README Freshness Checker
description: Verifies README accuracy against actual codebase state
on:
  schedule: weekly on wednesday
  workflow_dispatch:
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [repos, issues]
safe-outputs:
  create-issue:
    title-prefix: "[Docs] "
    close-older-issues: true
    labels: [documentation]
    max: 1
timeout-minutes: 15
---

# README Freshness Checker

Audit the README for accuracy.

## Checks

1. **Installation instructions** — Do the listed commands still work?
   Check if referenced packages, CLI tools, or versions exist.
2. **Code examples** — Do function names, imports, and APIs in examples
   match the actual source code? Read referenced files to verify.
3. **File references** — Are all referenced file paths (`src/`, `docs/`,
   config files) still present in the repo?
4. **Badge URLs** — Do CI badge URLs point to existing workflows?
5. **Section completeness** — Are there major features in the code that
   the README doesn't mention?

## Output

If issues are found, create a single issue:
```
## README Audit Results

### Outdated Content
- [ ] Section "Installation": references `v2.0` but latest is `v3.1`
- [ ] Code example in "Quick Start" uses `oldFunction()` which was renamed

### Missing Documentation
- [ ] New `--verbose` flag is not documented
- [ ] The `config.yaml` format changed but docs still show old format

### Dead References
- [ ] Link to `docs/API.md` — file does not exist
```

If the README is up to date, do nothing.
```

---

### 15. API Docs Generator

**What it does:** When a PR modifies API-related files, generates or updates API
documentation and posts it as a PR comment.

**Save as:** `.github/workflows/api-docs.md`

```markdown
---
name: API Docs Generator
description: Generates API documentation from code changes in PRs
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "src/api/**"
      - "routes/**"
      - "handlers/**"
      - "controllers/**"
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  add-comments:
    max: 1
timeout-minutes: 10
---

# API Docs Generator

When API-related files change, generate documentation for new or modified endpoints.

## Steps

1. Get the list of changed files in the PR.
2. Filter to API-related files (routes, handlers, controllers).
3. For each changed file, read the new version.
4. Extract endpoint information:
   - HTTP method and path
   - Request parameters (path, query, body)
   - Response format and status codes
   - Authentication requirements
5. Post a comment with the generated docs:
   ```
   ## API Changes in This PR

   ### New Endpoints

   #### `POST /api/users`
   Creates a new user account.

   **Request Body:**
   | Field | Type | Required | Description |
   |-------|------|----------|-------------|
   | email | string | yes | User email |
   | name | string | yes | Display name |

   **Response:** `201 Created`

   ### Modified Endpoints

   #### `GET /api/users/:id`
   - Added `include` query parameter for related resources

   ### Removed Endpoints
   - `DELETE /api/legacy/users` — removed in favor of `/api/users/:id`
   ```
```

---

## Slash Commands

### 16. /deploy Command

**What it does:** When someone comments `/deploy staging` or `/deploy production` on a PR,
validates the target environment and triggers a deployment.

**Save as:** `.github/workflows/deploy-command.md`

```markdown
---
name: Deploy Command
description: Triggers deployment from PR comments using /deploy command
on:
  slash_command:
    name: deploy
    events: [pull_request_comment]
permissions:
  contents: read
  pull-requests: read
  actions: read
engine: copilot
roles:
  - admin
  - maintainer
tools:
  github:
    toolsets: [pull_requests, repos, actions]
safe-outputs:
  add-comments:
    max: 1
  dispatch-workflow:
    max: 1
timeout-minutes: 5
---

# Deploy Command Handler

Triggered by `/deploy [environment]` in a PR comment.

## Supported Environments

- `staging` — deploys to staging environment
- `production` — deploys to production (requires admin role)

## Steps

1. Parse the command: extract the target environment from
   `needs.activation.outputs.text` (text after `/deploy`).
2. Validate:
   - Environment must be `staging` or `production`.
   - PR must be in `open` state with all checks passing.
   - For `production`: verify the commenter has admin or maintainer role.
3. If valid: dispatch the deployment workflow using `dispatch-workflow` safe output
   with the target environment as input. Comment:
   "Deployment to **[environment]** initiated. Track progress in the Actions tab."
4. If invalid: comment with the reason.
   "Cannot deploy: [reason]. Usage: `/deploy staging` or `/deploy production`."
```

---

### 17. /explain Command

**What it does:** When someone comments `/explain` on an issue or PR, the agent provides
a plain-English explanation of the code, change, or error discussed.

**Save as:** `.github/workflows/explain-command.md`

```markdown
---
name: Explain Command
description: Explains code, changes, or errors in plain English
on:
  slash_command:
    name: explain
    events: [issue_comment, pull_request_comment]
permissions:
  contents: read
  issues: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [repos, issues, pull_requests]
safe-outputs:
  add-comments:
    max: 1
timeout-minutes: 10
---

# Explain Command

Provide a clear, beginner-friendly explanation when `/explain` is used.

## Context Detection

Determine what needs explaining based on where the command was used:

- **On a PR:** Explain what the PR changes, why, and how it works.
  Read the diff and summarize each file's changes in plain English.
- **On an issue:** Explain the problem described, possible causes,
  and what area of the codebase is likely involved.
- **With a file path** (`/explain src/auth.ts`): Read the file and explain
  what it does, its role in the project, and key functions.

## Response Format

Post a comment with:
```
### Explanation

[2-4 paragraph plain-English explanation]

### Key Points
- [Bullet point 1]
- [Bullet point 2]
- [Bullet point 3]

### Related Files
- `path/to/file.ts` — [one-line description]
```

Write for someone unfamiliar with the codebase. Avoid jargon.
```

---

### 18. /refactor Command

**What it does:** When `/refactor` is used on a PR, the agent suggests concrete
refactoring improvements for the changed code.

**Save as:** `.github/workflows/refactor-command.md`

```markdown
---
name: Refactor Suggestions
description: Suggests code refactoring improvements for PR changes
on:
  slash_command:
    name: refactor
    events: [pull_request_comment]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
safe-outputs:
  create-pull-request-review-comment:
    max: 8
  add-comments:
    max: 1
  messages:
    footer: "> Refactoring suggestions by [{workflow_name}]({run_url})"
timeout-minutes: 15
---

# Refactor Command

Analyze the PR diff and suggest refactoring opportunities.

## What to Look For

- **Extract method** — repeated logic that could be a function
- **Simplify conditionals** — nested ifs that could be guard clauses or early returns
- **Remove duplication** — copy-pasted blocks across files
- **Improve naming** — vague names like `data`, `result`, `temp`, `x`
- **Reduce complexity** — functions doing too many things
- **Use language idioms** — non-idiomatic patterns for the language

## Output

- Post inline review comments on specific lines with the suggestion.
  Each comment: what to change, why, and a brief code snippet showing the improvement.
- Post one summary comment:
  ```
  ## Refactoring Suggestions

  Found [N] opportunities across [M] files:
  - [count] extract method
  - [count] simplify conditional
  - [count] naming improvement

  These are suggestions, not requirements. Apply what makes sense.
  ```
```

---

### 19. /merge-main Command

**What it does:** When `/merge-main` is commented on a PR, the agent merges the base
branch into the PR branch and pushes the result.

**Save as:** `.github/workflows/merge-main.md`

```markdown
---
name: Merge Main
description: Merges base branch into PR branch on /merge-main command
on:
  slash_command:
    name: merge-main
    events: [pull_request_comment]
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [pull_requests, repos]
  bash:
    - "git *"
  edit:
safe-outputs:
  push-to-pull-request-branch:
  add-comments:
    max: 1
timeout-minutes: 10
steps:
  - name: Configure Git
    run: |
      git config user.name "github-actions[bot]"
      git config user.email "github-actions[bot]@users.noreply.github.com"
---

# Merge Main into PR Branch

When `/merge-main` is used, merge the base branch into this PR's branch.

## Steps

1. Get PR details to find the head branch and base branch names.
2. Fetch latest from origin.
3. Checkout the PR branch.
4. Run `git merge origin/<base-branch> --no-edit`.
5. If the merge succeeds cleanly:
   - Push via `push-to-pull-request-branch` safe output.
   - Comment: "Merged `<base>` into `<head>` successfully. [N] commits merged."
6. If there are merge conflicts:
   - Attempt to resolve simple conflicts (generated files, lock files).
   - For code conflicts that require judgment, abort the merge and comment:
     "Merge conflicts detected in [files]. Please resolve manually."

## Rules

- Never force push.
- Never resolve ambiguous code conflicts automatically.
- Always report what happened in a comment.
```

---

## Scheduled Reports

### 20. Daily Repository Health Report

**What it does:** Runs daily. Produces a discussion summarizing open issues, PR backlog,
CI status, and contributor activity.

**Save as:** `.github/workflows/daily-health.md`

```markdown
---
name: Daily Health Report
description: Generates a daily summary of repository health metrics
on:
  schedule: daily at 9am
permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read
engine: copilot
tools:
  github:
    toolsets: [default, actions]
safe-outputs:
  create-discussion:
    title-prefix: "[Health Report] "
    category: announcements
    close-older-discussions: true
    max: 1
timeout-minutes: 15
---

# Daily Health Report

Generate a repository health summary as a discussion post.

## Metrics to Collect

1. **Issues**: total open, opened today, closed today, oldest unresolved
2. **Pull Requests**: open count, draft count, avg age, oldest open PR
3. **CI Status**: last 5 workflow runs on main (pass/fail), current streak
4. **Response Times**: median time to first response on issues/PRs (last 7 days)
5. **Activity**: commits to main (last 24h), unique contributors (last 7 days)

## Report Format

```
### Repository Health — [date]

| Metric | Value | Trend |
|--------|-------|-------|
| Open Issues | 42 | +3 from yesterday |
| Open PRs | 12 | -1 from yesterday |
| CI Status | Passing | 15 run streak |
| Median Response Time | 4.2 hours | Improved |

### Attention Needed
- PR #89 has been open for 21 days with no review
- Issue #201 is marked critical with no assignee
- CI failed 3 times this week on the `lint` job

### Wins
- Closed 8 issues this week
- Average PR merge time improved to 1.2 days
```

Keep it concise. Focus on actionable items and trends.
```

---

### 21. Weekly Contributor Summary

**What it does:** Posts a weekly discussion recognizing contributors and summarizing merged
work.

**Save as:** `.github/workflows/weekly-contributors.md`

```markdown
---
name: Weekly Contributor Summary
description: Recognizes contributors and summarizes merged work each week
on:
  schedule: weekly on friday at 4pm
permissions:
  contents: read
  pull-requests: read
engine: copilot
tools:
  github:
    toolsets: [repos, pull_requests]
safe-outputs:
  create-discussion:
    title-prefix: "[Weekly] "
    category: announcements
    close-older-discussions: true
    max: 1
timeout-minutes: 10
---

# Weekly Contributor Summary

Generate a weekly summary recognizing everyone who contributed.

## Data to Collect

1. All PRs merged in the last 7 days.
2. For each PR: author, title, files changed, lines added/removed.
3. Group by contributor.

## Report Format

```
### Week of [date range]

**[N] PRs merged by [M] contributors**

### Contributors

**@alice** — 3 PRs
- Fix authentication timeout (#123)
- Add retry logic to API client (#124)
- Update deployment docs (#125)

**@bob** — 1 PR
- Refactor database connection pooling (#130)

### By the Numbers
- Lines added: +1,234
- Lines removed: -567
- Files changed: 42
- Most active area: src/api/

Thanks to everyone who contributed this week!
```
```

---

## Advanced Patterns

### 22. Multi-Engine Workflow (Claude)

**What it does:** Demonstrates using the Claude engine with MCP servers for tool access.
Performs deep code analysis on demand.

**Save as:** `.github/workflows/deep-analysis.md`

```markdown
---
name: Deep Code Analysis
description: In-depth code analysis using Claude engine with MCP tools
on:
  slash_command:
    name: analyze
    events: [issue_comment, pull_request_comment]
permissions:
  contents: read
  issues: read
  pull-requests: read
engine: claude
tools:
  github:
    toolsets: [repos, issues, pull_requests]
  web-search: {}
  web-fetch: {}
safe-outputs:
  add-comments:
    max: 1
network:
  allowed:
    - defaults
    - github
timeout-minutes: 20
---

# Deep Code Analysis (Claude)

When triggered by `/analyze`, perform an in-depth analysis.

## Context-Aware Behavior

- **On an issue**: Analyze the reported problem. Read relevant source files,
  trace the likely code path, and explain the root cause with references
  to specific lines.
- **On a PR**: Go beyond a surface review. Analyze architectural impact,
  identify subtle bugs, check for performance regressions, and evaluate
  test coverage gaps.
- **With arguments** (`/analyze src/auth/`): Focus analysis on the specified
  path. Explain the module's design, identify tech debt, and suggest improvements.

## Output

Post a detailed comment with:
1. **Executive Summary** — 2-3 sentences
2. **Detailed Analysis** — organized by topic
3. **Recommendations** — prioritized action items
4. **References** — links to relevant files and lines
```

---

### 23. Workflow with Custom MCP Server

**What it does:** Shows how to connect a custom MCP server (e.g., a database query tool)
to give the agent access to external data sources.

**Save as:** `.github/workflows/db-health.md`

```markdown
---
name: Database Health Check
description: Queries database metrics via custom MCP server and reports findings
on:
  schedule: daily at 6am
  workflow_dispatch:
permissions:
  contents: read
  issues: read
engine: copilot
tools:
  github:
    toolsets: [issues]
  postgres-mcp:
    type: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-postgres"]
    env:
      DATABASE_URL: "${{ secrets.DATABASE_URL }}"
    allowed: ["query"]
safe-outputs:
  create-issue:
    title-prefix: "[DB Health] "
    close-older-issues: true
    labels: [infrastructure]
    max: 1
timeout-minutes: 10
---

# Database Health Check

Query database metrics and create an issue if problems are found.

## Checks

Using the `query` tool from the postgres MCP server:

1. **Table sizes** — Find tables over 1GB.
2. **Slow queries** — Check `pg_stat_statements` for queries over 1s avg.
3. **Connection count** — Compare active connections to `max_connections`.
4. **Unused indexes** — Find indexes with zero scans.
5. **Dead tuples** — Tables with high dead tuple ratios needing VACUUM.

## Reporting

- If all checks pass: do nothing.
- If any check fails, create an issue:
  ```
  ## Database Health Alert

  ### Problems Found

  **Table Sizes (>1GB)**
  | Table | Size |
  |-------|------|
  | events | 2.3 GB |

  **Slow Queries**
  | Query | Avg Time | Calls |
  |-------|----------|-------|

  ### Recommended Actions
  - [ ] Run VACUUM ANALYZE on [table]
  - [ ] Add index on [table.column]
  - [ ] Review and optimize [query]
  ```
```

---

### 24. Shared Config via Imports

**What it does:** Shows how to create reusable shared configurations that multiple
workflows can import to reduce duplication.

**Create shared config:** `.github/workflows/shared/standard-tools.md`

```markdown
---
tools:
  github:
    toolsets: [default]
  cache-memory: true
network:
  allowed:
    - defaults
    - github
---
```

**Create shared config:** `.github/workflows/shared/reporting.md`

```markdown
---
safe-outputs:
  create-discussion:
    category: audits
    close-older-discussions: true
    max: 1
  messages:
    footer: "> Generated by [{workflow_name}]({run_url})"
---
```

**Workflow using imports:** `.github/workflows/audit-workflow.md`

```markdown
---
name: Code Audit
description: Weekly code quality audit with shared tool and reporting config
on:
  schedule: weekly on monday
permissions:
  contents: read
engine: copilot
imports:
  - shared/standard-tools.md
  - shared/reporting.md
timeout-minutes: 15
---

# Weekly Code Audit

This workflow imports shared tool and reporting configurations.
It inherits: GitHub tools, cache memory, network rules, and discussion output.

## Task

1. Scan the repository for code quality issues:
   - Files over 500 lines
   - Functions over 50 lines
   - TODO/FIXME/HACK comments
   - Unused imports or variables (based on naming conventions)
2. Create a discussion with findings using the imported reporting config.
```

The `imports` field performs BFS resolution — imported configs are merged into the parent.
Tools, safe-outputs, permissions, and markdown bodies all merge together. This lets you
define your organization's standard tool setup once and reuse it everywhere.

---

### 25. Rate-Limited Scheduled Workflow

**What it does:** Demonstrates rate limiting, stop-after, skip conditions, and manual
approval — all the safety features for production workflows.

**Save as:** `.github/workflows/production-audit.md`

```markdown
---
name: Production Security Audit
description: Comprehensive security audit with production safety controls
on:
  schedule: daily at 2am
  workflow_dispatch:
  stop-after: "+90d"
  skip-if-match:
    query: "is:issue is:open label:security-audit"
    max: 3
  manual-approval: production
  reaction: eyes
rate-limit:
  max: 3
  window: 1440
  ignored-roles: [admin]
permissions:
  contents: read
  security-events: read
  issues: read
engine: copilot
strict: true
network:
  allowed:
    - defaults
tools:
  github:
    toolsets: [repos, code_security, issues]
    read-only: true
safe-outputs:
  create-issue:
    title-prefix: "[Security Audit] "
    labels: [security-audit, automated]
    max: 1
  messages:
    run-started: "Starting security audit... [{workflow_name}]({run_url})"
    run-success: "Audit complete. [{workflow_name}]({run_url})"
    run-failure: "Audit failed. [{workflow_name}]({run_url}) {status}"
timeout-minutes: 30
---

# Production Security Audit

Comprehensive security review with safety controls.

## Safety Controls Explained

This workflow demonstrates several production safety features:
- **`stop-after: +90d`** — auto-disables after 90 days (forces re-evaluation)
- **`skip-if-match`** — skips if 3+ audit issues are already open (prevents flood)
- **`manual-approval: production`** — requires environment approval before running
- **`rate-limit`** — max 3 runs per day, admins exempt
- **`strict: true`** — strict validation mode
- **`read-only: true`** — tools can only read, never write
- **`reaction: eyes`** — shows acknowledgment on trigger

## Audit Scope

1. **Code Scanning Alerts** — summarize open alerts by severity
2. **Dependency Vulnerabilities** — check for known CVEs in dependencies
3. **Permission Audit** — review workflow permissions for least-privilege
4. **Secret Usage** — verify secrets are not logged or exposed in outputs

## Output

Create one issue summarizing all findings with severity ratings.
```

---

## Quick Reference: Frontmatter Cheat Sheet

```yaml
# --- Required ---
on:                              # Trigger(s)

# --- Common ---
name: "Workflow Name"            # Display name
description: "What it does"      # Description
engine: copilot                  # copilot | claude | codex | custom
permissions:                     # GitHub token permissions
  contents: read
  issues: write
tools:                           # AI tools available
  github:
    toolsets: [default]
safe-outputs:                    # Allowed write operations
  add-labels:
    max: 10
timeout-minutes: 15              # Max execution time

# --- Safety ---
rate-limit:                      # Prevent runaway execution
  max: 5
  window: 60
strict: true                     # Strict validation
network:                         # Network allowlist
  allowed: [defaults]
roles: [admin, maintainer]       # Who can trigger

# --- Advanced ---
imports:                         # Reuse shared configs
  - shared/tools.md
  - path: shared/config.md
    inputs:
      count: 10
stop-after: "+30d"               # Auto-disable date
skip-if-match: "is:issue label:x"  # Skip condition
manual-approval: production      # Require env approval
reaction: eyes                   # React to trigger event
```

---

## Next Steps

1. **Start simple.** Pick one workflow from the Issue Management section.
2. **Compile and test.** Use `gh aw compile` and push to see it run.
3. **Iterate.** Tune the instructions based on the agent's actual output.
4. **Scale.** Use imports to share configs across your growing set of workflows.
5. **Go to production.** Add rate limits, stop-after, and manual approval for critical workflows.

For full configuration reference, see the [Architecture Guide](ARCHITECTURE.md).
For step-by-step tutorials, see the [User Guide](USER_GUIDE.md).
