# Developer Guide - GitHub Agentic Workflows

A comprehensive, step-by-step guide for developers who want to understand,
build, test, and contribute to the gh-aw codebase. Written for developers
with traditional C/C++/Java backgrounds who are new to Go and full-stack
web development.

## Table of Contents

- [Technology Primer](#technology-primer)
- [Environment Setup](#environment-setup)
- [Building the Project](#building-the-project)
- [Running Tests](#running-tests)
- [Project Structure Walkthrough](#project-structure-walkthrough)
- [How the Code Works](#how-the-code-works)
- [Adding a New CLI Command](#adding-a-new-cli-command)
- [Adding a New Safe-Output Type](#adding-a-new-safe-output-type)
- [Adding a New AI Engine](#adding-a-new-ai-engine)
- [Modifying the Compiler](#modifying-the-compiler)
- [Working with Tests](#working-with-tests)
- [Debugging](#debugging)
- [Code Style and Conventions](#code-style-and-conventions)
- [Common Pitfalls](#common-pitfalls)
- [Pre-Commit Checklist](#pre-commit-checklist)

---

## Technology Primer

If you come from C/C++ or Java, here's how the key technologies map to
concepts you already know.

### Go Language Basics

| Go Concept | C/C++/Java Equivalent |
|------------|----------------------|
| `package main` | `int main()` / `public static void main` |
| `go build` | `gcc` / `javac` + `jar` |
| `go test` | JUnit / Google Test |
| `go.mod` | `pom.xml` / `CMakeLists.txt` (dependency file) |
| `interface{}` (or `any`) | `void*` / `Object` |
| `struct` | `class` (with fields only) |
| Methods on structs | Member functions |
| Goroutines (`go func()`) | `std::thread` / `Thread` |
| Channels (`chan`) | Thread-safe queues |
| `defer` | RAII / `finally` block |
| `error` return values | Exceptions (but explicit) |

### Go Project Layout

```
Go projects follow a convention (not enforced by the compiler):

my-project/
+-- go.mod              # Like pom.xml - declares module + dependencies
+-- go.sum              # Like package-lock.json - checksums of dependencies
+-- cmd/                # Main programs (each subdir = one binary)
|   +-- myapp/
|       +-- main.go     # Entry point (package main, func main())
+-- pkg/                # Library packages (importable by other projects)
|   +-- mylib/
|       +-- mylib.go    # Library code (package mylib)
|       +-- mylib_test.go  # Tests (same package, _test.go suffix)
+-- internal/           # Private packages (not importable externally)
```

### What is Cobra?

Cobra is Go's most popular CLI framework. Think of it as "argparse for Go"
or "Apache Commons CLI for Go".

```go
// Cobra creates a tree of commands:
rootCmd          // gh aw
  +-- compile    // gh aw compile
  +-- run        // gh aw run
  +-- new        // gh aw new
  +-- logs       // gh aw logs

// Each command has:
//   Use:   "compile [file]"         - Usage pattern
//   Short: "Brief help"              - One-line description
//   Long:  "Detailed help..."        - Full help text
//   RunE:  func(cmd, args) error     - The actual implementation
//   Flags: --verbose, --output, etc. - Command-line flags
```

### What is YAML Frontmatter?

Frontmatter is metadata at the top of a markdown file, enclosed in `---`:

```markdown
---
title: "My Document"     <-- This is YAML frontmatter
date: 2024-01-01
tags: [go, cli]
---

# My Document            <-- This is the markdown body

Regular content here.
```

gh-aw uses frontmatter for workflow configuration (triggers, engine, tools)
and the markdown body for AI instructions.

### What is MCP (Model Context Protocol)?

MCP is a standard protocol for AI models to communicate with external tools.
Think of it as a REST API specifically designed for AI agents.

```
AI Agent  <-- MCP (JSON-RPC) -->  Tool Server

Example: AI Agent wants to create a GitHub issue
  1. Agent sends MCP call: {"method": "create_issue", "params": {...}}
  2. Tool server creates the issue via GitHub API
  3. Tool server returns: {"result": {"id": 123, "url": "..."}}
```

### What is GitHub Actions?

GitHub Actions is GitHub's CI/CD platform. It runs workflows defined in YAML
files when events occur (code pushed, issue opened, etc.).

```
Event (issue opened) --> Workflow (.yml file) --> Jobs --> Steps
                                                          |
                                                   Runs on GitHub's
                                                   servers (Ubuntu VMs)
```

gh-aw **generates** these YAML workflow files from your markdown.

---

## Environment Setup

### Prerequisites

You need these tools installed on your machine:

| Tool | Version | Purpose | Install |
|------|---------|---------|---------|
| **Go** | 1.25.0+ | Main language | https://go.dev/dl/ |
| **Node.js** | 20+ | JavaScript runtime | https://nodejs.org/ |
| **Git** | Any recent | Version control | https://git-scm.com/ |
| **GitHub CLI** | 2.0+ | GitHub integration | https://cli.github.com/ |
| **Make** | Any | Build automation | Pre-installed on Mac/Linux |

### Step-by-Step Setup

#### 1. Install Go

**macOS:**
```bash
brew install go
```

**Windows:**
Download and run the installer from https://go.dev/dl/

**Linux:**
```bash
wget https://go.dev/dl/go1.25.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

**Verify:**
```bash
go version
# Should print: go version go1.25.0 ...
```

#### 2. Install Node.js

**macOS:**
```bash
brew install node@20
```

**Windows:**
Download LTS from https://nodejs.org/

**Linux:**
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

**Verify:**
```bash
node --version   # Should print v20.x.x or higher
npm --version    # Should print 10.x.x or higher
```

#### 3. Install GitHub CLI

```bash
# macOS
brew install gh

# Windows
winget install GitHub.cli

# Linux
sudo apt install gh
```

**Authenticate:**
```bash
gh auth login
```

#### 4. Clone the Repository

```bash
git clone https://github.com/github/gh-aw.git
cd gh-aw
```

#### 5. Install Dependencies

```bash
# Basic dependencies (Go modules + npm packages)
make deps           # Takes ~1.5 minutes on first run

# Full development dependencies (includes linter and tools)
make deps-dev       # Takes ~5-8 minutes on first run
```

What `make deps` does:
1. Downloads Go module dependencies (`go mod download`)
2. Tidies the module file (`go mod tidy`)
3. Installs npm packages for JavaScript files (`cd actions/setup/js && npm ci`)

What `make deps-dev` adds:
1. Everything from `make deps`
2. Installs `golangci-lint` (Go linter)
3. Downloads GitHub Actions JSON schema for validation
4. Installs build tools from `tools.go`

#### 6. Verify Your Setup

```bash
# Build the binary
make build
# Should create ./gh-aw (or gh-aw.exe on Windows)

# Run the binary
./gh-aw --help
# Should print the help message

# Run tests
make test-unit
# Should pass all tests (takes ~3 minutes)

# Run the linter
make lint
# Should pass with no errors
```

If all four commands succeed, your development environment is ready.

---

## Building the Project

### Basic Build

```bash
make build
```

This does three things:
1. Syncs action pin files (SHA hashes for supply chain security)
2. Syncs install scripts to the actions directory
3. Compiles the Go binary with version information

The output is a single binary: `./gh-aw` (or `gh-aw.exe` on Windows).

### What Happens During Build

```
make build
  |
  +-- sync-action-pins
  |   Copies .github/aw/actions-lock.json
  |   to pkg/workflow/data/action_pins.json
  |
  +-- sync-action-scripts
  |   Copies install-gh-aw.sh
  |   to actions/setup-cli/install.sh
  |
  +-- go build -ldflags "-s -w -X main.version=..." ./cmd/gh-aw
      Compiles Go code into single binary
      -s -w: Strip debug info (smaller binary)
      -X main.version=...: Inject version from git tag
```

### Cross-Platform Build

```bash
make build-all    # Build for all platforms

# Or individual platforms:
make build-linux    # Linux amd64 + arm64
make build-darwin   # macOS amd64 + arm64
make build-windows  # Windows amd64
```

### Install Locally for Testing

```bash
make install
# Installs as: gh aw
# Now you can test with: gh aw --help
```

---

## Running Tests

### Quick Reference

```bash
# FASTEST: Run a single test by name
go test -v -run "TestCompileSimpleWorkflow" ./pkg/workflow/

# FAST: Run tests matching a pattern
go test -v -run "TestCompile.*" ./pkg/workflow/

# MEDIUM: Run all unit tests (~3 minutes)
make test-unit

# SLOW: Run full suite including integration (>5 minutes)
make test

# ALL: Go + JavaScript + Copilot client
make test-all
```

### How Go Tests Work (for C++/Java developers)

Go tests live next to the code they test:

```
pkg/workflow/
+-- compiler.go           # Source code
+-- compiler_test.go      # Tests for compiler.go
```

Test functions must:
1. Be in files ending with `_test.go`
2. Be in functions starting with `Test`
3. Accept `*testing.T` as the only parameter

```go
// Example test (like JUnit @Test or Google Test TEST())
func TestMyFunction(t *testing.T) {
    result := MyFunction("input")
    if result != "expected" {
        t.Errorf("got %q, want %q", result, "expected")
    }
}
```

This project uses `testify` for assertions (like Hamcrest or Google Mock):

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestMyFunction(t *testing.T) {
    // assert: test continues if this fails (like EXPECT_EQ in gtest)
    assert.Equal(t, "expected", result, "should match")

    // require: test STOPS if this fails (like ASSERT_EQ in gtest)
    require.NotNil(t, obj, "object must exist")
}
```

### Running Specific Tests

```bash
# Single test by exact name
go test -v -run "TestCompileSimpleWorkflow" ./pkg/workflow/

# Pattern matching (regex)
go test -v -run "TestCompile|TestValidate" ./pkg/workflow/

# All tests in a directory
go test -v ./pkg/cli/

# With race detection (catches concurrency bugs)
go test -race ./pkg/workflow/
```

### Test Organization in This Project

```
Test type             Build tag              Run with
---------             ---------              --------
Unit tests            //go:build !integration  make test-unit
Integration tests     //go:build integration   make test-integration
JavaScript tests      (npm test)               make test-js
Copilot client tests  (npm test)               make test-copilot-client
Security tests        (TestSecurity prefix)     make test-security
Performance tests     (Benchmark prefix)        make bench
Fuzz tests            (Fuzz prefix)             make fuzz
```

---

## Project Structure Walkthrough

Here's every significant directory and what it does:

```
gh-aw/
|
+-- cmd/gh-aw/main.go
|   THE ENTRY POINT. This file:
|   - Defines the root "gh aw" command
|   - Registers all subcommands (compile, run, new, etc.)
|   - Parses global flags (--verbose, --banner)
|   - Handles error formatting and exit codes
|
+-- pkg/cli/
|   COMMAND IMPLEMENTATIONS (largest package, ~200 files)
|   Every gh aw subcommand has its logic here.
|
|   Key patterns:
|   - *_command.go: Command definition (NewXCommand)
|   - *_command_test.go: Tests
|   - compile_*.go: Compilation orchestration
|   - add_*.go: Adding workflows from sources
|   - interactive.go: Interactive TUI mode
|   - flags.go: Shared flag helper functions
|
+-- pkg/workflow/
|   CORE COMPILATION ENGINE (~150 files)
|   The heart of the system. Transforms markdown to YAML.
|
|   Sub-areas:
|   - compiler*.go: Main compilation pipeline
|   - *_engine.go: AI engine implementations
|   - create_*.go: Safe-output type handlers
|   - safe_output*.go: Safe-output system
|   - mcp_*.go: MCP server configuration
|   - validation.go: Workflow validation
|   - expressions.go: Expression building
|   - frontmatter_types.go: Typed config structs
|
+-- pkg/parser/
|   FRONTMATTER PARSING (~20 files)
|   - frontmatter_content.go: Extract YAML from markdown
|   - import_processor.go: Resolve and merge imports
|   - schema_compiler.go: JSON schema validation
|   - schemas/: Embedded validation schemas (318KB)
|
+-- pkg/console/
|   OUTPUT FORMATTING (~15 files)
|   All user-facing output goes through this package.
|   - console.go: FormatError, FormatSuccessMessage, etc.
|   - render.go: Struct rendering with tags
|   - input.go: User prompts (text input)
|   - select.go: Selection menus
|   - table.go: Table rendering
|   - spinner.go: Progress spinners
|
+-- pkg/constants/
|   SEMANTIC TYPES AND CONSTANTS
|   Type aliases that add meaning to primitives:
|   - WorkflowID, EngineName, Version, JobName, StepID
|   - CLIExtensionPrefix = "gh aw"
|
+-- pkg/logger/
|   DEBUG LOGGING
|   - Namespace-based: logger.New("cli:compile")
|   - Controlled by DEBUG environment variable
|   - Shows time deltas between calls
|
+-- pkg/fileutil/, pkg/stringutil/, pkg/gitutil/, etc.
|   SMALL UTILITY PACKAGES
|   Each focuses on one concern (file paths, strings, git, etc.)
|
+-- actions/setup/
|   RUNTIME COMPONENTS (copied to GitHub Actions runner)
|   - js/*.cjs: JavaScript handlers (~117 files)
|     These run DURING workflow execution on GitHub's servers.
|     They handle safe-outputs (create_issue, add_comment, etc.)
|   - sh/*.sh: Shell scripts (~31 files)
|     Infrastructure setup (start MCP servers, download Docker, etc.)
|
+-- copilot-client/
|   TYPESCRIPT COPILOT SDK CLIENT
|   A Node.js program that drives Copilot sessions.
|   - src/: TypeScript source
|   - dist/: Bundled JavaScript (~190KB)
|
+-- internal/tools/
|   BUILD TOOLS (not part of the main binary)
|   - actions-build: Bundles JavaScript actions
|   - generate-action-metadata: Auto-generates action.yml files
|
+-- .github/
|   GITHUB CONFIGURATION
|   - workflows/: Sample and CI workflows (.md + .lock.yml)
|   - aw/: Agentic workflow configuration files
|   - agents/: Agent definition files
|   - copilot/instructions/: Performance guidelines
|
+-- Makefile
|   BUILD AUTOMATION (75+ targets)
|   The central command for all development tasks.
|
+-- go.mod / go.sum
|   DEPENDENCY MANAGEMENT
|   Like pom.xml (Maven) or CMakeLists.txt dependencies.
|
+-- tools.go
    BUILD TOOL DEPENDENCIES
    Blank imports that track tool versions in go.mod.
```

---

## How the Code Works

### The Compile Command (Most Important Flow)

When you run `gh aw compile my-workflow`, here's exactly what happens:

```
1. cmd/gh-aw/main.go
   compileCmd.RunE is called with args=["my-workflow"]
   Parses all flags (--strict, --verbose, etc.)
   Calls cli.CompileWorkflows(ctx, config)

2. pkg/cli/compile_orchestrator.go
   CompileWorkflows() is the orchestrator
   - Creates a Compiler instance
   - Finds the .md file matching "my-workflow"
   - Calls compiler.CompileWorkflow(path)

3. pkg/workflow/compiler.go
   CompileWorkflow() runs the 4-phase pipeline:

   Phase 1 - PARSE:
     pkg/parser/frontmatter_content.go
     ExtractFrontmatterFromContent() splits YAML from markdown

     pkg/parser/import_processor.go
     ProcessImportsFromFrontmatter() resolves and merges imports

   Phase 2 - BUILD:
     pkg/workflow/compiler.go
     Builds WorkflowData struct with all config

     pkg/workflow/engine.go
     Selects the engine (copilot/claude/codex/custom)

   Phase 3 - VALIDATE:
     pkg/workflow/validation.go
     Runs 12+ validation checks

   Phase 4 - GENERATE:
     pkg/workflow/compiler_yaml.go
     Generates the complete YAML output

     pkg/workflow/compiler_jobs.go
     Builds: activation -> agent -> safe_outputs -> conclusion

4. Output:
   Writes .github/workflows/my-workflow.lock.yml
   Displays success/error message via pkg/console/
```

### How a Safe-Output is Generated

When the frontmatter says `safe-outputs: { create-issue: { max: 5 } }`:

```
1. pkg/workflow/safe_outputs_config.go
   Parses the "create-issue" section from frontmatter

2. pkg/workflow/create_issue.go
   CreateIssuesConfig struct defines the schema
   parseIssuesConfig() extracts: max, target, labels, etc.

3. pkg/workflow/compiler_safe_output_jobs.go
   buildSafeOutputsJobs() creates a GitHub Actions job
   Each safe-output becomes a step in the job

4. Runtime (actions/setup/js/create_issue.cjs)
   When the workflow runs on GitHub Actions,
   this JavaScript file actually creates the issue
   via the GitHub API (with limit enforcement)
```

---

## Adding a New CLI Command

Follow this step-by-step guide to add a new command to `gh aw`.

### Step 1: Create the command file

Create `pkg/cli/mycommand_command.go`:

```go
//go:build !integration

package cli

import (
    "fmt"
    "os"

    "github.com/github/gh-aw/pkg/console"
    "github.com/github/gh-aw/pkg/logger"
    "github.com/spf13/cobra"
)

// Logger with namespace following cli:command_name convention
var mycommandLog = logger.New("cli:mycommand")

// NewMyCommandCommand creates the mycommand command
func NewMyCommandCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "mycommand <arg>",
        Short: "Brief description under 80 chars",
        Long: `Detailed description.

Examples:
  gh aw mycommand foo        # Basic usage
  gh aw mycommand foo -v     # Verbose
  gh aw mycommand foo --json # JSON output`,
        Args: cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            verbose, _ := cmd.Flags().GetBool("verbose")
            return RunMyCommand(args[0], verbose)
        },
    }
    return cmd
}

// RunMyCommand is the testable logic (separated from cobra)
func RunMyCommand(arg string, verbose bool) error {
    mycommandLog.Printf("Starting with arg=%s", arg)

    if arg == "" {
        return fmt.Errorf("argument cannot be empty")
    }

    // Your logic here

    fmt.Fprintln(os.Stderr, console.FormatSuccessMessage("Done!"))
    return nil
}
```

### Step 2: Create the test file

Create `pkg/cli/mycommand_command_test.go`:

```go
//go:build !integration

package cli

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestRunMyCommand(t *testing.T) {
    tests := []struct {
        name      string
        arg       string
        shouldErr bool
    }{
        {"valid input", "test", false},
        {"empty input", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := RunMyCommand(tt.arg, false)
            if tt.shouldErr {
                assert.Error(t, err, "should return error")
            } else {
                assert.NoError(t, err, "should succeed")
            }
        })
    }
}
```

### Step 3: Register the command

In `cmd/gh-aw/main.go`, add to the `init()` function:

```go
func init() {
    // ... existing code ...

    myCmd := cli.NewMyCommandCommand()
    myCmd.GroupID = "utilities"  // or "setup", "development", etc.
    rootCmd.AddCommand(myCmd)
}
```

### Step 4: Build and test

```bash
make build
./gh-aw mycommand test
go test -v -run "TestRunMyCommand" ./pkg/cli/
```

---

## Adding a New Safe-Output Type

### Step 1: Add to the schema

Edit `pkg/parser/schemas/main_workflow_schema.json` and add your new type
to the `safe-outputs` section.

### Step 2: Create the Go implementation

Create `pkg/workflow/create_myentity.go`:

```go
package workflow

// CreateMyEntityConfig holds configuration for creating my entities
type CreateMyEntityConfig struct {
    BaseSafeOutputConfig
    SafeOutputTargetConfig
    // Add your custom fields
    CustomField string
}

func (c *Compiler) parseMyEntityConfig(raw map[string]any) *CreateMyEntityConfig {
    config := &CreateMyEntityConfig{}
    // Parse configuration from raw map
    return config
}

func (c *Compiler) generateCreateMyEntityJob(config *CreateMyEntityConfig) map[string]any {
    // Generate the GitHub Actions job steps
    return map[string]any{
        "name": "Create My Entity",
        // ...
    }
}
```

### Step 3: Create the JavaScript handler

Create `actions/setup/js/create_myentity.cjs`:

```javascript
const { getOctokit } = require('./github_api.cjs');

async function createMyEntity(config) {
    const octokit = getOctokit();
    // Implement the actual API call
}

module.exports = { createMyEntity };
```

### Step 4: Wire it into the safe-outputs system

Update `pkg/workflow/safe_outputs_config.go` to parse your new type.
Update `pkg/workflow/compiler_safe_output_jobs.go` to generate the job.

### Step 5: Build and test

```bash
make build
make test-unit
make recompile    # Important! Regenerate all .lock.yml files
```

---

## Working with Tests

### Test Patterns Used in This Project

**Table-Driven Tests** (most common pattern):
```go
func TestSomething(t *testing.T) {
    tests := []struct {
        name     string
        input    string
        expected string
        wantErr  bool
    }{
        {"basic", "hello", "HELLO", false},
        {"empty", "", "", true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := DoSomething(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.expected, got)
            }
        })
    }
}
```

**Golden File Tests** (for output verification):
```bash
# Update golden files when output changes intentionally
make update-golden
```

### Build Tags (Important!)

Every test file MUST have a build tag on the first line:

```go
//go:build !integration    // For unit tests (default)

package cli
// ... tests ...
```

```go
//go:build integration     // For integration tests

package cli
// ... tests ...
```

If you forget the build tag, the linter will catch it.

---

## Debugging

### Debug Logging

The project uses a namespace-based debug logger:

```bash
# Enable all debug logs
DEBUG=* ./gh-aw compile

# Enable specific package
DEBUG=cli:* ./gh-aw compile

# Multiple packages
DEBUG=cli:*,workflow:* ./gh-aw compile

# Exclude a specific namespace
DEBUG=*,-workflow:test ./gh-aw compile
```

### Verbose Mode

Most commands support `--verbose` / `-v`:

```bash
./gh-aw compile my-workflow -v
./gh-aw run my-workflow -v
./gh-aw logs my-workflow -v
```

### Running a Single Test with Output

```bash
# -v shows test output, -count=1 disables caching
go test -v -count=1 -run "TestMyFunction" ./pkg/cli/
```

### Using Go's Built-in Profiler

```bash
make bench-memory
# Creates mem.prof and cpu.prof files

# Interactive analysis
go tool pprof -http=:8080 mem.prof
go tool pprof -http=:8080 cpu.prof
```

---

## Code Style and Conventions

### Output Rules

```go
// ALL diagnostic output goes to stderr:
fmt.Fprintln(os.Stderr, console.FormatSuccessMessage("Done"))
fmt.Fprintln(os.Stderr, console.FormatErrorMessage("Failed"))
fmt.Fprintln(os.Stderr, console.FormatInfoMessage("Processing..."))
fmt.Fprintln(os.Stderr, console.FormatWarningMessage("Caution"))

// ONLY structured data goes to stdout:
fmt.Println(string(jsonBytes))  // JSON output
fmt.Println(hash)               // Hash output
```

### Error Handling

```go
// ALWAYS wrap errors with context:
if err != nil {
    return fmt.Errorf("failed to compile workflow: %w", err)
}

// Validation error format: [what's wrong]. [expected]. [example].
return fmt.Errorf("invalid engine '%s'. Expected: copilot, claude, codex. Example: engine: copilot", engine)
```

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Files | `snake_case.go` | `compile_command.go` |
| Packages | `lowercase` | `pkg/workflow` |
| Types | `PascalCase` | `CompileConfig` |
| Functions | `PascalCase` (exported) | `CompileWorkflows()` |
| Functions | `camelCase` (unexported) | `parseConfig()` |
| Constants | `PascalCase` | `MaxExpressionSize` |
| Logger | `pkg:filename` | `logger.New("cli:compile")` |

### Use `any` Not `interface{}`

```go
// CORRECT:
func Process(data map[string]any) error { ... }

// INCORRECT (old style):
func Process(data map[string]interface{}) error { ... }
```

---

## Common Pitfalls

### 1. Forgetting to recompile workflows

If you change the compiler code, the `.lock.yml` files are stale:
```bash
make build && make recompile
```

### 2. Forgetting to rebuild after schema changes

JSON schemas are embedded in the binary:
```bash
make build    # Must rebuild after changing schemas
```

### 3. Missing build tags on test files

Every `*_test.go` must start with a build tag:
```go
//go:build !integration   // <-- FIRST LINE, no exceptions
```

### 4. Outputting to stdout instead of stderr

```go
// WRONG:
fmt.Println("Processing...")

// RIGHT:
fmt.Fprintln(os.Stderr, console.FormatInfoMessage("Processing..."))
```

### 5. Not running the linter

The linter catches many issues that compile but are wrong:
```bash
make lint
```

---

## Pre-Commit Checklist

Before every commit, run this single command:

```bash
make agent-finish
```

This runs ALL validation steps: `deps-dev`, `fmt`, `lint`, `build`,
`test-all`, `fix`, `recompile`, `dependabot`, `generate-schema-docs`,
`generate-agent-factory`, and `security-scan`.

If `make agent-finish` takes too long during development, at minimum run:

```bash
make fmt          # Format code
make build        # Verify it compiles
go test -v -run "TestMyChanges" ./pkg/mypackage/   # Test your changes
```

Then run `make agent-finish` before pushing.
