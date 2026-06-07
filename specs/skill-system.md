# Skill System 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

Skill System 让 Agent 能够调用和组合预定义的技能（Skills）。在 Agent Protocol v3 中，Skill 是一种特殊的 Agent 类型，具有以下特点：

- ✅ **单一职责** - 一个 Skill 只做一件事
- ✅ **可复用** - 可被多个 Agent 调用
- ✅ **参数化** - 通过参数配置行为
- ✅ **可组合** - 多个 Skills 组合成复杂功能

---

## 1. Skill vs Agent

### 区别

| 特性 | Skill | Agent |
|------|-------|-------|
| **复杂度** | 简单，单一功能 | 复杂，多步骤工作流 |
| **Pipeline** | 1-3 步 | 多步骤，有条件分支 |
| **用途** | 被其他 Agent 调用 | 独立运行或调用 Skills |
| **示例** | "总结文本"、"翻译" | "代码审查"、"项目管理" |

### Skill 示例

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "description": "Summarizes text into concise bullet points"
  },
  "type": "skill",  // ← 标记为 skill
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

```yaml
# worker.yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: summarize
    tool: llm_chat
    args:
      system_prompt: "Summarize text into 3-5 bullet points."
      prompt: "{{text}}"
      max_tokens: 200
    output: summary
```

---

## 2. 在 Agent 中使用 Skills

### 方式 1: 作为 Subagent

将 Skill 作为 Subagent 加载。

**Agent 结构**:
```
my-agent/
├── agent.json
├── orchestrator.yaml      # 主工作流
└── skills/
    ├── text-summarizer/   # Skill 1
    │   ├── agent.json
    │   └── worker.yaml
    └── translator/        # Skill 2
        ├── agent.json
        └── worker.yaml
```

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "content-processor",
    "description": "Processes content with summarization and translation"
  },
  "entry": {
    "main_subagent": "orchestrator"
  },
  "subagents": [
    {
      "name": "orchestrator",
      "path": "orchestrator.yaml"
    },
    {
      "name": "text-summarizer",
      "path": "skills/text-summarizer/worker.yaml",
      "description": "Summarizes text"
    },
    {
      "name": "translator",
      "path": "skills/translator/worker.yaml",
      "description": "Translates text"
    }
  ]
}
```

**orchestrator.yaml**:
```yaml
tools:
  - name: read_file
    type: builtin
  - name: skill_text_summarizer
    type: subagent
    subagent: text-summarizer
  - name: skill_translator
    type: subagent
    subagent: translator

pipeline:
  - step: read_content
    tool: read_file
    args:
      path: "{{file_path}}"
    output: content

  # 调用 Skill 1: 总结
  - step: summarize
    tool: skill_text_summarizer
    args:
      text: "{{steps.read_content.output}}"
    output: summary

  # 调用 Skill 2: 翻译
  - step: translate
    tool: skill_translator
    args:
      text: "{{steps.summarize.output}}"
      target_lang: "{{target_lang}}"
    output: translated

  - step: result
    tool: bash
    args:
      command: "echo '{{steps.translated.output}}'"
    output: final_result
```

### 方式 2: 通过 Skill Registry

从 Skill Registry 动态加载 Skills。

**Skill Registry 配置** (~/.agent-deploy/skills.json):
```json
{
  "skills": [
    {
      "name": "text-summarizer",
      "path": "/usr/local/share/agent-skills/text-summarizer",
      "version": "1.0.0"
    },
    {
      "name": "translator",
      "path": "/usr/local/share/agent-skills/translator",
      "version": "2.1.0"
    }
  ]
}
```

**worker.yaml**:
```yaml
tools:
  - name: text_summarizer
    type: skill
    skill_name: text-summarizer  # 从 registry 加载

pipeline:
  - step: summarize
    tool: text_summarizer
    args:
      text: "{{content}}"
    output: summary
```

**运行时行为**:
```typescript
// src/runtime/skills/loader.ts
export class SkillLoader {
  async loadSkill(skillName: string): Promise<Agent> {
    // 1. 从 registry 查找
    const skillInfo = this.registry.find(skillName);
    if (!skillInfo) {
      throw new Error(`Skill not found: ${skillName}`);
    }

    // 2. 加载 Skill Agent
    const skillAgent = await this.agentLoader.load(skillInfo.path);

    // 3. 验证是 Skill 类型
    if (skillAgent.type !== "skill") {
      throw new Error(`Not a skill: ${skillName}`);
    }

    return skillAgent;
  }

  async executeSkill(
    skillAgent: Agent,
    args: Record<string, any>
  ): Promise<any> {
    // 执行 Skill 的 pipeline
    return await skillAgent.run(args);
  }
}
```

---

## 3. Skill Tool 实现

### SkillTool 类

```typescript
// src/runtime/tools/skill-tool.ts
export class SkillTool implements BuiltinTool {
  name: string;
  type = "skill";
  
  constructor(
    private skillName: string,
    private skillLoader: SkillLoader
  ) {
    this.name = `skill_${skillName}`;
  }

  async execute(args: any, context: ExecutionContext): Promise<any> {
    // 1. 加载 Skill
    const skillAgent = await this.skillLoader.loadSkill(this.skillName);

    // 2. 创建 Skill 执行上下文
    const skillContext = {
      ...context,
      initialArgs: args,
      steps: new Map()  // 隔离的步骤上下文
    };

    // 3. 执行 Skill pipeline
    const result = await skillAgent.run(args);

    return result;
  }
}
```

### 在 Pipeline 中使用

```yaml
tools:
  - name: text_summarizer
    type: skill
    skill_name: text-summarizer

pipeline:
  - step: summarize
    tool: text_summarizer
    args:
      text: "{{content}}"
      max_length: 200
    output: summary
    on_fail: abort
```

---

## 4. Skill 市场集成

### 从 Market 安装 Skills

```bash
# 搜索 Skills
agent-deploy search --type skill summarizer

# 输出:
# text-summarizer (v1.0.0) - Summarizes text into bullet points
# ai-summarizer (v2.3.0) - Advanced AI-powered summarization

# 安装 Skill
agent-deploy install text-summarizer

# 安装到:
# ~/.agent-deploy/skills/text-summarizer/
```

### Skill 注册

```typescript
// src/runtime/skills/registry.ts
export class SkillRegistry {
  private skills = new Map<string, SkillInfo>();

  async discover() {
    // 1. 扫描系统 Skill 目录
    const systemSkills = await this.scanDirectory(
      "/usr/local/share/agent-skills"
    );

    // 2. 扫描用户 Skill 目录
    const userSkills = await this.scanDirectory(
      "~/.agent-deploy/skills"
    );

    // 3. 注册所有 Skills
    for (const skill of [...systemSkills, ...userSkills]) {
      this.register(skill);
    }
  }

  register(skillInfo: SkillInfo) {
    this.skills.set(skillInfo.name, skillInfo);
  }

  find(name: string): SkillInfo | null {
    return this.skills.get(name) || null;
  }

  list(): SkillInfo[] {
    return Array.from(this.skills.values());
  }
}
```

---

## 5. Skill 参数验证

### 定义参数 Schema

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "text-summarizer"
  },
  "type": "skill",
  "parameters": {
    "text": {
      "type": "string",
      "required": true,
      "description": "Text to summarize"
    },
    "max_length": {
      "type": "number",
      "default": 200,
      "description": "Maximum summary length"
    },
    "format": {
      "type": "string",
      "enum": ["bullets", "paragraph"],
      "default": "bullets",
      "description": "Output format"
    }
  }
}
```

### 运行时验证

```typescript
// src/runtime/skills/validator.ts
export class SkillParameterValidator {
  validate(
    args: Record<string, any>,
    parameters: ParameterSchema
  ): ValidationResult {
    const errors: string[] = [];

    // 1. 检查必填参数
    for (const [key, schema] of Object.entries(parameters)) {
      if (schema.required && !(key in args)) {
        errors.push(`Missing required parameter: ${key}`);
      }
    }

    // 2. 检查类型
    for (const [key, value] of Object.entries(args)) {
      const schema = parameters[key];
      if (!schema) {
        errors.push(`Unknown parameter: ${key}`);
        continue;
      }

      if (!this.checkType(value, schema.type)) {
        errors.push(
          `Invalid type for ${key}: expected ${schema.type}, got ${typeof value}`
        );
      }

      // 3. 检查枚举
      if (schema.enum && !schema.enum.includes(value)) {
        errors.push(
          `Invalid value for ${key}: must be one of ${schema.enum.join(", ")}`
        );
      }
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }
}
```

---

## 6. 内置 Skills

### 推荐的内置 Skills

| Skill | 功能 | 参数 |
|-------|------|------|
| `text-summarizer` | 文本摘要 | text, max_length |
| `translator` | 文本翻译 | text, target_lang |
| `json-parser` | JSON 解析 | json_string |
| `markdown-formatter` | Markdown 格式化 | text |
| `code-formatter` | 代码格式化 | code, language |
| `url-extractor` | 提取 URL | text |
| `date-parser` | 日期解析 | date_string |

### 示例：translator Skill

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "translator",
    "version": "1.0.0",
    "description": "Translates text between languages"
  },
  "type": "skill",
  "parameters": {
    "text": {
      "type": "string",
      "required": true
    },
    "target_lang": {
      "type": "string",
      "required": true,
      "enum": ["en", "zh", "es", "fr", "de", "ja"]
    },
    "source_lang": {
      "type": "string",
      "default": "auto"
    }
  },
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

**worker.yaml**:
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: translate
    tool: llm_chat
    args:
      system_prompt: |
        You are a professional translator.
        Translate accurately and naturally.
      prompt: |
        Translate the following text from {{source_lang}} to {{target_lang}}:
        
        {{text}}
        
        Only return the translated text, no explanations.
      temperature: 0.3
    output: translation
```

---

## 7. Skill 组合模式

### 模式 1: 串行组合

```yaml
# Agent: content-pipeline
pipeline:
  # Skill 1: 总结
  - step: summarize
    tool: skill_text_summarizer
    args:
      text: "{{content}}"
    output: summary

  # Skill 2: 翻译（使用 Skill 1 的输出）
  - step: translate
    tool: skill_translator
    args:
      text: "{{steps.summarize.output}}"
      target_lang: "zh"
    output: translated

  # Skill 3: 格式化（使用 Skill 2 的输出）
  - step: format
    tool: skill_markdown_formatter
    args:
      text: "{{steps.translate.output}}"
    output: formatted
```

### 模式 2: 并行组合（未来）

```yaml
# v3.1+ 支持
pipeline:
  - parallel:
      # Skill 1: 总结
      - step: summarize
        tool: skill_text_summarizer
        args:
          text: "{{content}}"
      
      # Skill 2: 提取关键词
      - step: extract_keywords
        tool: skill_keyword_extractor
        args:
          text: "{{content}}"
      
      # Skill 3: 情感分析
      - step: sentiment
        tool: skill_sentiment_analyzer
        args:
          text: "{{content}}"
    
    output: parallel_results

  # 合并结果
  - step: combine
    tool: llm_chat
    args:
      prompt: |
        Combine these analyses:
        Summary: {{steps.parallel_results[0]}}
        Keywords: {{steps.parallel_results[1]}}
        Sentiment: {{steps.parallel_results[2]}}
    output: combined_report
```

### 模式 3: 条件组合

```yaml
pipeline:
  # 检测语言
  - step: detect_language
    tool: skill_language_detector
    args:
      text: "{{content}}"
    output: detected_lang

  # 如果不是英文，先翻译
  - step: translate_to_english
    tool: skill_translator
    args:
      text: "{{content}}"
      target_lang: "en"
    output: english_text
    when: "{{steps.detect_language.output}} != 'en'"

  # 使用英文版本进行分析
  - step: analyze
    tool: skill_sentiment_analyzer
    args:
      text: "{{steps.translate_to_english.output}}"
    when: "{{steps.translate_to_english.success}}"
```

---

## 8. Skill 开发最佳实践

### ✅ DO - 推荐做法

**1. 单一职责**:
```json
// ✅ Good - 一个 Skill 做一件事
{
  "name": "text-summarizer",
  "description": "Summarizes text"
}

// ❌ Bad - 太多职责
{
  "name": "text-processor",
  "description": "Summarizes, translates, and formats text"
}
```

**2. 参数化**:
```json
// ✅ Good - 可配置
{
  "parameters": {
    "max_length": {"type": "number", "default": 200},
    "format": {"type": "string", "enum": ["bullets", "paragraph"]}
  }
}

// ❌ Bad - 硬编码
// （在 worker.yaml 中硬编码 max_length: 200）
```

**3. 错误处理**:
```yaml
# ✅ Good
- step: process
  tool: llm_chat
  args: {...}
  on_fail: abort  # 明确失败行为

# ❌ Bad
- step: process
  tool: llm_chat
  args: {...}
  # 没有 on_fail，不清楚失败时行为
```

**4. 文档清晰**:
```json
// ✅ Good
{
  "parameters": {
    "text": {
      "type": "string",
      "required": true,
      "description": "The text content to summarize. Can be up to 10,000 characters."
    }
  }
}

// ❌ Bad
{
  "parameters": {
    "text": {"type": "string"}  // 没有 description
  }
}
```

### ❌ DON'T - 避免做法

**1. 避免副作用**:
```yaml
# ❌ Bad - Skill 修改文件系统
- step: save_result
  tool: write_file  # Skill 不应该有副作用

# ✅ Good - Skill 只返回结果，由调用者决定是否保存
- step: process
  tool: llm_chat
  output: result
```

**2. 避免状态依赖**:
```yaml
# ❌ Bad - 依赖外部状态
- step: process
  tool: llm_chat
  args:
    prompt: "Process based on previous result"  # 依赖外部状态

# ✅ Good - 所有输入都通过参数传递
- step: process
  tool: llm_chat
  args:
    prompt: "Process: {{input_text}}"
```

**3. 避免过度依赖**:
```json
// ❌ Bad - 依赖太多 MCP tools
{
  "dependencies": {
    "mcp_tools": ["tapd", "dify", "gitlab", "jira", "slack"]
  }
}

// ✅ Good - 最小依赖
{
  "dependencies": {
    "llm_provider": "anthropic"
  }
}
```

---

## 9. Skill 发现与安装

### CLI 命令

```bash
# 列出本地 Skills
agent-deploy skill list

# 搜索 Skills
agent-deploy skill search summarizer

# 安装 Skill
agent-deploy skill install text-summarizer

# 卸载 Skill
agent-deploy skill uninstall text-summarizer

# 查看 Skill 信息
agent-deploy skill info text-summarizer

# 测试 Skill
agent-deploy skill test text-summarizer \
  --args text="Long text here..." \
  --args max_length=100
```

### Skill Info 输出

```bash
$ agent-deploy skill info text-summarizer

📦 text-summarizer v1.0.0
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Description: Summarizes text into concise bullet points
Author: Agent Protocol Team
License: MIT

Parameters:
  text (string, required)
    The text content to summarize
  
  max_length (number, default: 200)
    Maximum length of summary in characters
  
  format (string, default: "bullets")
    Output format: bullets | paragraph

Dependencies:
  llm_provider: anthropic

Location: ~/.agent-deploy/skills/text-summarizer/

Usage:
  agent-deploy skill test text-summarizer \
    --args text="Your text here"
```

---

## 10. 完整示例

### Skill: code-formatter

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "code-formatter",
    "version": "1.0.0",
    "display_name": "Code Formatter",
    "description": "Formats code in various programming languages",
    "author": "Agent Protocol Team",
    "tags": ["code", "formatting", "skill"]
  },
  "type": "skill",
  "parameters": {
    "code": {
      "type": "string",
      "required": true,
      "description": "The code to format"
    },
    "language": {
      "type": "string",
      "required": true,
      "enum": ["typescript", "javascript", "python", "go", "rust"],
      "description": "Programming language"
    },
    "style": {
      "type": "string",
      "enum": ["standard", "google", "airbnb"],
      "default": "standard",
      "description": "Code style guide"
    }
  },
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

**worker.yaml**:
```yaml
tools:
  - name: bash
    type: builtin
  - name: llm_chat
    type: builtin

pipeline:
  # 尝试使用工具格式化
  - step: try_formatter
    tool: bash
    args:
      command: |
        case "{{language}}" in
          typescript|javascript)
            echo "{{code}}" | npx prettier --parser typescript
            ;;
          python)
            echo "{{code}}" | python -m black -
            ;;
          go)
            echo "{{code}}" | gofmt
            ;;
          rust)
            echo "{{code}}" | rustfmt
            ;;
        esac
    output: formatted_code
    on_fail: continue

  # Fallback: 使用 LLM
  - step: llm_format
    tool: llm_chat
    args:
      system_prompt: |
        You are a code formatter.
        Format the code according to {{style}} style guide.
        Only return the formatted code, no explanations.
      prompt: |
        Format this {{language}} code:
        
        ```{{language}}
        {{code}}
        ```
      temperature: 0.1
    output: formatted_code
    when: "{{steps.try_formatter.success}} == false"
```

### 使用 code-formatter Skill

**在 Agent 中使用**:
```yaml
tools:
  - name: read_file
    type: builtin
  - name: write_file
    type: builtin
  - name: code_formatter
    type: skill
    skill_name: code-formatter

pipeline:
  - step: read_code
    tool: read_file
    args:
      path: "{{file_path}}"
    output: raw_code

  - step: format
    tool: code_formatter
    args:
      code: "{{steps.read_code.output}}"
      language: "typescript"
      style: "airbnb"
    output: formatted

  - step: write_back
    tool: write_file
    args:
      path: "{{file_path}}"
      content: "{{steps.format.output}}"
```

---

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)
- [Subagent System 规范](./subagent-system.md)

---

**Skill System - 可复用的单一功能模块** 🧩
