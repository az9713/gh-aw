<!--

Hello fellow agent! Welcome to GitHub Agentic Workflows = Actions + Agent + Safety.
Here are some pointers to get you started in using this tool.

- Create a new workflow: https://raw.githubusercontent.com/github/gh-aw/main/create.md
- Install: https://raw.githubusercontent.com/github/gh-aw/main/install.md
- Reference: https://raw.githubusercontent.com/github/gh-aw/main/.github/aw/github-agentic-workflows.md

-->

> **This is a clone of the original [`gh-aw`](https://github.com/github/gh-aw) repository by GitHub.**
> The original project lives at **https://github.com/github/gh-aw** — go there for the latest version, issues, and contributions.
> This fork adds a detailed [Use Cases Guide](docs/USE_CASES.md) with 25 copy-paste-ready workflow examples and beginner-friendly setup instructions.

---

# GitHub Agentic Workflows (gh-aw)

Write agentic workflows in natural language markdown, and run them as GitHub Actions.

```
+---------------------------+        +-----------+        +-------------------+
|  Markdown Workflow (.md)  | -----> |  gh-aw    | -----> | GitHub Actions    |
|  (natural language +      |compile |  compiler |        | (.lock.yml YAML)  |
|   YAML frontmatter)      |        +-----------+        +-------------------+
+---------------------------+                                      |
                                                                   v
                                                          +-------------------+
                                                          | AI Agent executes |
                                                          | (Copilot, Claude, |
                                                          |  Codex, Custom)   |
                                                          +-------------------+
```

## Contents

- [What Is This?](#what-is-this)
- [Quick Start](#quick-start)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Your First Workflow](#your-first-workflow)
- [Key Concepts](#key-concepts)
- [Documentation](#documentation)
- [Guardrails and Security](#guardrails-and-security)
- [Contributing](#contributing)
- [Share Feedback](#share-feedback)
- [Related Projects](#related-projects)

## What Is This?

GitHub Agentic Workflows lets you write instructions for AI agents in plain
markdown files and run them automatically through GitHub Actions. Instead of
writing complex YAML workflows by hand, you describe what you want in natural
language and the tool compiles it into a secure, executable GitHub Actions
workflow.

**Example**: You write a markdown file saying "When a new issue is opened, triage
it by adding appropriate labels based on the content" and gh-aw compiles it into
a complete GitHub Actions workflow that runs an AI agent to do exactly that.

### Who Is This For?

- **Repository maintainers** who want to automate issue triage, PR reviews, or
  documentation updates using AI
- **Teams** who want AI-powered automation without writing complex CI/CD
  pipelines
- **Developers** who want to experiment with AI agents in a safe, sandboxed
  environment

## Quick Start

### 1. Install the GitHub CLI (if you don't have it)

```bash
# macOS
brew install gh

# Windows
winget install GitHub.cli

# Linux (Debian/Ubuntu)
sudo apt install gh
```

Then authenticate:
```bash
gh auth login
```

### 2. Install the gh-aw Extension

```bash
gh extension install github/gh-aw
```

Verify it works:
```bash
gh aw --help
```

### 3. Initialize Your Repository

Navigate to your GitHub repository and run:
```bash
gh aw init
```

### 4. Create Your First Workflow

```bash
gh aw new my-first-workflow
```

This creates `.github/workflows/my-first-workflow.md` with a template you can
customize.

### 5. Compile and Run

```bash
gh aw compile       # Compiles .md to .lock.yml
gh aw run my-first-workflow   # Triggers on GitHub Actions
```

For the full quick start experience with detailed walkthroughs, see the
[Quick Start Guide](docs/USER_GUIDE.md).

## How It Works

```
You write this:                      gh-aw produces this:
+-------------------------------+    +----------------------------------+
| .github/workflows/triage.md  |    | .github/workflows/triage.lock.yml|
|-------------------------------|    |----------------------------------|
| ---                           |    | name: triage                     |
| on:                           |    | on:                              |
|   issues:                     |    |   issues:                        |
|     types: [opened]           |    |     types: [opened]              |
| engine: copilot               |    | permissions:                     |
| tools:                        |    |   contents: read                 |
|   github:                     |    | jobs:                            |
|     toolsets: [issues]        |    |   activation:                    |
| safe-outputs:                 |    |     runs-on: ubuntu-latest       |
|   add-labels:                 |    |     steps:                       |
|     max: 5                    |    |       - name: Check permissions  |
| ---                           |    |       - name: Check rate limits  |
|                               |    |   agent:                         |
| # Issue Triage Agent          |    |     runs-on: ubuntu-latest       |
|                               |    |     steps:                       |
| Read the new issue and add    |    |       - name: Setup environment  |
| appropriate labels based on   |    |       - name: Run AI agent       |
| the content.                  |    |   safe_outputs:                  |
+-------------------------------+    |     runs-on: ubuntu-latest       |
                                     |     steps:                       |
                                     |       - name: Add labels         |
                                     +----------------------------------+
```

1. **You write markdown** with YAML frontmatter (configuration) and natural
   language instructions
2. **gh-aw compiles** it into a secure GitHub Actions workflow file
3. **GitHub Actions runs** the workflow, launching an AI agent
4. **The AI agent** reads your instructions and uses the configured tools
5. **Safe-outputs** ensure write operations are sanitized and limited

## Installation

### Prerequisites

- [GitHub CLI](https://cli.github.com/) (`gh`) version 2.0 or later
- A GitHub account with access to GitHub Actions

### Install via GitHub CLI

```bash
gh extension install github/gh-aw
```

### Install a Specific Version

```bash
gh extension install github/gh-aw@v0.37.18
```

### Verify Installation

```bash
gh aw version
gh aw --help
```

### Update to Latest Version

```bash
gh aw upgrade
```

## Your First Workflow

See the [User Guide](docs/USER_GUIDE.md) for 10 hands-on tutorials that walk
you through creating real workflows from scratch, including:

1. Hello World workflow
2. Automatic issue labeling
3. PR review assistant
4. Scheduled reports
5. Slash command responder
6. And more...

## Key Concepts

| Concept | What It Means |
|---------|---------------|
| **Workflow** | A `.md` file with YAML config + natural language instructions |
| **Frontmatter** | The YAML section between `---` delimiters at the top |
| **Engine** | The AI model that runs the workflow (copilot, claude, codex) |
| **Safe-Outputs** | Controlled write operations (create issue, add label, etc.) |
| **Safe-Inputs** | Custom tools the AI agent can use |
| **MCP** | Model Context Protocol - how AI tools communicate |
| **Lock File** | The compiled `.lock.yml` GitHub Actions workflow |
| **Compile** | Converting `.md` to `.lock.yml` |

## Documentation

| Document | Audience | Description |
|----------|----------|-------------|
| [User Guide](docs/USER_GUIDE.md) | Users | Step-by-step tutorials with 10 use cases |
| [Architecture Guide](docs/ARCHITECTURE.md) | Developers | System design with ASCII diagrams |
| [Developer Guide](docs/DEVELOPER_GUIDE.md) | Developers | Setup, build, test, and contribute |
| [CLAUDE.md](CLAUDE.md) | AI Agents | Machine-readable project context |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Contributors | Contribution process and guidelines |
| [DEVGUIDE.md](DEVGUIDE.md) | Developers | Detailed development reference |

## Guardrails and Security

Security is foundational to GitHub Agentic Workflows:

- **Read-only by default** - Workflows start with minimal permissions
- **Safe-outputs** - All write operations go through sanitized handlers with
  configurable limits
- **Sandboxed execution** - AI agents run in isolated environments
- **Network isolation** - Configurable domain allowlists
- **Supply chain security** - All action dependencies are SHA-pinned
- **Tool allow-listing** - Explicit tool configuration required
- **Compile-time validation** - Errors caught before deployment
- **Human approval gates** - Optional approval for critical operations

**Use agentic workflows with caution and human supervision.** Even with
guardrails, AI agents can produce unexpected results.

## Contributing

For development setup and contribution guidelines, see
[CONTRIBUTING.md](CONTRIBUTING.md).

## Share Feedback

We welcome your feedback on GitHub Agentic Workflows!

- [Community Feedback Discussions](https://github.com/orgs/community/discussions/186451)
- [GitHub Next Discord](https://gh.io/next-discord)

## Peli's Agent Factory

See [Peli's Agent Factory](https://github.github.com/gh-aw/blog/2026-01-12-welcome-to-pelis-agent-factory/)
for a guided tour through many uses of agentic workflows.

## Related Projects

- **[Agent Workflow Firewall (AWF)](https://github.com/github/gh-aw-firewall)** -
  Network egress control for AI agents
- **[MCP Gateway](https://github.com/github/gh-aw-mcpg)** - Unified Model
  Context Protocol gateway for centralized access management
