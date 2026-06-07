# Migration Guide: agent.json v2 to v3

**Target Audience**: Developers with existing v2 agents  
**Estimated Time**: 15-30 minutes per agent  
**Difficulty**: Easy to Moderate

---

## What Changed in v3?

### Major Changes

| Feature | v2 | v3 | Migration Required? |
|---------|----|----|---------------------|
| **Pipeline Workflow** | ❌ None | ✅ worker.yaml | ✅ Yes |
| **Subagents** | ❌ Not supported | ✅ entry + subagents | ✅ Yes |
| **Static Instructions** | ✅ instructions field | ⚠️ Optional (compatibility) | ⚠️ Recommended |
| **Builtin Tools** | ❌ None | ✅ llm_chat, read_file, etc. | ⚠️ Optional |
| **Dependencies** | ❌ Implicit | ✅ Explicit declaration | ⚠️ Optional |

### Backward Compatibility

✅ **v2 agents still work** - The runtime automatically converts them to v3 format internally.

⚠️ **But you should migrate** - To leverage new features like pipelines, builtin tools, and subagents.

---

## Migration Paths

### Path 1: Minimal Migration (5 minutes)

Keep static instructions, add v3 structure.

**Before (v2)**:
```json
{
  "schema_version": "2.0",
  "identity": {
    "name": "code-reviewer",
    "version": "1.0.0",
    "description": "Reviews code for bugs",
    "author": "You"
  },
  "instructions": {
    "format": "markdown",
    "source": "inline",
    "content": "You are a code reviewer. Find bugs and suggest fixes."
  }
}
```

**After (v3 - minimal)**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "code-reviewer",
    "version": "1.0.1",
    "description": "Reviews code for bugs",
    "author": "You"
  },
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml",
      "description": "Main workflow"
    }
  ],
  "instructions": {
    "format": "markdown",
    "source": "inline",
    "content": "You are a code reviewer. Find bugs and suggest fixes."
  }
}
```

**worker.yaml (auto-generated)**:
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: process
    tool: llm_chat
    args:
      system_prompt: "You are a code reviewer. Find bugs and suggest fixes."
      prompt: "{{user_input}}"
    output: result
```

**Result**: ✅ Agent works in both v2 and v3 environments.

---

### Path 2: Full Migration (30 minutes)

Convert to pipeline workflow with builtin tools.

**Before (v2 - static instructions)**:
```json
{
  "schema_version": "2.0",
  "identity": {
    "name": "file-processor",
    "version": "1.0.0",
    "description": "Processes text files",
    "author": "You"
  },
  "instructions": {
    "content": "Read the file at {{file_path}}, process it, and save results to {{output_path}}"
  }
}
```

**After (v3 - pipeline workflow)**:

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "file-processor",
    "version": "2.0.0",
    "description": "Processes text files with structured pipeline",
    "author": "You",
    "tags": ["file", "processing", "pipeline"]
  },
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml",
      "description": "File processing pipeline"
    }
  ],
  "dependencies": {
    "llm_provider": "anthropic"
  },
  "category": "productivity",
  "type": "agent"
}
```

**worker.yaml**:
```yaml
tools:
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin
  - name: write_file
    type: builtin

pipeline:
  - step: read_input
    tool: read_file
    args:
      path: "{{file_path}}"
    output: raw_content
    on_fail: abort

  - step: process
    tool: llm_chat
    args:
      system_prompt: "You are a file processor. Extract key information."
      prompt: "Process this file:\n\n{{steps.read_input.output}}"
    output: processed_content
    on_fail: abort

  - step: write_output
    tool: write_file
    args:
      path: "{{output_path}}"
      content: "{{steps.process.output}}"
    on_fail: abort
```

**Result**: ✅ Declarative pipeline with error handling, tool integration, and clear data flow.

---

## Step-by-Step Migration

### Step 1: Update agent.json

1. Change `schema_version` from `"2.0"` to `"3.0"`
2. Add `entry` field:
   ```json
   "entry": {
     "main_subagent": "worker"
   }
   ```
3. Add `subagents` array:
   ```json
   "subagents": [
     {
       "name": "worker",
       "path": "worker.yaml",
       "description": "Main workflow"
     }
   ]
   ```
4. (Optional) Add `dependencies`:
   ```json
   "dependencies": {
     "llm_provider": "anthropic"
   }
   ```
5. (Optional) Add `category` and `type`:
   ```json
   "category": "development",
   "type": "agent"
   ```

### Step 2: Create worker.yaml

**Template**:
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: process
    tool: llm_chat
    args:
      system_prompt: "<your instructions here>"
      prompt: "{{user_input}}"
    output: result
```

**From v2 instructions**:
```javascript
// Extract v2 instructions
const v2Instructions = agentV2.instructions.content;

// Create worker.yaml
const workerYaml = `
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: process
    tool: llm_chat
    args:
      system_prompt: |
${v2Instructions.split('\n').map(line => '        ' + line).join('\n')}
      prompt: "{{user_input}}"
    output: result
`;
```

### Step 3: Test the Migration

```bash
# Test v3 agent
agent-deploy run ./my-agent --args user_input="test"

# Compare with v2 behavior
agent-deploy run ./my-agent-v2 --args user_input="test"
```

### Step 4: Update Version

Increment version in `agent.json`:
```json
{
  "identity": {
    "version": "2.0.0"  // Major bump for v3 migration
  }
}
```

---

## Common Migration Patterns

### Pattern 1: Simple LLM Agent

**v2**:
```json
{
  "instructions": "You are a translator. Translate to {{target_lang}}."
}
```

**v3**:
```yaml
# worker.yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: translate
    tool: llm_chat
    args:
      system_prompt: "You are a translator."
      prompt: "Translate to {{target_lang}}: {{text}}"
    output: translation
```

### Pattern 2: File Processing Agent

**v2**:
```json
{
  "instructions": "Read {{file_path}}, analyze it, write to {{output_path}}"
}
```

**v3**:
```yaml
# worker.yaml
tools:
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin
  - name: write_file
    type: builtin

pipeline:
  - step: read
    tool: read_file
    args:
      path: "{{file_path}}"
    output: content

  - step: analyze
    tool: llm_chat
    args:
      prompt: "Analyze: {{steps.read.output}}"
    output: analysis

  - step: write
    tool: write_file
    args:
      path: "{{output_path}}"
      content: "{{steps.analyze.output}}"
```

### Pattern 3: Multi-Step Agent

**v2**:
```json
{
  "instructions": "1. Read config\n2. Process data\n3. Generate report"
}
```

**v3**:
```yaml
# worker.yaml
tools:
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin
  - name: write_file
    type: builtin

pipeline:
  - step: read_config
    tool: read_file
    args:
      path: "config.json"
    output: config

  - step: process_data
    tool: llm_chat
    args:
      prompt: "Process data using config: {{steps.read_config.output}}"
    output: processed

  - step: generate_report
    tool: write_file
    args:
      path: "report.md"
      content: "{{steps.process_data.output}}"
```

---

## Automated Migration Tool

```bash
# Run migration assistant
agent-deploy migrate ./my-v2-agent

# What it does:
# 1. Reads agent.json v2
# 2. Generates agent.json v3
# 3. Creates default worker.yaml
# 4. Backs up original files
# 5. Validates new structure
```

**Output**:
```
✅ Migrated agent.json v2 → v3
✅ Created worker.yaml from instructions
✅ Backup saved to: .backup/agent.json.v2
⚠️  Manual review recommended for:
   - Complex instructions with multiple steps
   - Custom file paths and arguments
   - Error handling requirements
```

---

## Checklist

Use this checklist to ensure complete migration:

- [ ] Updated `schema_version` to `"3.0"`
- [ ] Added `entry.main_subagent`
- [ ] Added `subagents` array with at least one entry
- [ ] Created `worker.yaml` file
- [ ] Declared all required tools in `worker.yaml`
- [ ] Converted instructions to pipeline steps
- [ ] Added error handling (`on_fail`)
- [ ] Added dependencies (if using LLM)
- [ ] Tested with sample inputs
- [ ] Incremented version number
- [ ] Updated README/documentation
- [ ] Committed changes to version control

---

## FAQ

### Q: Do I need to migrate immediately?

**A**: No. v2 agents still work and will be auto-converted at runtime. But migrating unlocks:
- Pipeline workflows
- Builtin tools
- Better error handling
- Subagent composition

### Q: Can I keep using static instructions?

**A**: Yes. The `instructions` field is still supported in v3 for backward compatibility. But pipelines are more powerful.

### Q: What if my agent is complex?

**A**: Break it into multiple subagents. See [multi-subagent example](../examples/multi-subagent/).

### Q: How do I test both versions?

**A**: Keep v2 in a separate directory:
```bash
my-agent/
├── v2/
│   └── agent.json
└── v3/
    ├── agent.json
    └── worker.yaml
```

### Q: Can I mix v2 and v3 fields?

**A**: Yes, for compatibility. But avoid it long-term:
```json
{
  "schema_version": "3.0",
  "entry": {...},
  "subagents": [...],
  "instructions": "..."  // ⚠️ Deprecated, use worker.yaml
}
```

---

## Support

- 📖 [agent.json v3 spec](../specs/agent-json-v3.md)
- 📖 [worker.yaml spec](../specs/worker-yaml.md)
- 💬 [Discussions](https://github.com/agent-protocol/discussions)
- 🐛 [Issues](https://github.com/agent-protocol/issues)

---

**Migration Complete? Welcome to Agent Protocol v3!** 🎉
