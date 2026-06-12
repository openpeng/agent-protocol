# Agent Protocol Specification

**Version**: 3.0.0  
**Status**: Active  
**Last Updated**: 2026-06-12

---

## 概述

Agent Protocol 是一套完整的 AI Agent 定义、开发、运行和分发的标准协议。它定义了：

1. **agent.json v3** - Agent 元数据和结构定义
2. **worker.yaml** - Pipeline 工作流编排
3. **Builtin Tools** - 标准工具系统（9 工具）
4. **Subagent System** - 多子 Agent 组合 + invoke_parallel 并行
5. **Dynamic Overrides** - 运行时 instructions/skills/MCP 覆盖
6. **Deployment Targets** - 跨 9 平台部署适配

---

## 为什么需要 Agent Protocol？

### 当前问题

**Agent 定义碎片化**：
- Claude Code 用 `.md` 文件
- Cursor 用 `.cursorrules`
- Windsurf 用 `.windsurfrules`
- 各平台互不兼容

**缺乏运行时能力**：
- 只有静态指令，无法编排复杂工作流
- 无法调用工具（LLM、文件、命令等）
- 无法组合多个子 Agent

**无标准分发机制**：
- 没有统一的 Agent Market
- 无法跨平台共享和复用

### Agent Protocol 解决方案

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Protocol v3                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ agent.json   │───▶│ worker.yaml  │───▶│ Builtin      │ │
│  │ (元数据)      │    │ (Pipeline)   │    │ Tools        │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│         │                    │                    │         │
│         └────────────────────┴────────────────────┘         │
│                              │                              │
│                    ┌─────────▼─────────┐                    │
│                    │  Agent Runtime    │                    │
│                    │  (执行引擎)        │                    │
│                    └─────────┬─────────┘                    │
│                              │                              │
│         ┌────────────────────┼────────────────────┐         │
│         │                    │                    │         │
│  ┌──────▼──────┐    ┌────────▼────────┐    ┌─────▼─────┐  │
│  │ Local Run   │    │ Deploy to Tool  │    │ MCP Server│  │
│  │ (本地运行)   │    │ (部署到平台)     │    │ (作为工具) │  │
│  └─────────────┘    └─────────────────┘    └───────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 核心特性

### ✅ 1. 标准化定义

**agent.json v3**：
- 完整的元数据（identity、author、tags）
- 入口和子 Agent 定义
- 依赖声明（Python、Node.js 等）
- 向后兼容 v2

### ✅ 2. 工作流编排

**worker.yaml**：
- Pipeline 步骤编排（串行/并行）
- 条件执行 (when clauses) 和错误处理
- 模板变量系统 ({{var}}, {{steps.X.output}}, {{shared.key}})
- 步骤间数据传递
- 调用级重试 (exponential backoff)
- invoke_parallel 并行子Agent调用

### ✅ 3. 工具系统

**Builtin Tools (9)**：
- `llm_chat` - 调用大模型（多 Provider 自动降级）
- `read_file` / `write_file` - 文件操作
- `bash` - 命令执行（安全沙箱 + denyList）
- `glob` - 文件匹配 (ripgrep 引擎)
- `web_fetch` / `web_search` - 网络访问（SSRF 防护）
- `invoke_agent` - 调用子 Agent（支持动态 overrides）
- `list_agents` - 发现可用 Agent

### ✅ 4. 多模式运行

**运行模式**：
- **Local Run** - 本地执行 pipeline
- **Deploy** - 部署到 Claude Code/Cursor 等平台
- **MCP Server** - 作为 MCP Tool 暴露

### ✅ 5. 跨平台兼容

**部署目标 (9 个)**：
- Claude Code, Cursor, CodeBuddy
- GitHub Copilot, Windsurf, OpenCode
- Trae, Aider, AGENTS.md

---

## 快速开始

### 最小 Agent 示例

**agent.json**：
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "hello-agent",
    "version": "1.0.0",
    "description": "A simple hello world agent",
    "author": "Your Name"
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
  ]
}
```

**worker.yaml**：
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: greet
    tool: llm_chat
    args:
      prompt: "Say hello to {{user_name}}"
      system_prompt: "You are a friendly assistant"
    output: greeting

  - step: output
    tool: write_file
    args:
      path: "output.txt"
      content: "{{steps.greet.output}}"
```

### 运行 Agent

```bash
# 本地运行
agent-deploy run ./hello-agent --args user_name=Alice

# 部署到工具
agent-deploy deploy ./hello-agent -t claude_code

# 作为 MCP Server（提供 9 个工具）
agent-deploy
```

---

## 文档结构

### 规范文档

| 文件 | 说明 |
|------|------|
| [specs/agent-json-v3.md](./specs/agent-json-v3.md) | agent.json v3 完整规范 |
| [specs/worker-yaml.md](./specs/worker-yaml.md) | Pipeline 工作流规范 |
| [specs/builtin-tools.md](./specs/builtin-tools.md) | 内置工具系统规范 |
| [specs/skill-system.md](./specs/skill-system.md) | Skill 系统规范 |
| [specs/mcp-integration.md](./specs/mcp-integration.md) | MCP 集成规范 |
| [specs/discovery.md](./specs/discovery.md) | Agent 发现规范 |
| [specs/memory-system.md](./specs/memory-system.md) | 记忆系统规范 |

### JSON Schemas

| 文件 | 说明 |
|------|------|
| [schemas/agent.schema.json](./schemas/agent.schema.json) | agent.json JSON Schema |
| [schemas/worker.schema.json](./schemas/worker.schema.json) | worker.yaml JSON Schema |

### 示例 Agents

| 目录 | 说明 |
|------|------|
| [examples/minimal-agent/](./examples/minimal-agent/) | 最小可运行 Agent |
| [examples/file-summarizer/](./examples/file-summarizer/) | 文件摘要生成器（完整示例） |
| [examples/multi-subagent/](./examples/multi-subagent/) | 多子 Agent 组合示例 |

### 兼容性

| 文件 | 说明 |
|------|------|
| [compatibility/migration-v2-to-v3.md](./compatibility/migration-v2-to-v3.md) | v2 到 v3 迁移指南 |

---

## 设计原则

### 1. 声明式优于命令式

✅ **Good** - 声明你想要什么：
```yaml
- step: summarize
  tool: llm_chat
  args:
    prompt: "Summarize: {{content}}"
```

❌ **Bad** - 描述如何做：
```yaml
- step: manual
  tool: bash
  args:
    command: "curl ... | python process.py"
```

### 2. 组合优于单体

✅ **Good** - 多个小 Agent 组合：
```json
{
  "subagents": [
    {"name": "reader", "path": "reader.yaml"},
    {"name": "analyzer", "path": "analyzer.yaml"},
    {"name": "writer", "path": "writer.yaml"}
  ]
}
```

❌ **Bad** - 一个巨大的 Agent：
```json
{
  "subagents": [
    {"name": "monolith", "path": "everything.yaml"}
  ]
}
```

### 3. 显式优于隐式

✅ **Good** - 明确的依赖声明：
```json
{
  "dependencies": {
    "python3": ">=3.10",
    "llm_provider": "anthropic"
  }
}
```

❌ **Bad** - 隐式假设：
```yaml
# 假设用户已经配置了 API Key...
```

### 4. 兼容优于重写

✅ **Good** - 平滑升级路径：
```json
{
  "schema_version": "3.0",
  // v2 字段仍然支持
  "instructions": "..." 
}
```

❌ **Bad** - 强制迁移：
```
Error: v2 format not supported, please rewrite
```

---

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| **3.0** | 2026-06-07 | 引入 Pipeline、Subagent、Builtin Tools |
| **2.0** | 2026-05-15 | 标准化 identity 结构，引入 instructions 字段 |
| **1.0** | 2026-04-01 | 初始版本（SKILL.md 格式） |

---

## 参考实现

- **agent-hub** - Agent 运行时、Market 服务、MCP Server
  - Repository: https://github.com/openpeng/agent-hub
  - 支持 agent.json v3、Pipeline 执行、9 平台部署、9 MCP 工具、动态 Overrides

---

## 贡献指南

### 提议新特性

1. 在 [Issues](https://github.com/openpeng/agent-protocol/issues) 中创建提议
2. 说明使用场景和动机
3. 提供示例和预期行为
4. 等待社区讨论和投票

### 提交 PR

1. Fork 本仓库
2. 创建特性分支
3. 更新相关规范文档和 JSON Schema
4. 添加示例（如果适用）
5. 提交 PR 并说明变更原因

---

## 许可证

MIT License

---

## 联系方式

- **GitHub**: https://github.com/openpeng/agent-protocol
- **Issues**: https://github.com/openpeng/agent-protocol/issues
- **Discussions**: https://github.com/openpeng/agent-protocol/discussions

---

**Agent Protocol v3 - 让 AI Agent 标准化、可组合、可分发** 🚀
