# Skill System 规范

**版本**: 3.1.0
**状态**: Draft
**最后更新**: 2026-06-23

---

## 概述

Skill System 让 Agent 能够包含和调用可复用的技能（Skills）。在 Agent Protocol v3 中，**Skill 是 Agent 的可复用能力模块，使用 `skill.json` 定义**。

v3.1 扩展引入了**市场引用模式**，Agent 可以通过 `ref` 引用市场中的 Skill，运行时自动解析和下载，实现 Skill 的独立分发与跨 Agent 复用。

### 核心理念

- **Skills 打包在 Agent 内** - 作为独立能力单元一起发布
- **自包含** - Agent 包含所有需要的 Skills，无需额外下载
- **可复用** - 多个 subagents 可以使用相同的 Skill
- **市场引用（v3.1 新增）** - 通过 `ref` 引用市场 Skill，运行时自动解析与缓存
- **混合模式（推荐）** - 内联打包与市场引用灵活组合，兼顾自包含与复用
- **独立格式** - Skill 是独立的能力单元，使用 `skill.json` 定义，与 Agent 格式解耦

---

## 1. Skill vs Agent

### 区别

| 特性 | Skill | Agent |
|------|-------|-------|
| **复杂度** | 简单，单一功能 | 复杂，多步骤工作流 |
| **Pipeline** | 1-3 步 | 多步骤，有条件分支 |
| **用途** | 被其他 subagents 调用 | 独立发布或包含 Skills |
| **发布方式** | 作为 Agent 的一部分 | 独立发布到 Market |
| **定义文件** | `skill.json` | `agent.json` |
| **示例** | "总结文本"、"翻译" | "内容处理器"（含多个 Skills） |

### Skill 标记

Skill 使用独立的 `skill.json` 文件定义，不再依赖 `agent.json` 中的 `type` 字段：

```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "description": "Summarizes text into concise bullet points"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-summarization",
    "bullet-point-generation"
  ]
}
```

---

## 2. Agent 包含 Skills

### Agent 目录结构

```
content-processor/              # 主 Agent
├── agent.json                  # 声明包含的 Skills
├── orchestrator.yaml           # 主工作流
├── skills/                     # Skills 目录
│   ├── text-summarizer/        # Skill 1
│   │   ├── skill.json          # Skill 定义
│   │   └── worker.yaml
│   ├── translator/             # Skill 2
│   │   ├── skill.json
│   │   └── worker.yaml
│   └── markdown-formatter/     # Skill 3
│       ├── skill.json
│       └── worker.yaml
└── README.md
```

### agent.json 声明

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "content-processor",
    "version": "1.0.0",
    "display_name": "Content Processor",
    "description": "Process content with summarization, translation, and formatting",
    "author": "Your Name",
    "tags": ["content", "nlp", "processing"]
  },
  "entry": {
    "main_subagent": "orchestrator"
  },
  "subagents": [
    {
      "name": "orchestrator",
      "path": "orchestrator.yaml",
      "description": "Main workflow coordinator"
    }
  ],
  "skills": [
    {
      "name": "text-summarizer",
      "display_name": "Text Summarizer",
      "description": "Summarizes text",
      "version": "1.0.0",
      "source": "local"
    },
    {
      "name": "translator",
      "display_name": "Translator",
      "description": "Translates text",
      "version": "1.0.0",
      "source": "local"
    },
    {
      "name": "markdown-formatter",
      "display_name": "Markdown Formatter",
      "description": "Formats markdown",
      "version": "1.0.0",
      "source": "local"
    }
  ]
}
```

### orchestrator.yaml 使用 Skills

```yaml
tools:
  - name: read_file
    type: builtin
  - name: write_file
    type: builtin

  # Skill 作为 subagent tool
  - name: summarize
    type: subagent
    subagent: text-summarizer

  - name: translate
    type: subagent
    subagent: translator

  - name: format_markdown
    type: subagent
    subagent: markdown-formatter

pipeline:
  # Step 1: 读取文件
  - step: read_content
    tool: read_file
    args:
      path: "{{file_path}}"
    output: raw_content

  # Step 2: 调用 Skill - 总结
  - step: summarize_content
    tool: summarize
    args:
      text: "{{steps.read_content.output}}"
      max_length: 200
    output: summary

  # Step 3: 调用 Skill - 翻译
  - step: translate_summary
    tool: translate
    args:
      text: "{{steps.summarize_content.output}}"
      target_lang: "{{target_lang}}"
    output: translated
    when: "{{target_lang}} != ''"

  # Step 4: 调用 Skill - 格式化
  - step: format_output
    tool: format_markdown
    args:
      text: "{{steps.translate_summary.output}}"
    output: formatted

  # Step 5: 写入文件
  - step: write_result
    tool: write_file
    args:
      path: "{{output_path}}"
      content: "{{steps.format_output.output}}"
```

---

## 3. Skill 定义

### skill.json 格式

Skill 使用独立的 `skill.json` 文件定义，格式如下：

```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "display_name": "Text Summarizer",
    "description": "Summarizes text into concise bullet points",
    "author": "Your Name",
    "license": "MIT",
    "tags": ["text", "summarization", "nlp"],
    "icon": "📝"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-summarization",
    "bullet-point-generation",
    "content-condensation"
  ],
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
  },
  "scripts": {
    "pre_install": "scripts/pre-install.sh",
    "post_run": "scripts/post-run.py"
  },
  "dependencies": {
    "nodejs": ">=18.0.0"
  }
}
```

#### 字段说明

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `schema_version` | string | 是 | Skill 规范版本，当前为 `1.0.0` |
| `identity` | object | 是 | Skill 身份标识 |
| `identity.name` | string | 是 | Skill 唯一名称（kebab-case） |
| `identity.version` | string | 是 | 语义化版本号 |
| `identity.display_name` | string | 否 | 展示名称 |
| `identity.description` | string | 是 | 一句话描述 |
| `identity.author` | string | 否 | 作者 |
| `identity.license` | string | 否 | 许可证 |
| `identity.tags` | string[] | 否 | 标签数组 |
| `identity.icon` | string | 否 | 图标（emoji 或 URL） |
| `content` | object | 是 | Skill 指令内容 |
| `content.format` | string | 是 | 内容格式：`markdown`、`text`、`yaml` |
| `content.source` | string | 是 | 内容来源：`inline`、`file` |
| `content.file` | string | 条件 | 当 `source` 为 `file` 时，指定文件路径 |
| `content.value` | string | 条件 | 当 `source` 为 `inline` 时，内联内容 |
| `capabilities` | string[] | 否 | Skill 提供的能力列表 |
| `parameters` | object | 否 | 参数定义（从 worker.yaml 移入） |
| `scripts` | object | 否 | 脚本钩子 |
| `dependencies` | object | 否 | 环境依赖 |

### Skill: text-summarizer

**skills/text-summarizer/skill.json**:
```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "description": "Summarizes text into concise bullet points"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-summarization",
    "bullet-point-generation"
  ],
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

**skills/text-summarizer/worker.yaml**:
```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: summarize
    tool: llm_chat
    args:
      system_prompt: |
        You are a professional summarizer.
        Summarize text into {{format}} format.
        Keep it under {{max_length}} characters.
      prompt: "{{text}}"
      temperature: 0.3
      max_tokens: 500
    output: summary
```

### Skill: translator

**skills/translator/skill.json**:
```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "translator",
    "version": "1.0.0",
    "description": "Translates text between languages"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-translation",
    "language-detection"
  ],
  "parameters": {
    "text": {
      "type": "string",
      "required": true,
      "description": "Text to translate"
    },
    "target_lang": {
      "type": "string",
      "required": true,
      "enum": ["en", "zh", "es", "fr", "de", "ja"],
      "description": "Target language"
    },
    "source_lang": {
      "type": "string",
      "default": "auto",
      "description": "Source language (auto-detect if not specified)"
    }
  }
}
```

**skills/translator/worker.yaml**:
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
        Translate from {{source_lang}} to {{target_lang}}:

        {{text}}

        Only return the translation, no explanations.
      temperature: 0.3
    output: translation
```

---

## 4. 发布和使用流程

### 发布 Agent (含 Skills)

```bash
cd content-processor/

# 目录结构:
# content-processor/
# ├── agent.json (声明 Skills)
# ├── orchestrator.yaml
# ├── skills/
# │   ├── text-summarizer/  ← Skill 1
# │   │   ├── skill.json
# │   │   └── worker.yaml
# │   ├── translator/       ← Skill 2
# │   │   ├── skill.json
# │   │   └── worker.yaml
# │   └── markdown-formatter/  ← Skill 3
# │       ├── skill.json
# │       └── worker.yaml
# └── README.md

# 发布到 Market
agent-deploy publish .

# 上传内容:
# ✅ agent.json
# ✅ orchestrator.yaml
# ✅ skills/ 目录（所有 Skills 一起打包）
```

### 用户下载和使用

```bash
# 下载 Agent
agent-deploy download content-processor

# 下载后目录:
# content-processor/
# ├── agent.json
# ├── orchestrator.yaml
# ├── skills/
# │   ├── text-summarizer/  ← Skills 已包含
# │   │   ├── skill.json
# │   │   └── worker.yaml
# │   ├── translator/
# │   │   ├── skill.json
# │   │   └── worker.yaml
# │   └── markdown-formatter/
# │       ├── skill.json
# │       └── worker.yaml
# └── README.md

# 运行 Agent (Skills 自动可用)
agent-deploy run ./content-processor \
  --args file_path=/data/report.txt \
  --args output_path=/tmp/summary.md \
  --args target_lang=zh

# 输出:
# 🚀 Running agent: content-processor
# 📂 Loading skills: text-summarizer, translator, markdown-formatter
# ✅ Skills loaded: 3
#
# [读取文件...]
# [调用 Skill: text-summarizer...]
# [调用 Skill: translator...]
# [调用 Skill: markdown-formatter...]
#
# ✅ Result written to: /tmp/summary.md
```

---

## 5. Runtime 实现

### Skill Loader

```typescript
// src/runtime/skills/loader.ts
export class SkillLoader {
  async loadSkillsFromAgent(agent: Agent): Promise<Map<string, Skill>> {
    const skills = new Map<string, Skill>();

    // 读取 agent.json 中的 skills 数组
    for (const skillDef of agent.skills || []) {
      // 加载 Skill 的 skill.json
      const skillPath = path.resolve(
        agent.basePath,
        "skills",
        skillDef.name,
        "skill.json"
      );

      const skill = await this.loadSkill(skillPath);
      skills.set(skillDef.name, skill);
    }

    return skills;
  }

  async loadSkill(skillPath: string): Promise<Skill> {
    const content = await fs.readFile(skillPath, "utf-8");
    const skillJson = JSON.parse(content);

    // 加载 SKILL.md 内容
    const skillDir = path.dirname(skillPath);
    const skillMdPath = path.join(skillDir, skillJson.content.file || "SKILL.md");
    const skillMdContent = await fs.readFile(skillMdPath, "utf-8");

    return {
      ...skillJson,
      skillMdContent
    };
  }
}
```

### Subagent Tool (用于调用 Skill)

```typescript
// src/runtime/tools/subagent-tool.ts
export class SubagentTool implements BuiltinTool {
  name: string;
  type = "subagent";

  constructor(
    private subagentName: string,
    private skill: Skill
  ) {
    this.name = `subagent_${subagentName}`;
  }

  async execute(args: any, context: ExecutionContext): Promise<any> {
    // 1. 验证参数
    this.validateParameters(args, this.skill.parameters);

    // 2. 创建隔离的执行上下文
    const subagentContext = {
      ...context,
      initialArgs: args,
      steps: new Map(),  // 隔离步骤上下文
      skill: this.skill
    };

    // 3. 执行 skill pipeline
    const engine = new PipelineEngine(context.toolRegistry, context.logger);
    const workerYaml = await this.loadWorkerYaml(this.skill);

    const result = await engine.execute(workerYaml, subagentContext);

    return result;
  }

  private validateParameters(
    args: Record<string, any>,
    parameters: ParameterSchema
  ) {
    if (!parameters) return;

    const validator = new SkillParameterValidator();
    const result = validator.validate(args, parameters);

    if (!result.valid) {
      throw new Error(
        `Invalid parameters for skill ${this.subagentName}: ` +
        result.errors.join(", ")
      );
    }
  }
}
```

### Parameter Validator

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
        // 允许额外参数
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

  private checkType(value: any, expectedType: string): boolean {
    switch (expectedType) {
      case "string": return typeof value === "string";
      case "number": return typeof value === "number";
      case "boolean": return typeof value === "boolean";
      case "array": return Array.isArray(value);
      case "object": return typeof value === "object" && !Array.isArray(value);
      default: return true;
    }
  }
}
```

---

## 6. Skill 组合模式

### 模式 1: 串行调用

```yaml
# orchestrator.yaml
pipeline:
  # Skill 1: 总结
  - step: summarize
    tool: summarize
    args:
      text: "{{content}}"
    output: summary

  # Skill 2: 翻译（使用 Skill 1 的输出）
  - step: translate
    tool: translate
    args:
      text: "{{steps.summarize.output}}"
      target_lang: "zh"
    output: translated

  # Skill 3: 格式化（使用 Skill 2 的输出）
  - step: format
    tool: format_markdown
    args:
      text: "{{steps.translate.output}}"
    output: formatted
```

### 模式 2: 条件调用

```yaml
pipeline:
  # 检测语言
  - step: detect_lang
    tool: llm_chat
    args:
      prompt: "Detect language of: {{content}}"
    output: lang

  # 如果不是英文，先翻译
  - step: translate_to_english
    tool: translate
    args:
      text: "{{content}}"
      target_lang: "en"
    output: english_text
    when: "{{steps.detect_lang.output}} != 'en'"

  # 使用英文版本进行分析
  - step: analyze
    tool: analyze_sentiment
    args:
      text: "{{steps.translate_to_english.output}}"
    when: "{{steps.translate_to_english.success}}"
```

### 模式 3: Skill 复用

```yaml
# 同一个 Skill 调用多次
pipeline:
  # 翻译成中文
  - step: translate_to_zh
    tool: translate
    args:
      text: "{{content}}"
      target_lang: "zh"
    output: zh_text

  # 翻译成日文
  - step: translate_to_ja
    tool: translate
    args:
      text: "{{content}}"
      target_lang: "ja"
    output: ja_text

  # 翻译成法文
  - step: translate_to_fr
    tool: translate
    args:
      text: "{{content}}"
      target_lang: "fr"
    output: fr_text
```

---

## 7. 完整示例

### Agent: content-processor

**项目结构**:
```
content-processor/
├── agent.json
├── orchestrator.yaml
├── skills/
│   ├── text-summarizer/
│   │   ├── skill.json
│   │   └── worker.yaml
│   ├── translator/
│   │   ├── skill.json
│   │   └── worker.yaml
│   └── code-formatter/
│       ├── skill.json
│       └── worker.yaml
└── README.md
```

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "content-processor",
    "version": "1.0.0",
    "display_name": "Content Processor",
    "description": "Multi-language content processing with summarization and formatting",
    "author": "Agent Protocol Team",
    "tags": ["content", "nlp", "translation"]
  },
  "entry": {
    "main_subagent": "orchestrator"
  },
  "subagents": [
    {
      "name": "orchestrator",
      "path": "orchestrator.yaml"
    }
  ],
  "skills": [
    {
      "name": "text-summarizer",
      "display_name": "Text Summarizer",
      "description": "Summarizes text into concise bullet points",
      "version": "1.0.0",
      "source": "local"
    },
    {
      "name": "translator",
      "display_name": "Translator",
      "description": "Translates text between languages",
      "version": "1.0.0",
      "source": "local"
    },
    {
      "name": "code-formatter",
      "display_name": "Code Formatter",
      "description": "Formats code in various languages",
      "version": "1.0.0",
      "source": "local"
    }
  ],
  "dependencies": {
    "llm_provider": "anthropic"
  }
}
```

**orchestrator.yaml**:
```yaml
tools:
  - name: read_file
    type: builtin
  - name: write_file
    type: builtin
  - name: summarize
    type: subagent
    subagent: text-summarizer
  - name: translate
    type: subagent
    subagent: translator
  - name: format_code
    type: subagent
    subagent: code-formatter

shared_context:
  output_dir: "{{output_dir}}"

pipeline:
  - step: read_input
    tool: read_file
    args:
      path: "{{file_path}}"
    output: content

  - step: summarize_content
    tool: summarize
    args:
      text: "{{steps.read_input.output}}"
      max_length: 300
      format: "bullets"
    output: summary

  - step: translate_summary
    tool: translate
    args:
      text: "{{steps.summarize_content.output}}"
      target_lang: "{{target_lang}}"
    output: translated
    when: "{{target_lang}} != ''"

  - step: write_output
    tool: write_file
    args:
      path: "{{shared_context.output_dir}}/summary.md"
      content: "{{steps.translated.output}}"
```

**skills/text-summarizer/skill.json**:
```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "display_name": "Text Summarizer",
    "description": "Summarizes text into concise bullet points"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-summarization",
    "bullet-point-generation"
  ],
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

**skills/translator/skill.json**:
```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "translator",
    "version": "1.0.0",
    "display_name": "Translator",
    "description": "Translates text between languages"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "text-translation",
    "language-detection"
  ],
  "parameters": {
    "text": {
      "type": "string",
      "required": true,
      "description": "Text to translate"
    },
    "target_lang": {
      "type": "string",
      "required": true,
      "enum": ["en", "zh", "es", "fr", "de", "ja"],
      "description": "Target language"
    },
    "source_lang": {
      "type": "string",
      "default": "auto",
      "description": "Source language (auto-detect if not specified)"
    }
  }
}
```

**skills/code-formatter/skill.json**:
```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "code-formatter",
    "version": "1.0.0",
    "display_name": "Code Formatter",
    "description": "Formats code in various languages"
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "code-formatting",
    "syntax-highlighting"
  ],
  "parameters": {
    "code": {
      "type": "string",
      "required": true,
      "description": "Code to format"
    },
    "language": {
      "type": "string",
      "required": true,
      "enum": ["javascript", "python", "go", "rust", "java"],
      "description": "Programming language"
    }
  }
}
```

### 使用示例

```bash
# 发布
cd content-processor/
agent-deploy publish .

# 下载
agent-deploy download content-processor

# 运行
agent-deploy run ./content-processor \
  --args file_path=/data/report.txt \
  --args output_dir=/tmp \
  --args target_lang=zh

# 输出:
# 🚀 Running agent: content-processor
# 📂 Loading 3 skills...
# ✅ Skills loaded: text-summarizer, translator, code-formatter
#
# [Pipeline 执行...]
# ✅ Summary written to: /tmp/summary.md
```

---

## 8. 最佳实践

### ✅ DO - 推荐做法

**1. Skills 打包在 Agent 内**:
```
my-agent/
├── agent.json
├── orchestrator.yaml
└── skills/           ← Skills 作为 Agent 的一部分
    ├── skill1/
    │   ├── skill.json
    │   └── worker.yaml
    └── skill2/
        ├── skill.json
        └── worker.yaml
```

**2. 单一职责**:
```json
// ✅ Good - 一个 Skill 做一件事
{
  "identity": {
    "name": "text-summarizer",
    "description": "Summarizes text"
  }
}

// ❌ Bad - 太多职责
{
  "identity": {
    "name": "text-processor",
    "description": "Summarizes, translates, and formats text"
  }
}
```

**3. 清晰的参数定义**:
```json
{
  "parameters": {
    "text": {
      "type": "string",
      "required": true,
      "description": "The text to summarize. Max 10,000 characters."
    },
    "max_length": {
      "type": "number",
      "default": 200,
      "description": "Maximum summary length in characters"
    }
  }
}
```

**4. 避免副作用**:
```yaml
# ✅ Good - Skill 只返回结果
- step: process
  tool: llm_chat
  output: result

# ❌ Bad - Skill 修改文件系统
- step: process_and_save
  tool: write_file
```

### ❌ DON'T - 避免做法

**1. 推荐市场引用模式实现 Skill 复用**:
```json
// ✅ Good - 通过市场引用复用 Skill
{
  "skills": [
    {
      "ref": "html-anything",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market"
    }
  ]
}

// ✅ Good - 混合模式：引用 + 本地
{
  "skills": [
    {
      "ref": "html-anything",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market"
    },
    {
      "name": "custom-skill",
      "display_name": "Custom Skill",
      "description": "Agent-specific custom skill",
      "version": "1.0.0",
      "source": "local"
    }
  ]
}
```

**2. 不要过度依赖**:
```json
// ❌ Bad - 依赖太多 Skills
{
  "skills": [
    {"name": "skill1"},
    {"name": "skill2"},
    // ... 20+ skills
  ]
}

// ✅ Good - 最小 Skills 集合
{
  "skills": [
    {"name": "summarizer", "source": "local"},
    {"name": "translator", "source": "local"}
  ]
}
```

**3. 不要 Skills 间相互依赖**:
```yaml
# ❌ Bad - Skill A 调用 Skill B
# skills/skill-a/worker.yaml
- step: call_skill_b
  tool: skill_b  # Skill 不应该调用其他 Skills

# ✅ Good - 由 orchestrator 协调
# orchestrator.yaml
- step: use_skill_a
  tool: skill_a
- step: use_skill_b
  tool: skill_b
```

---

## 9. 市场引用模式（v3.1 扩展）

v3.1 在原有内联打包的基础上，新增了**市场引用模式**，允许 Agent 通过引用标识从云端市场获取 Skill，运行时自动解析、下载并缓存。

### 9.1 三种使用模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **内联打包**（现有） | Skill 完整定义在 Agent 的 `skills/` 目录中 | Agent 专属 Skill、离线场景 |
| **市场引用**（新增） | 通过 `ref` 引用市场 Skill，运行时自动下载 | 通用 Skill、跨 Agent 复用 |
| **混合**（推荐） | 内联 + 引用灵活组合 | 大多数生产场景 |

```
模式 A: 内联打包（现有）          模式 B: 市场引用（新增）           模式 C: 混合（推荐）
┌─────────────────────┐          ┌─────────────────────┐          ┌─────────────────────┐
│ my-agent/           │          │ my-agent/           │          │ my-agent/           │
│ ├── agent.json      │          │ ├── agent.json      │          │ ├── agent.json      │
│ │   └── skills: [   │          │ │   └── skills: [   │          │ │   └── skills: [   │
│ │       {name,      │          │ │       {ref,       │          │ │       {ref,       │
│ │        display,   │          │ │        version,   │          │ │        version},  │
│ │        desc,      │          │ │        market_url}│          │ │       {name,      │
│ │        version,   │          │ │   ]               │          │ │        display,   │
│ │        source:    │          │                     │          │ │        ...}       │
│ │        "local"}   │          │ 运行时自动下载:      │          │ │   ]               │
│ │   ]               │          │ ~/.agent-hub/cache/ │          │ ├── skills/         │
│ └── skills/         │          │   └── skills/       │          │ │   └── local-skill/│
│     └── skill-a/    │          │       └── skill-a/  │          │ │       ├── skill.json│
│         ├── skill.json│        │           └── ...   │          │ │       └── worker.yaml│
│         └── worker.yaml│       │                     │          │ └─────────────────────┘
└─────────────────────┘          └─────────────────────┘
     自包含但臃肿                    精简但依赖网络
```

### 9.2 SkillRef 引用格式

在 agent.json 的 `skills` 数组中，使用以下字段实现市场引用：

```typescript
interface SkillRef {
  // 模式 A: 本地内联定义
  name?: string;
  display_name?: string;
  description?: string;
  version?: string;
  category?: string;
  icon?: string;
  parameters?: Record<string, any>;

  // 模式 B: 市场引用（v3.1 新增）
  ref?: string;           // 引用标识: "html-anything" 或 "openpeng/html-anything"
  version?: string;       // 版本约束: "^1.0.0", ">=2.0.0", "*"
  market_url?: string;    // 市场地址: "https://market.aitboy.cn"
  source?: "local" | "market";  // 来源类型
}
```

**解析规则**:
- 如果 `ref` 存在 → 市场引用模式
- 如果 `name` 存在且无 `ref` → 本地内联模式
- `source` 显式声明时以 `source` 为准

#### 版本约束语法

| 语法 | 含义 | 示例 |
|------|------|------|
| `1.0.0` | 精确版本 | 只匹配 1.0.0 |
| `^1.0.0` | 兼容版本 | 匹配 1.x.x，不匹配 2.0.0 |
| `~1.0.0` | 近似版本 | 匹配 1.0.x，不匹配 1.1.0 |
| `>=1.0.0` | 大于等于 | 匹配 1.0.0 及以上 |
| `*` | 任意版本 | 匹配最新版本 |

#### 引用示例

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "content-processor",
    "version": "1.0.0",
    "description": "Content processing with referenced skills"
  },
  "skills": [
    {
      "ref": "html-anything",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market"
    },
    {
      "ref": "text-summarizer",
      "version": "~2.1.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market"
    }
  ]
}
```

### 9.3 Skill 独立包规范

市场引用的 Skill 以独立包形式发布，包含以下文件：

```
html-anything-skill/
├── skill.json          # Skill 元数据（必需）
├── SKILL.md            # Skill 指令内容（必需）
├── scripts/            # 可执行脚本（可选）
│   ├── pre-install.sh  # 安装前钩子
│   └── post-run.py     # 运行后钩子
├── templates/          # 模板文件（可选）
│   └── default.html
└── README.md           # 使用说明（可选）
```

#### skill.json 格式

```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "html-anything",
    "version": "1.0.0",
    "display_name": "HTML Anything",
    "description": "将 Markdown、CSV、Excel 等内容转换为精美可发布的 HTML",
    "author": "Open Design Team",
    "license": "MIT",
    "tags": ["html", "design", "publish", "markdown"]
  },
  "content": {
    "format": "markdown",
    "source": "file",
    "file": "SKILL.md"
  },
  "capabilities": [
    "html-generation",
    "markdown-to-html",
    "presentation-creation"
  ],
  "scripts": {
    "pre_install": "scripts/pre-install.sh",
    "post_run": "scripts/post-run.py"
  },
  "dependencies": {
    "nodejs": ">=18.0.0"
  }
}
```

**打包规则**:
- tar.gz 格式，顶层目录名为 `{name}-v{version}/`
- 必须包含 `skill.json` 和 `SKILL.md`
- 可选包含 `scripts/`、`templates/`、`README.md`
- 总大小不超过 10MB

### 9.4 运行时引用解析

Agent 启动时，运行时自动解析所有 `ref` 引用并加载对应 Skill：

```
Agent 启动
  │
  ▼
读取 agent.json
  │
  ▼
解析 skills 数组 ──┬── 本地定义 → 直接从 skills/ 目录加载
                  └── 引用定义 → 解析引用
                                    │
                                    ▼
                              检查本地缓存
                              ~/.agent-hub/cache/skills/{ref}@{version}/
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                      缓存命中            缓存未命中
                          │                   │
                          ▼                   ▼
                      直接使用          从市场下载
                      加载 SKILL.md     解压到缓存目录
                      加载 scripts      验证 skill.json
                          │                   │
                          └─────────┬─────────┘
                                    ▼
                              合并到 Agent 上下文
                              (SKILL.md → system prompt)
```

### 9.5 缓存策略

```
~/.agent-hub/cache/
├── skills/
│   ├── html-anything@1.0.0/
│   │   ├── skill.json
│   │   ├── SKILL.md
│   │   └── scripts/
│   ├── text-summarizer@2.1.0/
│   └── ...
└── index.json          # 缓存索引: {ref: {version, path, downloaded_at, etag}}
```

**缓存规则**:
- 按 `ref@resolved_version` 目录存储
- 下载时记录 `etag`，启动时检查是否需要更新
- `version: "*"` 或 `^x.x.x` 时，每日检查一次最新版本
- 缓存清理: `agent-deploy skill cache clean --unused-for 30d`

### 9.6 CLI 命令

```bash
# 打包 Skill
agent-deploy skill pack <path> [-o, --output <file>]

# 验证 Skill 包
agent-deploy skill verify <file>

# 上传 Skill 到市场
agent-deploy skill upload <path> [-m, --market <url>] [-k, --api-key <key>] [-f, --force]

# 从市场下载 Skill
agent-deploy skill download <ref> [-v, --version <ver>] [-o, --output <dir>]

# 列出已缓存的 Skills
agent-deploy skill list --cached

# 搜索市场 Skills
agent-deploy skill search <query> [-m, --market <url>]

# 清理 Skill 缓存
agent-deploy skill cache clean [--unused-for <days>] [--ref <ref>]
```

### 9.7 向后兼容性

| 场景 | 兼容性 |
|------|------|
| 现有内联 skills 数组 | 完全兼容，无 `ref` 字段时按内联处理 |
| 现有 `skills/` 目录打包 | 完全兼容，`skill.json` 格式已更新 |
| 新引用模式 Agent | 需要运行时支持引用解析（agent-compose 升级后） |

---

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)
- [Subagent System 规范](./subagent-system.md)

---

**Skill System - Skills 打包在 Agent 内，开箱即用** 🧩
