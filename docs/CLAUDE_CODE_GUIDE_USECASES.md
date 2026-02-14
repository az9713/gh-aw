# 50 Use Cases for the `claude-code-guide` Subagent

Practical, non-trivial use cases with exact CLI prompts and testing instructions.

> **How to use**: Copy the prompt into your Claude Code CLI session. Claude will use
> the `claude-code-guide` subagent internally to research the best approach, then
> implement it for you.

---

## What Is the `claude-code-guide` Subagent?

The `claude-code-guide` is a **built-in specialized subagent** that ships with
Claude Code. It is not a plugin, not from a third-party repository, and not
something you install separately. It is baked into Claude Code as one of the
available `subagent_type` options when using the internal Task tool.

### Purpose

It answers questions about three domains:

| Domain | Examples |
|--------|----------|
| **Claude Code (the CLI tool)** | Hooks, slash commands, MCP servers, settings, IDE integrations, keyboard shortcuts, permission modes, status lines, plugins |
| **Claude Agent SDK** | Building custom agents, agent teams, inter-agent communication, task delegation, permission boundaries |
| **Claude API (Anthropic API)** | API usage, tool use schemas, streaming, prompt caching, extended thinking, batch processing, multimodal inputs, Anthropic SDK usage |

### Tools It Has Access To

The `claude-code-guide` agent has access to five tools:

| Tool | What It Does |
|------|-------------|
| `Glob` | Finds files by name pattern (e.g., `**/*.json`, `settings*.json`) |
| `Grep` | Searches file contents with regex (e.g., find all hook configurations) |
| `Read` | Reads file contents (e.g., your current `settings.json`) |
| `WebFetch` | Fetches and processes content from a specific URL |
| `WebSearch` | Searches the web for documentation, guides, and solutions |

It does **not** have access to `Write`, `Edit`, `Bash`, or `Task` — it is a
read-only research agent. It gathers information and returns findings to the
main Claude Code session, which then performs the actual implementation.

### How It Works

```
You ask a question          Claude Code delegates           Guide agent researches
about Claude Code      -->  to claude-code-guide       -->  using its 5 tools
features or setup           subagent internally             (Glob, Grep, Read,
                                                             WebFetch, WebSearch)
                                                                    |
                                                                    v
You see the final      <--  Claude Code implements     <--  Returns findings to
working implementation      based on the research           the main session
```

1. You type a prompt mentioning Claude Code features (hooks, MCP, settings, API, etc.)
2. Claude Code internally spawns the `claude-code-guide` subagent via the Task tool
3. The guide agent searches official documentation, your local config files, and web sources
4. It returns a comprehensive answer with configuration details, schemas, and examples
5. The main Claude Code session uses those findings to implement the actual solution

### When Does Claude Use It?

Claude Code automatically uses the `claude-code-guide` agent when your question
involves Claude Code's own features. You can also explicitly request it:

```
Use the claude-code-guide agent to help me with [your question]
```

Typical triggers:
- "How do I set up hooks for..."
- "Configure MCP to..."
- "What Claude API parameters control..."
- "Set up a custom slash command that..."
- "How does prompt caching work..."

### What It Is NOT

- **Not a plugin** — it's a core part of Claude Code, always available
- **Not an external tool** — it doesn't require installation or configuration
- **Not a code executor** — it researches and reports, it doesn't make changes itself
- **Not a general-purpose agent** — it specializes in Claude Code, Agent SDK, and Claude API topics only
- **Not the same as community projects** — there are third-party repos with similar names, but this is the official built-in subagent

### Other Built-in Subagent Types

The `claude-code-guide` is one of many built-in subagent types. Others include:

| Subagent Type | Purpose |
|---------------|---------|
| `Explore` | Fast codebase exploration — find files, search code, understand architecture |
| `Plan` | Design implementation plans with step-by-step strategies |
| `general-purpose` | Full-capability agent for complex, multi-step tasks |
| `builder` | Write and modify code following plans and conventions |
| `reviewer` | Perform code reviews and identify issues |
| `fixer` | Implement targeted bug fixes with minimal changes |
| `test-writer` | Create comprehensive unit and integration tests |
| `refactorer` | Restructure code for maintainability without changing behavior |
| `security-auditor` | Identify vulnerabilities and provide remediation guidance |
| `doc-fetcher` | Fetch external documentation and API references from the web |

The `claude-code-guide` is unique among these because it is the only one
specifically designed to answer questions about Claude Code itself.

---

## Category 1: Hooks & Automation

### 1. Auto-format on Save

Run a code formatter automatically every time Claude writes or edits a file.

**Prompt:**
```
Use the claude-code-guide agent to set up a PostToolUse hook that runs my
project's formatter (prettier for JS/TS, black for Python, gofmt for Go)
after every Write or Edit tool call. It should only format the file that
was just modified, not the entire project.
```

**How to test:**
- Ask Claude to create a deliberately messy file: `write a JS file with inconsistent indentation and missing semicolons`
- Check the file after Claude writes it — it should already be formatted
- Verify in the hook output that the formatter ran (check stderr/stdout in the terminal)
- Intentionally break formatting in a file and ask Claude to edit one line — the whole file should be reformatted

---

### 2. Lint Gating Before Edits

Block Claude from editing files if there are existing lint errors in the project.

**Prompt:**
```
Use the claude-code-guide agent to create a PreToolUse hook that runs the
project linter (eslint, pylint, or golangci-lint) before allowing any Edit
or Write tool call. If the linter finds errors in the target file, the hook
should block the edit and show the lint errors so Claude can fix them first.
```

**How to test:**
- Introduce a lint error manually in a file (e.g., an unused variable)
- Ask Claude to edit that file for an unrelated change
- The hook should block the edit and Claude should see the lint errors
- After Claude fixes the lint errors, the subsequent edit should go through
- Check by running the linter manually to confirm zero errors

---

### 3. Commit Message Enforcement

Validate that all git commits follow conventional commit format.

**Prompt:**
```
Use the claude-code-guide agent to set up a PreToolUse hook on the Bash tool
that intercepts git commit commands and validates the commit message follows
conventional commits format (feat:, fix:, docs:, chore:, refactor:, test:).
If the message doesn't match, block the command and explain why.
```

**How to test:**
- Ask Claude: `commit the current changes with message "updated stuff"`
- The hook should block it and explain the format requirement
- Ask Claude: `commit with message "fix: resolve null pointer in user service"`
- This should succeed
- Check `git log --oneline -1` to verify the commit went through with the correct message

---

### 4. Auto-test Runner After Code Changes

Automatically run relevant tests after every code modification.

**Prompt:**
```
Use the claude-code-guide agent to configure a PostToolUse hook that detects
when a source file is modified (Write or Edit tool) and automatically runs
the corresponding test file. For example, editing src/utils.ts should trigger
tests/utils.test.ts. The hook should parse the file path to find the matching
test file and run it with the project's test runner.
```

**How to test:**
- Ask Claude to modify a source file that has a corresponding test
- Watch the terminal — the test runner should execute automatically after the edit
- Introduce a deliberate bug and ask Claude to edit the file
- The post-hook should run the test and the failure should appear in output
- Ask Claude to fix the bug — the test should pass on the next run

---

### 5. Security Scan on Dependency Changes

Run a security audit whenever package files are modified.

**Prompt:**
```
Use the claude-code-guide agent to create a PostToolUse hook that triggers
when package.json, package-lock.json, requirements.txt, go.mod, or Cargo.toml
is modified. The hook should run the appropriate security audit command
(npm audit, pip-audit, govulncheck, cargo audit) and display the results.
```

**How to test:**
- Ask Claude: `add the lodash package to this project`
- After the package file is modified, check that the audit ran in the terminal output
- Deliberately add a known vulnerable package version and ask Claude to install it
- The audit results should flag the vulnerability
- Remove the vulnerable package and verify a clean audit

---

### 6. Branch Protection

Prevent Claude from editing files when on protected branches.

**Prompt:**
```
Use the claude-code-guide agent to set up a PreToolUse hook that checks the
current git branch before allowing Write, Edit, or Bash (git commit/push)
operations. If the branch is main, master, or production, block the operation
and tell Claude to create a feature branch first.
```

**How to test:**
- Switch to `main` branch: `git checkout main`
- Ask Claude to edit any file — it should be blocked
- Ask Claude to create a feature branch and switch to it
- Now ask Claude to edit the same file — it should succeed
- Try `git push origin main` through Claude — it should be blocked

---

### 7. Slack/Discord Notification on Task Completion

Send a message to your team channel when Claude finishes a long task.

**Prompt:**
```
Use the claude-code-guide agent to configure a Stop hook that sends a
notification to a Slack webhook URL when Claude finishes responding. The
notification should include the working directory and a summary. Use a
Slack incoming webhook. Store the webhook URL in an environment variable
SLACK_WEBHOOK_URL so it's not hardcoded.
```

**How to test:**
- Set the env var: `export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"`
- Ask Claude to perform a multi-step task (e.g., refactor a function)
- When Claude finishes, check your Slack channel for the notification
- Verify the message contains the working directory
- Test with webhook URL unset — the hook should fail silently without crashing Claude

---

### 8. Session Audit Logging

Record all tool invocations to a local log file for compliance or review.

**Prompt:**
```
Use the claude-code-guide agent to set up both PreToolUse and PostToolUse hooks
that log every tool invocation to ~/.claude/audit-log.jsonl. Each entry should
include: timestamp, tool name, session ID, working directory, and whether it
succeeded or failed. The log should be append-only JSONL format.
```

**How to test:**
- Start a new Claude session and perform several operations (read files, edit, run commands)
- Check the log file: `cat ~/.claude/audit-log.jsonl`
- Verify each entry has a timestamp, tool name, and session ID
- Confirm entries are valid JSON (one per line): `cat ~/.claude/audit-log.jsonl | python -m json.tool --no-ensure-ascii`
- Run multiple sessions and verify they have different session IDs

---

### 9. Token Usage Cost Tracker

Log estimated token usage per session for budget monitoring.

**Prompt:**
```
Use the claude-code-guide agent to create a SessionEnd hook that appends a
line to ~/.claude/cost-tracker.csv with columns: date, session_id, working
directory, and session duration. Also set up a Stop hook that increments a
turn counter in a temp file so I can track how many turns each session uses.
```

**How to test:**
- Complete a full Claude session with several interactions
- Exit the session and check `~/.claude/cost-tracker.csv`
- Verify the CSV has the correct date, session ID, and directory
- Run 3 different sessions and confirm 3 rows appear
- Open in a spreadsheet to verify columns parse correctly

---

### 10. Auto-backup Before Destructive Changes

Create a git stash snapshot before Claude makes potentially destructive changes.

**Prompt:**
```
Use the claude-code-guide agent to set up a PreToolUse hook for the Bash tool
that detects destructive commands (rm, git reset, git checkout ., git clean,
drop table, truncate). Before allowing the command, automatically create a
git stash with a descriptive message including the timestamp. If the working
tree is clean, skip the stash.
```

**How to test:**
- Make some uncommitted changes to a file
- Ask Claude: `delete all .tmp files in this directory`
- Check `git stash list` — there should be a new stash with a timestamp
- Run `git stash pop` to verify your changes are recoverable
- Ask Claude to run a non-destructive command (e.g., `ls`) — no stash should be created

---

## Category 2: MCP Server Configuration

### 11. Database Query MCP Server

Let Claude query your development database directly.

**Prompt:**
```
Use the claude-code-guide agent to help me configure an MCP server that
connects to my local PostgreSQL database. I want Claude to be able to run
SELECT queries but not INSERT, UPDATE, DELETE, or DROP. Set it up in my
project's .claude/settings.json with the connection string from the
DATABASE_URL environment variable.
```

**How to test:**
- Start a Claude session in the project directory
- Ask Claude: `query the database for the first 10 users`
- Verify it returns actual data from your database
- Ask Claude: `delete all records from the users table`
- The MCP server should reject the destructive query
- Check that the connection uses the env var, not a hardcoded string

---

### 12. Internal REST API Bridge

Wrap your team's internal API as an MCP tool.

**Prompt:**
```
Use the claude-code-guide agent to help me build a simple MCP server (in
Node.js or Python) that wraps our internal REST API at http://localhost:3000/api.
It should expose tools for GET requests to any endpoint, with proper error
handling and response formatting. Show me how to register it in Claude Code's
MCP settings.
```

**How to test:**
- Start your local API server
- Start a Claude session and ask: `use the API to get the list of projects`
- Verify Claude calls the MCP tool and gets real API data
- Stop the API server and ask Claude to make another call
- The MCP tool should return a clear error, not crash
- Check `.claude/settings.json` to verify the MCP config is correct

---

### 13. Scoped File System Access

Restrict Claude's file operations to specific directories.

**Prompt:**
```
Use the claude-code-guide agent to help me configure Claude Code so that file
read/write operations are restricted to only the src/ and tests/ directories
of my project. Use PreToolUse hooks to intercept Read, Write, Edit, and Glob
tools and block access to any path outside those directories, especially
.env files, config/secrets/, and node_modules/.
```

**How to test:**
- Ask Claude to read `src/index.ts` — should succeed
- Ask Claude to read `.env` — should be blocked
- Ask Claude to read `config/secrets/api-keys.json` — should be blocked
- Ask Claude to write a file in `tests/` — should succeed
- Ask Claude to write a file in the project root — should be blocked
- Try a path traversal: ask Claude to read `src/../../etc/passwd` — should be blocked

---

### 14. Docker Container Management MCP

Let Claude manage Docker containers during development.

**Prompt:**
```
Use the claude-code-guide agent to help me set up an MCP server that wraps
Docker CLI commands. It should expose tools for: listing containers, viewing
logs, restarting containers, and checking container health. It should NOT
allow deleting containers or images. Configure it in my Claude Code settings.
```

**How to test:**
- Start some Docker containers: `docker compose up -d`
- Ask Claude: `what containers are running?`
- Verify it lists the correct containers via the MCP tool
- Ask Claude: `show me the last 50 lines of logs from the web container`
- Verify real logs are returned
- Ask Claude: `delete the web container` — should be blocked
- Ask Claude: `restart the api container` — should succeed

---

### 15. Cloud Provider CLI MCP

Give Claude controlled access to AWS/GCP/Azure operations.

**Prompt:**
```
Use the claude-code-guide agent to help me configure an MCP server that wraps
the AWS CLI for read-only operations. It should allow: listing S3 buckets,
describing EC2 instances, checking CloudWatch metrics, and reading CloudFormation
stack status. It must block any write/delete/modify operations. Use my existing
AWS credentials from the environment.
```

**How to test:**
- Ensure AWS credentials are configured (`aws sts get-caller-identity` works)
- Ask Claude: `list my S3 buckets`
- Verify it returns real bucket names
- Ask Claude: `show me the status of my EC2 instances in us-east-1`
- Verify real instance data is returned
- Ask Claude: `delete the test-bucket S3 bucket` — should be blocked
- Ask Claude: `terminate instance i-12345` — should be blocked

---

### 16. Jira/Linear Ticket Integration

Let Claude read and update project management tickets.

**Prompt:**
```
Use the claude-code-guide agent to help me set up an MCP server for Jira
(or Linear) that lets Claude: read ticket details, list tickets assigned
to me, add comments, and update ticket status. It should use my API token
from the JIRA_API_TOKEN environment variable. Configure it so Claude can
reference tickets while coding.
```

**How to test:**
- Ask Claude: `what tickets are assigned to me?`
- Verify it returns your actual tickets
- Ask Claude: `show me the details of PROJ-123`
- Verify the ticket description and status are correct
- Ask Claude: `add a comment to PROJ-123 saying the fix is in PR #45`
- Check Jira/Linear to verify the comment appeared
- Ask Claude to implement a fix: `fix the bug described in PROJ-123` — it should read the ticket first

---

### 17. Team Documentation Bridge

Give Claude access to Confluence/Notion docs for context.

**Prompt:**
```
Use the claude-code-guide agent to help me configure an MCP server that
connects to our Confluence (or Notion) workspace. It should let Claude
search for pages, read page content, and list pages in a specific space.
Write access should be disabled. Use the API token from CONFLUENCE_TOKEN
environment variable.
```

**How to test:**
- Ask Claude: `search our docs for the authentication architecture`
- Verify it finds relevant pages from your workspace
- Ask Claude: `read the API design guidelines page`
- Verify the content matches what's in Confluence/Notion
- Ask Claude to implement a feature: `implement the auth flow described in our architecture docs`
- Verify Claude references actual documentation content
- Ask Claude to modify a doc — should be blocked

---

### 18. Custom Linter MCP Tool

Expose proprietary code rules as a tool Claude can invoke.

**Prompt:**
```
Use the claude-code-guide agent to help me build an MCP server that wraps
our custom linting rules. We have a script at ./scripts/custom-lint.sh that
checks for company-specific patterns (no console.log in production code,
required error codes, mandatory header comments). The MCP tool should accept
a file path and return any violations found.
```

**How to test:**
- Ask Claude: `check src/api/handler.ts against our custom lint rules`
- Verify it invokes the MCP tool and returns actual violations
- Fix the violations and run again — should return clean
- Ask Claude to write new code: `create a new API endpoint` — then ask it to lint the result
- Verify Claude can self-correct based on the linter output
- Test with a nonexistent file — should return a clear error

---

### 19. Database Migration MCP

Let Claude run and verify database migrations through a controlled interface.

**Prompt:**
```
Use the claude-code-guide agent to help me create an MCP server that wraps
our database migration tool (knex, alembic, goose, or prisma). It should
expose tools for: checking migration status, generating a new migration,
running pending migrations on the dev database only, and rolling back the
last migration. It must refuse to run against production.
```

**How to test:**
- Ask Claude: `what's the current migration status?`
- Verify it shows pending/applied migrations accurately
- Ask Claude: `create a migration to add an email column to the users table`
- Verify the migration file was generated correctly
- Ask Claude: `run the pending migrations`
- Check the database to verify the schema changed
- Ask Claude: `rollback the last migration` — verify it reverts cleanly
- Set `DATABASE_URL` to a production URL and try — should be blocked

---

### 20. Monitoring Dashboard MCP

Connect Claude to your observability stack for debugging.

**Prompt:**
```
Use the claude-code-guide agent to help me configure an MCP server that
connects to our Grafana instance (or Datadog). It should let Claude query
metrics for the last N hours, check for active alerts, and read recent
error logs. Use the GRAFANA_API_KEY environment variable for authentication.
Read-only access only.
```

**How to test:**
- Ask Claude: `are there any active alerts in our monitoring?`
- Verify it returns real alert status
- Ask Claude: `show me the API error rate for the last 2 hours`
- Verify the metrics data matches your dashboard
- Ask Claude: `check if there are any 500 errors in the logs from today`
- Verify real log entries are returned
- While debugging: ask Claude to `investigate why the /users endpoint is slow` — it should check metrics

---

## Category 3: Agent SDK & Custom Agents

### 21. Code Review Agent

Build an agent that reviews PRs against your team's coding standards.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom Claude Code agent
(in .claude/agents/) that performs code reviews. It should: read the git diff,
check against our CONTRIBUTING.md style guide, verify test coverage for changed
files, flag security concerns, and output a structured review with severity
levels (critical, warning, suggestion). Include the agent YAML config.
```

**How to test:**
- Create the agent, then invoke it: `/agent code-reviewer`
- Make a PR with an obvious bug (e.g., SQL injection) — the agent should flag it as critical
- Make a PR with style violations — should flag as warnings
- Make a clean PR that follows all guidelines — should approve
- Check that the review references your actual CONTRIBUTING.md rules
- Verify it checks for test files corresponding to changed source files

---

### 22. Framework Migration Agent

Design an agent that upgrades framework versions across a codebase.

**Prompt:**
```
Use the claude-code-guide agent to help me create a custom agent that
systematically migrates a codebase from one framework version to another
(e.g., React 17 to 18, Django 4 to 5, or Next.js 13 to 14). The agent
should: read the migration guide, identify affected files, make changes
one module at a time, and run tests after each change. Include rollback
capability if tests fail.
```

**How to test:**
- Set up a small test project with the old framework version
- Run the agent and let it perform the migration
- Verify each file was changed according to the migration guide
- Run the test suite — all tests should pass
- Check that deprecated APIs were replaced with new equivalents
- Intentionally break one test and verify the agent stops and reports the failure

---

### 23. API Documentation Generator Agent

An agent that reads source code and generates API documentation.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom agent that scans
my project's API route handlers, extracts endpoint information (method, path,
parameters, request/response types, auth requirements), and generates
OpenAPI/Swagger YAML documentation. The agent should also generate markdown
docs with examples for each endpoint.
```

**How to test:**
- Run the agent on your project
- Open the generated OpenAPI YAML and validate it: `swagger-cli validate openapi.yaml`
- Import the YAML into Swagger UI and verify all endpoints are listed
- Check that request/response schemas match your actual TypeScript/Python types
- Verify auth requirements are correctly documented
- Add a new endpoint and re-run — the new endpoint should appear in the docs

---

### 24. Test Coverage Gap Finder Agent

An agent that identifies untested code and writes missing tests.

**Prompt:**
```
Use the claude-code-guide agent to help me create a custom agent that: runs
the test suite with coverage, parses the coverage report to find uncovered
functions/branches, prioritizes them by risk (public APIs first, then internal
logic), and writes test cases for the top uncovered areas. It should follow
the existing test patterns in the project.
```

**How to test:**
- Run `npm test -- --coverage` (or equivalent) and note the current coverage percentage
- Run the agent and let it generate tests
- Run coverage again — the percentage should increase
- Verify the new tests actually test meaningful behavior (not just line coverage)
- Check that new tests follow the same patterns as existing tests (same assertions, same setup)
- Run all tests to ensure nothing is broken

---

### 25. Dependency Update Agent

An agent that evaluates, updates, and tests dependencies one at a time.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom agent that: lists
outdated dependencies, evaluates each update's changelog for breaking changes,
updates one dependency at a time, runs the test suite after each update, and
rolls back if tests fail. It should produce a report of what was updated and
what was skipped (with reasons).
```

**How to test:**
- Pin a few dependencies to older versions intentionally
- Run the agent and let it process updates
- Check the report: each dependency should have update status and reasoning
- Verify tests pass after all updates
- Verify that a dependency with known breaking changes was either skipped or handled
- Check `package.json` / `go.mod` for the actual version bumps
- Run `git diff` to see exactly what changed

---

### 26. Incident Response Agent

An agent that correlates errors with recent changes to suggest fixes.

**Prompt:**
```
Use the claude-code-guide agent to help me create a custom agent for incident
response. When given an error message or stack trace, it should: search the
codebase for the relevant code, check git log for recent changes to those
files, identify the likely cause, and suggest a fix with a confidence level.
It should also check if similar errors have been fixed before.
```

**How to test:**
- Take a real stack trace from your application logs
- Feed it to the agent: `/agent incident-responder` then paste the stack trace
- Verify it identifies the correct file and function
- Check that it found relevant recent git commits
- Verify the suggested fix is reasonable
- Test with a known fixed bug — the agent should find the previous fix commit
- Test with a nonsensical stack trace — it should say it can't identify the cause

---

### 27. Developer Onboarding Agent

An agent that answers questions about your codebase for new team members.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom agent that acts as
an onboarding buddy. It should have deep knowledge of: the project's architecture
(from ARCHITECTURE.md), coding conventions (from CONTRIBUTING.md), key abstractions
and patterns, how to run/test/deploy, and common gotchas. It should answer
questions conversationally and point to relevant files.
```

**How to test:**
- Ask the agent: `how is authentication handled in this project?`
- Verify it points to the actual auth files and explains the flow
- Ask: `how do I add a new API endpoint?`
- Follow its instructions and verify they work
- Ask: `what's the testing strategy here?`
- Verify it references your actual test setup
- Ask a question about a part of the codebase that doesn't exist — it should say so honestly

---

### 28. Release Notes Generator Agent

An agent that generates changelogs from git history.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom agent that generates
release notes. It should: read git log between two tags (or from last tag to
HEAD), categorize commits by type (features, fixes, docs, internal), group
related commits, write user-facing descriptions (not raw commit messages),
and highlight breaking changes. Output in markdown.
```

**How to test:**
- Tag the current commit: `git tag v1.0.0`
- Make several commits with different types (feat, fix, docs)
- Run the agent targeting `v1.0.0..HEAD`
- Verify all commits are categorized correctly
- Verify breaking changes are highlighted
- Check that the descriptions are user-friendly, not raw commit messages
- Compare with your actual CHANGELOG.md format

---

### 29. Multi-Agent Team Orchestration

Design a team of specialized agents that collaborate on features.

**Prompt:**
```
Use the claude-code-guide agent to help me set up a multi-agent team workflow
where: Agent 1 (planner) reads the requirements and creates a task breakdown,
Agent 2 (implementer) writes the code for each task, Agent 3 (reviewer) reviews
each implementation, and Agent 4 (tester) writes and runs tests. Show me how
to configure agent teams, task delegation, and inter-agent communication.
```

**How to test:**
- Give the team a small feature to implement (e.g., add a new API endpoint with validation)
- Watch the agents coordinate — planner should produce tasks, implementer should code, reviewer should review
- Verify the final code passes all tests
- Check that the reviewer caught at least one issue that the implementer fixed
- Verify task handoffs happened correctly (no agent worked on unassigned tasks)
- Check the task list to see the completed workflow

---

### 30. Agent Permission Boundaries

Configure fine-grained permissions for different agents in a team.

**Prompt:**
```
Use the claude-code-guide agent to help me configure permission boundaries
for my agent team. The research agent should only have Read/Grep/Glob access.
The implementer should have Read/Write/Edit but no Bash. The tester should
have Read/Bash (for running tests) but no Write/Edit. Show me how to enforce
these boundaries in agent configurations.
```

**How to test:**
- Start the research agent and ask it to edit a file — should be blocked
- Start the implementer and ask it to run a shell command — should be blocked
- Start the tester and ask it to write to a file — should be blocked
- Verify each agent can perform its allowed operations
- Try to escalate privileges from within an agent (e.g., ask research agent to spawn a subagent with write access) — should fail
- Check agent config files to verify tool restrictions are properly set

---

## Category 4: Claude API Integration

### 31. Streaming Response Implementation

Implement proper SSE streaming from the Claude API.

**Prompt:**
```
Use the claude-code-guide agent to show me how to implement streaming responses
from the Claude API using the Anthropic SDK. I need: proper SSE event handling,
incremental text display, error handling mid-stream, cancellation support, and
token counting from stream events. Show me the implementation in
[TypeScript/Python/Go].
```

**How to test:**
- Run the implementation and send a prompt that generates a long response
- Verify text appears incrementally (not all at once)
- Kill the request mid-stream — verify no hanging connections or crashes
- Send a prompt that triggers an API error — verify the error is caught and displayed
- Check that token counts are reported after the stream completes
- Send 5 concurrent streaming requests — verify they don't interfere with each other

---

### 32. Complex Tool Definitions

Design JSON schemas for tools with nested parameters and validation.

**Prompt:**
```
Use the claude-code-guide agent to help me design Claude API tool definitions
for a complex use case: a project management tool with nested parameters
(project > task > subtask), enum constraints, optional fields with defaults,
array parameters with item validation, and dependent parameters (field B
required only if field A is set). Show the full JSON schema and example calls.
```

**How to test:**
- Make an API call with the tool definition and a prompt that should trigger tool use
- Verify Claude generates valid tool calls with all required fields
- Send a prompt that should trigger nested parameters — verify the structure is correct
- Verify enum constraints work: Claude should only use allowed values
- Test optional fields: verify defaults are applied when not specified
- Validate the JSON schema against the Anthropic API spec

---

### 33. System Prompt Architecture

Structure multi-section system prompts with caching optimization.

**Prompt:**
```
Use the claude-code-guide agent to help me design a system prompt architecture
for my application. I need: a static base section (cacheable), a dynamic
context section (per-request), role-specific instructions, tool usage guidelines,
and output format constraints. Show me how to structure this for maximum prompt
cache hits and minimum token waste.
```

**How to test:**
- Make 10 identical API calls and check the `cache_creation_input_tokens` vs `cache_read_input_tokens` in the response
- After the first call, subsequent calls should show cache hits (cache_read > 0)
- Change only the dynamic section and make another call — the static section should still cache
- Measure token counts with and without caching — cached version should be significantly cheaper
- Verify the system prompt produces correct behavior for each role
- Test with different dynamic contexts to ensure the static cache persists

---

### 34. Context Window Management

Implement smart conversation truncation for long-running chats.

**Prompt:**
```
Use the claude-code-guide agent to help me implement context window management
for a chatbot. I need: token counting before each API call, smart truncation
that keeps the system prompt and recent messages, summarization of older messages
before dropping them, and a sliding window that maintains conversation coherence.
Use the Anthropic SDK token counting API.
```

**How to test:**
- Start a conversation and send 50+ messages until you exceed the context limit
- Verify the system doesn't crash — it should truncate gracefully
- After truncation, ask about something mentioned in the first few messages
- Verify the summary retained the key information
- Check token counts: each API call should stay within the model's context limit
- Verify the system prompt is never truncated
- Test with very long individual messages — they should be handled without breaking

---

### 35. Robust API Retry Logic

Build retry logic with exponential backoff for production use.

**Prompt:**
```
Use the claude-code-guide agent to help me implement production-grade retry
logic for Claude API calls. I need: exponential backoff with jitter, different
strategies for different error types (429 rate limit vs 500 server error vs
network timeout), maximum retry limits, circuit breaker pattern for sustained
failures, and detailed logging of retry attempts.
```

**How to test:**
- Simulate a 429 by sending many rapid requests — verify retries with backoff
- Check logs to see retry attempts with increasing delays
- Verify jitter: run 10 retries and confirm delay times aren't identical
- Test circuit breaker: after N consecutive failures, verify it stops retrying temporarily
- Verify non-retryable errors (400 bad request) are NOT retried
- Measure total retry time stays within acceptable bounds (not infinite)
- Test recovery: after failures, verify the circuit breaker resets

---

### 36. Multi-turn Conversation State

Design conversation memory management for a chatbot application.

**Prompt:**
```
Use the claude-code-guide agent to help me design a conversation state manager
for a multi-turn chatbot. I need: persistent conversation storage (database or
file), message history retrieval with pagination, conversation forking (branch
from any point), metadata per message (timestamps, token counts, tool usage),
and conversation export/import. Show me the data model and implementation.
```

**How to test:**
- Start a conversation, send 10 messages, then close and reopen — history should persist
- Verify message metadata (timestamps, token counts) is stored correctly
- Fork a conversation from message 5 — verify both branches work independently
- Export a conversation to JSON and import it — verify all messages and metadata survive
- Test with concurrent conversations — verify they don't interfere
- Delete a conversation and verify it's fully removed from storage

---

### 37. Multimodal Document Processing

Implement API calls that process images and PDFs.

**Prompt:**
```
Use the claude-code-guide agent to help me implement multimodal Claude API
calls that can process: screenshots (PNG/JPEG), PDF documents (with page
selection), and mixed content (text + images in one request). Show me how
to encode images, handle large files, and extract structured data from
visual content. Include proper error handling for unsupported formats.
```

**How to test:**
- Send a screenshot of a UI mockup and ask Claude to describe the components
- Send a PDF invoice and ask Claude to extract the line items as JSON
- Send a mix of text and images in one request — verify both are processed
- Send a very large image (>20MB) — verify proper error handling
- Send an unsupported format (e.g., .svg) — verify clear error message
- Compare extraction accuracy: verify structured output matches actual document content

---

### 38. Batch API Processing

Set up batch processing for high-volume, async workloads.

**Prompt:**
```
Use the claude-code-guide agent to help me set up Claude's Batch API for
processing a large dataset. I need: batch request formatting, job submission,
status polling, result retrieval, error handling for partial failures, and
progress tracking. Show me how to process 1000+ items efficiently with proper
rate limiting and result aggregation.
```

**How to test:**
- Prepare a batch of 100 test prompts in JSONL format
- Submit the batch and verify you get a batch ID
- Poll for status and verify it progresses from `in_progress` to `ended`
- Retrieve results and verify all 100 responses are present
- Intentionally include 5 malformed requests — verify they fail individually without killing the batch
- Check cost: batch processing should be ~50% cheaper than real-time
- Verify results can be mapped back to the original requests

---

### 39. Prompt Caching Strategy

Optimize API costs by maximizing cache hits.

**Prompt:**
```
Use the claude-code-guide agent to help me design a prompt caching strategy
for my application. I need to understand: what qualifies for caching, how to
structure prompts for maximum cache reuse, cache TTL behavior, how to monitor
cache hit rates, and how to structure system prompts with cache_control
breakpoints. Show me before/after cost comparisons.
```

**How to test:**
- Make the same API call 5 times and log `cache_creation_input_tokens` and `cache_read_input_tokens`
- First call: should show cache creation tokens, zero cache read
- Calls 2-5: should show cache read tokens, zero cache creation
- Calculate cost savings: cache reads are 90% cheaper — verify the math
- Change the system prompt slightly and verify the cache is invalidated
- Wait beyond TTL (5 minutes) and retry — verify cache is refreshed
- Structure with breakpoints and verify partial caching works

---

### 40. Extended Thinking Configuration

Tune thinking parameters for complex reasoning tasks.

**Prompt:**
```
Use the claude-code-guide agent to help me configure extended thinking for
Claude API calls. I need to understand: how to enable thinking, budget_tokens
parameter tuning, when thinking helps vs hurts, how to access thinking content
in the response, streaming with thinking blocks, and how to use thinking for
different task types (coding, analysis, math). Show practical examples.
```

**How to test:**
- Send a complex math problem with thinking enabled — verify the thinking block shows reasoning steps
- Compare responses with and without thinking on a hard coding problem — thinking version should be more thorough
- Set budget_tokens to a very low value (1024) — verify Claude still responds but may truncate thinking
- Set budget_tokens high (16000) for a complex problem — verify deeper reasoning
- Stream a response with thinking — verify thinking blocks arrive before the final answer
- Verify thinking content is accessible in the API response under the correct field

---

## Category 5: IDE & Workflow Integration

### 41. VS Code Keyboard Shortcuts

Set up efficient keybindings for Claude Code within VS Code.

**Prompt:**
```
Use the claude-code-guide agent to help me set up VS Code keyboard shortcuts
for Claude Code. I want: a shortcut to open Claude Code terminal, a shortcut
to send the current file to Claude for review, a shortcut to send the selected
text as a prompt, and a shortcut to accept/reject Claude's suggested changes.
Show me the keybindings.json configuration.
```

**How to test:**
- Add the keybindings to VS Code's `keybindings.json`
- Press the "open Claude Code" shortcut — verify the terminal opens with Claude Code running
- Select some code and press the "send selection" shortcut — verify Claude receives it
- Press the "review current file" shortcut — verify Claude reviews the active file
- Verify shortcuts don't conflict with existing VS Code keybindings
- Test on both Windows and Mac (if applicable) — verify modifier keys are correct

---

### 42. Project-scoped Settings

Configure per-project permissions and tool restrictions.

**Prompt:**
```
Use the claude-code-guide agent to help me set up project-specific Claude Code
settings in .claude/settings.json. I want: specific Bash commands allowed
without prompting (npm test, npm run build, make), specific files/directories
blocked from editing (.env, secrets/, migrations/), and custom MCP servers
only for this project. Also set up .claude/settings.local.json for my personal
overrides that won't be committed.
```

**How to test:**
- Verify `npm test` runs without a permission prompt
- Verify `npm run build` runs without a permission prompt
- Ask Claude to edit `.env` — should be blocked
- Ask Claude to edit `secrets/config.json` — should be blocked
- Ask Claude to edit `src/index.ts` — should be allowed
- Commit `.claude/settings.json` and verify `.claude/settings.local.json` is in `.gitignore`
- Clone the repo fresh and verify project settings apply

---

### 43. CLAUDE.md Optimization

Structure CLAUDE.md for maximum agent productivity.

**Prompt:**
```
Use the claude-code-guide agent to help me optimize my CLAUDE.md file. I want
to understand: what information has the highest impact on agent productivity,
how to structure it for minimal token usage, what should be in CLAUDE.md vs
separate files, how to use nested CLAUDE.md files in subdirectories, and how
to keep it maintained as the project evolves. Review my current CLAUDE.md and
suggest specific improvements.
```

**How to test:**
- Start a new Claude session and ask Claude to describe the project architecture
- Verify it uses information from CLAUDE.md without needing to read extra files
- Ask Claude to add a new feature — verify it follows the conventions in CLAUDE.md
- Create a subdirectory CLAUDE.md (e.g., `src/api/CLAUDE.md`) and verify Claude uses it when working in that directory
- Measure: ask the same question before and after CLAUDE.md optimization — the optimized version should produce a better answer with fewer tool calls
- Check token count of CLAUDE.md — should be under 2000 tokens for efficiency

---

### 44. Custom Slash Commands

Create project-specific commands for repetitive workflows.

**Prompt:**
```
Use the claude-code-guide agent to help me create custom slash commands for
my project. I want: /deploy (runs the deployment pipeline), /db-reset (resets
the dev database), /check (runs lint + type-check + tests in sequence),
/new-feature <name> (scaffolds a new feature with files in the right places),
and /release <version> (creates a release tag and changelog). Show me where
to define these and the implementation for each.
```

**How to test:**
- Type `/deploy` in Claude Code — verify it runs the deployment steps
- Type `/db-reset` — verify the database is reset (check by querying it)
- Type `/check` — verify lint, type-check, and tests all run in order
- Type `/new-feature user-profile` — verify the correct files are scaffolded in the right directories
- Type `/release v1.2.0` — verify a git tag is created and changelog is updated
- Type an invalid command `/nonexistent` — verify helpful error message

---

### 45. Git Workflow Automation

Configure Claude's git integration for your branching strategy.

**Prompt:**
```
Use the claude-code-guide agent to help me configure Claude Code to follow
our team's git workflow: GitFlow with feature/, bugfix/, hotfix/ branch
prefixes. Set up hooks so that: commits on feature branches require a Jira
ticket reference (e.g., PROJ-123), Claude auto-creates feature branches
when starting new work, and Claude never pushes directly to develop or main.
Include a PreToolUse hook for git operations.
```

**How to test:**
- Ask Claude to start working on ticket PROJ-456 — should create `feature/PROJ-456`
- Make changes and commit — message should include the Jira reference
- Try to commit without a ticket reference — should be blocked
- Ask Claude to push to `develop` — should be blocked
- Ask Claude to push to the feature branch — should succeed
- Create a bugfix: verify Claude uses `bugfix/` prefix
- Verify the branch naming follows your GitFlow convention

---

### 46. Multi-repo Workspace

Configure Claude Code to work across multiple related repositories.

**Prompt:**
```
Use the claude-code-guide agent to help me set up Claude Code to work across
our monorepo-style workspace with multiple related repos. I have: frontend/
(React app), backend/ (Node.js API), shared/ (TypeScript types), and infra/
(Terraform). I want Claude to understand the relationships between them, be
able to make coordinated changes, and know which repo's test suite to run
when making changes.
```

**How to test:**
- Ask Claude to add a new API field — verify it updates the backend endpoint AND the shared types AND the frontend consumer
- Run tests in each repo after the change — all should pass
- Ask Claude: `where is the User type defined?` — it should find it in the shared/ repo
- Ask Claude to make a frontend-only change — verify it only runs frontend tests
- Verify Claude understands the dependency graph: shared types are consumed by both frontend and backend
- Make a breaking change in shared/ and verify Claude updates both consumers

---

### 47. CI/CD Hook Integration

Connect Claude Code hooks to your CI/CD pipeline.

**Prompt:**
```
Use the claude-code-guide agent to help me create a hook that triggers a
lightweight CI check locally before Claude pushes code. The PreToolUse hook
should intercept git push commands and run: lint, type-check, and unit tests
locally first. If any fail, block the push. Also set up a PostToolUse hook
that posts the commit SHA to our CI dashboard API after a successful push.
```

**How to test:**
- Introduce a type error and ask Claude to push — should be blocked
- Fix the type error and push again — should succeed
- Check your CI dashboard API for the posted commit SHA
- Verify lint and tests both run before the push (check terminal output)
- Time the hook: it should complete within a reasonable time (< 60 seconds for small projects)
- Test with a fast-forward push (no changes) — should skip tests since nothing changed

---

### 48. Permission Mode Tuning

Choose the right permission configuration for each project.

**Prompt:**
```
Use the claude-code-guide agent to help me understand and configure Claude
Code's permission modes. I want: default mode for client projects (maximum
safety), a custom mode for my personal projects where file reads and common
build commands are auto-approved but writes still need confirmation, and
documentation of when to use each mode. Show me how to configure per-project
permission overrides in .claude/settings.json.
```

**How to test:**
- In a client project: verify every tool call asks for permission
- In a personal project: verify file reads happen without prompting
- In a personal project: verify `npm test` and `make build` run without prompting
- In a personal project: verify file writes still ask for permission
- Switch between projects and verify the correct mode activates
- Check that `.claude/settings.json` in each project has the right configuration
- Verify dangerous commands (rm -rf, git push --force) always require permission regardless of mode

---

### 49. Custom Plugin Development

Build a plugin that adds domain-specific tools for your team.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom Claude Code plugin
for our team. The plugin should add: a slash command /api-test that runs our
API integration tests, a PreToolUse hook that enforces our naming conventions
on new files, and a custom MCP server that provides access to our internal
service registry. Show me the full plugin structure, manifest, and how to
distribute it to the team.
```

**How to test:**
- Install the plugin locally: verify it appears in Claude Code's plugin list
- Type `/api-test` — verify integration tests run
- Create a file with a wrong naming convention — the hook should block it
- Create a file with the correct convention — should succeed
- Ask Claude about a service in the registry — verify the MCP tool returns data
- Share the plugin with a teammate — verify they can install and use it
- Disable the plugin and verify all custom functionality is removed

---

### 50. Custom Status Line

Build a status line showing project health metrics.

**Prompt:**
```
Use the claude-code-guide agent to help me build a custom Claude Code status
line that shows: current git branch, test suite status (pass/fail with count),
last build time, number of lint warnings, and TypeScript error count. The
status line should update after every tool call and use color coding (green
for pass, red for fail, yellow for warnings). Implement as a PowerShell
script since I'm on Windows.
```

**How to test:**
- Start a Claude session — verify the status line appears with current branch name
- Run tests through Claude — verify the status line updates with pass/fail counts
- Introduce a lint warning — verify the warning count increases (yellow)
- Introduce a TypeScript error — verify the error count increases (red)
- Fix all issues — verify everything shows green
- Switch git branches — verify the branch name updates
- Check that the status line doesn't slow down Claude's operations (< 500ms refresh)

---

## Quick Reference: How to Invoke the Guide Agent

All of the above prompts use the same pattern. You type the prompt directly into
Claude Code. When Claude sees a question about Claude Code features, hooks, MCP,
settings, or the Claude API, it automatically uses the `claude-code-guide` subagent
to research the answer before implementing.

If Claude doesn't use the guide agent automatically, you can be explicit:

```
Use the claude-code-guide agent to help me with [your question]
```

## Tips for Best Results

1. **Be specific about your stack** — mention your language, framework, and OS
2. **Mention existing config** — if you already have hooks or settings, tell Claude
3. **Test incrementally** — implement one hook/tool at a time, test, then add more
4. **Check settings.json after** — always verify the configuration was written correctly
5. **Restart the session** — some settings changes require a new Claude Code session
