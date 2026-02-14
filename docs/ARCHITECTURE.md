# Architecture Guide - GitHub Agentic Workflows (gh-aw)

This document describes the architecture of `gh-aw` at multiple abstraction
levels with ASCII diagrams. It is intended for developers who want to understand
how the system works before modifying the code.

## Table of Contents

- [System Overview](#system-overview)
- [High-Level Architecture](#high-level-architecture)
- [Compilation Pipeline](#compilation-pipeline)
- [Package Architecture](#package-architecture)
- [Engine Architecture](#engine-architecture)
- [Safe-Outputs System](#safe-outputs-system)
- [MCP Server Architecture](#mcp-server-architecture)
- [Runtime Execution Flow](#runtime-execution-flow)
- [Data Flow Diagrams](#data-flow-diagrams)
- [File Organization](#file-organization)
- [Technology Stack](#technology-stack)

---

## System Overview

gh-aw is a **compiler** that translates human-readable markdown files into
executable GitHub Actions workflows. It is a **GitHub CLI extension** written
in **Go** that runs locally on your machine.

```
DEVELOPMENT TIME                          RUNTIME (GitHub Actions)
=================                         ========================

+-------------------+                     +-----------------------------+
|  Developer writes |                     | GitHub Actions Runner       |
|  .md workflow     |                     |                             |
+--------+----------+                     | +-------------------------+ |
         |                                | | Activation Job          | |
         v                                | | - Permission checks     | |
+-------------------+                     | | - Rate limit checks     | |
|  gh-aw compile    |                     | | - Time limit checks     | |
|  (Go binary)      |                     | +------------+------------+ |
+--------+----------+                     |              |              |
         |                                |              v              |
         v                                | +-------------------------+ |
+-------------------+   git push +-----+ | | Agent Job               | |
|  .lock.yml file   | ---------> | Git | | | - Setup MCP servers     | |
|  (GitHub Actions) |            +-----+ | | - Run AI engine         | |
+-------------------+                    | | - Process instructions  | |
                                         | +------------+------------+ |
                                         |              |              |
                                         |              v              |
                                         | +-------------------------+ |
                                         | | Safe-Outputs Job        | |
                                         | | - Create issues         | |
                                         | | - Add comments          | |
                                         | | - Add labels            | |
                                         | +-------------------------+ |
                                         +-----------------------------+
```

### What This Diagram Shows

- **Left side**: What happens on your local machine (development time)
- **Right side**: What happens on GitHub's servers (runtime)
- The `.lock.yml` file is the bridge between the two worlds

---

## High-Level Architecture

The system consists of three major layers:

```
+=====================================================================+
|                        USER INTERFACE LAYER                          |
|                                                                      |
|  gh aw init | new | add | compile | run | logs | audit | ...        |
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
| - Compile config  |  | - Import        |  | - Safe-outputs   |
| - File watching   |  |   resolution    |  | - MCP setup      |
| - Console output  |  | - JSON schema   |  | - Validation     |
|                  |  |   validation    |  | - YAML gen       |
+==================+  +==================+  +==================+
         |                    |                    |
         v                    v                    v
+=====================================================================+
|                     UTILITY PACKAGES                                 |
|                                                                      |
| console | logger | constants | fileutil | stringutil | gitutil      |
| envutil | repoutil | timeutil | sliceutil | mathutil | tty | styles |
| testutil | types                                                     |
+=====================================================================+
         |
         v
+=====================================================================+
|                     RUNTIME COMPONENTS                                |
|                                                                      |
| actions/setup/js/*.cjs    - JavaScript handlers (safe I/O, GitHub)  |
| actions/setup/sh/*.sh     - Shell scripts (MCP servers, Docker)     |
| copilot-client/           - TypeScript Copilot SDK client           |
+=====================================================================+
```

### Layer Responsibilities

| Layer | Responsibility | Key Files |
|-------|---------------|-----------|
| **User Interface** | Parse CLI args, dispatch commands | `cmd/gh-aw/main.go` |
| **CLI** | Orchestrate operations, manage state | `pkg/cli/*.go` |
| **Parser** | Extract and validate frontmatter | `pkg/parser/*.go` |
| **Workflow** | Compile markdown to YAML | `pkg/workflow/*.go` |
| **Utilities** | Shared helpers used across packages | `pkg/*/` |
| **Runtime** | Execute at workflow runtime on GitHub | `actions/setup/` |

---

## Compilation Pipeline

This is the most important architectural flow. It transforms a markdown file
into a GitHub Actions workflow.

### Phase Diagram

```
+------------------+     +------------------+     +------------------+
|  PHASE 1: PARSE  |---->|  PHASE 2: BUILD  |---->| PHASE 3: VALIDATE|
|                  |     |                  |     |                  |
| Read .md file    |     | Build workflow   |     | Check expressions|
| Extract YAML     |     | data structure   |     | Validate schema  |
| Process imports  |     | Configure engine |     | Check permissions|
| Merge configs    |     | Setup tools      |     | Verify security  |
+------------------+     +------------------+     +------------------+
                                                          |
                                                          v
+------------------+     +------------------+     +------------------+
|  PHASE 6: DONE   |<----|  PHASE 5: WRITE  |<----| PHASE 4: GENERATE|
|                  |     |                  |     |                  |
| Display results  |     | Write .lock.yml  |     | Generate YAML    |
| Report warnings  |     | Check file size  |     | Build jobs       |
| Update stats     |     | Diff with old    |     | Build steps      |
+------------------+     +------------------+     +------------------+
```

### Detailed Phase Breakdown

#### Phase 1: Parse (pkg/parser/)

```
Input: .github/workflows/my-workflow.md
       +-----------------------------------------+
       | ---                                      |
       | name: "My Workflow"                      |    <-- YAML frontmatter
       | on:                                      |
       |   issues:                                |
       |     types: [opened]                      |
       | engine: copilot                          |
       | imports:                                 |
       |   - shared/tools-config.md               |
       | ---                                      |
       |                                          |
       | # My Workflow                            |    <-- Markdown body
       |                                          |
       | When a new issue is opened, analyze it   |
       | and add appropriate labels.              |
       +-----------------------------------------+

Step 1: ExtractFrontmatterFromContent()
  - Scans for --- delimiters (line by line)
  - Parses YAML between delimiters into map[string]any
  - Separates markdown body

Step 2: ProcessImportsFromFrontmatter()
  - Resolves import paths (local: ./shared/x.md, remote: owner/repo/path@ref)
  - Uses BFS (breadth-first search) for import ordering
  - Detects import cycles
  - Merges: tools, safe-outputs, permissions, engines, markdown

Step 3: ValidateAgainstSchema()
  - Validates merged frontmatter against JSON schema
  - Schema embedded via //go:embed (pkg/parser/schemas/)
  - Reports errors with file:line:column positions

Output: FrontmatterResult{Frontmatter, Markdown, ImportPaths}
```

**Key files:**
- `pkg/parser/frontmatter_content.go` - Core extraction
- `pkg/parser/import_processor.go` - Import resolution
- `pkg/parser/schema_compiler.go` - JSON schema validation

#### Phase 2: Build (pkg/workflow/)

```
Input: FrontmatterResult from Phase 1

Step 1: Extract engine config
  - String format:  engine: "copilot"
  - Object format:  engine: {id: copilot, model: gpt-5}
  - Resolves via EngineRegistry

Step 2: Parse tools configuration
  - github: {toolsets: [issues], allowed: [create-issue]}
  - bash: ["jq *", "curl *"]
  - playwright: {version: v1.41.0}
  - Custom MCP servers

Step 3: Parse safe-outputs
  - create-issue: {max: 5, target: triggering}
  - add-comment: {max: 10}
  - add-labels: {max: 5}

Step 4: Build WorkflowData struct
  - Assembles all parsed components
  - Stores references to action cache
  - Extracts custom YAML sections

Output: WorkflowData{Name, Engine, Tools, SafeOutputs, Network, ...}
```

**Key files:**
- `pkg/workflow/compiler.go` - Main compilation logic
- `pkg/workflow/compiler_orchestrator_workflow.go` - Orchestration
- `pkg/workflow/engine.go` - Engine configuration types

#### Phase 3: Validate (pkg/workflow/)

```
Input: WorkflowData from Phase 2

Validation checks (in order):
  1. Expression safety (no injection in ${{ }})
  2. Runtime-import file existence
  3. Feature flags validity
  4. Permission combinations (no dangerous combos)
  5. Agent file existence (.github/agents/*.md)
  6. Sandbox configuration consistency
  7. Safe-output target validity
  8. Network domain allowlist
  9. Label/concurrency/tool validity
  10. Permission vs. MCP toolset compatibility
  11. ID-token scope warnings
  12. Tool/toolset compatibility cross-checks

Each validation returns errors with:
  - File path, line number, column number
  - Error type (error, warning, info)
  - Actionable suggestion message

Output: []CompilerError or nil (validation passed)
```

**Key files:**
- `pkg/workflow/validation.go` - Core validation
- `pkg/workflow/strict_mode_validation.go` - Security checks
- `pkg/workflow/expression_safety.go` - Expression injection checks

#### Phase 4: Generate (pkg/workflow/)

```
Input: Validated WorkflowData

Job generation order:
  1. Activation job (pre-flight checks)
     +------------------------------------------+
     | activation:                               |
     |   steps:                                  |
     |     - Check user permissions              |
     |     - Check API rate limits               |
     |     - Check stop-time deadline            |
     |     - Check skip conditions               |
     |     - Validate configuration              |
     +------------------------------------------+

  2. Agent job (main AI execution)
     +------------------------------------------+
     | agent:                                    |
     |   needs: [activation]                     |
     |   steps:                                  |
     |     - Setup gh-aw CLI                     |
     |     - Setup runtime (Node, Python, etc.)  |
     |     - Download Docker images              |
     |     - Start MCP servers                   |
     |     - Configure AI engine                 |
     |     - Run AI agent with prompt            |
     |     - Save agent output                   |
     +------------------------------------------+

  3. Safe-outputs job (write operations)
     +------------------------------------------+
     | safe_outputs:                             |
     |   needs: [agent]                          |
     |   steps:                                  |
     |     - Parse agent output                  |
     |     - Create issues (if configured)       |
     |     - Add comments (if configured)        |
     |     - Add labels (if configured)          |
     |     - Update PRs (if configured)          |
     +------------------------------------------+

  4. Conclusion job (final status)
     +------------------------------------------+
     | conclusion:                               |
     |   needs: [safe_outputs]                   |
     |   steps:                                  |
     |     - Report final status                 |
     |     - Clean up artifacts                  |
     +------------------------------------------+

Output: Complete GitHub Actions YAML string
```

**Key files:**
- `pkg/workflow/compiler_yaml.go` - YAML generation
- `pkg/workflow/compiler_jobs.go` - Job generation
- `pkg/workflow/compiler_safe_output_jobs.go` - Safe output jobs

---

## Package Architecture

### Package Dependency Graph

```
cmd/gh-aw/main.go
    |
    +---> pkg/cli/          (Command implementations)
    |        |
    |        +---> pkg/workflow/    (Core compilation)
    |        |        |
    |        |        +---> pkg/parser/     (Frontmatter parsing)
    |        |        +---> pkg/constants/  (Shared types)
    |        |        +---> pkg/types/      (MCP types)
    |        |
    |        +---> pkg/console/    (Output formatting)
    |        +---> pkg/logger/     (Debug logging)
    |        +---> pkg/fileutil/   (File operations)
    |        +---> pkg/stringutil/ (String helpers)
    |        +---> pkg/gitutil/    (Git helpers)
    |        +---> pkg/envutil/    (Environment vars)
    |        +---> pkg/repoutil/   (Repository helpers)
    |
    +---> pkg/console/      (Banner printing)
    +---> pkg/constants/     (CLI prefix)
    +---> pkg/parser/        (Engine validation)
    +---> pkg/workflow/      (Engine registry)
```

### Package Details

#### pkg/cli/ - CLI Command Implementations

The largest package. Contains one file per command following this pattern:

```
Pattern: NewXCommand() -> *cobra.Command
         RunX(config)  -> error

File naming:
  *_command.go       - Command definition + logic
  *_command_test.go  - Tests
  *_config.go        - Config types (for complex commands)
  *_helpers.go       - Helper functions
  *_orchestrator.go  - Multi-step orchestration
```

Key command files:
```
pkg/cli/
+-- add_command.go                  # Add workflows from repos
+-- compile_orchestrator.go         # Compile all workflows
+-- compile_config.go               # CompileConfig type
+-- compile_batch_operations.go     # Batch compilation
+-- compile_watch.go                # File watching
+-- run_command.go (in main.go)     # Run on GitHub Actions
+-- logs_command.go                 # Download/view logs
+-- audit.go                        # Debug failed runs
+-- health_command.go               # Health checks
+-- mcp_command.go                  # MCP management
+-- mcp_server_command.go           # MCP server tools
+-- fix_command.go                  # Auto-fix workflows
+-- secrets_command.go              # Secret management
+-- interactive.go                  # Interactive workflow builder
+-- flags.go                        # Shared flag helpers
+-- commands.go                     # Shared command utilities
```

#### pkg/workflow/ - Core Compilation Engine

```
pkg/workflow/
+-- Compiler types & orchestration
|   +-- compiler.go                  # Compiler struct, main CompileWorkflow()
|   +-- compiler_types.go            # WorkflowData, EngineConfig types
|   +-- compiler_orchestrator_*.go   # Multi-phase orchestration
|
+-- YAML generation
|   +-- compiler_yaml.go             # Generate GitHub Actions YAML
|   +-- compiler_jobs.go             # Build job definitions
|   +-- compiler_steps.go            # Build step definitions
|
+-- Engine implementations
|   +-- agentic_engine.go            # Engine interface (11 sub-interfaces)
|   +-- copilot_engine.go            # GitHub Copilot engine
|   +-- claude_engine.go             # Claude Code engine
|   +-- codex_engine.go              # OpenAI Codex engine
|   +-- custom_engine.go             # Custom engine support
|   +-- engine_helpers.go            # Shared engine utilities
|   +-- engine_registry.go           # Engine lookup registry
|
+-- Safe-outputs system
|   +-- safe_outputs_config.go       # Config parsing
|   +-- safe_outputs_steps.go        # Step builders
|   +-- safe_outputs_jobs.go         # Job assembly
|   +-- safe_output_builder.go       # Target/filter config
|   +-- compiler_safe_outputs.go     # Compiler integration
|   +-- compiler_safe_output_jobs.go # Job generation
|   +-- create_issue.go              # Issue creation
|   +-- create_pull_request.go       # PR creation
|   +-- create_discussion.go         # Discussion creation
|   +-- (25+ more create_*.go files)
|
+-- MCP server configuration
|   +-- mcp_setup_generator.go       # MCP setup steps
|   +-- mcp_gateway_config.go        # Gateway configuration
|   +-- mcp_config_renderer.go       # Config file generation
|
+-- Validation
|   +-- validation.go                # Core validation
|   +-- strict_mode_validation.go    # Security checks
|   +-- expression_safety.go         # Expression injection
|   +-- expressions.go               # Expression building
|
+-- Data & schemas
    +-- schemas/                      # Embedded JSON schemas
    +-- data/                         # Action pins, defaults
    +-- js/                           # safe_outputs_tools.json
```

#### pkg/parser/ - Frontmatter Parsing

```
pkg/parser/
+-- frontmatter_content.go   # Core YAML extraction
+-- frontmatter_hash.go      # Frontmatter hashing
+-- import_processor.go       # Import resolution (BFS)
+-- schema_compiler.go        # JSON schema validation
+-- schemas/                  # Embedded JSON schemas
    +-- main_workflow_schema.json  (318KB)
    +-- mcp_config_schema.json
```

#### pkg/console/ - Output Formatting

```
pkg/console/
+-- console.go       # FormatError, FormatSuccessMessage, etc.
+-- render.go        # Struct rendering with tags
+-- input.go         # User prompts
+-- select.go        # Selection prompts
+-- form.go          # Form building
+-- confirm.go       # Confirmation dialogs
+-- progress.go      # Progress tracking
+-- spinner.go       # Spinner animation
+-- table.go         # Table rendering
```

---

## Engine Architecture

Each AI engine implements a set of interfaces. The system uses **interface
segregation** (many small interfaces) rather than one large interface.

### Interface Hierarchy

```
+---------------------+
|     Engine          |  (identity)
| - GetID()           |
| - GetDisplayName()  |
| - GetDescription()  |
| - IsExperimental()  |
+---------------------+
         ^
         |
+---------------------+
| CapabilityProvider  |  (feature detection)
| - SupportsToolsAllowlist()   |
| - SupportsHTTPTransport()    |
| - SupportsMaxTurns()         |
| - SupportsWebFetch()         |
| - SupportsFirewall()         |
| - SupportsPlugins()          |
+---------------------+
         ^
         |
+---------------------+
|  WorkflowExecutor   |  (compilation)
| - GetInstallationSteps()     |
| - GetExecutionSteps()        |
| - GetDeclaredOutputFiles()   |
+---------------------+
         ^
         |
    (optional interfaces)
         |
+---------------------+   +---------------------+
| MCPConfigProvider   |   | SecurityProvider    |
| - RenderMCPConfig() |   | - GetDefaultDetectionModel()  |
+---------------------+   | - GetRequiredSecretNames()    |
                           +---------------------+

All composed into:
+---------------------+
| CodingAgentEngine   |  (backward-compatible composite)
+---------------------+
```

### Engine Registry Flow

```
main.go                    pkg/workflow/
+-----+                    +------------------+
| CLI | -- engine:"copilot" --> | EngineRegistry   |
+-----+                    | engines map      |
                           |  "copilot" -> CopilotEngine  |
                           |  "claude"  -> ClaudeEngine    |
                           |  "codex"   -> CodexEngine     |
                           |  "custom"  -> CustomEngine    |
                           +------------------+
                                    |
                                    v
                           +------------------+
                           | Selected Engine  |
                           | .GetInstallationSteps() |
                           | .GetExecutionSteps()    |
                           | .RenderMCPConfig()      |
                           +------------------+
```

### How Engines Generate Steps

Each engine produces GitHub Actions steps as arrays of YAML strings:

```
CopilotEngine.GetExecutionSteps(workflowData)
  |
  v
Returns: []GitHubActionStep{
  ["- name: \"Run Copilot Agent\"",
   "  run: |",
   "    copilot-cli agent --prompt $PROMPT",
   "  env:",
   "    COPILOT_GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}"],
}
  |
  v
Compiler inserts these steps into the agent job YAML
```

**Key files:**
- `pkg/workflow/agentic_engine.go` - Interface definitions
- `pkg/workflow/copilot_engine.go` - Copilot implementation
- `pkg/workflow/claude_engine.go` - Claude implementation
- `pkg/workflow/codex_engine.go` - Codex implementation
- `pkg/workflow/engine_helpers.go` - Shared utilities

---

## Safe-Outputs System

Safe-outputs is the security layer that controls all write operations. The AI
agent cannot directly modify GitHub resources. Instead, it writes structured
output that the safe-outputs job processes through validated handlers.

### Safe-Outputs Flow

```
+-------------------+
| AI Agent          |  Runs in the "agent" job
| (Copilot/Claude)  |
+--------+----------+
         |
         | Writes structured output (JSON)
         | to /tmp/gh-aw/agent/safe-outputs.json
         v
+-------------------+     +---------------------+
| Safe-Outputs MCP  |     | Safe-Inputs MCP     |
| Server (Node.js)  |     | Server (Node.js)    |
| Port 3001         |     | Port 3000           |
+--------+----------+     +---------------------+
         |
         | Validated output passed to safe_outputs job
         v
+----------------------------------------------+
| Safe-Outputs Job (GitHub Actions)            |
|                                              |
| For each configured output:                  |
|                                              |
|  create-issue:                               |
|    +-- parse output JSON                     |
|    +-- validate against limits (max: 5)      |
|    +-- sanitize content                      |
|    +-- call GitHub API via github-script     |
|                                              |
|  add-labels:                                 |
|    +-- parse output JSON                     |
|    +-- validate label names                  |
|    +-- call GitHub API                       |
|                                              |
|  add-comment:                                |
|    +-- parse output JSON                     |
|    +-- validate length (65,536 char max)     |
|    +-- sanitize mentions (max 10)            |
|    +-- call GitHub API                       |
+----------------------------------------------+
```

### Supported Safe-Output Types

Each type has its own implementation file in `pkg/workflow/`:

```
pkg/workflow/create_issue.go             - Create GitHub issues
pkg/workflow/create_pull_request.go      - Create pull requests
pkg/workflow/create_discussion.go        - Create discussions
pkg/workflow/create_code_scanning_alert.go - Code scanning
pkg/workflow/create_agent_task.go        - Agent tasks

(Runtime JavaScript handlers in actions/setup/js/):
add_comment.cjs                          - Add comments
add_labels.cjs                           - Add labels
add_reaction.cjs                         - Add reactions
add_reviewer.cjs                         - Add PR reviewers
assign_issue.cjs                         - Assign issues
close_issue.cjs                          - Close issues
close_pull_request.cjs                   - Close PRs
update_issue.cjs                         - Update issues
update_pr.cjs                            - Update PRs
update_discussion.cjs                    - Update discussions
```

---

## MCP Server Architecture

MCP (Model Context Protocol) is how AI agents communicate with external tools.
gh-aw sets up multiple MCP servers during workflow execution.

### MCP Server Topology

```
+-----------------------------------------------------------+
|  GitHub Actions Runner                                     |
|                                                            |
|  +------------------+     +--------------------+           |
|  | AI Agent         |     | MCP Gateway        |           |
|  | (Copilot/Claude) | <-> | (Port 80)          |           |
|  +------------------+     +----+----+----+-----+           |
|                                |    |    |                  |
|          +---------+-----------+    |    +-------+          |
|          |         |                |            |          |
|          v         v                v            v          |
|  +------------+ +------------+ +----------+ +----------+   |
|  | GitHub MCP | | Safe-Out   | | Safe-In  | | Playwright|  |
|  | Server     | | MCP Server | | MCP Srv  | | MCP Srv  |   |
|  | (remote or | | (Port 3001)| | (Port    | | (Docker) |   |
|  |  Docker)   | | (Node.js)  | |  3000)   | |          |   |
|  +-----+------+ +-----+------+ +----+-----+ +----+-----+   |
|        |               |             |            |          |
|        v               v             v            v          |
|  GitHub API      Agent Output   Custom Tools   Browsers     |
+-----------------------------------------------------------+
```

### MCP Server Types

| Server | Purpose | Transport | Port |
|--------|---------|-----------|------|
| **GitHub MCP** | GitHub API access (issues, PRs, repos) | HTTP or Docker | Remote |
| **Safe-Outputs** | Validated write operations | HTTP (Node.js) | 3001 |
| **Safe-Inputs** | Custom tool definitions | HTTP (Node.js) | 3000 |
| **Playwright** | Browser automation | Docker | Container |
| **Serena** | Code search/analysis | Docker | Container |
| **MCP Gateway** | Unified proxy for all servers | HTTP | 80 |
| **Custom** | User-defined MCP servers | HTTP/stdio | Custom |

### MCP Configuration Generation

```
Frontmatter:                       Generated MCP Config:
+------------------------+        +---------------------------+
| tools:                 |        | {                         |
|   github:              | -----> |   "mcpServers": {         |
|     toolsets: [issues] |        |     "github": {           |
|   bash:                |        |       "type": "http",     |
|     - "jq *"           |        |       "url": "...",       |
|   playwright:          |        |       "tools": ["*"]      |
|     version: v1.41.0   |        |     },                    |
+------------------------+        |     "safe-outputs": {     |
                                  |       "type": "http",     |
                                  |       "url": "localhost:3001"|
                                  |     },                    |
                                  |     "playwright": {       |
                                  |       "type": "http",     |
                                  |       "url": "..."        |
                                  |     }                     |
                                  |   }                       |
                                  | }                         |
                                  +---------------------------+
```

**Key files:**
- `pkg/workflow/mcp_setup_generator.go` - Generates MCP setup steps
- `pkg/workflow/mcp_gateway_config.go` - Gateway configuration
- `pkg/workflow/mcp_config_renderer.go` - Config file rendering

---

## Runtime Execution Flow

When a workflow runs on GitHub Actions, this is the complete execution sequence:

```
                    GitHub Actions Trigger
                    (issue opened, PR created, etc.)
                              |
                              v
                  +------------------------+
                  | ACTIVATION JOB         |
                  |                        |
                  | 1. Copy JS/Shell files |
                  |    from actions/setup/ |
                  |    to /tmp/gh-aw/      |
                  |                        |
                  | 2. check_permissions   |
                  |    .cjs                |
                  |    (verify user access) |
                  |                        |
                  | 3. check_rate_limit    |
                  |    .cjs                |
                  |    (GitHub API limits) |
                  |                        |
                  | 4. check_stop_time     |
                  |    .cjs                |
                  |    (time budget)       |
                  |                        |
                  | 5. check_skip_if_match |
                  |    .cjs                |
                  |    (conditional skip)  |
                  +----------+-------------+
                             |
                             | (all checks pass)
                             v
                  +------------------------+
                  | AGENT JOB              |
                  |                        |
                  | 1. Install gh-aw CLI   |
                  |    (install-gh-aw.sh)  |
                  |                        |
                  | 2. Setup Node/Python   |
                  |    runtime             |
                  |                        |
                  | 3. Download Docker     |
                  |    images (if needed)  |
                  |                        |
                  | 4. Start MCP servers   |
                  |    - Safe-outputs      |
                  |    - Safe-inputs       |
                  |    - GitHub MCP        |
                  |    - MCP Gateway       |
                  |                        |
                  | 5. Build AI prompt     |
                  |    from markdown body  |
                  |                        |
                  | 6. RUN AI ENGINE       |
                  |    (copilot/claude/    |
                  |     codex/custom)      |
                  |                        |
                  | 7. Save agent output   |
                  +----------+-------------+
                             |
                             v
                  +------------------------+
                  | SAFE-OUTPUTS JOB       |
                  |                        |
                  | 1. Parse agent output  |
                  |    (parse_copilot.cjs  |
                  |     or parse_claude.cjs)|
                  |                        |
                  | 2. For each operation: |
                  |    a. Validate limits  |
                  |    b. Sanitize content |
                  |    c. Execute via      |
                  |       GitHub API       |
                  |                        |
                  | 3. Report results      |
                  +----------+-------------+
                             |
                             v
                  +------------------------+
                  | CONCLUSION JOB         |
                  | (status reporting)     |
                  +------------------------+
```

### Runtime File Structure

```
/tmp/gh-aw/                    (Created at runtime)
+-- actions/                    (Copied from actions/setup/)
|   +-- *.cjs                   (JavaScript handlers)
|   +-- *.sh                    (Shell scripts)
+-- agent/                      (Agent workspace)
|   +-- safe-outputs.json       (Agent output)
|   +-- logs/                   (Execution logs)
+-- safe-inputs/                (Safe-inputs server)
|   +-- logs/                   (Server logs)
+-- safe-outputs/               (Safe-outputs server)
|   +-- config.json             (Server configuration)
|   +-- tools.json              (Tool definitions)
|   +-- validation.json         (Validation rules)
+-- cache-memory/               (Agent memory/context)
+-- sandbox/                    (Sandboxed execution)
```

---

## Data Flow Diagrams

### Compilation Data Flow

```
my-workflow.md
     |
     | (raw text)
     v
ExtractFrontmatterFromContent()    pkg/parser/frontmatter_content.go
     |
     | FrontmatterResult{map[string]any, markdownBody}
     v
ProcessImports()                   pkg/parser/import_processor.go
     |
     | ImportsResult{mergedTools, mergedSafeOutputs, ...}
     v
ParseFrontmatterConfig()           pkg/workflow/frontmatter_types.go
     |
     | FrontmatterConfig{Tools, SafeOutputs, Network, Engine, ...}
     v
BuildWorkflowData()                pkg/workflow/compiler.go
     |
     | WorkflowData{all config + engine + tools + markdown}
     v
ValidateWorkflowData()             pkg/workflow/validation.go
     |
     | errors or nil
     v
GenerateYAML()                     pkg/workflow/compiler_yaml.go
     |
     | string (YAML content)
     v
WriteFile()                        pkg/workflow/compiler.go
     |
     v
my-workflow.lock.yml
```

### Command Dispatch Flow

```
User types: gh aw compile --strict my-workflow

     +----------------+
     | main.go        |
     | rootCmd.Execute |
     +-------+--------+
             |
             v
     +----------------+
     | compileCmd     |
     | .RunE          |
     +-------+--------+
             |
             v
     +-------------------+
     | cli.CompileWorkflows |
     | (pkg/cli/compile_   |
     |  orchestrator.go)   |
     +-------+-------------+
             |
             | For each .md file:
             v
     +-------------------+
     | compiler.          |
     | CompileWorkflow()  |
     | (pkg/workflow/     |
     |  compiler.go)      |
     +-------------------+
```

### Interactive Mode Flow

```
User types: gh aw new

     +------------------+
     | newCmd.RunE      |
     +--------+---------+
              |
              v
     +------------------+
     | CreateWorkflow   |
     | Interactively()  |
     | (pkg/cli/        |
     |  interactive.go) |
     +--------+---------+
              |
              | Uses charmbracelet/huh for prompts:
              v
     +---------------------------+
     | 1. What trigger?          |
     |    [issues, PR, schedule] |
     |                           |
     | 2. Which AI engine?       |
     |    [copilot, claude, ...] |
     |                           |
     | 3. Which tools?           |
     |    [github, bash, ...]    |
     |                           |
     | 4. Which safe-outputs?    |
     |    [create-issue, ...]    |
     |                           |
     | 5. Network access?        |
     |    [defaults, ecosystem]  |
     +---------------------------+
              |
              | Generates .md file
              | Compiles to .lock.yml
              v
     +---------------------------+
     | Workflow ready to use!    |
     +---------------------------+
```

---

## File Organization

### Repository Root

```
gh-aw/
+-- cmd/
|   +-- gh-aw/
|       +-- main.go              # CLI entry point (700 lines)
|
+-- pkg/                          # Go library packages
|   +-- cli/                      # ~200 files, command implementations
|   +-- workflow/                  # ~150 files, compilation engine
|   +-- parser/                   # ~20 files, frontmatter parsing
|   +-- console/                  # ~15 files, output formatting
|   +-- constants/                # Semantic types and constants
|   +-- logger/                   # Debug logging
|   +-- fileutil/                 # File path security
|   +-- stringutil/               # String manipulation
|   +-- gitutil/                  # Git helpers
|   +-- envutil/                  # Environment variable parsing
|   +-- repoutil/                 # Repository slug parsing
|   +-- timeutil/                 # Duration formatting
|   +-- sliceutil/                # Generic slice operations
|   +-- mathutil/                 # Min/Max helpers
|   +-- tty/                      # Terminal detection
|   +-- styles/                   # Adaptive color themes
|   +-- testutil/                 # Test helpers
|   +-- types/                    # Shared MCP types
|
+-- actions/                      # GitHub Actions (runtime)
|   +-- setup/
|   |   +-- js/                   # ~117 JavaScript files (.cjs)
|   |   +-- sh/                   # ~31 shell scripts
|   |   +-- md/                   # Prompt templates
|   |   +-- action.yml            # Action metadata
|   |   +-- setup.sh              # Main setup script
|   +-- setup-cli/
|       +-- install.sh            # CLI installer
|       +-- action.yml            # Action metadata
|
+-- copilot-client/               # TypeScript Copilot SDK client
|   +-- src/                      # TypeScript source
|   +-- dist/                     # Bundled output (~190KB)
|   +-- package.json              # ESM module, Node 24+
|
+-- internal/tools/               # Build tools
|   +-- actions-build/            # Action bundler
|   +-- generate-action-metadata/ # Metadata generator
|
+-- .github/
|   +-- workflows/                # Sample + CI workflows (.md + .lock.yml)
|   +-- aw/                       # Agentic workflow configs
|   +-- agents/                   # Agent definitions
|   +-- copilot/instructions/     # Copilot performance guidelines
|
+-- docs/                         # Astro documentation site
+-- Makefile                      # 75+ build targets
+-- go.mod                        # Go 1.25.0
+-- tools.go                      # Build tool dependencies
```

---

## Technology Stack

| Technology | Purpose | Version |
|------------|---------|---------|
| **Go** | Main language for CLI and compiler | 1.25.0 |
| **Cobra** | CLI framework (commands, flags, help) | spf13/cobra |
| **goccy/go-yaml** | YAML parsing (GitHub Actions compat) | 1.19.2 |
| **jsonschema v6** | Frontmatter schema validation | santhosh-tekuri |
| **lipgloss** | Terminal styling and themes | charmbracelet |
| **bubbletea/huh** | Interactive TUI prompts | charmbracelet |
| **Node.js** | Runtime JavaScript handlers | 20+ |
| **TypeScript** | Copilot SDK client | ESM/Node 24+ |
| **Docker** | MCP server containers (Playwright, etc.) | Runtime |
| **GitHub Actions** | Workflow execution platform | Runtime |
| **MCP** | Model Context Protocol for tool communication | JSON-RPC |

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Go for compiler | Fast compilation, single binary, cross-platform |
| JavaScript for runtime | GitHub Actions scripts use Node.js natively |
| `map[string]any` for frontmatter | Dynamic YAML structure, validated post-parse |
| Interface segregation for engines | Optional capabilities, backward compatibility |
| BFS for import resolution | Deterministic merge ordering |
| Embedded schemas (`//go:embed`) | Zero-configuration validation, no external files |
| SHA-pinned actions | Supply chain security |
| Safe-outputs pattern | Security boundary between AI and GitHub API |
| MCP protocol | Standard tool communication across AI engines |
