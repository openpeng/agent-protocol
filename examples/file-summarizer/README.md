# File Summarizer

A complete agent that reads text files and generates concise summaries using LLM.

## Features

- ✅ Reads any text file
- ✅ Checks file length before processing
- ✅ Generates structured summaries with key points
- ✅ Fallback for short files
- ✅ Fallback if LLM unavailable
- ✅ Writes output to file

## Structure

```
file-summarizer/
├── agent.json      # Agent metadata
├── worker.yaml     # Pipeline workflow
└── README.md       # This file
```

## Usage

### Basic usage

```bash
agent-deploy run ./file-summarizer \
  --args file_path=/path/to/document.txt \
  --args output_path=/tmp/summary.md \
  --args timestamp="2026-06-07 10:30"
```

### Example with actual file

```bash
# Create a test file
cat > /tmp/report.txt << 'EOF'
This is a long document about AI agents.
Agent Protocol v3 introduces pipeline workflows, builtin tools, and subagent composition.
The new architecture enables declarative agent development with clear separation of concerns.
Key features include llm_chat for model calls, conditional execution with when clauses,
and robust error handling with on_fail strategies.
EOF

# Run summarization
agent-deploy run ./file-summarizer \
  --args file_path=/tmp/report.txt \
  --args output_path=/tmp/summary.md \
  --args timestamp="$(date)"

# View result
cat /tmp/summary.md
```

**Expected output** (`/tmp/summary.md`):
```markdown
# Summary of /tmp/report.txt

Agent Protocol v3 introduces a declarative workflow system for AI agent development.

Key points:
- Pipeline workflows for structured agent execution
- Builtin tools including llm_chat for model integration
- Conditional execution via when clauses
- Robust error handling with on_fail strategies
- Subagent composition for modular design

---
Generated: 2026-06-07 10:30:15
Original length: 342 characters
```

## Pipeline Breakdown

### Step 1: Read File

```yaml
- step: read_input
  tool: read_file
  args:
    path: "{{file_path}}"
  output: raw_content
  on_fail: abort  # Critical: must succeed
```

Reads the input file. If file doesn't exist, aborts.

### Step 2: Check Length

```yaml
- step: check_length
  tool: bash
  args:
    command: "echo '{{steps.read_input.output}}' | wc -c"
  output: content_length
```

Counts characters to decide if summarization is needed.

### Step 3: Generate Summary (Conditional)

```yaml
- step: generate_summary
  tool: llm_chat
  args:
    system_prompt: "..."
    prompt: "..."
  when: "{{steps.check_length.output}} > 1000"
  on_fail: continue
```

Only runs if file > 1000 chars. If LLM fails, continues (fallback will handle).

### Step 4: Short File Fallback

```yaml
- step: short_file_message
  tool: bash
  args:
    command: "echo 'File is too short...'"
  when: "{{steps.check_length.output}} <= 1000"
```

Runs if file ≤ 1000 chars.

### Step 5: LLM Fallback

```yaml
- step: fallback_summary
  tool: bash
  args:
    command: "echo '{{steps.read_input.output}}' | head -c 500"
  when: "{{steps.generate_summary.success}} == false"
```

Runs if LLM failed. Simply truncates to 500 chars.

### Step 6: Write Output

```yaml
- step: write_output
  tool: write_file
  args:
    path: "{{output_path}}"
    content: "..."
  on_fail: abort
```

Writes final summary to file.

## Configuration

### Customize summary length

Edit `shared_context` in `worker.yaml`:
```yaml
shared_context:
  max_summary_length: 1000  # Increase to 1000 chars
```

### Change LLM model

Edit the `generate_summary` step:
```yaml
- step: generate_summary
  tool: llm_chat
  args:
    model: "claude-3-opus-20240229"  # Use Opus for better quality
```

### Adjust threshold

Edit the `when` condition:
```yaml
when: "{{steps.check_length.output}} > 500"  # Lower threshold
```

## Error Handling

| Scenario | Behavior |
|----------|----------|
| File not found | Abort immediately |
| File too short | Skip LLM, use short message |
| LLM fails | Continue, use truncation fallback |
| Cannot write output | Abort with error |

## Deploy to Platforms

### Claude Code

```bash
agent-deploy deploy ./file-summarizer -t claude_code
```

Then in Claude Code:
```
/file-summarizer file_path=/data/report.txt output_path=/tmp/summary.md
```

### Cursor

```bash
agent-deploy deploy ./file-summarizer -t cursor
```

## Next Steps

- See [multi-subagent](../multi-subagent/) for advanced composition
- Read [worker.yaml spec](../../specs/worker-yaml.md)
- Read [builtin-tools spec](../../specs/builtin-tools.md)
