# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Agentic Workflows (`gh-aw`) is a **Go-based GitHub CLI extension** that compiles
natural language markdown workflows into GitHub Actions YAML. Developers write instructions
for AI agents in plain markdown files with YAML frontmatter configuration, and `gh-aw`
compiles them into secure, executable GitHub Actions workflows (`.lock.yml` files).

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

**Key terminology**:
- **Workflow** - A `.md` file in `.github/workflows/` with YAML frontmatter + instructions
- **Lock file** - The compiled `.lock.yml` GitHub Actions workflow (must be tracked in git)
- **Engine** - The AI model backend (copilot, claude, codex, custom)
- **Safe-outputs** - Controlled write operations (create issue, add label, etc.)
- **MCP** - Model Context Protocol for AI-tool communication

## Build & Development Commands

```bash
# First-time setup
make deps           # Install Go + npm dependencies (~1.5min)
make deps-dev       # Install dev tools including golangci-lint (~5-8min)

# Build
make build          # Build binary (auto-syncs action pins and scripts)

# Test - prefer selective testing during development
go test -v -run "TestName" ./pkg/cli/          # Single test (fastest, PREFERRED)
go test -v -run "TestFoo|TestBar" ./pkg/cli/   # Pattern matching
make test-unit                                  # All unit tests (~3min)
make test                                       # Full suite incl. integration (>5min)

# Lint & Format
make fmt             # Format Go, JS, and JSON
make lint            # Full linting (format check + golangci-lint)
make golint-incremental BASE_REF=origin/main   # Lint only changed files (50-75% faster)

# Pre-commit validation (MANDATORY before every commit)
make agent-finish    # Runs: deps-dev, fmt, lint, build, test-all, fix, recompile, security-scan

# Workflow operations
make recompile       # Recompile all .md workflows to .lock.yml (required after compiler changes)
make watch           # Auto-compile on file changes
./gh-aw compile .github/workflows/my-workflow.md   # Compile single workflow
```

## Architecture

### System Layers

```
+=====================================================================+
|                        USER INTERFACE LAYER                          |
|  gh aw init | new | compile | run | logs | audit | mcp | ...        |
|  (cmd/gh-aw/main.go - Cobra commands)                               |
+=====================================================================+
         |                    |                    |
         v                    v                    v
+==================+  +==================+  +==================+
| CLI PACKAGE      |  | PARSER PACKAGE   |  | WORKFLOW PACKAGE |
| (pkg/cli/)       |  | (pkg/parser/)    |  | (pkg/workflow/)  |
|                  |  |                  |  |                  |
| - Command impl   |  | - Frontmatter   |  | - Compiler       |
| - Interactive     |  |   extraction    |  | - Engines        |
| - File watching   |  | - Import        |  | - Safe-outputs   |
| - Console output  |  |   resolution    |  | - MCP setup      |
|                  |  | - JSON schema   |  | - Validation     |
|                  |  |   validation    |  | - YAML gen       |
+==================+  +==================+  +==================+
         |                    |                    |
         v                    v                    v
+=====================================================================+
|                     UTILITY PACKAGES                                 |
| console | logger | constants | fileutil | stringutil | gitutil      |
| envutil | repoutil | timeutil | sliceutil | mathutil | tty | styles |
+=====================================================================+
```

### Compilation Pipeline (4 Phases)

```
Markdown        Phase 1         Phase 2          Phase 3          Phase 4
Input           PARSE           BUILD            VALIDATE         GENERATE
 .md  --------> Extract  -----> Resolve   -----> Schema    -----> Emit
 file            YAML            imports          check            .lock.yml
                 frontmatter     + merge          + engine
                 + body          configs          validation
```

1. **Parse** (`pkg/parser/`): Extracts YAML frontmatter and markdown body from `.md` files
2. **Build** (`pkg/workflow/`): Resolves `imports:` via BFS, merges configs, selects engine
3. **Validate** (`pkg/workflow/`): JSON schema validation, engine-specific checks, security rules
4. **Generate** (`pkg/workflow/`): Emits `.lock.yml` with activation, agent, and safe-output jobs

### Package Structure

- **`cmd/gh-aw/`** - CLI entry point using Cobra. Root command + all subcommand registration.
- **`pkg/cli/`** - Command implementations (largest package). Each command follows:
  `NewXCommand()` returns a cobra command, `RunX()` contains testable logic.
- **`pkg/workflow/`** - Core compilation engine. Contains engine implementations (copilot,
  claude, codex, custom), expression building, validation, safe-output generation, MCP setup.
- **`pkg/parser/`** - Markdown frontmatter parsing, BFS import resolution, JSON schema
  validation. Schemas are embedded via `//go:embed`.
- **`pkg/console/`** - Console formatting utilities (success, info, warning, error messages).
  All CLI output must use these formatters.
- **`pkg/constants/`** - Semantic type aliases (WorkflowID, EngineName, Version, etc.) and
  shared constants with `String()` and `IsValid()` methods.
- **`pkg/logger/`** - Debug logging with namespace filtering (`DEBUG=cli:* gh aw compile`).
- **`pkg/fileutil/`**, **`pkg/stringutil/`**, **`pkg/gitutil/`**, **`pkg/envutil/`**, etc. -
  Small focused utility packages.

### Engine Architecture

Each AI engine has its own file in `pkg/workflow/`:

| Engine | File | Description |
|--------|------|-------------|
| Copilot | `copilot_engine.go` | GitHub Copilot (default). Cannot access api.github.com directly - must use GitHub MCP. |
| Claude | `claude_engine.go` | Anthropic Claude. Uses MCP servers for tool access. |
| Codex | `codex_engine.go` | OpenAI Codex. |
| Custom | `custom_engine.go` | User-defined engine configurations. |

All engines implement common methods via the engine registry (`agentic_engine.go`). Shared
utilities live in `engine_helpers.go`.

### Safe-Outputs System

Write operations go through sanitized safe-output handlers. Each entity type has its own
`create_*.go` file in `pkg/workflow/`:

- `create_issue.go` - Issue creation with label, assignee, sub-issue support
- `create_pull_request.go` - PR creation with reviewers, base branch
- `create_discussion.go` - Discussion creation
- `create_pr_review_comment.go` - PR review comments
- `create_project.go` / `create_project_status_update.go` - Project management
- `create_agent_session.go` - Agent session management
- `create_code_scanning_alert.go` - Security alert creation

### JavaScript/Shell Scripts (Runtime)

Source of truth: `actions/setup/js/*.cjs` and `actions/setup/sh/*.sh`

These are **NOT** embedded in the binary. At runtime, the `actions/setup` action copies
them to `/tmp/gh-aw/actions` where workflow jobs use them via `require()` or direct execution.

### CLI Command Groups

| Group | Commands |
|-------|----------|
| Setup | `init`, `new`, `add`, `remove`, `update`, `upgrade`, `secrets` |
| Development | `compile`, `mcp`, `status`, `list`, `fix` |
| Execution | `run`, `enable`, `disable`, `trial` |
| Analysis | `logs`, `audit`, `health` |
| Utilities | `mcp-server`, `pr`, `completion`, `hash`, `project` |

## Code Conventions

### Output Routing (Unix Conventions)
- **Diagnostic output** (messages, warnings, errors) -> `stderr` via `console.Format*`
- **Structured data** (JSON, hashes, graphs) -> `stdout`
- All CLI output must use `fmt.Fprintln(os.Stderr, console.FormatXMessage(...))` for diagnostics

### Error Handling
- Wrap errors with context: `fmt.Errorf("failed to X: %w", err)`
- Validation errors follow: `[what's wrong]. [what's expected]. [example]`
- Use `console.FormatErrorMessage(err.Error())` for user-facing errors

### Go Style
- **Use `any` not `interface{}`** throughout the codebase
- **Logger namespacing**: `logger.New("pkg:filename")` e.g., `logger.New("cli:compile_command")`
- **Command naming**: Files as `*_command.go`, constructors as `NewXCommand()`, runners as `RunX()`

### Testing
- **Build tags**: Every `*_test.go` must have `//go:build !integration` (unit) or
  `//go:build integration` as the **first line**, followed by blank line
- **Style**: Table-driven with `t.Run()`. Use `require.*` for setup, `assert.*` for validations
- **No mocks** - test real component interactions
- **Selective testing preferred**: `go test -v -run "TestName" ./pkg/package/` during development

### YAML Libraries
- **`goccy/go-yaml`** - For GitHub Actions YAML (1.1/1.2 compatibility). Used in workflow
  compilation, frontmatter parsing.
- **`go.yaml.in/yaml/v3`** - For simple marshaling. Use canonical import path (not
  deprecated `gopkg.in/yaml.v3`).

## Important Warnings

- **Never add `.lock.yml` to `.gitignore`** - compiled workflows must be tracked in git
- **Always run `make recompile`** after changing workflow compilation logic
- **Always run `make build`** after modifying JSON schemas (they're `//go:embed`-ed)
- **Copilot engine cannot access `api.github.com` directly** - must use GitHub MCP server
- **Cache-memory filenames must avoid colons** (NTFS limitation) - use `YYYY-MM-DD-HH-MM-SS`
- **ANSI escape codes in YAML**: Never copy-paste from colored terminal output into workflows.
  The compiler strips ANSI codes automatically, but prevent them at the source.
- **Node.js 20+** required for JavaScript tooling (`make check-node-version`)

## Debugging

```bash
# Enable debug logging
DEBUG=* gh aw compile                    # All debug logs
DEBUG=cli:* gh aw compile                # CLI package only
DEBUG=cli:*,workflow:* gh aw compile     # Multiple packages
DEBUG=*,-workflow:test gh aw compile     # All except specific logger

# Verbose mode
./gh-aw compile --verbose

# Performance profiling
make bench-performance                    # Critical benchmarks
make bench-memory                         # Memory profiling (creates mem.prof, cpu.prof)
go tool pprof -http=:8080 mem.prof       # View profile
```

## Documentation Reference

| Document | Purpose |
|----------|---------|
| [User Guide](docs/USER_GUIDE.md) | Step-by-step tutorials with 10 use cases |
| [Architecture Guide](docs/ARCHITECTURE.md) | System design with ASCII diagrams at multiple abstraction levels |
| [Developer Guide](docs/DEVELOPER_GUIDE.md) | Environment setup, build, test, and contribute |
| [AGENTS.md](AGENTS.md) | Detailed agent development guidelines, skills directory |
| [DEVGUIDE.md](DEVGUIDE.md) | Quick-reference development tasks |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Contribution process and guidelines |

## Performance Copilot Instructions

The `.github/copilot/instructions/` directory contains performance guidelines for builds,
CI, CLI, and workflows that agents should follow when making changes in those areas.

## Key File Paths Quick Reference

| What | Where |
|------|-------|
| CLI entry point | `cmd/gh-aw/main.go` |
| Command implementations | `pkg/cli/*_command.go` |
| Compiler core | `pkg/workflow/compiler.go` |
| Engine implementations | `pkg/workflow/*_engine.go` |
| Safe-output handlers | `pkg/workflow/create_*.go` |
| Frontmatter parser | `pkg/parser/parser.go` |
| JSON schemas (embedded) | `pkg/parser/schemas/` |
| Frontmatter types | `pkg/workflow/frontmatter_types.go` |
| Console formatters | `pkg/console/` |
| Runtime JS handlers | `actions/setup/js/*.cjs` |
| Runtime shell scripts | `actions/setup/sh/*.sh` |
| Sample workflows | `.github/workflows/*.md` |
| Compiled workflows | `.github/workflows/*.lock.yml` |
| Action pin registry | `.github/aw/actions-lock.json` |
| Copilot client | `copilot-client/` (TypeScript) |
