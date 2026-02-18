# Use Cases - GitHub Agentic Workflows

**25 real-world AI workflows you can copy-paste into your project.** Each one
is a complete, working example with step-by-step instructions. No prior
experience with Git, GitHub, or GitHub Actions is required.

---

## Table of Contents

- [Before You Start (Required First-Time Setup)](#before-you-start-required-first-time-setup)
- [How Every Use Case Works (Read This First)](#how-every-use-case-works-read-this-first)
- **Issue Management**
  1. [Auto-Triage New Issues](#1-auto-triage-new-issues)
  2. [Stale Issue Closer](#2-stale-issue-closer)
  3. [Bug Report Validator](#3-bug-report-validator)
  4. [Duplicate Issue Detector](#4-duplicate-issue-detector)
- **Pull Request Automation**
  5. [PR Review Bot](#5-pr-review-bot)
  6. [PR Welcome Message](#6-pr-welcome-message)
  7. [Stale Draft PR Cleanup](#7-stale-draft-pr-cleanup)
  8. [Breaking Change Detector](#8-breaking-change-detector)
- **CI/CD & DevOps**
  9. [CI Failure Doctor](#9-ci-failure-doctor)
  10. [Dependency Update Bundler](#10-dependency-update-bundler)
  11. [Release Notes Generator](#11-release-notes-generator)
- **Security**
  12. [Code Scanning Auto-Fixer](#12-code-scanning-auto-fixer)
  13. [Secret Leak Scanner](#13-secret-leak-scanner)
- **Documentation**
  14. [README Freshness Checker](#14-readme-freshness-checker)
  15. [API Docs Generator](#15-api-docs-generator)
- **Slash Commands**
  16. [/deploy Command](#16-deploy-command)
  17. [/explain Command](#17-explain-command)
  18. [/refactor Command](#18-refactor-command)
  19. [/merge-main Command](#19-merge-main-command)
- **Scheduled Reports**
  20. [Daily Repository Health Report](#20-daily-repository-health-report)
  21. [Weekly Contributor Summary](#21-weekly-contributor-summary)
- **Advanced Patterns**
  22. [Multi-Engine Workflow (Claude)](#22-multi-engine-workflow-claude)
  23. [Workflow with Custom MCP Server](#23-workflow-with-custom-mcp-server)
  24. [Shared Config via Imports](#24-shared-config-via-imports)
  25. [Rate-Limited Scheduled Workflow](#25-rate-limited-scheduled-workflow)
- [Quick Reference: Frontmatter Cheat Sheet](#quick-reference-frontmatter-cheat-sheet)

---

## Before You Start (Required First-Time Setup)

You only need to do this once. If you have already completed these steps, skip
ahead to [How Every Use Case Works](#how-every-use-case-works-read-this-first).

### What You Need

- A computer with a terminal (Terminal on Mac, PowerShell or Git Bash on Windows,
  any terminal on Linux)
- An internet connection

### Step 1: Create a GitHub Account

If you already have one, skip this step.

1. Go to https://github.com in your browser.
2. Click **Sign up** and follow the prompts (email, password, username).
3. Verify your email address.

### Step 2: Create a Repository

A **repository** (or "repo") is a project folder on GitHub that tracks your files
and their history.

1. Go to https://github.com/new in your browser.
2. Fill in:
   - **Repository name**: `my-project` (or any name you like)
   - **Description**: (optional)
   - **Public** or **Private**: either is fine
   - Check **Add a README file**
3. Click **Create repository**.
4. You now have a repo at `https://github.com/YOUR_USERNAME/my-project`.

### Step 3: Install the GitHub CLI

The **GitHub CLI** (`gh`) is a tool that lets you interact with GitHub from your
terminal instead of the browser.

**macOS:**
```bash
brew install gh
```

**Windows:**
```bash
winget install GitHub.cli
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install gh
```

**Verify it installed:**
```bash
gh --version
# You should see something like: gh version 2.x.x
```

### Step 4: Log In to GitHub from Your Terminal

```bash
gh auth login
```

Follow the prompts:
1. Select **GitHub.com**
2. Select **HTTPS**
3. Select **Login with a web browser**
4. Copy the one-time code shown in your terminal
5. Press Enter to open your browser
6. Paste the code and click **Authorize**

**Verify:**
```bash
gh auth status
# Should show: Logged in to github.com as YOUR_USERNAME
```

### Step 5: Clone Your Repository

**Cloning** downloads your GitHub repository to your computer so you can work
on it locally.

```bash
gh repo clone YOUR_USERNAME/my-project
cd my-project
```

Replace `YOUR_USERNAME` with your actual GitHub username and `my-project` with
your repository name.

### Step 6: Install gh-aw

You have two options: install the published release from GitHub, or build and
install from a local copy of the source code. Pick whichever applies to you.

#### Option A: Install from GitHub (Easiest)

If you do not have the gh-aw source code on your computer, install the published
extension directly:

```bash
gh extension install github/gh-aw
```

**Verify:**
```bash
gh aw --help
# You should see a help message listing commands like compile, init, new, etc.
```

Skip ahead to [Step 7](#step-7-initialize-gh-aw-in-your-repository).

#### Option B: Build and Install from Local Source Code

If you already have a local clone of the gh-aw repository (for example, at
`C:\Users\you\Downloads\gh-aw-main` or `~/gh-aw-main`), you can build and
install it locally instead of downloading from GitHub. This is useful if you want
to use a modified or development version.

##### B1. Install the Go Programming Language

gh-aw is written in Go. You need Go installed to build it from source.

**Windows:**
```bash
winget install GoLang.Go
```

**macOS:**
```bash
brew install go
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install golang
```

**After installing, close and reopen your terminal** so the `go` command is
available in your PATH.

**Verify Go is installed:**
```bash
go version
# Should show something like: go version go1.24.x windows/amd64
```

##### B2. Build the gh-aw Binary

Open a terminal and navigate to your local gh-aw source code folder:

```bash
cd C:\Users\you\Downloads\gh-aw-main    # Windows — use your actual path
# cd ~/gh-aw-main                        # macOS/Linux — use your actual path
```

Build the binary:

```bash
go build -o gh-aw.exe ./cmd/gh-aw       # Windows
# go build -o gh-aw ./cmd/gh-aw          # macOS/Linux
```

This compiles the source code and creates a `gh-aw.exe` (or `gh-aw`) file in
the current folder.

**Verify the build succeeded:**
```bash
./gh-aw --help                            # macOS/Linux
# .\gh-aw.exe --help                      # Windows (PowerShell)
# ./gh-aw.exe --help                      # Windows (Git Bash)
# You should see a help message listing commands like compile, init, new, etc.
```

##### B3. Install as a Local gh Extension

Still inside the gh-aw source folder, run:

```bash
gh extension install .
```

The `.` (dot) means "install from the current directory." This creates a symbolic
link so that `gh aw` commands use your locally built binary.

**Verify it is installed:**
```bash
gh aw --help
# Should show the same help message — but now it is running your local build
```

> **Tip:** If you later make changes to the gh-aw source code and rebuild
> (`go build -o gh-aw.exe ./cmd/gh-aw`), the `gh aw` command will automatically
> pick up the new binary because it is linked to your local folder.

### Step 7: Initialize gh-aw in Your Repository

Navigate to the repository where you want to use gh-aw workflows. This is the
target project — **not** the gh-aw source code folder.

For example, if your project is at `C:\Users\you\Downloads\my-project`:

```bash
cd C:\Users\you\Downloads\my-project     # Windows — use your actual path
# cd ~/my-project                         # macOS/Linux — use your actual path
```

Then initialize gh-aw:

```bash
gh aw init
```

This creates the necessary configuration files inside your repo.

You are now ready to use any of the use cases below.

---

## How Every Use Case Works (Read This First)

Every use case in this document follows the same pattern. Understanding this
pattern once means you can set up any of the 25 examples.

### The Big Picture

```
You write a              gh-aw compiles it          You push it to           An event triggers
.md file with     --->   into a .lock.yml    --->   GitHub with       --->   the AI agent,
instructions             (GitHub Actions)           git push                 which runs
for an AI agent          workflow file                                       automatically
```

### What is a `.md` Workflow File?

It is a markdown file with two parts:

1. **YAML frontmatter** (between `---` markers at the top) — configuration that
   tells the system *when* to run, *which AI* to use, *what tools* it can access,
   and *what actions* it is allowed to take.
2. **Markdown body** (everything after the second `---`) — plain English
   instructions telling the AI agent *what to do*.

### What is a `.lock.yml` File?

When you run `gh aw compile`, it reads your `.md` file and generates a
`.lock.yml` file. This is a standard GitHub Actions workflow that GitHub knows
how to execute. You never edit `.lock.yml` files by hand — they are always
generated from your `.md` file.

### What is a "Trigger"?

A trigger is the event that starts your workflow. Examples:
- Someone opens a new issue (`on: issues`)
- Someone opens a pull request (`on: pull_request`)
- Someone types a slash command like `/review` in a comment (`on: slash_command`)
- A schedule runs every day at 9am (`on: schedule: daily at 9am`)
- You click a button manually in the GitHub UI (`on: workflow_dispatch`)

### What are "Safe-Outputs"?

Safe-outputs are the actions the AI agent is *allowed* to take. Without them, the
agent can only read — it cannot write anything. Examples:
- `add-labels` — add labels to issues or PRs
- `add-comments` — post comments on issues or PRs
- `create-issue` — create new issues
- `create-pull-request` — create new pull requests
- `close-issue` — close an issue

Each safe-output has a `max` limit to prevent the agent from doing too much in
one run.

### Step-by-Step Pattern (Same for Every Use Case)

These are the exact steps you will follow for every single use case below. Each
use case tells you what to paste in Step 2 — everything else is the same.

**Step 1. Create the workflow file.**

In your terminal, inside your repository folder, run:

```bash
gh aw new WORKFLOW_NAME
```

This creates a new file at `.github/workflows/WORKFLOW_NAME.md`.

**Step 2. Paste the workflow content.**

Open `.github/workflows/WORKFLOW_NAME.md` in any text editor (VS Code, Notepad,
nano, vim — anything). Delete whatever is in it and paste the workflow content
from the use case.

**Step 3. Compile.**

In your terminal, run:

```bash
gh aw compile .github/workflows/WORKFLOW_NAME.md
```

This reads your `.md` file and creates a `.lock.yml` file next to it. You should
see a success message.

**Step 4. Push to GitHub.**

```bash
git add .github/workflows/WORKFLOW_NAME.md .github/workflows/WORKFLOW_NAME.lock.yml
git commit -m "Add WORKFLOW_NAME agentic workflow"
git push
```

**What these commands do:**
- `git add` — tells Git to include these files in the next save point
- `git commit` — creates a save point (a "commit") with a description message
- `git push` — uploads your save point to GitHub so the workflow is live

**Step 5. Trigger the workflow.**

This depends on the use case:
- If the trigger is `issues: [opened]` — go to your repo on GitHub, click the
  **Issues** tab, and click **New issue**. Write anything and submit it.
- If the trigger is `pull_request: [opened]` — create a pull request.
- If the trigger is `schedule` — it will run automatically at the scheduled time.
  You can also add `workflow_dispatch:` to test it immediately (see below).
- If the trigger is `workflow_dispatch` — go to your repo on GitHub, click the
  **Actions** tab, select your workflow on the left, and click
  **Run workflow** > **Run workflow**.
- If the trigger is `slash_command` — go to an issue or PR and type the command
  (e.g., `/review`) in a comment.

**Step 6. See the results.**

- Go to your repo on GitHub.
- Click the **Actions** tab to see the workflow run (a green checkmark means
  success, a red X means failure — click it for details).
- Check the relevant place for output: the Issues tab for new labels/comments,
  the Pull Requests tab for review comments, the Discussions tab for reports.

---

## Issue Management

### 1. Auto-Triage New Issues

**What it does and why you want it:**

When someone opens an issue on your GitHub project (a bug report, feature
request, question, etc.), a human maintainer normally has to read it and
manually add labels like `bug`, `enhancement`, or `question`. This takes time,
and issues often sit unlabeled for days.

This workflow makes an AI agent do that job automatically. The moment an issue
is opened (or edited), the agent reads the issue's title and body text, decides
what kind of issue it is, and adds the appropriate labels. It also runs every
6 hours to catch any issues that were missed.

**How the agent "reads" the issue:** GitHub automatically tells the workflow
which issue triggered it (via `github.event.issue`). The agent uses the GitHub
API (through the `issues` toolset) to read the issue's title, body, and
existing labels. All of this happens on GitHub's servers — nothing runs on your
computer.

**Setup Instructions:**

1. In your terminal, inside your repository folder:
   ```bash
   gh aw new auto-triage
   ```

2. Open `.github/workflows/auto-triage.md` in your text editor and replace
   everything with the content below.

3. Compile:
   ```bash
   gh aw compile .github/workflows/auto-triage.md
   ```

4. Push to GitHub:
   ```bash
   git add .github/workflows/auto-triage.md .github/workflows/auto-triage.lock.yml
   git commit -m "Add auto-triage workflow"
   git push
   ```

5. **Test it:** Go to your repository on GitHub (https://github.com/YOUR_USERNAME/YOUR_REPO).
   Click the **Issues** tab. Click **New issue**. Type a title like
   "The login button crashes when I click it" and a body like
   "Steps to reproduce: click login. Expected: login screen. Actual: app crashes."
   Click **Submit new issue**.

6. **See the result:** Within a minute or two, go back to the issue you just
   created. You should see labels like `bug` automatically added by the agent.
   You can also click the **Actions** tab to see the workflow run.

**Workflow File Contents** (paste this into `.github/workflows/auto-triage.md`):

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

You are an issue triage agent for the repository ${{ github.repository }}.

## How You Are Triggered

You run in two situations:

1. **When an issue is opened or edited.** The issue that triggered you is
   available at `github.event.issue`. Use the GitHub tools to read its
   title (`github.event.issue.title`), body (`github.event.issue.body`),
   and current labels (`github.event.issue.labels`).

2. **On a schedule (every 6 hours).** When running on schedule, there is no
   single triggering issue. Instead, use the GitHub `issues` toolset to search
   for all open issues that have zero labels. Process up to 10 of them.

## What To Do

For each issue you analyze:

1. Read the issue title and body.
2. Decide which labels to apply based on the classification rules below.
3. Use the `add_labels` tool to apply them. For the triggering issue, you can
   target it directly. For scheduled runs, specify the issue number.

## Classification Rules

Apply one **type** label:
- `bug` — the issue describes something broken. Look for words like: "error",
  "fail", "crash", "broken", "doesn't work", "not working", stack traces, or
  error messages.
- `enhancement` — the issue asks for something new. Look for: "feature",
  "add", "support", "would be nice", "suggestion", "allow", "enable".
- `documentation` — the issue is about docs. Look for: "docs", "readme",
  "guide", "tutorial", "clarify", "explain".
- `question` — the issue is a question. Look for: titles starting with "How",
  "Why", "What", question marks, "how do I".

Apply one **priority** label:
- `priority: critical` — words like "blocking", "urgent", data loss, security
- `priority: high` — affects many users, security adjacent
- `priority: medium` — default for most bugs
- `priority: low` — cosmetic issues, minor improvements

Apply **component** labels when the topic is clear:
- `area: frontend`, `area: backend`, `area: api`, `area: ci`

When you are not confident about the classification, apply `needs-triage`
instead and do not guess.

## Limits

You can apply a maximum of 10 label operations per run (configured in
safe-outputs). If you have more than 10 issues to process during a scheduled
run, handle the first 10 and leave the rest for the next run.
```

**How to customize it:**

- **Change the labels:** Edit the classification rules in the markdown body.
  The label names (like `bug`, `enhancement`) must match labels that exist in
  your GitHub repo. To create labels, go to your repo > Issues > Labels > New label.
- **Change the schedule:** Replace `every 6h` with `daily`, `hourly`,
  `every 30 minutes`, `weekly on monday`, etc.
- **Change the limit:** Edit `max: 10` under `add-labels` to allow more or
  fewer label operations per run.

---

### 2. Stale Issue Closer

**What it does and why you want it:**

Over time, GitHub repositories accumulate issues that nobody is working on.
These "stale" issues clutter the tracker and make it hard to find what matters.

This workflow runs once a day. It finds issues that have had no activity
(no comments, no label changes, no assignments) for 30 days and posts a warning
comment. If there is still no activity after 45 days, it closes the issue.
Issues with special labels like `keep-open` or `in-progress` are never touched.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new stale-issues
   ```

2. Open `.github/workflows/stale-issues.md` and replace everything with the
   content below.

3. Compile:
   ```bash
   gh aw compile .github/workflows/stale-issues.md
   ```

4. Push to GitHub:
   ```bash
   git add .github/workflows/stale-issues.md .github/workflows/stale-issues.lock.yml
   git commit -m "Add stale issue closer workflow"
   git push
   ```

5. **Test it:** This workflow runs daily. To test immediately, you can also add
   `workflow_dispatch:` to the `on:` section (see the note below the workflow
   content), then go to Actions > Stale Issue Closer > Run workflow.

6. **See the result:** Check the Issues tab — stale issues will have a warning
   comment and a `stale` label. After 45 days, they will be closed.

**Workflow File Contents** (paste into `.github/workflows/stale-issues.md`):

```markdown
---
name: Stale Issue Closer
description: Warns and closes issues that have been inactive for extended periods
on:
  schedule: daily at 8am
  workflow_dispatch:
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

You manage stale issues in the repository ${{ github.repository }}.

## How You Are Triggered

You run on a daily schedule (8am UTC) and can also be triggered manually from
the Actions tab using the "Run workflow" button.

## What To Do

1. Use the GitHub `issues` toolset to search for all open issues, sorted by
   the `updated_at` field (oldest first).
2. For each issue, check the `updated_at` timestamp (this is automatically
   updated by GitHub whenever there is any activity: comments, label changes,
   assignments, or edits).
3. Calculate how many days have passed since `updated_at`.

### Skip These Issues (Exempt)

Do NOT touch issues that have any of these labels:
- `keep-open`
- `blocked`
- `in-progress`
- `priority: critical`

### Warning Phase (30-44 Days Inactive)

If an issue has been inactive for 30-44 days and does NOT have the `stale`
label:
- Add the `stale` label.
- Post this comment:
  "This issue has been inactive for 30 days. It will be closed in 15 days
  unless there is new activity. Add the `keep-open` label to prevent this."

### Closing Phase (45+ Days Inactive)

If an issue has been inactive for 45 or more days:
- Close the issue.
- Post this comment:
  "Closing due to 45 days of inactivity. If this is still relevant, you can
  reopen it at any time."

## Limits

- Maximum 20 label additions per run.
- Maximum 20 comments per run.
- Maximum 10 issue closures per run.
If limits are hit, prioritize the oldest issues first.
```

> **Tip:** The `workflow_dispatch:` line in the `on:` section lets you test
> this workflow immediately by going to Actions > Stale Issue Closer >
> Run workflow, instead of waiting for the daily schedule.

---

### 3. Bug Report Validator

**What it does and why you want it:**

Incomplete bug reports waste everyone's time. Developers ask follow-up
questions, wait days for answers, and sometimes give up. This workflow
checks every new issue that looks like a bug report and immediately asks for
missing information (steps to reproduce, expected behavior, etc.) via a
comment with a checklist.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new bug-validator
   ```

2. Open `.github/workflows/bug-validator.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/bug-validator.md
   git add .github/workflows/bug-validator.md .github/workflows/bug-validator.lock.yml
   git commit -m "Add bug report validator workflow"
   git push
   ```

4. **Test it:** Open a new issue with a vague title like "It's broken" and a
   body that says only "nothing works". The agent should add a `needs-info`
   label and post a comment asking for steps to reproduce, expected behavior,
   etc.

5. **See the result:** Check the issue you just created — a comment should
   appear within 1-2 minutes.

**Workflow File Contents** (paste into `.github/workflows/bug-validator.md`):

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

You check whether newly opened issues in ${{ github.repository }} are complete
bug reports.

## How You Are Triggered

You run every time a new issue is opened. The issue is available at
`github.event.issue`. Use the GitHub tools to read its title
(`github.event.issue.title`) and body (`github.event.issue.body`).

## What To Do

### Step 1: Decide If This Is a Bug Report

Read the title and body. If the issue is clearly a feature request, question,
or documentation request — do nothing and stop. Only validate issues that
appear to be reporting a problem or bug.

### Step 2: Check for Required Sections

A complete bug report should contain:
- **Steps to Reproduce** — numbered steps or a clear description of how to
  trigger the problem.
- **Expected Behavior** — what the user expected to happen.
- **Actual Behavior** — what actually happened instead.
- **Environment** — any mention of OS, browser, version, or runtime.

### Step 3: Take Action

- If ALL four sections are present: add the `validated` label.
- If ANY sections are missing: add the `needs-info` label and post a single
  comment listing only the missing items as a checklist:

  ```
  Thanks for the report! To help us investigate, could you add:
  - [ ] Steps to reproduce
  - [ ] Expected behavior
  - [ ] Actual behavior
  - [ ] Environment info (OS, version, browser, etc.)
  ```

  Only include the items that are actually missing — do not list items that
  the user already provided.
```

---

### 4. Duplicate Issue Detector

**What it does and why you want it:**

Users often report the same problem without realizing someone else already did.
Maintainers waste time investigating known issues. This workflow automatically
searches for similar existing issues when a new one is opened and posts links
if it finds likely duplicates.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new duplicate-detector
   ```

2. Open `.github/workflows/duplicate-detector.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/duplicate-detector.md
   git add .github/workflows/duplicate-detector.md .github/workflows/duplicate-detector.lock.yml
   git commit -m "Add duplicate issue detector workflow"
   git push
   ```

4. **Test it:** Open a new issue with a title/body similar to an existing issue
   in your repo. The agent should add a `possible-duplicate` label and post a
   comment with links to the similar issues.

5. **See the result:** Check the new issue for a comment from the bot.

**Workflow File Contents** (paste into `.github/workflows/duplicate-detector.md`):

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

You check newly opened issues in ${{ github.repository }} for potential
duplicates.

## How You Are Triggered

You run every time a new issue is opened. The new issue is available at
`github.event.issue`.

## What To Do

### Step 1: Read the New Issue

Use the GitHub tools to read the title and body of
`github.event.issue`.

### Step 2: Extract Key Terms

Identify 3-5 meaningful keywords from the title and body. Ignore generic words
like "the", "is", "not working", "please", "help". Focus on specific nouns,
error messages, function names, or feature names.

### Step 3: Search for Similar Issues

Use the GitHub issues search to look for open AND recently closed issues
(from the last 90 days) in this repository that match your keywords.

### Step 4: Evaluate Candidates

For each candidate issue found, compare:
- Title similarity (do the titles describe the same problem?)
- Problem description overlap (is the root cause the same?)
- Error messages or stack traces in common

Only flag issues that are genuinely similar — not just issues that share a
single common word.

### Step 5: Report (Only If Duplicates Found)

If you find likely duplicates with high confidence:
- Add the `possible-duplicate` label to the new issue.
- Post one comment listing the duplicates:
  ```
  This may be related to an existing issue:
  - #123 — [issue title]
  - #456 — [issue title]
  Please check if any of these address your problem.
  ```

If you do NOT find duplicates, do nothing. Do not post a "no duplicates found"
comment — that just adds noise.
```

---

## Pull Request Automation

### 5. PR Review Bot

**What it does and why you want it:**

Code review is essential but time-consuming. This workflow gives you an
on-demand AI code reviewer. When anyone comments `/review` on a pull request,
the agent reads the code diff (the changes made in the PR) and posts targeted
review comments on specific lines pointing out bugs, security issues,
performance problems, and style concerns.

**What is a Pull Request?** A pull request (PR) is a proposal to merge code
changes into the main codebase. Other developers review the changes before
they are accepted.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new pr-review
   ```

2. Open `.github/workflows/pr-review.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/pr-review.md
   git add .github/workflows/pr-review.md .github/workflows/pr-review.lock.yml
   git commit -m "Add PR review bot workflow"
   git push
   ```

4. **Test it:** Open a pull request on your repo (or use an existing one). Go
   to the PR page and scroll to the bottom where the comment box is. Type
   `/review` and click **Comment**. Wait 1-2 minutes.

5. **See the result:** The agent will post inline review comments on specific
   lines of your code changes, plus a summary comment.

**Workflow File Contents** (paste into `.github/workflows/pr-review.md`):

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

You are a thorough code reviewer for ${{ github.repository }}.

## How You Are Triggered

Someone commented `/review` on pull request
#${{ github.event.issue.number }}. The full text of the triggering comment
is: "${{ needs.activation.outputs.text }}"

## What To Do

### Step 1: Get the Pull Request Details

Use the GitHub `pull_requests` toolset to:
- Get the PR details for PR #${{ github.event.issue.number }}.
- Get the list of files changed in the PR.
- Read the diff for each changed file (this shows which lines were added,
  removed, or modified).

### Step 2: Review the Code Changes

For each changed file, read the diff carefully. Look for issues in this
priority order:

1. **Security** — injection vulnerabilities, hardcoded secrets, unsafe
   deserialization, missing input validation.
2. **Bugs** — null pointer dereference, off-by-one errors, race conditions,
   resource leaks, logic errors.
3. **Performance** — N+1 queries, unnecessary allocations, missing indexes,
   redundant computation.
4. **Error handling** — swallowed errors, missing validation, unhandled edge
   cases.
5. **Readability** — unclear variable names, overly complex logic, missing
   context.

Only review the changed lines (the diff). Do not review the entire file.

### Step 3: Post Review Comments

For each issue found, use the `create-pull-request-review-comment` tool to
post an inline comment on the specific file and line. Each comment should:
- State the problem clearly.
- Explain why it matters.
- Suggest a fix.
- Start with a severity prefix: `🔴 Critical:`, `🟡 Warning:`, or
  `💡 Suggestion:`.

### Step 4: Post a Summary

After reviewing all files, post one summary comment with:
- Total issues found by severity (critical / warning / suggestion).
- Overall quality impression (1-2 sentences).
- What was done well (acknowledge good patterns — reviewers should not only
  point out problems).

If the code is clean and you find no issues, say so briefly.

## Limits

- Maximum 10 inline review comments per run.
- Maximum 1 summary comment.
- Focus on the most important issues. Quality over quantity.
```

---

### 6. PR Welcome Message

**What it does and why you want it:**

First-time contributors to open source projects are often unsure if they did
everything right. This workflow detects when someone opens their very first
pull request to your repo and automatically posts a friendly welcome message
with a contribution checklist.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new pr-welcome
   ```

2. Open `.github/workflows/pr-welcome.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/pr-welcome.md
   git add .github/workflows/pr-welcome.md .github/workflows/pr-welcome.lock.yml
   git commit -m "Add PR welcome message workflow"
   git push
   ```

4. **Test it:** Have someone who has never submitted a PR to your repo open
   one, or use a second GitHub account.

5. **See the result:** A welcome comment and `first-contribution` label should
   appear on the PR.

**Workflow File Contents** (paste into `.github/workflows/pr-welcome.md`):

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

You greet first-time contributors to ${{ github.repository }}.

## How You Are Triggered

You run every time a pull request is opened. The PR details are available at
`github.event.pull_request`.

## What To Do

### Step 1: Identify the Author

The PR author's username is `github.event.pull_request.user.login`.

### Step 2: Check If This Is Their First PR

Use the GitHub tools to search for previous pull requests by the same author
in this repository (both merged and closed). For example, search:
`is:pr author:USERNAME repo:OWNER/REPO`

### Step 3: Take Action

If this is their **first PR** (zero previous PRs found):
- Add the `first-contribution` label to the PR.
- Post this comment (replacing USERNAME with their actual GitHub username):
  ```
  Welcome to the project, @USERNAME! Thanks for your first contribution.

  Here's a quick checklist before we review:
  - [ ] Tests pass locally
  - [ ] Code follows the project style guide
  - [ ] Changes are documented (if applicable)

  A maintainer will review your PR soon. Feel free to ask questions!
  ```

If they have previous PRs, do nothing. Do not post a message.
```

---

### 7. Stale Draft PR Cleanup

**What it does and why you want it:**

Draft pull requests (PRs marked as "not ready for review") pile up over time.
This workflow runs daily, warns authors of drafts inactive for 10+ days, and
closes them at 14 days. This keeps your PR list focused on active work.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new draft-cleanup
   ```

2. Open `.github/workflows/draft-cleanup.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/draft-cleanup.md
   git add .github/workflows/draft-cleanup.md .github/workflows/draft-cleanup.lock.yml
   git commit -m "Add draft PR cleanup workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **Draft PR Cleanup** > **Run workflow** to
   trigger it manually.

5. **See the result:** Check the Pull Requests tab for warning comments and
   `stale-draft` labels on inactive draft PRs.

**Workflow File Contents** (paste into `.github/workflows/draft-cleanup.md`):

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

You clean up stale draft pull requests in ${{ github.repository }}.

## How You Are Triggered

You run daily on a schedule, and can also be triggered manually from the
Actions tab.

## What To Do

### Step 1: Find All Open Draft PRs

Use the GitHub tools to search for pull requests matching:
`is:pr is:open is:draft` in this repository.

For each draft PR, get:
- PR number and title
- Author
- The `updated_at` timestamp (when the PR last had any activity)
- Current labels

### Step 2: Check Exemptions

Skip any PR that has one of these labels: `keep-draft`, `blocked`,
`awaiting-review`. These PRs are intentionally being kept open.

### Step 3: Calculate Inactivity

For each non-exempt draft PR, calculate how many days have passed since
`updated_at` (GitHub updates this automatically on any activity: commits,
comments, label changes, review requests).

### Step 4: Apply the Policy

| Days Since Last Activity | Action |
|--------------------------|--------|
| 0-9 days | Do nothing |
| 10-13 days (no `stale-draft` label) | Add `stale-draft` label + warning comment |
| 10-13 days (has `stale-draft` label) | Do nothing (already warned) |
| 14+ days | Close the PR with an explanation comment |

**Warning comment:**
"This draft PR has been inactive for [X] days. It will be automatically
closed in [Y] days unless there is new activity. To prevent this: push a
commit, add a comment, or add the `keep-draft` label."

**Closing comment:**
"Closing this draft PR due to 14+ days of inactivity. This is not a
rejection — feel free to reopen it when work continues."

## Limits

- Maximum 20 label operations per run
- Maximum 20 comments per run
- Maximum 10 PR closures per run
If limits are hit, process the oldest (most stale) PRs first.
```

---

### 8. Breaking Change Detector

**What it does and why you want it:**

When someone submits a PR that removes a public function, changes a method
signature, or deletes an exported type, downstream users of your code will
break. This workflow scans every PR targeting `main` and posts a warning if
breaking changes are detected.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new breaking-changes
   ```

2. Open `.github/workflows/breaking-changes.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/breaking-changes.md
   git add .github/workflows/breaking-changes.md .github/workflows/breaking-changes.lock.yml
   git commit -m "Add breaking change detector workflow"
   git push
   ```

4. **Test it:** Open a PR that removes a public function or renames an export.

5. **See the result:** A `breaking-change` label and a comment listing the
   breaking changes should appear on the PR.

**Workflow File Contents** (paste into `.github/workflows/breaking-changes.md`):

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

You analyze PRs targeting `main` in ${{ github.repository }} for potential
breaking changes.

## How You Are Triggered

You run when a pull request is opened or updated (new commits pushed) that
targets the `main` branch. The PR is available at
`github.event.pull_request`.

## What To Do

### Step 1: Get Changed Files

Use the GitHub tools to list all files changed in PR
#${{ github.event.pull_request.number }} and read the diff for each file.

### Step 2: Look for Breaking Changes

Scan each file's diff for these patterns:

- **Removed exports** — public functions, classes, types, or constants deleted
- **Changed signatures** — parameters added, removed, or reordered in public
  functions
- **Renamed identifiers** — public symbols renamed without backward-compatible
  aliases
- **Changed return types** — functions that now return a different type
- **Deleted files** — files that other packages may import
- **Config format changes** — altered config file schemas or required fields

### Step 3: Ignore These Files

Do NOT flag changes in:
- Test files (`*_test.go`, `*.test.ts`, `*.spec.js`)
- Internal/private modules (paths containing `/internal/`)
- Generated files (`.lock.yml`, `package-lock.json`, `go.sum`)

### Step 4: Report (Only If Breaking Changes Found)

If breaking changes are found:
- Add the `breaking-change` label.
- Post a single comment listing each breaking change:
  ```
  **Potential breaking changes detected:**

  - `path/to/file.ts:42` — Removed export `functionName`
  - `path/to/api.go:15` — Changed signature of `HandleRequest` (added
    required parameter)

  Please verify these are intentional and update the changelog.
  ```

If no breaking changes are found, do nothing.
```

---

## CI/CD & DevOps

### 9. CI Failure Doctor

**What it does and why you want it:**

When your CI (Continuous Integration — automated tests that run on every push)
fails, someone has to dig through log output to figure out what went wrong.
This workflow does that automatically: when your CI workflow fails on `main`,
the agent pulls the logs, identifies the root cause, and creates an issue with
the diagnosis and suggested fix.

**Prerequisites:** This workflow monitors another workflow called "CI". If your
CI workflow has a different name, change `workflows: ["CI"]` to match your
actual workflow name (visible in the Actions tab).

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new ci-doctor
   ```

2. Open `.github/workflows/ci-doctor.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/ci-doctor.md
   git add .github/workflows/ci-doctor.md .github/workflows/ci-doctor.lock.yml
   git commit -m "Add CI failure doctor workflow"
   git push
   ```

4. **Test it:** Wait for a CI failure on main (or intentionally break a test
   and push to main).

5. **See the result:** After the CI workflow fails, check the Issues tab — a
   new issue titled "[CI Doctor] ..." should appear with a diagnosis.

**Workflow File Contents** (paste into `.github/workflows/ci-doctor.md`):

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

You investigate failed CI runs in ${{ github.repository }} and create
diagnostic issues.

## How You Are Triggered

You run when the "CI" workflow completes on the `main` branch. The workflow
run details are available at `github.event.workflow_run`.

## What To Do

### Step 1: Verify This Is a Failure

Check `github.event.workflow_run.conclusion`. If it is NOT `failure`, call
the `noop` tool and stop. Do not investigate successful runs.

### Step 2: Get Workflow Details

Use the GitHub Actions tools:
1. `get_workflow_run` — get full details of run
   `${{ github.event.workflow_run.id }}`.
2. `list_workflow_jobs` — find which specific jobs failed.
3. `get_job_logs` with `failed_only=true` — get the log output from failed
   jobs.

### Step 3: Analyze the Failure

Read the logs and identify the root cause. Common categories:
- **Test failures** — which test, what assertion, what was expected vs actual
- **Build errors** — compilation errors, missing dependencies
- **Infrastructure** — runner issues, network timeouts
- **Flaky tests** — intermittent failures, timing-dependent

### Step 4: Check for Existing Issues

Search the Issues tab for open issues with "[CI Doctor]" in the title. If a
very similar diagnosis already exists as an open issue, add a comment to it
instead of creating a new one.

### Step 5: Create an Issue

Create a new issue with this structure:

```
## Summary
[One sentence describing the failure]

## Failed Jobs
- **[Job name]**: [error summary]

## Root Cause Analysis
[Detailed explanation of what went wrong and why]

## Suggested Fix
- [ ] [Specific, actionable step]
- [ ] [Another step if needed]

## Relevant Logs
<details>
<summary>Click to expand log output</summary>

[Include only the key error lines, not the entire log]

</details>

## Context
- **Run:** ${{ github.event.workflow_run.html_url }}
- **Commit:** ${{ github.event.workflow_run.head_sha }}
```
```

---

### 10. Dependency Update Bundler

**What it does and why you want it:**

If you use Dependabot (GitHub's automatic dependency updater), you may get
flooded with individual PRs for every package update. This workflow runs
weekly, finds all open Dependabot PRs, and creates a summary issue grouping
them by ecosystem (npm, Go, Python, etc.) so you can see the full picture at
a glance.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new dep-bundler
   ```

2. Open `.github/workflows/dep-bundler.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/dep-bundler.md
   git add .github/workflows/dep-bundler.md .github/workflows/dep-bundler.lock.yml
   git commit -m "Add dependency update bundler workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **Dependency Update Bundler** > **Run workflow**.

5. **See the result:** Check the Issues tab for a new issue summarizing
   Dependabot PRs.

**Workflow File Contents** (paste into `.github/workflows/dep-bundler.md`):

```markdown
---
name: Dependency Update Bundler
description: Groups open Dependabot PRs into a summary issue per runtime
on:
  schedule: weekly on monday
  workflow_dispatch:
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

You create summary issues for open Dependabot PRs in ${{ github.repository }}.

## How You Are Triggered

You run weekly on Monday, or manually from the Actions tab.

## What To Do

### Step 1: Find Dependabot PRs

Use the GitHub tools to search for open PRs authored by `dependabot[bot]`
or `app/dependabot` in this repository.

### Step 2: Group by Ecosystem

Categorize each PR by the files it modifies:
- **npm** — changes to `package.json` or `package-lock.json`
- **Go** — changes to `go.mod` or `go.sum`
- **pip** — changes to `requirements.txt` or `pyproject.toml`
- **GitHub Actions** — changes to `.github/workflows/*.yml`
- **Other** — anything that does not fit the above categories

### Step 3: Create Summary Issues

For each ecosystem that has open PRs, create one issue:

```
## Dependency Updates — [Ecosystem Name]

| Package | From | To | PR | Type |
|---------|------|----|----|------|
| lodash | 4.17.20 | 4.17.21 | #123 | patch |
| express | 4.18.0 | 5.0.0 | #124 | major |

### Recommended Action
- **Patch** updates: generally safe to merge
- **Minor** updates: review the changelog before merging
- **Major** updates: test thoroughly — may contain breaking changes
```

If there are no open Dependabot PRs, do nothing.
```

---

### 11. Release Notes Generator

**What it does and why you want it:**

Writing release notes by hand is tedious. This workflow runs automatically when
you publish a new release on GitHub. It reads all commits since the last release,
categorizes them (features, bug fixes, docs, etc.), and updates the release
description with structured notes.

**What is a "release"?** A release is a tagged snapshot of your code (like v1.0.0)
that you publish on GitHub. Users can download releases. You create them from the
**Releases** section of your repo on GitHub.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new release-notes
   ```

2. Open `.github/workflows/release-notes.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/release-notes.md
   git add .github/workflows/release-notes.md .github/workflows/release-notes.lock.yml
   git commit -m "Add release notes generator workflow"
   git push
   ```

4. **Test it:** Go to your repo on GitHub > Releases > Draft a new release.
   Create a new tag (e.g., `v0.1.0`), add a title, and click **Publish release**.

5. **See the result:** The release description should update with categorized notes.

**Workflow File Contents** (paste into `.github/workflows/release-notes.md`):

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

You generate release notes for ${{ github.repository }}.

## How You Are Triggered

You run when a new release is published. The release details are available at
`github.event.release`.

## What To Do

### Step 1: Get Release Info

- Current tag: `github.event.release.tag_name`
- Use the GitHub tools to find the previous release tag (the most recent
  release before this one).

### Step 2: List Commits Between Tags

Use the GitHub repos tools to get the list of commits between the previous
release tag and the current one.

### Step 3: Categorize Changes

For each commit, look at the commit message and any associated PR:
- **Features** — messages starting with `feat:`, or PRs with `enhancement` label
- **Bug Fixes** — messages starting with `fix:`, or PRs with `bug` label
- **Performance** — messages starting with `perf:`
- **Documentation** — messages starting with `docs:`
- **Breaking Changes** — messages containing `BREAKING CHANGE:`
- **Other** — everything else

### Step 4: Update the Release

Use the `update-release` tool to set the release body:

```
## What's Changed

### Breaking Changes
- [description] (#PR) by @author

### Features
- [description] (#PR) by @author

### Bug Fixes
- [description] (#PR) by @author

### Other Changes
- [description] (#PR) by @author

**Full Changelog**: [link to compare between tags]
```

Omit any empty sections (e.g., don't include "Breaking Changes" if there are none).
```

---

## Security

### 12. Code Scanning Auto-Fixer

**What it does and why you want it:**

GitHub Code Scanning detects security vulnerabilities in your code. But
finding vulnerabilities is only half the battle — fixing them takes developer
time. This workflow picks the highest-severity unfixed alert, writes a fix,
and opens a pull request for you to review.

**Prerequisites:** Code Scanning must be enabled on your repository (Settings >
Code security and analysis > Code scanning).

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new security-fixer
   ```

2. Open `.github/workflows/security-fixer.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/security-fixer.md
   git add .github/workflows/security-fixer.md .github/workflows/security-fixer.lock.yml
   git commit -m "Add security alert fixer workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **Security Alert Fixer** > **Run workflow**.

5. **See the result:** Check the Pull Requests tab for a new PR with the fix.

**Workflow File Contents** (paste into `.github/workflows/security-fixer.md`):

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

You fix code scanning alerts in ${{ github.repository }}.

## How You Are Triggered

You are triggered manually from the Actions tab. The `skip-if-match` rule
prevents this from running if there is already an open PR with "[security-fix]"
in the title (to avoid duplicate fixes).

## What To Do

### Step 1: List Open Alerts

Use the `code_security` toolset to list all open code scanning alerts.
Sort by severity: critical > high > medium > low.

### Step 2: Select the Top Alert

Pick the highest-severity alert. If no open alerts exist, stop and report
"No open security alerts found."

### Step 3: Understand the Vulnerability

- Read the alert details: rule ID, CWE, file path, line number.
- Read the affected file using the `repos` toolset.
- Understand the vulnerability type and the code context (at least 20 lines
  around the flagged line).

### Step 4: Write a Fix

- Use the `edit` tool to modify the affected file.
- Make a minimal, surgical fix that addresses the root cause.
- Follow security best practices for the specific vulnerability type.
- Do not change unrelated code.

### Step 5: Run Tests

Use the `bash` tool to run the project test suite to verify your fix does
not break anything. If tests fail, undo your change and stop.

### Step 6: Create a Pull Request

Create a PR with this structure:
```
## Security Fix: [Rule ID]

**Alert:** #[number] | **Severity:** [level] | **CWE:** [id]

### Vulnerability
[What the issue is and why it matters]

### Fix Applied
[What was changed and why]

### Testing
- [ ] Existing tests pass
- [ ] Fix addresses the root cause
```

## Rules

- Fix one alert per run. Quality over speed.
- Never introduce new vulnerabilities while fixing one.
- If the fix requires large architectural changes, explain that in the PR body
  instead of making risky automated changes.
```

---

### 13. Secret Leak Scanner

**What it does and why you want it:**

Accidentally committing API keys, passwords, or tokens to a PR is a serious
security risk. This workflow scans every PR diff for patterns that look like
secrets and warns the author before the PR is merged.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new secret-scanner
   ```

2. Open `.github/workflows/secret-scanner.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/secret-scanner.md
   git add .github/workflows/secret-scanner.md .github/workflows/secret-scanner.lock.yml
   git commit -m "Add secret leak scanner workflow"
   git push
   ```

4. **Test it:** Open a PR that adds a line like `API_KEY = "sk-abc123def456..."`.

5. **See the result:** A `security-review` label and warning comment should
   appear on the PR.

**Workflow File Contents** (paste into `.github/workflows/secret-scanner.md`):

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

You scan PR diffs in ${{ github.repository }} for accidentally committed secrets.

## How You Are Triggered

You run when a pull request is opened or updated (new commits pushed). The PR
details are at `github.event.pull_request`.

## What To Do

### Step 1: Get Changed Files

Use the GitHub tools to list files changed in PR
#${{ github.event.pull_request.number }} and read the diff for each file.

### Step 2: Scan Added Lines

For each file, look at only the **added lines** (lines prefixed with `+` in
the diff). Check for patterns like:

- **AWS keys**: strings matching `AKIA` followed by 16 uppercase letters/digits
- **API tokens**: strings starting with `sk-`, `ghp_`, `gho_`, `xoxb-`, `xoxp-`
- **Generic secrets**: variable names like `password`, `secret`, `api_key`,
  `token`, `auth` assigned a literal string value (not a variable reference
  or environment variable)
- **Private keys**: `-----BEGIN PRIVATE KEY-----` or similar
- **Connection strings**: `mysql://`, `postgres://`, `mongodb://` containing
  username:password

### Step 3: Filter Out False Positives

Ignore:
- Files in test directories (`test/`, `__tests__/`, `fixtures/`, `mocks/`)
- Placeholder values (`xxx`, `your-key-here`, `REPLACE_ME`, `changeme`)
- Documentation that explains how to set up secrets
- `.gitignore` files
- Lines that reference environment variables (`process.env.`, `os.Getenv(`)

### Step 4: Report (Only If Secrets Found)

If any potential secrets are found:
- Add the `security-review` label.
- Post one comment:
  ```
  **Potential secret detected in this PR.**

  | File | Line | Pattern |
  |------|------|---------|
  | path/to/file.js | 42 | AWS Access Key (AKIA...) |
  | path/to/config.py | 17 | Hardcoded password |

  Please remove these before merging. Use environment variables or a
  secrets manager instead.
  ```

If no secrets are found, do nothing.
```

---

## Documentation

### 14. README Freshness Checker

**What it does and why you want it:**

READMEs go stale: install commands reference old versions, code examples use
renamed functions, links point to deleted files. This workflow audits your
README weekly against the actual codebase and creates an issue listing anything
that is out of date.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new readme-checker
   ```

2. Open `.github/workflows/readme-checker.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/readme-checker.md
   git add .github/workflows/readme-checker.md .github/workflows/readme-checker.lock.yml
   git commit -m "Add README freshness checker workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **README Freshness Checker** > **Run workflow**.

5. **See the result:** If the README has issues, check the Issues tab for a
   new issue titled "[Docs] ...".

**Workflow File Contents** (paste into `.github/workflows/readme-checker.md`):

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

You audit the README.md in ${{ github.repository }} for accuracy.

## How You Are Triggered

You run weekly on Wednesday, or manually from the Actions tab.

## What To Do

### Step 1: Read the README

Use the GitHub `repos` toolset to read the `README.md` file from the
repository's default branch.

### Step 2: Audit Each Section

Go through the README and check:

1. **Installation instructions** — Are the listed commands, package names, and
   version numbers still correct? Check if referenced packages exist.
2. **Code examples** — Do function names, import paths, and API calls in
   examples match the actual source code? Read the referenced source files
   to verify.
3. **File path references** — Does every file path mentioned in the README
   (like `src/`, `docs/guide.md`, `config.yaml`) actually exist in the repo?
   Use the repos toolset to check.
4. **Badge URLs** — Do CI/status badge URLs point to existing workflows?
5. **Completeness** — Are there important features, commands, or config options
   in the code that the README does not mention?

### Step 3: Report (Only If Issues Found)

If you find any outdated, incorrect, or missing content, create one issue:

```
## README Audit Results

### Outdated Content
- [ ] Section "Installation": references `v2.0` but latest is `v3.1`
- [ ] Code example in "Quick Start" uses `oldFunction()` which was renamed

### Missing Documentation
- [ ] New `--verbose` flag is not documented
- [ ] Config file format changed but docs still show old format

### Dead References
- [ ] Link to `docs/API.md` — file does not exist
```

If the README is fully up to date, do nothing.
```

---

### 15. API Docs Generator

**What it does and why you want it:**

Keeping API documentation synchronized with code changes is error-prone. This
workflow triggers when a PR modifies API-related files and automatically posts
a comment documenting new, modified, or removed endpoints.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new api-docs
   ```

2. Open `.github/workflows/api-docs.md` and paste the content below.

3. **Customize the `paths:` filter** to match where your API code lives. The
   example below uses `src/api/**`, `routes/**`, `handlers/**`, and
   `controllers/**` — change these to your actual project paths.

4. Compile and push:
   ```bash
   gh aw compile .github/workflows/api-docs.md
   git add .github/workflows/api-docs.md .github/workflows/api-docs.lock.yml
   git commit -m "Add API docs generator workflow"
   git push
   ```

5. **Test it:** Open a PR that modifies a file in one of the paths listed
   in the `paths:` filter.

6. **See the result:** A comment on the PR documenting the API changes.

**Workflow File Contents** (paste into `.github/workflows/api-docs.md`):

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

You document API changes in PRs for ${{ github.repository }}.

## How You Are Triggered

You run when a PR is opened or updated that modifies files matching the path
filters (src/api/, routes/, handlers/, controllers/). The PR details are at
`github.event.pull_request`.

## What To Do

### Step 1: Get Changed API Files

Use the GitHub tools to list files changed in PR
#${{ github.event.pull_request.number }}. Filter to files that define API
endpoints (routes, handlers, controllers).

### Step 2: Read the Changed Files

For each API-related file, read the new version (after the changes).

### Step 3: Extract Endpoint Information

For each endpoint, extract:
- HTTP method (`GET`, `POST`, `PUT`, `DELETE`, etc.)
- URL path (`/api/users/:id`)
- Request parameters (path params, query params, request body fields)
- Response format and status codes
- Authentication requirements (if visible from code)

### Step 4: Post Documentation Comment

Post one comment on the PR:

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
- `DELETE /api/legacy/users` — removed in favor of `DELETE /api/users/:id`
```

Omit empty sections. If no meaningful API changes are found, do nothing.
```

---

## Slash Commands

> **What is a slash command?** A slash command is a comment starting with `/`
> (like `/review` or `/deploy staging`) that triggers a workflow. You type it
> in the comment box on an issue or pull request and click Comment.

### 16. /deploy Command

**What it does and why you want it:**

Lets team members trigger deployments directly from a PR comment. Type
`/deploy staging` or `/deploy production` and the agent validates the
environment, checks permissions, and initiates the deployment workflow.

**Prerequisites:** You need a separate deployment workflow (a standard GitHub
Actions `.yml` file) that actually performs the deployment. This workflow just
triggers it.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new deploy-command
   ```

2. Open `.github/workflows/deploy-command.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/deploy-command.md
   git add .github/workflows/deploy-command.md .github/workflows/deploy-command.lock.yml
   git commit -m "Add deploy command workflow"
   git push
   ```

4. **Test it:** Go to an open PR, type `/deploy staging` in a comment, and
   click Comment.

5. **See the result:** A reply comment confirming or denying the deployment.

**Workflow File Contents** (paste into `.github/workflows/deploy-command.md`):

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

You handle `/deploy` commands in ${{ github.repository }}.

## How You Are Triggered

Someone commented on a pull request. Their comment text is:
"${{ needs.activation.outputs.text }}"

The `roles` configuration restricts this to admins and maintainers only.

## What To Do

### Step 1: Parse the Command

Extract the target environment from the comment text. The expected format is:
`/deploy [environment]`

Valid environments: `staging`, `production`

### Step 2: Validate

Check:
- Is the environment valid (`staging` or `production`)?
- Is the PR open and not already merged?

### Step 3: Take Action

If valid:
- Use the `dispatch-workflow` safe output to trigger the deployment workflow
  with the target environment as input.
- Post a comment: "Deployment to **[environment]** initiated. Track progress
  in the Actions tab."

If invalid:
- Post a comment explaining what went wrong.
  "Cannot deploy: [reason]. Usage: `/deploy staging` or `/deploy production`"
```

---

### 17. /explain Command

**What it does and why you want it:**

When someone is confused by an issue or PR, they can type `/explain` and the
agent posts a beginner-friendly explanation of what is happening.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new explain-command
   ```

2. Open `.github/workflows/explain-command.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/explain-command.md
   git add .github/workflows/explain-command.md .github/workflows/explain-command.lock.yml
   git commit -m "Add explain command workflow"
   git push
   ```

4. **Test it:** Go to any issue or PR, type `/explain` in a comment, and
   click Comment.

5. **See the result:** A reply comment with a plain-English explanation.

**Workflow File Contents** (paste into `.github/workflows/explain-command.md`):

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

You provide beginner-friendly explanations in ${{ github.repository }}.

## How You Are Triggered

Someone typed `/explain` in a comment on an issue or PR. The full comment
text is: "${{ needs.activation.outputs.text }}"

## What To Do

Determine what needs explaining based on where the command was used:

### On a Pull Request

Read the PR diff (the code changes) and explain:
- What the PR is changing, in plain English.
- Why these changes are being made (infer from the PR title/description).
- How the changes work at a high level.

### On an Issue

Read the issue title and body and explain:
- What the problem is.
- What part of the codebase is likely involved.
- Possible causes or next steps.

### With a File Path (e.g., `/explain src/auth.ts`)

If the comment includes a file path after `/explain`:
- Read the specified file.
- Explain what it does, its role in the project, and its key functions.

### Response Format

Post a comment:
```
### Explanation

[2-4 paragraphs in plain English. No jargon. Write for someone who is new to
the project.]

### Key Points
- [Point 1]
- [Point 2]
- [Point 3]
```
```

---

### 18. /refactor Command

**What it does and why you want it:**

Type `/refactor` on a PR and the agent analyzes the code changes for
refactoring opportunities — repeated logic, complex conditionals, unclear
names — and posts specific suggestions.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new refactor-command
   ```

2. Open `.github/workflows/refactor-command.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/refactor-command.md
   git add .github/workflows/refactor-command.md .github/workflows/refactor-command.lock.yml
   git commit -m "Add refactor command workflow"
   git push
   ```

4. **Test it:** Go to a PR, type `/refactor` in a comment, click Comment.

5. **See the result:** Inline review comments with suggestions + a summary.

**Workflow File Contents** (paste into `.github/workflows/refactor-command.md`):

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

You suggest refactoring improvements for PR #${{ github.event.issue.number }}
in ${{ github.repository }}.

## How You Are Triggered

Someone typed `/refactor` in a comment on a pull request.

## What To Do

### Step 1: Read the PR Diff

Use the GitHub tools to get the list of changed files and their diffs.

### Step 2: Look for Refactoring Opportunities

- **Extract method** — repeated logic in multiple places that should be a
  shared function
- **Simplify conditionals** — deeply nested if/else that could use guard
  clauses, early returns, or switch/match
- **Remove duplication** — copy-pasted code across files
- **Improve naming** — vague variable names like `data`, `result`, `temp`
- **Reduce complexity** — functions longer than 50 lines or doing too many
  things
- **Use language idioms** — patterns that could be written more naturally

### Step 3: Post Inline Comments

For each suggestion, post an inline review comment on the specific line:
- State what could be improved.
- Explain why (readability, maintainability, performance).
- Show a brief code snippet of the improved version.

### Step 4: Post a Summary

```
## Refactoring Suggestions

Found [N] opportunities across [M] files:
- [count] extract method
- [count] simplify conditional
- [count] naming improvement

These are suggestions, not requirements. Apply what makes sense for your
codebase.
```
```

---

### 19. /merge-main Command

**What it does and why you want it:**

When your PR branch falls behind `main`, you need to merge the latest changes
in. Type `/merge-main` and the agent does it for you: fetches main, merges it
into your PR branch, and pushes the result.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new merge-main
   ```

2. Open `.github/workflows/merge-main.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/merge-main.md
   git add .github/workflows/merge-main.md .github/workflows/merge-main.lock.yml
   git commit -m "Add merge-main command workflow"
   git push
   ```

4. **Test it:** Go to a PR that is behind main, type `/merge-main`, click Comment.

5. **See the result:** New commits appear on the PR branch + a summary comment.

**Workflow File Contents** (paste into `.github/workflows/merge-main.md`):

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

You merge the base branch into the current PR branch in ${{ github.repository }}.

## How You Are Triggered

Someone typed `/merge-main` on PR #${{ github.event.issue.number }}.

## What To Do

### Step 1: Get PR Branch Info

Use the GitHub tools to get the PR details:
- `head.ref` — the PR branch name (e.g., `feature/new-login`)
- `base.ref` — the base branch (e.g., `main`)
- `state` — must be `open`

### Step 2: Fetch and Merge

Run these git commands (using the `bash` tool):
```bash
git fetch origin
git checkout <PR_BRANCH>
git pull origin <PR_BRANCH>
git merge origin/<BASE_BRANCH> --no-edit
```

Replace `<PR_BRANCH>` and `<BASE_BRANCH>` with the actual branch names from
Step 1.

### Step 3: Handle the Result

**If the merge succeeds (no conflicts):**
- Push using the `push-to-pull-request-branch` safe output.
- Post a comment: "Merged `<base>` into `<head>` successfully."

**If there are merge conflicts:**
- For generated files (like lock files), accept the base branch version and
  re-run any generators.
- For code conflicts that require human judgment, abort the merge with
  `git merge --abort` and post a comment:
  "Merge conflicts detected in the following files — please resolve manually:
  [list of conflicted files]"

## Rules

- Never force push.
- Never resolve ambiguous code conflicts automatically.
- Always report what happened in a comment.
```

---

## Scheduled Reports

### 20. Daily Repository Health Report

**What it does and why you want it:**

Every morning, this workflow gathers key metrics about your repository (open
issues, PR backlog, CI status) and posts a summary as a GitHub Discussion.
This gives your team a daily pulse check without anyone having to manually
count things.

**What is a Discussion?** GitHub Discussions are like a forum built into your
repository. You can enable them in Settings > Features > Discussions.

**Prerequisites:** Enable Discussions on your repository (Settings > General >
Features > check "Discussions"). Create a category called "Announcements" if
one does not exist already.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new daily-health
   ```

2. Open `.github/workflows/daily-health.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/daily-health.md
   git add .github/workflows/daily-health.md .github/workflows/daily-health.lock.yml
   git commit -m "Add daily health report workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **Daily Health Report** > **Run workflow**.

5. **See the result:** Check the Discussions tab for a new post in the
   Announcements category.

**Workflow File Contents** (paste into `.github/workflows/daily-health.md`):

```markdown
---
name: Daily Health Report
description: Generates a daily summary of repository health metrics
on:
  schedule: daily at 9am
  workflow_dispatch:
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

You generate a daily repository health summary for ${{ github.repository }}.

## How You Are Triggered

You run daily at 9am UTC, or manually from the Actions tab. Each run creates
a new Discussion and closes the previous day's report (via
`close-older-discussions`).

## What To Do

### Step 1: Collect Metrics

Use the GitHub tools to gather:

1. **Issues**: how many are open, how many were opened today, how many were
   closed today, which open issue is the oldest.
2. **Pull Requests**: how many are open, how many are drafts, what is the
   average age, which PR has been open the longest.
3. **CI Status**: look at the last 5 workflow runs on the default branch —
   how many passed, how many failed.
4. **Activity**: how many commits were pushed to the default branch in the
   last 24 hours.

### Step 2: Create the Report

Post a Discussion in the "announcements" category:

```
### Repository Health — [today's date]

| Metric | Value | Trend |
|--------|-------|-------|
| Open Issues | 42 | +3 from yesterday |
| Open PRs | 12 | -1 from yesterday |
| CI Status | Passing | 15 run streak |

### Needs Attention
- PR #89 has been open 21 days with no review
- Issue #201 is critical with no assignee
- CI has failed 3 times this week on the `lint` job

### Wins
- Closed 8 issues this week
- Average PR merge time improved to 1.2 days
```

Keep it concise. Focus on items that need action.
```

---

### 21. Weekly Contributor Summary

**What it does and why you want it:**

Recognizes everyone who contributed during the week. Posted as a Discussion
every Friday, listing who merged what.

**Setup Instructions:**

1. Create the file:
   ```bash
   gh aw new weekly-contributors
   ```

2. Open `.github/workflows/weekly-contributors.md` and paste the content below.

3. Compile and push:
   ```bash
   gh aw compile .github/workflows/weekly-contributors.md
   git add .github/workflows/weekly-contributors.md .github/workflows/weekly-contributors.lock.yml
   git commit -m "Add weekly contributor summary workflow"
   git push
   ```

4. **Test it:** Click **Actions** > **Weekly Contributor Summary** >
   **Run workflow**.

5. **See the result:** Check the Discussions tab.

**Workflow File Contents** (paste into `.github/workflows/weekly-contributors.md`):

```markdown
---
name: Weekly Contributor Summary
description: Recognizes contributors and summarizes merged work each week
on:
  schedule: weekly on friday at 4pm
  workflow_dispatch:
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

You generate a weekly contributor summary for ${{ github.repository }}.

## How You Are Triggered

You run every Friday at 4pm UTC, or manually from the Actions tab.

## What To Do

### Step 1: Get Merged PRs

Use the GitHub tools to find all pull requests merged in the last 7 days.
For each PR, get: author, title, number, files changed, lines added/removed.

### Step 2: Group by Contributor

Organize the PRs by author.

### Step 3: Post the Summary

Create a Discussion:

```
### Week of [start date] — [end date]

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

Thanks to everyone who contributed this week!
```

If no PRs were merged, post: "No PRs were merged this week."
```

---

## Advanced Patterns

### 22. Multi-Engine Workflow (Claude)

**What it does and why you want it:**

All previous examples use the default `copilot` engine (GitHub Copilot). This
example uses the `claude` engine (Anthropic Claude) for a deep code analysis
slash command. Different engines may have different strengths — Claude is known
for thorough analysis and long-form reasoning.

**Setup Instructions:**

Same as all other workflows. Create, paste, compile, push.

```bash
gh aw new deep-analysis
# Paste content below into .github/workflows/deep-analysis.md
gh aw compile .github/workflows/deep-analysis.md
git add .github/workflows/deep-analysis.md .github/workflows/deep-analysis.lock.yml
git commit -m "Add deep analysis workflow (Claude engine)"
git push
```

**Test it:** Type `/analyze` on an issue or PR comment.

**Workflow File Contents** (paste into `.github/workflows/deep-analysis.md`):

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

You perform in-depth code analysis for ${{ github.repository }} using the
Claude AI engine.

## How You Are Triggered

Someone typed `/analyze` on an issue or PR. The comment text is:
"${{ needs.activation.outputs.text }}"

## What To Do

### On an Issue

Read the issue body. Identify which area of the codebase is involved. Read
the relevant source files (use the `repos` toolset). Trace the likely code
path, identify the root cause, and explain it with references to specific
files and line numbers.

### On a Pull Request

Read the PR diff. Go beyond surface-level review:
- Analyze the architectural impact of the changes.
- Identify subtle bugs that a quick review might miss.
- Check for performance implications.
- Evaluate test coverage gaps.

### With Arguments (e.g., `/analyze src/auth/`)

Focus analysis on the specified path. Read the files in that directory.
Explain the module's design, identify tech debt, and suggest improvements.

### Response Format

Post one detailed comment:
1. **Executive Summary** — 2-3 sentences
2. **Detailed Analysis** — organized by topic
3. **Recommendations** — prioritized list of action items
4. **References** — links to specific files and line numbers
```

**What is different about this workflow?**
- `engine: claude` — uses Claude instead of Copilot.
- `web-search: {}` and `web-fetch: {}` — gives the agent access to search
  the web and fetch documentation, which is useful for looking up error codes
  or library docs.
- `network: allowed: [defaults, github]` — restricts network access to only
  GitHub and default domains for security.

---

### 23. Workflow with Custom MCP Server

**What it does and why you want it:**

This shows how to give the AI agent access to external tools beyond GitHub —
in this case, a PostgreSQL database. The agent can query your database directly
to check for performance issues.

**What is an MCP server?** MCP (Model Context Protocol) is a standard for
connecting AI agents to tools. An MCP server is a program that exposes tools
(like "run a SQL query") that the AI can call. You configure MCP servers in
the `tools:` section.

**Prerequisites:** You need a `DATABASE_URL` secret configured in your
repository (Settings > Secrets and variables > Actions > New repository secret).

**Setup Instructions:**

Same pattern. Create, paste, compile, push.

```bash
gh aw new db-health
# Paste content below into .github/workflows/db-health.md
gh aw compile .github/workflows/db-health.md
git add .github/workflows/db-health.md .github/workflows/db-health.lock.yml
git commit -m "Add database health check workflow"
git push
```

**Test it:** Click **Actions** > **Database Health Check** > **Run workflow**.

**Workflow File Contents** (paste into `.github/workflows/db-health.md`):

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

You check database health for ${{ github.repository }}'s infrastructure.

## How You Are Triggered

You run daily at 6am UTC, or manually from the Actions tab.

## What Tools You Have

You have access to a PostgreSQL MCP server with one tool: `query`. Use it
to run SQL queries against the database. Example:
```
Tool: query
Input: { "sql": "SELECT count(*) FROM pg_stat_activity" }
```

## What To Do

### Step 1: Run Health Checks

Use the `query` tool to run these checks:

1. **Table sizes** — `SELECT schemaname, tablename, pg_total_relation_size(...)
   FROM pg_tables WHERE schemaname = 'public'`. Flag tables over 1GB.
2. **Active connections** — `SELECT count(*) FROM pg_stat_activity`. Compare
   to `max_connections` from `SHOW max_connections`.
3. **Dead tuples** — `SELECT relname, n_dead_tup FROM pg_stat_user_tables
   WHERE n_dead_tup > 10000`. These tables need VACUUM.
4. **Unused indexes** — `SELECT indexrelname FROM pg_stat_user_indexes
   WHERE idx_scan = 0`. These waste space.

### Step 2: Report (Only If Problems Found)

If any check finds issues, create one issue:

```
## Database Health Alert — [today's date]

### Problems Found

**Large Tables (>1GB)**
| Table | Size |
|-------|------|
| events | 2.3 GB |

**High Dead Tuples**
| Table | Dead Tuples |
|-------|-------------|
| sessions | 45,000 |

### Recommended Actions
- [ ] Run `VACUUM ANALYZE` on [table]
- [ ] Drop unused index [index_name]
- [ ] Review connection pool settings
```

If everything is healthy, do nothing.
```

**What is different about this workflow?**
- The `postgres-mcp:` block under `tools:` defines a custom MCP server. When
  the workflow runs, GitHub Actions starts this server automatically and the
  AI agent can call its `query` tool.
- `allowed: ["query"]` restricts the agent to only the `query` tool (no
  creating tables, dropping data, etc.).
- `${{ secrets.DATABASE_URL }}` references a secret you configure in your
  repository settings.

---

### 24. Shared Config via Imports

**What it does and why you want it:**

If you have many workflows that share the same tools, network rules, or output
settings, you can put the shared configuration in a separate file and `import`
it. This avoids copy-pasting the same YAML blocks across 10 different workflows.

**How imports work:** The `imports:` field lists paths to other `.md` files. At
compile time, `gh aw` reads those files and merges their frontmatter into your
workflow. Tools, safe-outputs, permissions, and even markdown bodies all merge
together.

**Setup Instructions:**

You create three files: two shared configs and one workflow that imports them.

1. Create the shared configs directory:
   ```bash
   mkdir -p .github/workflows/shared
   ```

2. Create `.github/workflows/shared/standard-tools.md` with this content:
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

3. Create `.github/workflows/shared/reporting.md` with this content:
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

4. Create the workflow that uses them:
   ```bash
   gh aw new audit-workflow
   ```

5. Open `.github/workflows/audit-workflow.md` and paste:
   ```markdown
   ---
   name: Code Audit
   description: Weekly code quality audit with shared tool and reporting config
   on:
     schedule: weekly on monday
     workflow_dispatch:
   permissions:
     contents: read
   engine: copilot
   imports:
     - shared/standard-tools.md
     - shared/reporting.md
   timeout-minutes: 15
   ---

   # Weekly Code Audit

   You audit code quality in ${{ github.repository }}.

   ## How You Are Triggered

   You run weekly on Monday, or manually from the Actions tab.

   ## Tools and Outputs

   Your tools and reporting configuration are imported from shared configs.
   You have: GitHub tools (default toolset), cache memory, and the ability
   to create a Discussion in the "audits" category.

   ## What To Do

   1. Scan the repository for code quality issues:
      - Files over 500 lines
      - Functions over 50 lines
      - TODO/FIXME/HACK comments older than 30 days
      - Unused imports or variables (based on naming conventions)
   2. Create a Discussion summarizing your findings in a table format.
   3. If no issues found, create a brief "all clear" Discussion.
   ```

6. Compile all three files:
   ```bash
   gh aw compile .github/workflows/audit-workflow.md
   ```

7. Push:
   ```bash
   git add .github/workflows/shared/ .github/workflows/audit-workflow.md .github/workflows/audit-workflow.lock.yml
   git commit -m "Add code audit workflow with shared imports"
   git push
   ```

**Why use imports?** When you change `shared/standard-tools.md`, every workflow
that imports it picks up the change on the next compile. This is much better
than editing 10 separate workflow files when you want to add a new tool or
change a network rule.

---

### 25. Rate-Limited Scheduled Workflow

**What it does and why you want it:**

For production-critical workflows, you want safety guardrails: rate limiting
(prevent runaway execution), auto-expiry (force periodic review), skip
conditions (prevent flooding), and approval gates. This example combines all
of them in a security audit workflow.

**Prerequisites:** Create a GitHub Environment called `production` in your
repository (Settings > Environments > New environment). You can optionally
add reviewers who must approve before the workflow runs.

**Setup Instructions:**

```bash
gh aw new production-audit
# Paste content below into .github/workflows/production-audit.md
gh aw compile .github/workflows/production-audit.md
git add .github/workflows/production-audit.md .github/workflows/production-audit.lock.yml
git commit -m "Add production security audit workflow"
git push
```

**Test it:** Click **Actions** > **Production Security Audit** >
**Run workflow**. If you configured environment reviewers, you will need to
approve the run before it proceeds.

**Workflow File Contents** (paste into `.github/workflows/production-audit.md`):

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

You perform a comprehensive security audit of ${{ github.repository }}.

## Safety Controls (What They Do)

This workflow has several safety features configured in the frontmatter:

- **`stop-after: +90d`** — this workflow auto-disables after 90 days. You
  must re-enable it, forcing you to review whether it is still needed.
- **`skip-if-match`** — if there are already 3+ open issues with the
  `security-audit` label, the workflow skips (prevents flooding).
- **`manual-approval: production`** — requires someone to approve the run
  in the GitHub UI before it executes.
- **`rate-limit: max: 3, window: 1440`** — maximum 3 runs per day (1440
  minutes). Admins are exempt.
- **`strict: true`** — enables strict validation at compile time.
- **`read-only: true`** (on tools) — the agent can only read, never write
  to the repository.
- **`reaction: eyes`** — adds an "eyes" reaction to the triggering event to
  acknowledge the workflow received it.

## How You Are Triggered

Daily at 2am UTC (with all the safety checks above), or manually.

## What To Do

### Step 1: Check Code Scanning Alerts

Use the `code_security` toolset to list all open code scanning alerts.
Summarize by severity: how many critical, high, medium, low.

### Step 2: Review Dependency Vulnerabilities

Check for known CVEs in the repository's dependencies.

### Step 3: Permission Audit

Review the workflow files in the repository. Check that each one follows
the principle of least privilege (no unnecessary `write` permissions).

### Step 4: Create Report

Create one issue summarizing all findings:

```
## Security Audit — [today's date]

### Code Scanning Alerts
| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 2 |
| Medium | 5 |

### Dependency Vulnerabilities
- [List any known CVEs]

### Permission Audit
- [List any workflows with overly broad permissions]

### Recommendations
- [ ] [Specific action items]
```
```

---

## Quick Reference: Frontmatter Cheat Sheet

This is a summary of all the YAML frontmatter options available. Copy the parts
you need.

```yaml
# === REQUIRED ===
on:                                # What triggers the workflow

# === COMMON OPTIONS ===
name: "Workflow Name"              # Display name (defaults to filename)
description: "What it does"        # Description
engine: copilot                    # AI engine: copilot | claude | codex | custom
permissions:                       # GitHub token permissions
  contents: read
  issues: read                     # read | write | none
  pull-requests: read
  actions: read
  security-events: read
tools:                             # What tools the AI can use
  github:
    toolsets: [default]            # GitHub tool groups
  cache-memory: true               # Enable prompt caching
  bash:                            # Allowed shell commands
    - "npm test *"
  edit:                            # File editing tool
  web-search: {}                   # Web search tool
  web-fetch: {}                    # URL fetch tool
safe-outputs:                      # What actions the AI can take
  add-labels:
    max: 10                        # Maximum operations per run
  add-comments:
    max: 5
  create-issue:
    title-prefix: "[Bot] "
    max: 1
timeout-minutes: 15                # Maximum run time

# === SAFETY OPTIONS ===
rate-limit:                        # Prevent runaway execution
  max: 5                           # Max runs per time window
  window: 60                       # Window in minutes
strict: true                       # Enable strict validation
network:                           # Network allowlist
  allowed: [defaults]
roles: [admin, maintainer]         # Who can trigger this workflow

# === ADVANCED OPTIONS ===
imports:                           # Reuse shared configurations
  - shared/tools.md
stop-after: "+30d"                 # Auto-disable after 30 days
skip-if-match: "is:issue label:x"  # Skip if GitHub search matches
manual-approval: production        # Require environment approval
reaction: eyes                     # React to the triggering event
```

---

## Getting Help

- **Something not working?** Check the **Actions** tab in your repository for
  error details. Click the failed run to see logs.
- **Full configuration reference:** See the [Architecture Guide](ARCHITECTURE.md).
- **Step-by-step beginner tutorials:** See the [User Guide](USER_GUIDE.md).
- **Report a bug:** Open an issue on the gh-aw repository.
