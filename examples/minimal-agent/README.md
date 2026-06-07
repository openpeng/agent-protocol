# Minimal Agent Example

A minimal agent demonstrating the basic Agent Protocol v3 structure.

## Structure

```
minimal-agent/
├── agent.json      # Agent metadata
├── worker.yaml     # Pipeline workflow
└── README.md       # This file
```

## What it does

This agent accepts a `user_name` parameter and generates a friendly greeting using an LLM.

## Usage

### Run locally

```bash
agent-deploy run ./minimal-agent --args user_name=Alice
```

**Output**:
```
Hello Alice! How can I help you today?
```

### Deploy to Claude Code

```bash
agent-deploy deploy ./minimal-agent -t claude_code
```

Then in Claude Code:
```
/minimal-agent user_name=Bob
```

## Files Explained

### agent.json

Defines the agent's identity and structure:
- **identity**: Name, version, description, author
- **entry**: Entry point (which subagent to run first)
- **subagents**: List of worker.yaml files

### worker.yaml

Defines the execution pipeline:
- **tools**: Declares `llm_chat` builtin tool
- **pipeline**: One step that calls LLM with a prompt

## Customization

### Change the greeting style

Edit `worker.yaml`:
```yaml
- step: greet
  tool: llm_chat
  args:
    prompt: "Say hello to {{user_name}} in a formal way"
    system_prompt: "You are a professional business assistant"
```

### Add more steps

```yaml
pipeline:
  - step: greet
    tool: llm_chat
    args:
      prompt: "Say hello to {{user_name}}"
    output: greeting

  - step: write_output
    tool: write_file
    args:
      path: "greeting.txt"
      content: "{{steps.greet.output}}"
```

## Next Steps

- See [file-summarizer](../file-summarizer/) for a more complete example
- Read the [agent.json v3 spec](../../specs/agent-json-v3.md)
- Read the [worker.yaml spec](../../specs/worker-yaml.md)
