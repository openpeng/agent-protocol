# Multi-Subagent Example: Code Auditor

A comprehensive example demonstrating multi-subagent composition with an orchestrator pattern.

## Architecture

```
code-auditor/
├── agent.json           # Main agent definition
├── orchestrator.yaml    # Entry point - coordinates workflow
├── security.yaml        # Security analysis subagent
├── performance.yaml     # Performance analysis subagent
├── quality.yaml         # Code quality subagent
└── README.md           # This file
```

## How it works

### Orchestrator Pattern

The **orchestrator** is the entry point that:
1. Finds all source files
2. Runs each specialist subagent
3. Combines results into a single report

### Specialist Subagents

Each subagent focuses on one dimension:
- **security.yaml** - OWASP Top 10 vulnerabilities
- **performance.yaml** - Bottlenecks and optimization opportunities
- **quality.yaml** - Code smells and best practices

## Usage

```bash
agent-deploy run ./code-auditor --args timestamp="2026-06-07 11:00"
```

**Output**: `audit-report.md` with all findings

## Workflow Diagram

```
┌─────────────────────────────────────────────────┐
│         Orchestrator (orchestrator.yaml)        │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. Find files (glob src/**/*.{ts,js})         │
│       ↓                                         │
│  2. Count files                                 │
│       ↓                                         │
│  ┌───────────┬───────────────┬──────────────┐  │
│  │           │               │              │  │
│  ▼           ▼               ▼              │  │
│  Security    Performance    Quality         │  │
│  (security.yaml)  (performance.yaml)        │  │
│  │           │               │              │  │
│  └───────────┴───────────────┴──────────────┘  │
│       ↓                                         │
│  3. Combine results                             │
│       ↓                                         │
│  4. Write report (audit-report.md)              │
└─────────────────────────────────────────────────┘
```

## Subagent Details

### security.yaml

**Focus**: OWASP Top 10 vulnerabilities

```yaml
pipeline:
  - step: analyze_security
    tool: llm_chat
    args:
      system_prompt: "You are a security expert..."
      prompt: "Review these files: {{file_list}}"
```

**Checks**:
- SQL Injection
- XSS (Cross-Site Scripting)
- CSRF (Cross-Site Request Forgery)
- Authentication bypasses
- Authorization flaws
- Sensitive data exposure

### performance.yaml

**Focus**: Performance bottlenecks

```yaml
pipeline:
  - step: analyze_performance
    tool: llm_chat
    args:
      system_prompt: "You are a performance expert..."
      prompt: "Analyze: {{file_list}}"
```

**Checks**:
- N+1 database queries
- Unnecessary loops
- Memory leaks
- Blocking I/O operations
- Large payload sizes

### quality.yaml

**Focus**: Code quality and maintainability

```yaml
pipeline:
  - step: analyze_quality
    tool: llm_chat
    args:
      system_prompt: "You are a code quality expert..."
      prompt: "Review quality: {{file_list}}"
```

**Checks**:
- Code smells
- Poor naming conventions
- Missing error handling
- Lack of tests
- Documentation gaps

## Extending the Example

### Add a new dimension

1. Create `accessibility.yaml`:
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: analyze_accessibility
    tool: llm_chat
    args:
      system_prompt: "Check WCAG compliance..."
      prompt: "Review: {{file_list}}"
    output: findings
```

2. Update `agent.json`:
```json
{
  "subagents": [
    {"name": "orchestrator", "path": "orchestrator.yaml"},
    {"name": "security", "path": "security.yaml"},
    {"name": "performance", "path": "performance.yaml"},
    {"name": "quality", "path": "quality.yaml"},
    {"name": "accessibility", "path": "accessibility.yaml"}
  ]
}
```

3. Update `orchestrator.yaml`:
```yaml
- step: run_accessibility
  tool: bash
  args:
    command: "agent-deploy run ./accessibility --args file_list='{{steps.find_files.output}}'"
  output: accessibility_report
```

### Parallel execution (future)

In v3.1+, subagents will run in parallel:

```yaml
- parallel:
    - step: run_security
      tool: subagent
      args:
        name: security
        input: "{{steps.find_files.output}}"
    
    - step: run_performance
      tool: subagent
      args:
        name: performance
        input: "{{steps.find_files.output}}"
    
    - step: run_quality
      tool: subagent
      args:
        name: quality
        input: "{{steps.find_files.output}}"
  
  output: all_reports
```

## Benefits of Multi-Subagent Architecture

### ✅ Separation of Concerns

Each subagent has a single responsibility, making them:
- Easier to develop
- Easier to test
- Easier to maintain
- Reusable across agents

### ✅ Composability

Mix and match subagents:
```bash
# Full audit
agent-deploy run ./code-auditor

# Security only
agent-deploy run ./code-auditor/security

# Custom combination
agent-deploy run ./my-custom-orchestrator
```

### ✅ Independent Evolution

Update one dimension without affecting others:
- Upgrade security checks to include new CVEs
- Add performance rules for new frameworks
- Refine quality standards

### ✅ Clear Contract

Each subagent expects `file_list` input and returns `findings` output.

## Limitations (Current)

⚠️ **Sequential Execution**: Subagents run one after another (no parallelism yet)

⚠️ **Subprocess Overhead**: Each subagent launches a separate process

⚠️ **No Direct Communication**: Subagents can't talk to each other (must go through orchestrator)

These will be addressed in future versions.

## Next Steps

- See [agent.json v3 spec](../../specs/agent-json-v3.md) for subagent definition
- See [subagent-system spec](../../specs/subagent-system.md) for advanced patterns
- Try creating your own multi-subagent workflow!

---

**Multi-Subagent Pattern - Compose complex workflows from simple agents** 🎯
