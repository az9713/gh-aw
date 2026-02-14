# Study Plan: Mastering GitHub Agentic Workflows (gh-aw)

A systematic, zero-to-hero learning plan for understanding the inner workings of
gh-aw. Each module pairs theory with implementation. Read the referenced files,
trace the code, and complete the exercises before moving on.

**Prerequisites assumed**: Basic programming experience (C, C++, or Java is fine).
No Go or web development experience required.

---

## Table of Contents

- [Phase 1 - Foundations](#phase-1---foundations)
- [Phase 2 - The Big Picture](#phase-2---the-big-picture)
- [Phase 3 - The CLI Layer](#phase-3---the-cli-layer)
- [Phase 4 - The Parser (Compiler Front-End)](#phase-4---the-parser-compiler-front-end)
- [Phase 5 - The Compiler Core (Middle and Back-End)](#phase-5---the-compiler-core-middle-and-back-end)
- [Phase 6 - The Engine Architecture](#phase-6---the-engine-architecture)
- [Phase 7 - The Safe-Outputs System](#phase-7---the-safe-outputs-system)
- [Phase 8 - MCP and Runtime Execution](#phase-8---mcp-and-runtime-execution)
- [Phase 9 - Utilities and Supporting Packages](#phase-9---utilities-and-supporting-packages)
- [Phase 10 - Testing and Quality](#phase-10---testing-and-quality)
- [Phase 11 - Security Architecture](#phase-11---security-architecture)
- [Phase 12 - Mastery Exercises](#phase-12---mastery-exercises)

---

## Phase 1 - Foundations

**Goal**: Get comfortable with the technologies this project uses before touching
any gh-aw code.

### Module 1.1 - Go Language Basics

If you come from C/C++/Java, Go will feel familiar but simpler. Focus on the
differences.

**Theory to learn** (use the official Go tour at https://go.dev/tour/):

| Go Concept | C/C++/Java Equivalent | Key Difference |
|---|---|---|
| `package main` | `public static void main` | Every Go file belongs to a package |
| `func` | function/method | Multiple return values are normal |
| `struct` | class (no inheritance) | Composition over inheritance |
| `interface` | interface (Java) | Implicit satisfaction - no `implements` keyword |
| `goroutine` | thread | Lightweight, thousands are cheap |
| `channel` | blocking queue | Built-in concurrency primitive |
| `defer` | try-finally | Runs when function returns |
| `error` | exception | Errors are values, not exceptions |
| `:=` | auto/var | Short variable declaration |
| `any` | `void*` / `Object` | Type alias for `interface{}` |

**Essential Go patterns used in this project**:

```go
// 1. Error handling (no try-catch in Go)
result, err := doSomething()
if err != nil {
    return fmt.Errorf("failed to do X: %w", err)  // wrap with context
}

// 2. Table-driven tests
tests := []struct {
    name     string
    input    string
    expected string
}{
    {"valid input", "hello", "HELLO"},
    {"empty input", "", ""},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := transform(tt.input)
        assert.Equal(t, tt.expected, got)
    })
}

// 3. Embedding files at compile time
//go:embed schemas/my-schema.json
var mySchema string  // file content becomes a string constant

// 4. Interface satisfaction (implicit)
type Engine interface {
    GetID() string
}
type CopilotEngine struct{}
func (e *CopilotEngine) GetID() string { return "copilot" }
// CopilotEngine satisfies Engine without saying "implements Engine"
```

**Exercise**: Write a Go program that reads a text file, splits it by `---`
delimiters, and prints the section between the first and second `---`. This is
exactly what the gh-aw parser does with frontmatter.

### Module 1.2 - YAML and Markdown

**Theory**:
- YAML is a data serialization format (like JSON but human-readable)
- Markdown is a text formatting language (headings, lists, code blocks)
- gh-aw uses files that combine BOTH: YAML "frontmatter" between `---` delimiters,
  followed by markdown content

**Example gh-aw workflow file** (this is the INPUT to the compiler):

```markdown
---
on:
  issues:
    types: [opened]
engine: copilot
tools:
  github:
    toolsets: [issues]
safe-outputs:
  add-labels:
    max: 5
---

# Issue Triage Agent

Read the new issue and add appropriate labels based on the content.
Look at existing labels in the repository for guidance.
```

**Exercise**: Create a file with YAML frontmatter and markdown body. Identify:
1. Where does the YAML config start and end?
2. What is the `on:` field? (It defines WHEN the workflow triggers)
3. What is the `engine:` field? (It defines WHICH AI model runs)
4. What is the markdown body used for? (Instructions for the AI agent)

### Module 1.3 - GitHub Actions Basics

**Theory**: GitHub Actions is a CI/CD platform that runs workflows in response
to events (push, issue opened, schedule, etc.). Workflows are YAML files in
`.github/workflows/`.

**Key concepts**:

```
Workflow (.yml file)
  └── triggered by: events (push, issues.opened, schedule, etc.)
  └── contains: Jobs
        └── each job has: Steps
              └── each step: runs a command or calls an action
```

**Read**: Pick any `.lock.yml` file in `.github/workflows/` and study:
1. The `on:` section (what triggers it?)
2. The `jobs:` section (what jobs does it define?)
3. The `steps:` within each job (what does each step do?)

**Key insight**: gh-aw GENERATES these `.lock.yml` files from simpler `.md` files.
The `.md` is what humans write; the `.lock.yml` is what GitHub Actions executes.

### Module 1.4 - The Cobra CLI Framework

**Theory**: Cobra is a Go library for building command-line tools. It provides:
- Command hierarchy (root command + subcommands)
- Flag parsing (`--verbose`, `-v`)
- Help text generation
- Argument validation

**Pattern used in gh-aw**:

```go
// Every command follows this pattern:
cmd := &cobra.Command{
    Use:   "compile [workflow]",     // how to invoke it
    Short: "Compile workflows",      // one-line description
    Long:  `Detailed description...`, // full help text
    RunE: func(cmd *cobra.Command, args []string) error {
        // This runs when the user types "gh aw compile"
        return RunCompile(args)
    },
}
cmd.Flags().BoolP("verbose", "v", false, "Enable verbose output")
```

**Read**: `cmd/gh-aw/main.go` lines 1-100 to see how the root command is set up.

---

## Phase 2 - The Big Picture

**Goal**: Understand what gh-aw does end-to-end before diving into code details.

### Module 2.1 - The Compiler Analogy

If you have C/C++ experience, you already understand compilers:

```
C Compiler:                        gh-aw Compiler:
===========                        ===============
Source code (.c)         <--->     Markdown workflow (.md)
Preprocessing (#include) <--->     Import resolution (imports:)
Parsing (AST)            <--->     Frontmatter extraction (YAML -> map)
Semantic analysis        <--->     Validation (schema + security)
Code generation (.o)     <--->     YAML generation (.lock.yml)
Executable (a.out)       <--->     GitHub Actions workflow (runs on cloud)
```

**The four phases of gh-aw compilation**:

```
Phase 1: PARSE          Phase 2: BUILD          Phase 3: VALIDATE       Phase 4: GENERATE
─────────────          ──────────────          ───────────────         ────────────────
Read .md file    --->  Resolve imports   --->  Schema check     --->  Emit .lock.yml
Extract YAML           Merge configs           Engine rules            with activation,
frontmatter            Select engine           Security checks         agent, and
Extract markdown       Build WorkflowData      Expression safety       safe-output jobs
body                   struct
```

**Read these files for the big picture**:
1. `docs/ARCHITECTURE.md` - Full architecture with ASCII diagrams
2. `CLAUDE.md` - Concise project overview and key file paths
3. `README.md` - User-facing explanation

### Module 2.2 - See It In Action

**Exercise**: Trace a real compilation by studying a workflow pair:

1. Pick a `.md` file from `.github/workflows/` (e.g., a simple one)
2. Open its corresponding `.lock.yml` file side by side
3. For each section in the `.lock.yml`, find where it came from:
   - `on:` -> directly from the YAML frontmatter
   - `permissions:` -> generated by the compiler based on tools and outputs
   - `jobs.activation:` -> generated security checks (membership, rate limits)
   - `jobs.agent:` -> generated AI engine execution steps
   - `jobs.safe_outputs:` -> generated from `safe-outputs:` config

**Key insight**: The `.md` file might be 30 lines. The `.lock.yml` might be 300+
lines. The compiler generates all the boilerplate, security checks, and
infrastructure.

### Module 2.3 - The Data Flow

```
                    gh-aw compile
                         |
                         v
    +-------------------------------------------+
    |  .md file                                 |
    |  (YAML frontmatter + markdown body)       |
    +-------------------------------------------+
                         |
              Phase 1: PARSE (pkg/parser/)
                         |
                         v
    +-------------------------------------------+
    |  FrontmatterResult                        |
    |  - Frontmatter: map[string]any            |
    |  - Markdown: string                       |
    +-------------------------------------------+
                         |
              Phase 2: BUILD (pkg/workflow/)
                         |
                         v
    +-------------------------------------------+
    |  WorkflowData                             |
    |  - Engine, Tools, SafeOutputs, Network    |
    |  - Merged imports, resolved configs       |
    +-------------------------------------------+
                         |
              Phase 3: VALIDATE (pkg/workflow/)
                         |
                         v
    +-------------------------------------------+
    |  Validation checks                        |
    |  - JSON Schema, expressions, permissions  |
    |  - Engine-specific rules                  |
    +-------------------------------------------+
                         |
              Phase 4: GENERATE (pkg/workflow/)
                         |
                         v
    +-------------------------------------------+
    |  .lock.yml                                |
    |  (complete GitHub Actions workflow)        |
    +-------------------------------------------+
```

---

## Phase 3 - The CLI Layer

**Goal**: Understand how user commands get dispatched to the compiler.

### Module 3.1 - Command Registration

**Read**: `cmd/gh-aw/main.go`

**What to look for**:
- Lines 77-101: The root command definition (`&cobra.Command{}`)
- Lines 416-435: Command groups (setup, development, execution, analysis, utilities)
- Lines 610-641: How each command is assigned to a group
- Lines 673-700: The `main()` function - program entry point

**Theory**: Cobra uses a tree of commands. The root is `gh aw`. Subcommands like
`compile`, `run`, `init` are children. When you type `gh aw compile`, Cobra
dispatches to the compile command's `RunE` function.

```
gh aw (root)
  ├── init      (setup group)
  ├── new       (setup group)
  ├── compile   (development group)
  ├── run       (execution group)
  ├── logs      (analysis group)
  └── ...
```

### Module 3.2 - The Command Pattern

**Read**: `pkg/cli/add_command.go` (a representative command)

**What to look for**:
- Line 17: Logger creation with namespace `cli:add_command`
- Lines 48-158: `NewAddCommand()` - creates the cobra command with flags
- Lines 123-137: Interactive mode detection (TTY, CI, automation flags)
- Lines 214-283: `AddWorkflows()` - the actual implementation

**The pattern every command follows**:

```
File: pkg/cli/X_command.go

1. var xLog = logger.New("cli:X_command")     // Debug logger
2. func NewXCommand() *cobra.Command { ... }  // Command definition + flags
3. func RunX(config XConfig) error { ... }    // Testable business logic
4. (internal helpers)                          // Private implementation
```

**Exercise**: Pick three `*_command.go` files in `pkg/cli/` and verify they all
follow this pattern. Note the constructor name, runner name, and logger namespace.

### Module 3.3 - The Compile Command (Most Important)

The compile command is the heart of gh-aw. It is split across multiple files:

| File | Responsibility |
|------|---------------|
| `compile_command.go` | Brief docs (logic split into other files) |
| `compile_config.go` | `CompileConfig` struct definition |
| `compile_orchestrator.go` | Main `CompileWorkflows()` entry point |
| `compile_helpers.go` | Utility functions |
| `compile_validation.go` | Validation logic |
| `compile_watch.go` | File-watching mode |

**Read**: `pkg/cli/compile_orchestrator.go` lines 17-91

**Trace the call chain**:
```
User types: gh aw compile
  --> cmd/gh-aw/main.go: compileCmd.RunE()
    --> pkg/cli/compile_orchestrator.go: CompileWorkflows()
      --> createAndConfigureCompiler(config)   // Create compiler instance
      --> compileSpecificFiles() or compileAllFilesInDirectory()
        --> pkg/workflow/compiler.go: CompileWorkflow(markdownPath)
          --> ParseWorkflowFile()              // Phase 1-2: Parse + Build
          --> CompileWorkflowData()            // Phase 3-4: Validate + Generate
```

### Module 3.4 - Console Output

**Read**: `pkg/console/console.go`

**Theory**: All user-facing output goes through console formatters. This ensures
consistent styling and proper output routing (diagnostic -> stderr, data -> stdout).

**Key types**:
- `CompilerError` (lines 21-34): Structured error with file position, like a C compiler error
- `FormatSuccessMessage()`, `FormatErrorMessage()`, etc.: Formatting helpers

**The output format is IDE-parseable**: `file:line:column: error: message`
(just like `gcc` errors - you already know this pattern from C/C++!)

---

## Phase 4 - The Parser (Compiler Front-End)

**Goal**: Understand how `.md` files are parsed into structured data.

### Module 4.1 - Frontmatter Extraction

**Read**: `pkg/parser/frontmatter_content.go` lines 15-85

**Theory**: This is the "lexer/tokenizer" of gh-aw. It splits the file at `---`
delimiters, just like a C preprocessor splits at `#include` boundaries.

**The algorithm**:
```
1. Check if file starts with "---"           (line 30)
2. Find the closing "---"                     (lines 42-47)
3. Extract YAML between the delimiters        (lines 54-69)
4. Parse YAML into map[string]any             (goccy/go-yaml library)
5. Extract markdown content after "---"       (lines 72-76)
6. Return FrontmatterResult struct            (lines 79-84)
```

**Data structure** (lines 15-22):
```go
type FrontmatterResult struct {
    Frontmatter      map[string]any   // Parsed YAML as a Go map
    Markdown         string           // Everything after the second ---
    FrontmatterLines []string         // Raw YAML lines (for error context)
    FrontmatterStart int              // Line number (for error reporting)
}
```

**Exercise**: Write a function that takes a string, finds content between two
`---` lines, and parses it as YAML. Compare your approach with the actual code.

### Module 4.2 - Import Resolution (BFS)

**Read**: `pkg/parser/import_processor.go` lines 108-676

**Theory**: Workflows can import other workflows via the `imports:` field. This
creates a dependency graph. gh-aw resolves it using Breadth-First Search (BFS) -
the same algorithm you learned in data structures class.

**Why BFS (not DFS)?**
- BFS gives deterministic, level-order processing
- Parent configs take precedence over child configs
- Import order matches declaration order

**The BFS algorithm**:
```
Input: A workflow with imports: [a.md, b.md]
       a.md imports: [c.md]
       b.md imports: [c.md, d.md]

Queue:  [main] -> [a, b] -> [c, d] -> []
Visit:  main -> a -> b -> c -> d
```

```
1. Initialize queue with direct imports from frontmatter  (lines 164-268)
2. Initialize visited set for cycle detection              (line 165)
3. While queue is not empty:                               (lines 270-641)
   a. Dequeue first item (FIFO)                            (line 273)
   b. Skip if already visited (cycle detection)
   c. Read the imported file
   d. Extract its frontmatter
   e. Merge its configs (tools, engines, safe-outputs)
   f. Find its own imports and add to queue
4. Topologically sort the result (Kahn's algorithm)        (lines 681-789)
```

**Key data structures**:
- `importQueueItem` (lines 86-92): Queue entry with path, section, inputs
- `ImportsResult` (lines 16-47): Accumulated result with all merged configs

**Exercise**: Draw the dependency graph for a workflow that imports three files,
where two of them share a common import. Trace the BFS processing order.

### Module 4.3 - Schema Validation

**Read**: `pkg/parser/schema_validation.go` lines 54-75

**Theory**: After parsing, the frontmatter is validated against a JSON Schema.
This catches typos and invalid configurations at compile time (like a type checker
in a compiler).

**How it works**:
```
1. JSON schemas are embedded in the binary via //go:embed    (line 21)
2. Schemas are compiled once using sync.Once pattern         (lines 41-46)
3. Frontmatter map is validated against the compiled schema  (line 68)
4. Engine-specific rules are applied on top                  (line 74)
```

**Also read**: `pkg/parser/schemas/` directory to see the actual JSON schema files.

**Analogy**: This is like a C compiler checking that you don't pass a `char*` to
a function expecting `int`. The schema defines what fields are valid and what
types they must be.

---

## Phase 5 - The Compiler Core (Middle and Back-End)

**Goal**: Understand how parsed data becomes a GitHub Actions YAML file.

### Module 5.1 - The WorkflowData Structure

**Read**: `pkg/workflow/compiler_types.go` lines 378-451

**Theory**: `WorkflowData` is the "Abstract Syntax Tree" (AST) of gh-aw. Just
like a C compiler builds an AST from tokens, gh-aw builds WorkflowData from
parsed frontmatter.

**Key fields**:
```go
type WorkflowData struct {
    // Identity
    Name, ID, Source string

    // Configuration
    Engine       string              // "copilot", "claude", "codex", "custom"
    EngineConfig *EngineConfig       // Extended engine settings
    On           map[string]any      // Event triggers

    // Tools and integrations
    Tools        map[string]any      // Tool configurations
    ParsedTools  *Tools              // Strongly-typed tool config

    // Security
    Permissions       map[string]any       // GitHub token permissions
    SafeOutputs       *SafeOutputsConfig   // Output route configuration
    NetworkPermissions *NetworkPermissions  // Domain allowlists

    // Content
    Markdown string                  // AI agent instructions
    Steps    []map[string]any        // Custom workflow steps
}
```

### Module 5.2 - The Orchestration Pipeline

**Read**: `pkg/workflow/compiler_orchestrator_workflow.go` lines 17-97

This is where Phase 1 and Phase 2 happen in sequence:

```
ParseWorkflowFile(markdownPath)
  |
  +--> parseFrontmatterSection()           // Extract YAML from .md
  +--> setupEngineAndImports()             // Detect engine, resolve imports
  +--> processToolsAndMarkdown()           // Configure tools, get markdown
  +--> buildInitialWorkflowData()          // Create WorkflowData struct
  +--> extractYAMLSections()               // Extract on:, permissions:, etc.
  +--> processAndMergeSteps()              // Merge custom steps
  +--> extractAdditionalConfigurations()   // Cache, safe-inputs, safe-outputs
  +--> processOnSectionAndFilters()        // Event triggers and filters
```

**Exercise**: Set `DEBUG=workflow:* ./gh-aw compile some-workflow.md` and trace
the log output through these stages.

### Module 5.3 - Validation

**Read**: `pkg/workflow/compiler.go` lines 116-371 (validation functions)

After `WorkflowData` is built, it goes through extensive validation:

| Validation | What It Checks | File |
|---|---|---|
| Expression safety | No injection in `${{ }}` expressions | `expression_safety.go` |
| Feature flags | Required features are enabled | `compiler.go:135` |
| Dangerous permissions | No write-all without justification | `compiler.go:155` |
| Agent file | `.github/agents/*.md` exists if referenced | `compiler.go:161` |
| Safe-output targets | Output routes are valid repos | `compiler.go:173` |
| Network domains | Allowlisted domains are valid | `compiler.go:185` |
| GitHub Actions schema | Generated YAML is valid Actions syntax | `schema_validation.go` |
| Container images | Docker images exist and are tagged | `docker_validation.go` |
| Runtime packages | npm/pip packages are valid | `npm_validation.go`, `pip_validation.go` |

**Analogy**: This is the semantic analysis phase of a compiler. The syntax is
valid (schema check), but does it make logical sense?

### Module 5.4 - YAML Generation

**Read**: `pkg/workflow/compiler_yaml.go` lines 20-237

This is the "code generation" phase - the final output.

**The generated `.lock.yml` has this structure**:

```yaml
# Header (comments with source info, hash, imports list)
name: workflow-name
on: { ... }              # From frontmatter
permissions: { ... }     # Computed from tools + safe-outputs

jobs:
  activation:            # Generated: security checks
    steps:
      - Check membership
      - Check rate limits
      - Check time limits

  agent:                 # Generated: AI engine execution
    steps:
      - Setup environment
      - Setup MCP servers
      - Run AI engine with prompt

  safe_outputs:          # Generated: controlled write operations
    steps:
      - Create issues
      - Add labels
      - Add comments
```

**Read**: `compiler_yaml.go` line 159 `generateYAML()` and trace:
1. `buildJobsAndValidate()` - Build all jobs
2. `generateWorkflowHeader()` - Emit comments
3. `generateWorkflowBody()` - Emit YAML structure
4. `generatePrompt()` (line 237) - Embed the markdown instructions

---

## Phase 6 - The Engine Architecture

**Goal**: Understand how different AI models are plugged in.

### Module 6.1 - The Interface Hierarchy

**Read**: `pkg/workflow/agentic_engine.go` lines 18-199

**Theory**: gh-aw uses the Interface Segregation Principle (from SOLID design).
Instead of one giant interface, there are focused interfaces that engines can
implement:

```
Engine                    // Identity: GetID(), GetDisplayName()
  |
CapabilityProvider        // Features: SupportsToolsAllowlist(), SupportsFirewall()
  |
WorkflowExecutor          // Actions: GetInstallationSteps(), GetExecutionSteps()
  |
MCPConfigProvider          // AI config: RenderMCPConfig()
  |
SecurityProvider           // Security: GetRequiredSecretNames()
  |
LogParser                  // Analysis: ParseLogMetrics()
  |
CodingAgentEngine          // Composite: combines ALL of the above
```

**Analogy**: In Java, this is like having `Readable`, `Writable`, `Closeable`
interfaces instead of one giant `Resource` interface. Each engine implements
exactly the capabilities it supports.

### Module 6.2 - The Engine Registry

**Read**: `pkg/workflow/agentic_engine.go` lines 297-381

**Theory**: The registry pattern maps engine names to implementations:

```go
type EngineRegistry struct {
    engines map[string]CodingAgentEngine  // "copilot" -> CopilotEngine{}
}
```

Registration happens at initialization (lines 308-324):
```go
registry.Register(NewCopilotEngine())     // "copilot"
registry.Register(NewClaudeEngine())      // "claude"
registry.Register(NewCodexEngine())       // "codex"
registry.Register(NewCustomEngine())      // "custom"
```

### Module 6.3 - A Concrete Engine

**Read**: `pkg/workflow/copilot_engine.go` (the default engine)

**What to look for**:
- How it embeds `BaseEngine` for default behavior
- How `GetInstallationSteps()` generates setup steps
- How `GetExecutionSteps()` generates the AI execution command
- How `RenderMCPConfig()` generates tool configuration

**Exercise**: Compare `copilot_engine.go` and `claude_engine.go`. What is
different? What is shared via `BaseEngine`?

---

## Phase 7 - The Safe-Outputs System

**Goal**: Understand the security layer that controls AI write operations.

### Module 7.1 - Why Safe-Outputs Exist

**Theory**: AI agents can be unpredictable. If you tell an agent "triage this
issue", you don't want it to accidentally close 100 issues or post spam comments.
Safe-outputs is a security layer that:

1. **Limits** how many write operations can happen (e.g., max 5 labels)
2. **Sanitizes** the content (prevent injection)
3. **Routes** operations through controlled handlers
4. **Validates** at compile time that configurations are safe

### Module 7.2 - Configuration Types

**Read**: `pkg/workflow/compiler_types.go` lines 460-511

```go
type SafeOutputsConfig struct {
    CreateIssues          *CreateIssuesConfig       // Create new issues
    AddComments           *AddCommentsConfig        // Comment on issues/PRs
    AddLabels             *AddLabelsConfig          // Add labels
    CreatePullRequests    *CreatePullRequestsConfig // Create PRs
    CreateDiscussions     *CreateDiscussionsConfig  // Create discussions
    DispatchWorkflow      *DispatchWorkflowConfig   // Trigger other workflows
    ThreatDetection       *ThreatDetectionConfig    // Security scanning
    // ... 20+ output types
}
```

### Module 7.3 - A Safe-Output Handler

**Read**: `pkg/workflow/create_issue.go`

**What to look for**:
- `CreateIssuesConfig` struct (lines 12-24): Fields like `Max`, `AllowedLabels`,
  `Assignees`, `Expires`
- `parseIssuesConfig()` (lines 27-77): How frontmatter is parsed into the config
- Default max = 1 if not specified (line 60)
- Validation of target-repo (line 65)

**Exercise**: Read `create_pull_request.go` and `create_discussion.go`. Compare
the patterns. What is shared across all safe-output types?

### Module 7.4 - Config Generation for Runtime

**Read**: `pkg/workflow/safe_outputs_config_generation.go` lines 58-120

At runtime, the safe-outputs configuration is passed to an MCP server as JSON.
The `generateSafeOutputsConfig()` function converts the Go structs into the JSON
format that the runtime JavaScript handlers consume.

---

## Phase 8 - MCP and Runtime Execution

**Goal**: Understand how AI agents communicate with tools at runtime.

### Module 8.1 - What Is MCP?

**Theory**: Model Context Protocol (MCP) is a JSON-RPC protocol that lets AI
models call tools. Think of it as a standardized API between the AI brain and
its hands.

```
+-------------+     JSON-RPC      +-------------+     API calls    +----------+
|  AI Engine  | <--------------> | MCP Server  | <-------------> | GitHub   |
|  (Copilot,  |   "call tool     | (safe-outputs| create issue,  | API      |
|   Claude)   |    create_issue"  |  server)     | add label...  |          |
+-------------+                  +-------------+                 +----------+
```

### Module 8.2 - MCP Server Setup

**Read**: `pkg/workflow/mcp_setup_generator.go` lines 1-60 (doc comment) and
lines 76-150

**The setup sequence** (generated into the `.lock.yml`):
```
1. Download Docker images           (for containerized tools)
2. Install gh-aw extension          (if agentic-workflows tool enabled)
3. Write safe-outputs config files  (config.json, tools.json)
4. Start safe-outputs HTTP server   (port 3001)
5. Write safe-inputs config         (custom tool files)
6. Start safe-inputs HTTP server    (port 3000)
7. Start MCP gateway               (port 80, routes all traffic)
8. Render engine-specific MCP config
```

### Module 8.3 - Runtime JavaScript Handlers

**Read**: `actions/setup/js/check_membership.cjs`

**Theory**: At runtime on GitHub Actions, JavaScript handlers execute the actual
operations. These files are NOT embedded in the Go binary - they are copied to
`/tmp/gh-aw/actions` at runtime by the `actions/setup` composite action.

**The runtime flow**:

```
GitHub Actions Runner
  |
  +--> actions/setup/action.yml (composite action)
  |      |
  |      +--> Copies js/*.cjs and sh/*.sh to /tmp/gh-aw/actions
  |
  +--> Agent job runs AI engine
  |      |
  |      +--> AI calls tools via MCP
  |      +--> MCP gateway routes to safe-outputs server
  |      +--> Safe-outputs server validates and executes
  |
  +--> Safe-outputs job
         |
         +--> Runs JavaScript handlers
         +--> Each handler calls GitHub API with sanitized inputs
```

**Exercise**: Read `actions/setup/js/runtime_import.cjs` lines 48-109. This file
defines the list of allowed GitHub Actions expressions. Compare it with
`pkg/constants/constants.go` - they must stay in sync.

---

## Phase 9 - Utilities and Supporting Packages

**Goal**: Understand the small but important utility packages.

### Module 9.1 - The Logger

**Read**: `pkg/logger/logger.go`

**Theory**: Debug logging uses namespaces with pattern matching (inspired by the
npm `debug` package).

```bash
DEBUG=*                    # See everything
DEBUG=cli:*                # Only CLI commands
DEBUG=workflow:compiler    # Only the compiler
DEBUG=*,-parser:*          # Everything except parser
```

Each logger gets a unique color based on its namespace hash. Time deltas between
log calls are shown (e.g., `+5ms`) for performance debugging.

### Module 9.2 - The Constants Package (Semantic Types)

**Read**: `pkg/constants/constants.go`

**Theory**: Instead of using raw `string` and `int` everywhere, gh-aw defines
semantic type aliases:

```go
type EngineName string     // "copilot", "claude", "codex", "custom"
type WorkflowID string     // "my-workflow" (basename without .md)
type JobName string        // "agent", "activation"
type StepID string         // "check_membership", "run_agent"
type Version string        // "v0.37.18"
type LineLength int        // 120 (max expression line length)
```

Each type has `String()` and `IsValid()` methods. This prevents bugs like
accidentally passing a `JobName` where a `StepID` is expected.

**Analogy**: In C, this is like using `typedef` to distinguish `user_id_t` from
`product_id_t` even though both are `int`.

### Module 9.3 - The Console Package

**Read**: `pkg/console/console.go` and `pkg/console/format.go`

**Rules**:
- All diagnostic output goes to `stderr` (not `stdout`)
- All output uses `console.Format*` functions for consistent styling
- Error format is IDE-parseable: `file:line:column: error: message`
- Colors are only applied when output is a TTY (not piped)

---

## Phase 10 - Testing and Quality

**Goal**: Understand how the project ensures correctness.

### Module 10.1 - Testing Patterns

**Read**: Any `*_test.go` file in `pkg/workflow/`

**Patterns used**:

1. **Table-driven tests** - The primary pattern:
```go
tests := []struct {
    name     string
    input    string
    expected string
    wantErr  bool
}{
    {"valid", "good input", "expected output", false},
    {"empty", "", "", true},
}
```

2. **Build tags** - Every test file starts with:
```go
//go:build !integration   // Unit test (runs with make test-unit)
// OR
//go:build integration    // Integration test (runs with make test)
```

3. **Assertion libraries**:
```go
require.NotNil(t, result)        // Stops test if nil (critical)
assert.Equal(t, expected, got)   // Continues test if not equal (validation)
assert.NoError(t, err)           // Continues test if error
```

### Module 10.2 - Running Tests

```bash
# Fastest: run one specific test
go test -v -run "TestCompileWorkflow" ./pkg/workflow/

# Fast: run tests matching a pattern
go test -v -run "TestCompile.*" ./pkg/workflow/

# Medium: all unit tests
make test-unit

# Slow: full suite including integration
make test

# Before committing (MANDATORY)
make agent-finish
```

**Exercise**: Run a single test with `-v` flag and study the output. Then run
it with `DEBUG=workflow:*` to see debug log output alongside test output.

---

## Phase 11 - Security Architecture

**Goal**: Understand the defense-in-depth approach.

### Module 11.1 - Security Layers

```
Layer 1: COMPILE-TIME VALIDATION
  ├── JSON Schema validation (catches invalid config)
  ├── Expression safety (prevents ${{ }} injection)
  ├── Permission validation (minimum required permissions)
  └── Domain allowlisting (network restrictions)

Layer 2: ACTIVATION CHECKS (runtime, before agent runs)
  ├── Membership verification (is actor authorized?)
  ├── Rate limiting (prevent abuse)
  ├── Time-based controls (stop-after duration)
  └── Bot verification (is this an allowed bot?)

Layer 3: SAFE-OUTPUTS (runtime, controls what agent writes)
  ├── Max operation limits (e.g., max 5 issues)
  ├── Content sanitization (prevent XSS, injection)
  ├── Target validation (correct repo, correct labels)
  └── Expiration controls (auto-close old items)

Layer 4: NETWORK ISOLATION (runtime, controls what agent accesses)
  ├── Domain allowlisting (only approved domains)
  ├── MCP gateway (all tool traffic goes through proxy)
  └── Sandbox/firewall (AWF network egress control)
```

**Read**: `pkg/workflow/expression_safety.go` to see how `${{ }}` expressions
are validated against an allowlist.

**Read**: `actions/setup/js/check_membership.cjs` to see how runtime permission
checks work.

---

## Phase 12 - Mastery Exercises

**Goal**: Prove you understand the system by extending it.

### Exercise 12.1 - Trace a Full Compilation

Pick a real workflow `.md` file and trace every step of compilation by hand:

1. Identify the frontmatter fields
2. Determine what imports are needed
3. Draw the import dependency graph
4. Determine which engine is selected
5. List what safe-outputs are configured
6. Predict what the generated `.lock.yml` will contain
7. Compare your prediction with the actual `.lock.yml`

### Exercise 12.2 - Add a New Safe-Output Type

Imagine you need to add `create-wiki-page` as a new safe-output type:

1. What file would you create? (Follow the `create_*.go` pattern)
2. What config struct would you define?
3. What validation would you add?
4. What runtime JavaScript handler would you need?
5. What JSON schema changes are needed?

Study the existing `create_issue.go` as your template.

### Exercise 12.3 - Add a New CLI Command

Create a hypothetical `gh aw stats` command that shows workflow statistics:

1. Create `pkg/cli/stats_command.go` following the command pattern
2. Create `NewStatsCommand()` and `RunStats()`
3. Add flags: `--json`, `--verbose`
4. Register it in `cmd/gh-aw/main.go`
5. Write table-driven tests in `stats_command_test.go`

### Exercise 12.4 - Add a New Engine

Imagine adding a "gemini" engine:

1. Create `pkg/workflow/gemini_engine.go`
2. Embed `BaseEngine` for defaults
3. Implement the `CodingAgentEngine` interface
4. Register in the engine registry
5. Add constants in `pkg/constants/constants.go`

Study `claude_engine.go` as your template (simpler than copilot).

### Exercise 12.5 - The Full Stack Trace

The ultimate test of understanding. Start from the user typing:

```bash
gh aw compile my-workflow
```

And trace the execution through EVERY layer:

```
main() in cmd/gh-aw/main.go
  -> Cobra dispatches to compileCmd.RunE
    -> CompileWorkflows() in pkg/cli/compile_orchestrator.go
      -> createAndConfigureCompiler()
        -> workflow.NewCompiler()
      -> compileSpecificFiles()
        -> compiler.CompileWorkflow("my-workflow.md")
          -> ParseWorkflowFile()
            -> parser.ExtractFrontmatterFromContent()
            -> parser.ProcessImportsFromFrontmatterWithManifest()
            -> engine detection via ExtractEngineConfig()
            -> tool processing
            -> WorkflowData construction
          -> CompileWorkflowData()
            -> validateWorkflowData()
              -> validateExpressionSafety()
              -> ValidatePermissions()
              -> (many more validators)
            -> generateAndValidateYAML()
              -> generateYAML()
                -> buildJobsAndValidate()
                -> generateWorkflowHeader()
                -> generateWorkflowBody()
                  -> generateMCPSetup()
                  -> engine.GetExecutionSteps()
                  -> generatePrompt()
                  -> generate safe-output jobs
              -> validateGitHubActionsSchema()
            -> writeWorkflowOutput()
              -> Write .lock.yml to disk
```

If you can trace this entire flow and explain what each function does, you have
mastered gh-aw.

---

## Recommended Reading Order Summary

For each phase, read the files in this order:

| Phase | Files to Read |
|-------|--------------|
| 1 | Go Tour, YAML spec, GitHub Actions docs |
| 2 | `README.md`, `docs/ARCHITECTURE.md`, `CLAUDE.md` |
| 3 | `cmd/gh-aw/main.go`, `pkg/cli/compile_orchestrator.go`, `pkg/console/console.go` |
| 4 | `pkg/parser/frontmatter_content.go`, `pkg/parser/import_processor.go`, `pkg/parser/schema_validation.go` |
| 5 | `pkg/workflow/compiler_types.go`, `pkg/workflow/compiler_orchestrator_workflow.go`, `pkg/workflow/compiler.go`, `pkg/workflow/compiler_yaml.go` |
| 6 | `pkg/workflow/agentic_engine.go`, `pkg/workflow/copilot_engine.go`, `pkg/workflow/claude_engine.go` |
| 7 | `pkg/workflow/create_issue.go`, `pkg/workflow/safe_outputs_config_generation.go` |
| 8 | `pkg/workflow/mcp_setup_generator.go`, `actions/setup/js/check_membership.cjs` |
| 9 | `pkg/logger/logger.go`, `pkg/constants/constants.go` |
| 10 | Any `*_test.go`, `Makefile` (test targets) |
| 11 | `pkg/workflow/expression_safety.go`, `actions/setup/js/runtime_import.cjs` |
| 12 | (Exercises - no new reading, apply everything) |
