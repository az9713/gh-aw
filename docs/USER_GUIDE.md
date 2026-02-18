# User Guide - GitHub Agentic Workflows

A step-by-step guide with 10 hands-on tutorials to get you started with
`gh-aw`. No prior experience with GitHub Actions or AI agents required.

## Table of Contents

- [Why Markdown? Why gh-aw?](#why-markdown-why-gh-aw)
- [Before You Start](#before-you-start)
- [Understanding the Basics](#understanding-the-basics)
- [Tutorial 1: Hello World Workflow](#tutorial-1-hello-world-workflow)
- [Tutorial 2: Automatic Issue Labeling](#tutorial-2-automatic-issue-labeling)
- [Tutorial 3: PR Welcome Bot](#tutorial-3-pr-welcome-bot)
- [Tutorial 4: Slash Command Responder](#tutorial-4-slash-command-responder)
- [Tutorial 5: Scheduled Daily Report](#tutorial-5-scheduled-daily-report)
- [Tutorial 6: Issue Triage with Multiple Outputs](#tutorial-6-issue-triage-with-multiple-outputs)
- [Tutorial 7: Code Review Assistant](#tutorial-7-code-review-assistant)
- [Tutorial 8: Bug Report Analyzer](#tutorial-8-bug-report-analyzer)
- [Tutorial 9: Documentation Checker](#tutorial-9-documentation-checker)
- [Tutorial 10: Multi-Tool Workflow](#tutorial-10-multi-tool-workflow)
- [Common Commands Reference](#common-commands-reference)
- [Troubleshooting](#troubleshooting)

---

## Why Markdown? Why gh-aw?

If you've seen regular GitHub Actions workflows (the `.yml` files in
`.github/workflows/`), you might wonder: **why does gh-aw use markdown files
instead? And why can't I just write the YAML myself?**

These are the right questions. This section answers them.

### The Short Answer

gh-aw workflows are **not** regular GitHub Actions workflows. They are
instructions for **AI agents** -- programs that use artificial intelligence to
read, think, and act on your repository. Writing the YAML for an AI agent
workflow by hand would be like writing machine code instead of Python: technically
possible, but impractical, error-prone, and unnecessary.

### What You Write vs. What gh-aw Produces

Here is a real workflow from the gh-aw project itself. The developer writes this
markdown file (simplified for clarity):

```markdown
---
name: Auto-Triage Issues
on:
  issues:
    types: [opened, edited]
  schedule: every 6h
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 10
  create-discussion:
    category: "audits"
    max: 1
---

# Auto-Triage Issues Agent

You are the Auto-Triage Issues Agent. Automatically categorize
and label GitHub issues to improve discoverability.

When an issue is opened or edited, classify it based on its
title and body. Apply labels like "bug", "enhancement",
"documentation", or "question". If uncertain, add "needs-triage".
```

That is about **40 lines of simple config** plus **plain English instructions**.

When you run `gh aw compile`, it produces a `.lock.yml` file that is over
**1,100 lines** of GitHub Actions YAML. That compiled file contains:

```
+------------------------------------------------------------------+
|  6 JOBS generated automatically:                                  |
|                                                                   |
|  1. pre_activation                                                |
|     - Checks if the user has permission to trigger this workflow  |
|     - Enforces rate limiting (max 5 runs per 60 minutes)          |
|     - Validates team membership                                   |
|                                                                   |
|  2. activation                                                    |
|     - Verifies the workflow file hasn't been tampered with        |
|     - Checks file timestamps                                     |
|                                                                   |
|  3. agent                                                         |
|     - Sets up an isolated sandbox with a network firewall         |
|     - Downloads and starts Docker containers                      |
|     - Configures MCP (Model Context Protocol) servers             |
|     - Starts a secure MCP gateway with API key authentication     |
|     - Validates the COPILOT_GITHUB_TOKEN secret                   |
|     - Installs the Copilot CLI                                    |
|     - Builds the AI prompt from your markdown instructions        |
|     - Substitutes runtime variables (issue number, actor, etc.)   |
|     - Executes the AI agent inside the firewall sandbox           |
|     - Collects and sanitizes the agent's output                   |
|     - Redacts secrets from all log files                          |
|     - Uploads artifacts for debugging                             |
|                                                                   |
|  4. detection                                                     |
|     - Runs a SECOND AI agent to check if the first agent's        |
|       output is safe (threat detection)                           |
|     - Blocks unsafe outputs from being applied                    |
|                                                                   |
|  5. safe_outputs                                                  |
|     - Only runs if threat detection passes                        |
|     - Processes the agent's output through sanitized handlers     |
|     - Validates each label operation against configured limits    |
|     - Actually applies the labels to GitHub issues                |
|     - Creates discussion reports with enforced title prefixes     |
|                                                                   |
|  6. conclusion                                                    |
|     - Reports success or failure                                  |
|     - Handles agent failures gracefully                           |
|     - Records missing tool reports for debugging                  |
+------------------------------------------------------------------+
```

You would need to write and maintain all of that yourself.

### What Does gh-aw Actually Do?

gh-aw is **not** just a markdown-to-YAML translator. It is a complete platform
for building, running, and monitoring AI agent workflows. Here is everything it
handles:

#### 1. Compilation (the .md to .yml translation)

This is the most visible feature. When you run `gh aw compile`, gh-aw:

- **Parses** your markdown frontmatter and validates it against a JSON schema
- **Resolves imports** -- workflows can inherit configuration from shared files
  (like `imports: [shared/mood.md]`), and gh-aw merges them using breadth-first
  search
- **Selects the AI engine** -- different engines (Copilot, Claude, Codex) need
  completely different setup steps, Docker images, and authentication flows.
  gh-aw abstracts all of this behind `engine: copilot`
- **Generates safe-output handlers** -- each write operation (create issue, add
  labels, post comment) gets its own validation schema, rate limits, and
  sanitization rules
- **Configures the network firewall** -- restricts which domains the AI agent
  can access, preventing data exfiltration
- **Sets up MCP servers** -- the Model Context Protocol is how AI agents access
  tools. gh-aw configures a gateway that routes tool calls through a secure proxy
- **Adds threat detection** -- a second AI agent that reviews the first agent's
  output before it's applied
- **Pins all dependencies** -- every GitHub Action is referenced by its SHA hash,
  not a version tag, to prevent supply chain attacks
- **Validates everything** -- 60+ validation rules catch errors at compile time
  instead of at runtime on GitHub's servers

#### 2. Security (what you cannot do yourself easily)

The security system is the main reason gh-aw exists. Writing secure AI agent
workflows by hand is extremely difficult because:

- **AI agents can be tricked.** A malicious issue title could instruct the AI to
  delete other issues, leak secrets, or modify code. gh-aw's safe-output system
  ensures the AI can only take the specific actions you configured, with enforced
  limits
- **Secrets must be protected.** gh-aw automatically redacts secrets from all log
  files, validates that required secrets exist before running, and masks them in
  GitHub Actions output
- **Network access must be restricted.** Without a firewall, an AI agent could
  send your repository data to any server on the internet. gh-aw runs agents
  inside a Docker-based firewall with explicit domain allowlists
- **Permission escalation must be prevented.** If you configure `add-labels`
  with `max: 10`, the AI cannot add 11 labels or create issues instead. Each
  safe-output type has its own validation, sanitization, and rate limiting
- **Expressions can be injected.** GitHub Actions expressions like
  `${{ github.event.issue.title }}` are evaluated before your code runs. A
  specially crafted issue title could inject arbitrary commands. gh-aw's
  expression safety checks prevent this

#### 3. Full CLI Toolset (not just compilation)

gh-aw provides commands for the entire workflow lifecycle:

| Command | What It Does |
|---------|-------------|
| `gh aw init` | Initialize gh-aw in your repository (creates config files, sets up `.gitattributes`) |
| `gh aw new` | Create a new workflow from a template |
| `gh aw compile` | Compile `.md` to `.lock.yml` (with `--watch` mode for auto-compile) |
| `gh aw run` | Execute a workflow directly (for testing before pushing) |
| `gh aw logs` | Download and analyze workflow execution logs (duration, cost, token usage) |
| `gh aw audit` | Deep-dive into a single workflow run (error detection, MCP tool usage stats) |
| `gh aw health` | Check repository health (validates all workflow configs) |
| `gh aw status` | Show compilation status of all workflows |
| `gh aw secrets` | Manage workflow secrets interactively |
| `gh aw fix` | Auto-fix common compilation errors |
| `gh aw upgrade` | Upgrade gh-aw to the latest version |
| `gh aw mcp` | Manage MCP servers (add, list, inspect) |
| `gh aw trial` | Run a workflow in trial mode on a test issue |
| `gh aw enable/disable` | Toggle workflow triggers on or off |

#### 4. Runtime Infrastructure (what runs on GitHub's servers)

When a compiled workflow executes on GitHub Actions, gh-aw's generated code
provides infrastructure that does not exist in regular workflows:

- **Activation gates** -- permission checks, rate limiting, and team membership
  validation run before the AI agent starts
- **MCP gateway** -- a Docker-based proxy that routes AI tool calls to their
  destinations (GitHub API, safe-output handlers) with authentication and logging
- **Agent sandbox** -- the AI runs inside a firewall container that blocks
  unauthorized network access
- **Safe-output processing** -- agent output is collected as structured JSONL,
  validated against schemas, threat-checked, and only then applied to GitHub
- **Artifact collection** -- prompts, logs, firewall records, and agent
  conversations are uploaded as workflow artifacts for debugging

### Why Not Just Write YAML?

To summarize the answer in one table:

| What You'd Have to Do Yourself | What gh-aw Does for You |
|-------------------------------|------------------------|
| Write 1,100+ lines of YAML per workflow | Write ~40 lines of config + plain English |
| Manually configure Docker containers, MCP servers, and firewalls | One line: `engine: copilot` |
| Build your own safe-output validation and sanitization | Declare: `safe-outputs: add-labels: { max: 10 }` |
| Implement rate limiting, permission checks, team membership gates | Declare: `rate-limit: { max: 5, window: 60 }` |
| Pin every action SHA manually and keep them updated | Automatic SHA pinning at compile time |
| Write threat detection to review AI output before applying | Built-in, automatic |
| Debug by reading raw GitHub Actions logs | `gh aw logs`, `gh aw audit` with parsed metrics |
| Know the differences between Copilot, Claude, and Codex setup | Change one word: `engine: claude` |
| Redact secrets from logs yourself | Automatic at every step |
| Validate workflow config by deploying and watching it fail | Compile-time validation with 60+ rules |

**The markdown file is not a limitation. It is the point.** You write what you
want in plain language, and gh-aw handles everything else -- the security, the
infrastructure, the plumbing, and the guardrails.

---

## Before You Start

### What You Need

1. **A GitHub account** - Sign up at https://github.com if you don't have one
2. **A GitHub repository** - Any repository where you have write access
3. **GitHub CLI** (`gh`) - The command-line tool for GitHub

### Step 1: Install GitHub CLI

The GitHub CLI is a tool that lets you interact with GitHub from your terminal.

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
# Should print something like: gh version 2.x.x
```

### Step 2: Log In to GitHub

```bash
gh auth login
```

Follow the prompts:
1. Select "GitHub.com"
2. Select "HTTPS"
3. Select "Login with a web browser"
4. Copy the one-time code shown in your terminal
5. Press Enter to open your browser
6. Paste the code in the browser

**Verify you're logged in:**
```bash
gh auth status
# Should show: Logged in to github.com as YOUR_USERNAME
```

### Step 3: Install gh-aw

```bash
gh extension install github/gh-aw
```

**Verify it installed:**
```bash
gh aw --help
```

You should see a help message listing all available commands.

### Step 4: Navigate to Your Repository

```bash
cd /path/to/your/repository
```

If you don't have a repository yet, create one:
```bash
mkdir my-test-repo
cd my-test-repo
git init
gh repo create my-test-repo --public --source=. --push
```

### Step 5: Initialize gh-aw in Your Repository

```bash
gh aw init
```

This sets up the necessary configuration files.

---

## Understanding the Basics

### What is a Workflow File?

A workflow file is a markdown file (`.md`) that you place in
`.github/workflows/`. It has two parts:

```
+-----------------------------------------------+
| ---                                            |
| name: "My Workflow"         <-- YAML config    |
| on:                              (frontmatter) |
|   issues:                                      |
|     types: [opened]                            |
| engine: copilot                                |
| ---                                            |
|                                                |
| # My Workflow               <-- Natural        |
|                                  language      |
| Read the issue and respond      instructions   |
| with a helpful comment.                        |
+-----------------------------------------------+
```

**The YAML frontmatter** (between `---` markers) tells gh-aw:
- **When** to run (`on:` section)
- **Which AI** to use (`engine:` section)
- **What tools** the AI can access (`tools:` section)
- **What actions** the AI can take (`safe-outputs:` section)

**The markdown body** tells the AI agent what to do in plain English.

### The Workflow Lifecycle

```
1. WRITE        2. COMPILE         3. PUSH          4. TRIGGER
+--------+      +--------+         +--------+       +--------+
| Edit   |  --> | gh aw  |  -->    | git    |  -->  | Event  |
| .md    |      | compile|         | push   |       | occurs |
| file   |      |        |         |        |       |        |
+--------+      +--------+         +--------+       +--------+
                     |                                    |
              Creates .lock.yml                    AI agent runs
              (GitHub Actions)                     on GitHub servers
```

### Key Terms

| Term | Meaning |
|------|---------|
| **Frontmatter** | YAML configuration between `---` markers |
| **Engine** | The AI model (copilot, claude, codex) |
| **Safe-outputs** | Write operations the AI is allowed to perform |
| **Compile** | Convert your `.md` into a `.lock.yml` Actions workflow |
| **Lock file** | The compiled `.lock.yml` that GitHub Actions executes |
| **Trigger** | The event that starts the workflow (issue opened, etc.) |

---

## Tutorial 1: Hello World Workflow

**Goal**: Create your first workflow that responds when you manually trigger it.

**What you'll learn**: Basic workflow structure, manual triggers, compiling, running.

### Step 1: Create the workflow file

```bash
gh aw new hello-world
```

### Step 2: Edit the file

Open `.github/workflows/hello-world.md` in your editor and replace the
contents with:

```markdown
---
name: "Hello World"
on:
  workflow_dispatch:
engine: copilot
tools:
  github:
    toolsets: [default]
safe-outputs:
  add-comment: {}
---

# Hello World Agent

You are a friendly greeting agent. When triggered, create a brief,
cheerful message and post it as a comment on the most recent open issue
in this repository.

Keep your message under 100 words and be encouraging.
```

### Step 3: Compile the workflow

```bash
gh aw compile hello-world
```

You should see a success message. This creates
`.github/workflows/hello-world.lock.yml`.

### Step 4: Commit and push

```bash
git add .github/workflows/
git commit -m "Add hello-world agentic workflow"
git push
```

### Step 5: Run the workflow

```bash
gh aw run hello-world
```

### Step 6: Check the results

```bash
gh aw logs hello-world
```

**Congratulations!** You've created and run your first agentic workflow.

---

## Tutorial 2: Automatic Issue Labeling

**Goal**: Automatically add labels to new issues based on their content.

**What you'll learn**: Issue triggers, label management, safe-output limits.

### Step 1: Create labels in your repository

First, make sure you have some labels. Create them via GitHub UI or:
```bash
gh label create "bug" --color "d73a4a" --description "Something isn't working"
gh label create "feature" --color "a2eeef" --description "New feature request"
gh label create "question" --color "d876e3" --description "Further information requested"
gh label create "documentation" --color "0075ca" --description "Documentation improvements"
```

### Step 2: Create the workflow

Create `.github/workflows/auto-label.md`:

```markdown
---
name: "Auto Label Issues"
on:
  issues:
    types: [opened]
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 3
---

# Automatic Issue Labeler

When a new issue is opened, read its title and body carefully.

Based on the content, add up to 3 appropriate labels:
- "bug" if the issue describes something broken or not working
- "feature" if the issue requests new functionality
- "question" if the issue is asking for help or clarification
- "documentation" if the issue is about docs improvements

Only add labels that clearly match the issue content. When in doubt,
don't add a label.
```

### Step 3: Compile and push

```bash
gh aw compile auto-label
git add .github/workflows/
git commit -m "Add automatic issue labeling workflow"
git push
```

### Step 4: Test it

Create a test issue:
```bash
gh issue create --title "The login button crashes on mobile" --body "When I tap the login button on iOS Safari, the app crashes with a white screen."
```

Wait a minute or two, then check the issue - it should have the "bug" label.

---

## Tutorial 3: PR Welcome Bot

**Goal**: Welcome first-time contributors with a friendly comment.

**What you'll learn**: Pull request triggers, comment creation.

Create `.github/workflows/pr-welcome.md`:

```markdown
---
name: "PR Welcome Bot"
on:
  pull_request:
    types: [opened]
engine: copilot
tools:
  github:
    toolsets: [pull_requests]
safe-outputs:
  add-comment:
    max: 1
---

# Welcome New Contributors

When a new pull request is opened, check if this is the author's first
pull request to this repository.

If it is their first PR, post a warm welcome comment that:
1. Thanks them for their contribution
2. Explains what will happen next (review process)
3. Points them to the contributing guide if one exists

If they have contributed before, post a brief thank-you acknowledging
their continued contributions.

Keep comments concise and friendly. Do not exceed 200 words.
```

Compile and push:
```bash
gh aw compile pr-welcome
git add .github/workflows/
git commit -m "Add PR welcome bot"
git push
```

---

## Tutorial 4: Slash Command Responder

**Goal**: Create a bot that responds to `/help` comments on issues.

**What you'll learn**: Comment triggers, slash commands, conditional execution.

Create `.github/workflows/help-command.md`:

```markdown
---
name: "Help Command"
on:
  issue_comment:
    types: [created]
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-comment:
    max: 1
---

# Help Command Handler

When someone comments "/help" on an issue, respond with a helpful
message that includes:

1. A brief summary of the issue (read the issue title and body)
2. Suggestions for how to resolve or investigate the issue
3. Links to relevant documentation if applicable

Only respond to comments that start with "/help". Ignore all other
comments.

Keep your response focused, practical, and under 300 words.
```

Compile and push:
```bash
gh aw compile help-command
git add .github/workflows/
git commit -m "Add /help slash command responder"
git push
```

**Test it**: Comment `/help` on any issue in your repository.

---

## Tutorial 5: Scheduled Daily Report

**Goal**: Generate a daily summary of repository activity.

**What you'll learn**: Scheduled triggers, creating issues as reports.

Create `.github/workflows/daily-report.md`:

```markdown
---
name: "Daily Activity Report"
on:
  schedule:
    - cron: "0 9 * * 1-5"
engine: copilot
tools:
  github:
    toolsets: [issues, pull_requests]
safe-outputs:
  create-issue:
    max: 1
---

# Daily Repository Activity Report

Generate a daily summary of repository activity. Create an issue
titled "Daily Report - [today's date]" that includes:

1. **New Issues**: List issues opened in the last 24 hours
2. **Closed Issues**: List issues closed in the last 24 hours
3. **Open PRs**: List currently open pull requests
4. **Merged PRs**: List PRs merged in the last 24 hours

Format the report with clear sections and counts. If there was no
activity in a category, say "No activity".

Add the label "report" to the created issue.
```

Compile and push:
```bash
gh aw compile daily-report
git add .github/workflows/
git commit -m "Add daily activity report"
git push
```

This will run automatically at 9:00 AM UTC on weekdays. You can also
trigger it manually:
```bash
gh aw run daily-report
```

---

## Tutorial 6: Issue Triage with Multiple Outputs

**Goal**: Triage issues by adding labels, assigning users, and commenting.

**What you'll learn**: Multiple safe-outputs, combining actions.

Create `.github/workflows/issue-triage.md`:

```markdown
---
name: "Issue Triage"
on:
  issues:
    types: [opened]
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 3
  add-comment:
    max: 1
  assign-issue:
    max: 1
---

# Issue Triage Agent

When a new issue is opened, perform a complete triage:

1. **Classify**: Read the issue and determine its type (bug, feature,
   question, documentation). Add appropriate labels (up to 3).

2. **Prioritize**: If the issue mentions "crash", "data loss", or
   "security", add a "priority: high" label.

3. **Acknowledge**: Post a comment thanking the author and explaining
   the triage results. Mention what labels were added and why.

Be efficient and accurate. Don't add labels you're not confident about.
```

Compile and push:
```bash
gh aw compile issue-triage
git add .github/workflows/
git commit -m "Add issue triage workflow"
git push
```

---

## Tutorial 7: Code Review Assistant

**Goal**: Automatically review PRs and provide feedback comments.

**What you'll learn**: PR events, reading code changes, review feedback.

Create `.github/workflows/code-review.md`:

```markdown
---
name: "Code Review Assistant"
on:
  pull_request:
    types: [opened, synchronize]
engine: copilot
tools:
  github:
    toolsets: [pull_requests]
safe-outputs:
  add-comment:
    max: 1
---

# Code Review Assistant

When a pull request is opened or updated, review the changes and
provide a helpful code review comment.

Your review should cover:
1. **Summary**: Brief description of what the PR changes
2. **Positive aspects**: What was done well
3. **Suggestions**: Potential improvements (be constructive)
4. **Questions**: Any unclear aspects that need clarification

Guidelines:
- Be respectful and constructive
- Focus on important issues, not nitpicks
- Explain the "why" behind suggestions
- Keep the review under 500 words
- Use code blocks for specific suggestions
```

Compile and push:
```bash
gh aw compile code-review
git add .github/workflows/
git commit -m "Add code review assistant"
git push
```

---

## Tutorial 8: Bug Report Analyzer

**Goal**: Analyze bug reports and request missing information.

**What you'll learn**: Conditional logic, structured analysis.

Create `.github/workflows/bug-analyzer.md`:

```markdown
---
name: "Bug Report Analyzer"
on:
  issues:
    types: [opened]
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-comment:
    max: 1
  add-labels:
    max: 2
---

# Bug Report Analyzer

When a new issue is opened with the "bug" label (or when the title
contains words like "bug", "broken", "error", "crash", "fix"):

1. **Analyze completeness**: Check if the bug report includes:
   - Steps to reproduce
   - Expected behavior
   - Actual behavior
   - Environment information (OS, browser, version)

2. **If information is missing**: Post a friendly comment asking for
   the specific missing details. Use a checklist format so the author
   knows exactly what to add.

3. **If complete**: Add a "triage: ready" label and post a comment
   confirming the report has all needed information.

4. **Severity assessment**: If the bug seems critical (crashes, data
   loss, security), add a "priority: high" label.

If the issue doesn't appear to be a bug report, do nothing.
```

---

## Tutorial 9: Documentation Checker

**Goal**: Check if PRs that change code also update documentation.

**What you'll learn**: File-based analysis, documentation best practices.

Create `.github/workflows/docs-checker.md`:

```markdown
---
name: "Documentation Checker"
on:
  pull_request:
    types: [opened, synchronize]
engine: copilot
tools:
  github:
    toolsets: [pull_requests]
safe-outputs:
  add-comment:
    max: 1
  add-labels:
    max: 1
---

# Documentation Checker

When a pull request is opened or updated, check if the changes
might need documentation updates.

Analysis steps:
1. Review the list of changed files
2. If code files are modified (.go, .js, .ts, .py, etc.) but no
   documentation files (.md, .txt, .rst) are included in the PR:
   - Post a gentle reminder comment asking if docs need updating
   - Add a "needs-docs-review" label
3. If both code and docs are modified:
   - Post a brief "thank you for including docs" comment
4. If only docs are modified:
   - Post a brief acknowledgment

Be concise. Don't be annoying - a gentle reminder is sufficient.
Only comment once per PR (check if you've already commented).
```

---

## Tutorial 10: Multi-Tool Workflow

**Goal**: Create a workflow that uses multiple tools together.

**What you'll learn**: Tool configuration, bash access, network settings.

Create `.github/workflows/release-notes.md`:

```markdown
---
name: "Release Notes Generator"
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version to generate notes for"
        required: true
engine: copilot
tools:
  github:
    toolsets: [issues, pull_requests]
  bash:
    - "git log *"
    - "git tag *"
safe-outputs:
  create-issue:
    max: 1
---

# Release Notes Generator

Generate comprehensive release notes for the specified version.

Steps:
1. Use git to find all commits since the last tag
2. Use GitHub API to find all merged PRs since the last release
3. Use GitHub API to find all closed issues since the last release

Create an issue titled "Release Notes - v${{ inputs.version }}" with:

## Changes
- List each PR with its title and number
- Group by category (features, fixes, docs, etc.)

## Contributors
- List unique contributors from merged PRs

## Issues Resolved
- List closed issues referenced by PRs

Format the notes professionally. Use markdown headers and bullet points.
```

Compile and push:
```bash
gh aw compile release-notes
git add .github/workflows/
git commit -m "Add release notes generator"
git push
```

Run with a version input:
```bash
gh aw run release-notes -F version=1.0.0
```

---

## Common Commands Reference

### Everyday Commands

| Command | What It Does |
|---------|-------------|
| `gh aw new <name>` | Create a new workflow from template |
| `gh aw new` | Create workflow interactively (guided) |
| `gh aw compile` | Compile all workflows |
| `gh aw compile <name>` | Compile a specific workflow |
| `gh aw run <name>` | Trigger a workflow on GitHub Actions |
| `gh aw run` | Interactively select and run a workflow |
| `gh aw logs <name>` | View execution logs |
| `gh aw status` | Show status of all workflows |
| `gh aw list` | List all workflows |

### Management Commands

| Command | What It Does |
|---------|-------------|
| `gh aw enable <name>` | Enable a workflow |
| `gh aw disable <name>` | Disable a workflow |
| `gh aw remove <name>` | Remove a workflow |
| `gh aw audit <run-id>` | Debug a failed workflow run |
| `gh aw health` | Check workflow health |
| `gh aw fix` | Auto-fix common workflow issues |

### Setup Commands

| Command | What It Does |
|---------|-------------|
| `gh aw init` | Initialize repository for agentic workflows |
| `gh aw add <source>` | Add a workflow from another repository |
| `gh aw update` | Update workflows to latest versions |
| `gh aw upgrade` | Update gh-aw itself to the latest version |
| `gh aw secrets` | Manage workflow secrets |

### Development Commands

| Command | What It Does |
|---------|-------------|
| `gh aw compile --watch` | Auto-recompile when files change |
| `gh aw compile --strict` | Compile with strict security checks |
| `gh aw compile --validate` | Compile with schema validation |
| `gh aw mcp list` | List MCP servers in workflows |
| `gh aw mcp inspect <name>` | Inspect MCP server configuration |

---

## Troubleshooting

### "Workflow did not run"

**Cause**: The workflow might be disabled or the trigger didn't match.

**Fix**:
```bash
# Check if the workflow is enabled
gh aw status

# Enable it if disabled
gh aw enable my-workflow

# Verify the .lock.yml was pushed
git log --oneline -5
```

### "Compilation failed"

**Cause**: The YAML frontmatter has syntax errors.

**Fix**:
```bash
# Compile with verbose output to see the error
gh aw compile my-workflow -v
```

Common YAML mistakes:
- Incorrect indentation (use spaces, not tabs)
- Missing quotes around special characters
- Incorrect trigger format

### "Permission denied"

**Cause**: The workflow doesn't have the required permissions.

**Fix**: Add the necessary permissions to your frontmatter:
```yaml
permissions:
  issues: write      # Needed for creating/modifying issues
  pull-requests: write  # Needed for PR operations
  contents: read     # Default, always present
```

### "Rate limit exceeded"

**Cause**: Too many API calls in a short period.

**Fix**: Wait a few minutes and try again. The workflow has built-in rate
limit checking.

### "Safe-output limit reached"

**Cause**: The AI tried to create more items than allowed.

**Fix**: Increase the `max:` value in your safe-outputs configuration:
```yaml
safe-outputs:
  add-labels:
    max: 10    # Increase from default
```

### Getting Help

- Run `gh aw --help` for command help
- Run `gh aw <command> --help` for specific command help
- Check logs: `gh aw logs <workflow-name>`
- Debug failures: `gh aw audit <run-id>`
- Community: [GitHub Discussions](https://github.com/orgs/community/discussions/186451)
- Discord: [GitHub Next Discord](https://gh.io/next-discord)
