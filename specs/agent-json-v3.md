# agent.json v3 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

`agent.json` 是 Agent Protocol 的核心元数据文件，定义了 Agent 的身份、结构、依赖和入口点。

v3 版本引入了以下重大变更：
- ✅ **entry** - 定义 Agent 入口点
- ✅ **subagents** - 支持多子 Agent 组合
- ✅ **dependencies** - 显式依赖声明
- ✅ 向后兼容 v2 的 `instructions` 字段

---

## 完整规范

### 顶层结构

```typescript
interface AgentJsonV3 {
  schema_version: "3.0";
  identity: Identity;
  entry: Entry;
  subagents: Subagent[];
  dependencies?: Dependencies;
  category?: Category;
  type?: AgentType;
  
  // v2 兼容（可选）
  instructions?: string | InstructionsObject;
  capabilities?: string[];
  compatibility?: Record<string, any>;
}
```

---

## 字段详解

### 1. schema_version

**类型**: `string`  
**必填**: ✅  
**取值**: `"3.0"`

指定 agent.json 的版本。

```json
{
  "schema_version": "3.0"
}
```

---

### 2. identity

**类型**: `Identity`  
**必填**: ✅

定义 Agent 的身份信息。

```typescript
interface Identity {
  name: string;              // ✅ 必填：Agent 唯一标识符（kebab-case）
  version: string;           // ✅ 必填：语义版本号（semver）
  display_name?: string;     // 展示名称（可含中文）
  description: string;       // ✅ 必填：简短描述（60-120 字符）
  author: string;            // ✅ 必填：作者名
  license?: string;          // 许可证（默认 MIT）
  homepage?: string;         // 主页 URL
  repository?: string;       // 仓库 URL
  tags?: string[];           // 搜索标签
}
```

#### 字段规则

**name**:
- 格式：`kebab-case`（小写，连字符分隔）
- 示例：`file-summarizer`, `code-reviewer`, `test-generator`
- 唯一性：在 Market 中必须唯一

**version**:
- 格式：语义版本号 `MAJOR.MINOR.PATCH`
- 示例：`1.0.0`, `2.3.1`, `0.1.0-beta`
- 升级规则：
  - MAJOR: 不兼容的 API 变更
  - MINOR: 向后兼容的功能新增
  - PATCH: 向后兼容的问题修复

**description**:
- 长度：60-120 字符
- 要求：一句话说清楚 Agent 的功能
- ✅ Good: "Summarizes text files into concise bullet points"
- ❌ Bad: "This agent is very powerful and can do many things"

**tags**:
- 数量：3-6 个
- 格式：小写，连字符分隔
- 分类：
  - 功能标签：`summarize`, `review`, `generate`, `test`
  - 领域标签：`code`, `text`, `data`, `security`
  - 语言标签：`typescript`, `python`, `java`

#### 示例

```json
{
  "identity": {
    "name": "file-summarizer",
    "version": "1.2.0",
    "display_name": "File Summarizer",
    "description": "Reads text files and generates concise summaries with key points",
    "author": "OpenPeng",
    "license": "MIT",
    "homepage": "https://github.com/openpeng/file-summarizer",
    "repository": "https://github.com/openpeng/file-summarizer",
    "tags": ["file", "summary", "text-processing", "llm"]
  }
}
```

---

### 3. entry

**类型**: `Entry`  
**必填**: ✅

定义 Agent 的入口点。

```typescript
interface Entry {
  main_subagent: string;  // ✅ 必填：入口子 Agent 名称
}
```

#### 说明

- `main_subagent` 指向 `subagents` 数组中某个子 Agent 的 `name`
- 运行 Agent 时，从该子 Agent 开始执行
- 如果只有一个子 Agent，通常命名为 `"worker"`

#### 示例

```json
{
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml"
    }
  ]
}
```

---

### 4. subagents

**类型**: `Subagent[]`  
**必填**: ✅

定义 Agent 包含的子 Agent 列表。

```typescript
interface Subagent {
  name: string;         // ✅ 必填：子 Agent 名称
  path: string;         // ✅ 必填：worker.yaml 文件路径
  description?: string; // 子 Agent 描述
}
```

#### 子 Agent 机制

**单子 Agent**（最常见）：
```json
{
  "entry": {"main_subagent": "worker"},
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml",
      "description": "Main workflow"
    }
  ]
}
```

**多子 Agent**（高级用法）：
```json
{
  "entry": {"main_subagent": "orchestrator"},
  "subagents": [
    {
      "name": "orchestrator",
      "path": "orchestrator.yaml",
      "description": "Coordinates the workflow"
    },
    {
      "name": "reader",
      "path": "reader.yaml",
      "description": "Reads and parses files"
    },
    {
      "name": "analyzer",
      "path": "analyzer.yaml",
      "description": "Analyzes content with LLM"
    },
    {
      "name": "writer",
      "path": "writer.yaml",
      "description": "Writes output files"
    }
  ]
}
```

#### 文件结构

```
my-agent/
├── agent.json
├── worker.yaml          # main_subagent 指向这个
├── reader.yaml          # 其他子 Agent（可选）
└── analyzer.yaml
```

---

### 5. dependencies

**类型**: `Dependencies`  
**必填**: ❌（可选）

声明 Agent 的运行时依赖。

```typescript
interface Dependencies {
  python3?: string;           // Python 版本要求
  nodejs?: string;            // Node.js 版本要求
  llm_provider?: LLMProvider; // LLM 提供商
  [key: string]: string | undefined;
}

type LLMProvider = "anthropic" | "openai" | "any";
```

#### 依赖类型

**运行时依赖**：
```json
{
  "dependencies": {
    "python3": ">=3.10",
    "nodejs": ">=18.0.0"
  }
}
```

**LLM 依赖**：
```json
{
  "dependencies": {
    "llm_provider": "anthropic"  // 需要 Claude API
  }
}
```

**自定义依赖**：
```json
{
  "dependencies": {
    "custom_tool": "^2.0.0"
  }
}
```

#### 版本范围语法

| 语法 | 含义 | 示例 |
|------|------|------|
| `>=3.10` | 大于等于 | `>=3.10` 允许 3.10, 3.11, 3.12... |
| `^2.0.0` | 兼容版本 | `^2.0.0` 允许 2.x.x，但不允许 3.x.x |
| `~1.5.0` | 近似版本 | `~1.5.0` 允许 1.5.x，但不允许 1.6.x |
| `*` | 任意版本 | `*` 允许所有版本 |

---

### 6. category

**类型**: `Category`  
**必填**: ❌（可选）

Agent 的功能分类。

```typescript
type Category = 
  | "development"      // 开发工具
  | "productivity"     // 生产力
  | "writing"          // 写作辅助
  | "research"         // 研究分析
  | "data"             // 数据处理
  | "security"         // 安全审计
  | "testing"          // 测试生成
  | "documentation"    // 文档生成
  | "refactoring"      // 代码重构
  | "general";         // 通用
```

#### 示例

```json
{
  "category": "development"
}
```

---

### 7. type

**类型**: `AgentType`  
**必填**: ❌（可选）

Agent 的类型分类。

```typescript
type AgentType = 
  | "agent"      // 完整的 Agent（有 pipeline）
  | "skill"      // 简单技能（单步操作）
  | "workflow";  // 工作流（多步编排）
```

#### 区别

**agent**（推荐）：
- 有完整的 pipeline
- 可以调用多个工具
- 有复杂的逻辑编排

**skill**（简化版）：
- 单一功能
- 通常只有一个 pipeline 步骤
- 适合简单任务

**workflow**（复杂版）：
- 多个子 Agent 协作
- 有条件分支和循环
- 适合复杂业务流程

---

### 8. instructions (v2 兼容)

**类型**: `string | InstructionsObject`  
**必填**: ❌（v3 中可选，v2 中必填）

静态指令文本（向后兼容 v2）。

```typescript
type Instructions = string | {
  format: "markdown" | "yaml" | "text";
  source: "inline" | "file";
  content?: string;
  file?: string;
};
```

#### v3 中的处理

在 v3 中，`instructions` 字段是**可选的**，因为指令现在定义在 `worker.yaml` 的 pipeline 中。

但为了向后兼容，运行时可以自动将 `instructions` 转换为 worker.yaml：

```json
// v2 格式
{
  "schema_version": "2.0",
  "identity": {...},
  "instructions": {
    "format": "markdown",
    "source": "inline",
    "content": "You are a code reviewer..."
  }
}

// 自动转换为 v3
{
  "schema_version": "3.0",
  "identity": {...},
  "entry": {"main_subagent": "worker"},
  "subagents": [
    {"name": "worker", "path": "worker.yaml"}
  ]
}

// 生成的 worker.yaml
// tools:
//   - name: llm_chat
//     type: builtin
// pipeline:
//   - step: process
//     tool: llm_chat
//     args:
//       system_prompt: "You are a code reviewer..."
//       prompt: "{{user_input}}"
```

---

## 完整示例

### 示例 1: 最小 Agent

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "hello-world",
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

### 示例 2: 完整的 Agent

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "file-summarizer",
    "version": "2.1.0",
    "display_name": "File Summarizer",
    "description": "Reads text files and generates concise summaries with key points",
    "author": "OpenPeng",
    "license": "MIT",
    "homepage": "https://github.com/openpeng/file-summarizer",
    "repository": "https://github.com/openpeng/file-summarizer",
    "tags": [
      "file",
      "summary",
      "text-processing",
      "llm",
      "productivity"
    ]
  },
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml",
      "description": "Main summarization workflow"
    }
  ],
  "dependencies": {
    "llm_provider": "anthropic"
  },
  "category": "productivity",
  "type": "agent"
}
```

### 示例 3: 多子 Agent

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "code-auditor",
    "version": "1.0.0",
    "display_name": "Code Auditor",
    "description": "Comprehensive code audit with security, performance, and quality checks",
    "author": "Security Team",
    "tags": ["security", "audit", "code-quality", "performance"]
  },
  "entry": {
    "main_subagent": "orchestrator"
  },
  "subagents": [
    {
      "name": "orchestrator",
      "path": "orchestrator.yaml",
      "description": "Coordinates the audit workflow"
    },
    {
      "name": "security-scanner",
      "path": "security-scanner.yaml",
      "description": "Scans for security vulnerabilities"
    },
    {
      "name": "performance-analyzer",
      "path": "performance-analyzer.yaml",
      "description": "Analyzes performance bottlenecks"
    },
    {
      "name": "quality-checker",
      "path": "quality-checker.yaml",
      "description": "Checks code quality and best practices"
    },
    {
      "name": "report-generator",
      "path": "report-generator.yaml",
      "description": "Generates comprehensive audit report"
    }
  ],
  "dependencies": {
    "llm_provider": "anthropic",
    "python3": ">=3.10"
  },
  "category": "security",
  "type": "workflow"
}
```

---

## 验证规则

### 必填字段检查

```typescript
function validateAgentJson(agent: any): ValidationResult {
  const errors: string[] = [];
  
  // schema_version
  if (agent.schema_version !== "3.0") {
    errors.push("schema_version must be '3.0'");
  }
  
  // identity
  if (!agent.identity?.name) {
    errors.push("identity.name is required");
  }
  if (!agent.identity?.version) {
    errors.push("identity.version is required");
  }
  if (!agent.identity?.description) {
    errors.push("identity.description is required");
  }
  if (!agent.identity?.author) {
    errors.push("identity.author is required");
  }
  
  // entry
  if (!agent.entry?.main_subagent) {
    errors.push("entry.main_subagent is required");
  }
  
  // subagents
  if (!agent.subagents || agent.subagents.length === 0) {
    errors.push("At least one subagent is required");
  }
  
  return {
    valid: errors.length === 0,
    errors
  };
}
```

### 引用完整性检查

```typescript
function validateReferences(agent: AgentJsonV3): ValidationResult {
  const errors: string[] = [];
  
  // 检查 entry.main_subagent 是否存在
  const mainExists = agent.subagents.some(
    sub => sub.name === agent.entry.main_subagent
  );
  if (!mainExists) {
    errors.push(
      `entry.main_subagent '${agent.entry.main_subagent}' ` +
      `not found in subagents`
    );
  }
  
  // 检查 subagent 名称唯一性
  const names = agent.subagents.map(sub => sub.name);
  const duplicates = names.filter((name, i) => names.indexOf(name) !== i);
  if (duplicates.length > 0) {
    errors.push(`Duplicate subagent names: ${duplicates.join(", ")}`);
  }
  
  return {
    valid: errors.length === 0,
    errors
  };
}
```

---

## v2 迁移

### 自动迁移

```typescript
function migrateV2ToV3(v2: AgentJsonV2): AgentJsonV3 {
  return {
    schema_version: "3.0",
    identity: v2.identity,
    entry: {
      main_subagent: "worker"
    },
    subagents: [
      {
        name: "worker",
        path: "worker.yaml",
        description: "Main workflow (migrated from v2)"
      }
    ],
    category: v2.identity.category,
    type: "agent",
    // 保留 v2 兼容字段
    instructions: v2.instructions,
    capabilities: v2.capabilities,
    compatibility: v2.compatibility
  };
}
```

### 生成默认 worker.yaml

```typescript
function generateDefaultWorkerYaml(instructions: string): string {
  return `
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: process
    tool: llm_chat
    args:
      system_prompt: |
${instructions.split('\n').map(line => '        ' + line).join('\n')}
      prompt: "{{user_input}}"
    output: result
`.trim();
}
```

---

## JSON Schema

完整的 JSON Schema 定义参见：[schemas/agent.schema.json](../schemas/agent.schema.json)

---

## 参考

- [worker.yaml 规范](./worker-yaml.md)
- [Builtin Tools 规范](./builtin-tools.md)
- [Subagent System 规范](./subagent-system.md)
- [v2 到 v3 迁移指南](../compatibility/migration-v2-to-v3.md)

---

**agent.json v3 - 标准化的 Agent 元数据定义** 🎯
