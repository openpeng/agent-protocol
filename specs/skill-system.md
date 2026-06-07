# Skill System 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

Skill System 让 Agent 能够包含和调用可复用的技能（Skills）。在 Agent Protocol v3 中，**Skills 作为 Agent 的一部分打包发布**，而不是从独立的 Skill 市场下载。

### 核心理念

✅ **Skills 打包在 Agent 内** - 作为 subagents 一起发布  
✅ **自包含** - Agent 包含所有需要的 Skills，无需额外下载  
✅ **可复用** - 多个 subagents 可以使用相同的 Skill

---

## 1. Skill vs Agent

### 区别

| 特性 | Skill | Agent |
|------|-------|-------|
| **复杂度** | 简单，单一功能 | 复杂，多步骤工作流 |
| **Pipeline** | 1-3 步 | 多步骤，有条件分支 |
| **用途** | 被其他 subagents 调用 | 独立发布或包含 Skills |
| **发布方式** | 作为 Agent 的一部分 | 独立发布到 Market |
| **示例** | "总结文本"、"翻译" | "内容处理器"（含多个 Skills） |

### Skill 标记

在 agent.json 中使用 `"type": "skill"` 标记：

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

---

## 2. Agent 包含 Skills

### Agent 目录结构

```
content-processor/              # 主 Agent
├── agent.json                  # 声明包含的 Skills
├── orchestrator.yaml           # 主工作流
├── skills/                     # Skills 目录
│   ├── text-summarizer/        # Skill 1
│   │   ├── agent.json          # type: "skill"
│   │   └── worker.yaml
│   ├── translator/             # Skill 2
│   │   ├── agent.json
│   │   └── worker.yaml
│   └── markdown-formatter/     # Skill 3
│       ├── agent.json
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
    },
    {
      "name": "text-summarizer",
      "path": "skills/text-summarizer/worker.yaml",
      "description": "Summarizes text",
      "type": "skill"  // ← 标记为 skill
    },
    {
      "name": "translator",
      "path": "skills/translator/worker.yaml",
      "description": "Translates text",
      "type": "skill"
    },
    {
      "name": "markdown-formatter",
      "path": "skills/markdown-formatter/worker.yaml",
      "description": "Formats markdown",
      "type": "skill"
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

### Skill: text-summarizer

**skills/text-summarizer/agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "text-summarizer",
    "version": "1.0.0",
    "description": "Summarizes text into concise bullet points"
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

**skills/translator/agent.json**:
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
# │   ├── translator/       ← Skill 2
# │   └── markdown-formatter/  ← Skill 3
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
# │   ├── translator/
# │   └── markdown-formatter/
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
  async loadSkillsFromAgent(agent: Agent): Promise<Map<string, Agent>> {
    const skills = new Map<string, Agent>();

    // 遍历 subagents，找到 type: "skill" 的
    for (const subagentDef of agent.subagents) {
      if (subagentDef.type !== "skill") {
        continue;
      }

      // 加载 Skill 的 agent.json
      const skillPath = path.resolve(
        agent.basePath,
        path.dirname(subagentDef.path),
        "agent.json"
      );

      const skillAgent = await this.loadAgent(skillPath);

      // 验证是 Skill 类型
      if (skillAgent.type !== "skill") {
        console.warn(`Subagent ${subagentDef.name} declared as skill but type is not "skill"`);
      }

      skills.set(subagentDef.name, skillAgent);
    }

    return skills;
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
    private subagentAgent: Agent
  ) {
    this.name = `subagent_${subagentName}`;
  }

  async execute(args: any, context: ExecutionContext): Promise<any> {
    // 1. 验证参数 (如果是 Skill)
    if (this.subagentAgent.type === "skill") {
      this.validateParameters(args, this.subagentAgent.parameters);
    }

    // 2. 创建隔离的执行上下文
    const subagentContext = {
      ...context,
      initialArgs: args,
      steps: new Map(),  // 隔离步骤上下文
      agent: this.subagentAgent
    };

    // 3. 执行 subagent pipeline
    const engine = new PipelineEngine(context.toolRegistry, context.logger);
    const entryYaml = this.subagentAgent.subagents.get(
      this.subagentAgent.entry.main_subagent
    );
    
    const result = await engine.execute(entryYaml, subagentContext);

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
│   │   ├── agent.json
│   │   └── worker.yaml
│   ├── translator/
│   │   ├── agent.json
│   │   └── worker.yaml
│   └── code-formatter/
│       ├── agent.json
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
    },
    {
      "name": "text-summarizer",
      "path": "skills/text-summarizer/worker.yaml",
      "type": "skill"
    },
    {
      "name": "translator",
      "path": "skills/translator/worker.yaml",
      "type": "skill"
    },
    {
      "name": "code-formatter",
      "path": "skills/code-formatter/worker.yaml",
      "type": "skill"
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
    └── skill2/
```

**2. 单一职责**:
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

**1. 不要依赖外部 Skill 市场**:
```json
// ❌ Bad - 依赖外部下载
{
  "dependencies": {
    "skills": ["marketplace://text-summarizer"]
  }
}

// ✅ Good - Skills 打包在 Agent 内
{
  "subagents": [
    {
      "name": "text-summarizer",
      "path": "skills/text-summarizer/worker.yaml",
      "type": "skill"
    }
  ]
}
```

**2. 不要过度依赖**:
```json
// ❌ Bad - 依赖太多 Skills
{
  "subagents": [
    {"name": "skill1", "type": "skill"},
    {"name": "skill2", "type": "skill"},
    // ... 20+ skills
  ]
}

// ✅ Good - 最小 Skills 集合
{
  "subagents": [
    {"name": "summarizer", "type": "skill"},
    {"name": "translator", "type": "skill"}
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

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)
- [Subagent System 规范](./subagent-system.md)

---

**Skill System - Skills 打包在 Agent 内，开箱即用** 🧩
