# Memory System 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

Memory System 让 Agent 能够记忆和学习，实现自我提升。记忆系统采用**分层存储架构**，将记忆与 Agent 代码分离，确保 Agent 的可移植性和可回滚性。

### 核心理念

✅ **记忆与代码分离** - 记忆存储在用户目录，不污染 Agent 源代码  
✅ **分层存储** - Agent 级（跨项目）和项目级（特定项目）  
✅ **可控可清除** - 用户随时查看、导出、清除记忆  
✅ **自我提升** - Agent 通过记忆学习用户偏好和成功模式

---

## 1. 记忆存储结构

### 目录布局

```
~/.agent-deploy/memory/
├── agents/                           # Agent 级记忆（跨项目）
│   ├── code-reviewer/
│   │   ├── long-term.json            # 长期记忆
│   │   └── patterns.json             # 学习到的模式
│   ├── file-summarizer/
│   │   └── long-term.json
│   └── tapd-task-manager/
│       └── long-term.json
│
└── projects/                         # 项目级记忆
    ├── abc123def456/                 # 项目路径 MD5 hash
    │   ├── meta.json                 # 项目元信息
    │   ├── agents/
    │   │   ├── code-reviewer.json    # 该项目中 code-reviewer 的记忆
    │   │   └── file-summarizer.json
    │   └── shared.json               # 项目共享记忆
    └── xyz789abc012/
        ├── meta.json
        └── agents/
            └── code-reviewer.json
```

### 项目路径映射

```typescript
// 项目路径 → hash
function hashProjectPath(path: string): string {
  return crypto
    .createHash("md5")
    .update(path.resolve(path))
    .digest("hex")
    .substring(0, 16);
}

// 示例:
// /home/user/my-project → abc123def456
```

### meta.json 格式

```json
{
  "project_path": "/home/user/my-project",
  "created_at": "2026-06-07T10:00:00Z",
  "last_accessed": "2026-06-07T15:30:00Z",
  "agents_used": ["code-reviewer", "file-summarizer"]
}
```

---

## 2. 记忆数据结构

### Memory 对象

```typescript
interface Memory {
  agent_name: string;           // Agent 名称
  project_path?: string;        // 项目路径（项目级记忆）
  created_at: string;           // ISO 8601 格式
  updated_at: string;
  entries: MemoryEntry[];       // 记忆条目列表
}
```

### MemoryEntry 对象

```typescript
interface MemoryEntry {
  id: string;                   // 唯一 ID，如 "mem_1733567890_xyz"
  type: MemoryType;
  content: string;              // 记忆内容
  metadata: MemoryMetadata;
  created_at: string;
  accessed_count: number;       // 访问次数
  last_accessed: string;
}

type MemoryType = 
  | "preference"    // 用户偏好
  | "error"         // 错误和修正
  | "pattern"       // 成功的模式
  | "context"       // 项目上下文
  | "learning";     // 学习到的知识

interface MemoryMetadata {
  confidence: number;           // 0-1，记忆的可信度
  source: "user" | "self" | "external";
  tags: string[];               // 标签，如 ["code-style", "user-preference"]
  related_to?: string[];        // 关联的其他记忆 ID
}
```

---

## 3. 记忆示例

### Agent 长期记忆（跨项目）

**路径**: `~/.agent-deploy/memory/agents/code-reviewer/long-term.json`

```json
{
  "agent_name": "code-reviewer",
  "created_at": "2026-06-01T10:00:00Z",
  "updated_at": "2026-06-07T15:30:00Z",
  "entries": [
    {
      "id": "mem_1733567890_abc",
      "type": "preference",
      "content": "用户喜欢简洁的审查报告，每个问题用一句话描述，不要冗长解释",
      "metadata": {
        "confidence": 0.95,
        "source": "user",
        "tags": ["report-style", "user-preference", "brevity"]
      },
      "created_at": "2026-06-01T10:30:00Z",
      "accessed_count": 15,
      "last_accessed": "2026-06-07T14:20:00Z"
    },
    {
      "id": "mem_1733567891_def",
      "type": "pattern",
      "content": "async/await 函数缺少 try-catch 是常见问题，需要重点检查",
      "metadata": {
        "confidence": 0.88,
        "source": "self",
        "tags": ["code-pattern", "error-handling", "async"]
      },
      "created_at": "2026-06-03T11:00:00Z",
      "accessed_count": 8,
      "last_accessed": "2026-06-07T15:10:00Z"
    },
    {
      "id": "mem_1733567892_ghi",
      "type": "learning",
      "content": "TypeScript 项目中，any 类型通常是代码异味，建议用 unknown 或具体类型",
      "metadata": {
        "confidence": 0.82,
        "source": "self",
        "tags": ["typescript", "type-safety", "best-practice"]
      },
      "created_at": "2026-06-05T09:15:00Z",
      "accessed_count": 5,
      "last_accessed": "2026-06-07T13:45:00Z"
    }
  ]
}
```

### 项目记忆（特定项目）

**路径**: `~/.agent-deploy/memory/projects/abc123def456/agents/code-reviewer.json`

```json
{
  "agent_name": "code-reviewer",
  "project_path": "/home/user/my-typescript-project",
  "created_at": "2026-06-05T10:00:00Z",
  "updated_at": "2026-06-07T15:30:00Z",
  "entries": [
    {
      "id": "mem_p_1733567893_jkl",
      "type": "context",
      "content": "这个项目使用 TypeScript strict mode，所有类型必须明确声明",
      "metadata": {
        "confidence": 1.0,
        "source": "self",
        "tags": ["project-config", "typescript", "strict-mode"]
      },
      "created_at": "2026-06-05T10:30:00Z",
      "accessed_count": 12,
      "last_accessed": "2026-06-07T15:20:00Z"
    },
    {
      "id": "mem_p_1733567894_mno",
      "type": "preference",
      "content": "团队约定：所有 public API 函数必须有完整的 JSDoc 注释",
      "metadata": {
        "confidence": 0.98,
        "source": "user",
        "tags": ["team-convention", "documentation", "jsdoc"]
      },
      "created_at": "2026-06-05T14:00:00Z",
      "accessed_count": 9,
      "last_accessed": "2026-06-07T14:50:00Z"
    },
    {
      "id": "mem_p_1733567895_pqr",
      "type": "error",
      "content": "之前遗漏了检查 React Hooks 依赖数组，导致 bug。现在需要重点检查",
      "metadata": {
        "confidence": 0.92,
        "source": "self",
        "tags": ["react", "hooks", "past-error"]
      },
      "created_at": "2026-06-06T11:20:00Z",
      "accessed_count": 4,
      "last_accessed": "2026-06-07T15:00:00Z"
    }
  ]
}
```

---

## 4. Builtin Tools

### memory_read - 读取记忆

```yaml
tools:
  - name: memory_read
    type: builtin
```

**参数**:
```typescript
{
  scope: "agent" | "project" | "all";    // 记忆范围
  tags?: string[];                        // 按标签过滤
  types?: MemoryType[];                   // 按类型过滤
  limit?: number;                         // 最多返回条数（默认 10）
  min_confidence?: number;                // 最小置信度（默认 0.5）
}
```

**返回**:
```typescript
MemoryEntry[]  // 记忆条目数组
```

**示例**:
```yaml
- step: recall_preferences
  tool: memory_read
  args:
    scope: "all"
    tags: ["user-preference", "report-style"]
    limit: 5
  output: preferences
```

### memory_write - 写入记忆

```yaml
tools:
  - name: memory_write
    type: builtin
```

**参数**:
```typescript
{
  scope: "agent" | "project";             // 存储范围
  type: MemoryType;                       // 记忆类型
  content: string;                        // 记忆内容
  tags?: string[];                        // 标签
  confidence?: number;                    // 置信度（默认 0.7）
  replace_tag?: string;                   // 如果存在相同 tag，替换
}
```

**返回**:
```typescript
{
  id: string;        // 新记忆的 ID
  replaced?: string; // 如果替换了旧记忆，返回旧 ID
}
```

**示例**:
```yaml
- step: save_learning
  tool: memory_write
  args:
    scope: "agent"
    type: "learning"
    content: "发现新的代码模式：React 组件使用 memo 优化性能"
    tags: ["react", "performance", "optimization"]
    confidence: 0.85
  output: saved_memory
```

### memory_search - 搜索记忆

```yaml
tools:
  - name: memory_search
    type: builtin
```

**参数**:
```typescript
{
  query: string;                          // 搜索关键词
  scope?: "agent" | "project" | "all";    // 范围（默认 "all"）
  limit?: number;                         // 最多返回条数（默认 10）
}
```

**返回**:
```typescript
{
  results: Array<{
    entry: MemoryEntry;
    score: number;      // 相关性分数 0-1
  }>;
}
```

**示例**:
```yaml
- step: search_related
  tool: memory_search
  args:
    query: "TypeScript 类型检查"
    scope: "all"
    limit: 5
  output: related_memories
```

### memory_forget - 删除记忆

```yaml
tools:
  - name: memory_forget
    type: builtin
```

**参数**:
```typescript
{
  id?: string;                            // 删除指定 ID
  tags?: string[];                        // 删除匹配标签的
  older_than?: string;                    // 删除早于此日期的（ISO 8601）
  confidence_below?: number;              // 删除置信度低于此值的
}
```

**返回**:
```typescript
{
  deleted_count: number;
  deleted_ids: string[];
}
```

**示例**:
```yaml
# 删除低置信度的旧记忆
- step: cleanup_memories
  tool: memory_forget
  args:
    older_than: "2026-01-01T00:00:00Z"
    confidence_below: 0.5
  output: cleanup_result
```

---

## 5. 在 Pipeline 中使用记忆

### 示例 1: 自学习的代码审查 Agent

```yaml
# code-reviewer/worker.yaml
tools:
  - name: memory_read
    type: builtin
  - name: memory_write
    type: builtin
  - name: llm_chat
    type: builtin

shared_context:
  code: "{{code}}"
  user_feedback: "{{user_feedback}}"

pipeline:
  # Step 1: 回忆相关记忆
  - step: recall_memories
    tool: memory_read
    args:
      scope: "all"
      tags: ["user-preference", "code-pattern", "past-error"]
      limit: 10
    output: memories

  # Step 2: 结合记忆进行代码审查
  - step: review_code
    tool: llm_chat
    args:
      system_prompt: |
        You are a code reviewer with learned knowledge.
        
        **Remembered preferences and patterns**:
        {% for memory in steps.recall_memories.output %}
        - [{{ memory.type }}] {{ memory.content }} (confidence: {{ memory.metadata.confidence }})
        {% endfor %}
        
        Apply these learned patterns while reviewing.
      prompt: |
        Review this code:
        
        ```
        {{ shared_context.code }}
        ```
        
        Provide a concise review report.
      temperature: 0.3
    output: review

  # Step 3: 如果有用户反馈，分析学习点
  - step: analyze_feedback
    tool: llm_chat
    args:
      prompt: |
        User feedback on the review: {{ shared_context.user_feedback }}
        Original review: {{ steps.review_code.output }}
        
        Extract what I should learn or remember for future reviews.
        Return JSON:
        {
          "learned": "...",          // What to remember
          "type": "preference|pattern|error",
          "tags": ["...", "..."],
          "confidence": 0.0-1.0
        }
        
        If no learning point, return: {"learned": null}
      temperature: 0.2
    output: learning
    when: "{{ shared_context.user_feedback }} != ''"
    on_fail: skip

  # Step 4: 保存学习结果
  - step: save_learning
    tool: memory_write
    args:
      scope: "agent"
      type: "{{ steps.analyze_feedback.output.type }}"
      content: "{{ steps.analyze_feedback.output.learned }}"
      tags: "{{ steps.analyze_feedback.output.tags }}"
      confidence: "{{ steps.analyze_feedback.output.confidence }}"
    when: "{{ steps.analyze_feedback.output.learned }} != null"
    on_fail: skip
```

### 示例 2: 项目上下文记忆

```yaml
# project-analyzer/worker.yaml
tools:
  - name: memory_read
    type: builtin
  - name: memory_write
    type: builtin
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin

pipeline:
  # Step 1: 读取项目配置
  - step: read_package_json
    tool: read_file
    args:
      path: "package.json"
    output: package_json
    on_fail: skip

  # Step 2: 提取项目上下文
  - step: extract_context
    tool: llm_chat
    args:
      prompt: |
        Analyze this package.json:
        {{ steps.read_package_json.output }}
        
        Extract key project context:
        - Framework and version
        - Key dependencies
        - Build tools
        - Code style tools
        
        Return JSON with extracted info.
    output: context

  # Step 3: 保存项目上下文记忆
  - step: save_context
    tool: memory_write
    args:
      scope: "project"
      type: "context"
      content: "{{ steps.extract_context.output }}"
      tags: ["project-config", "dependencies"]
      confidence: 1.0
      replace_tag: "project-config"  // 替换旧的项目配置
```

---

## 6. Runtime 实现

### MemoryManager 类

```typescript
// src/runtime/memory/manager.ts
import * as fs from "fs";
import * as path from "path";
import * as os from "os";
import * as crypto from "crypto";

export class MemoryManager {
  private basePath: string;

  constructor() {
    this.basePath = path.join(os.homedir(), ".agent-deploy", "memory");
  }

  // 加载 Agent 长期记忆
  async loadAgentMemory(agentName: string): Promise<Memory> {
    const memoryPath = path.join(
      this.basePath,
      "agents",
      agentName,
      "long-term.json"
    );

    if (!fs.existsSync(memoryPath)) {
      return this.createEmptyMemory(agentName);
    }

    return JSON.parse(fs.readFileSync(memoryPath, "utf-8"));
  }

  // 加载项目记忆
  async loadProjectMemory(
    projectPath: string,
    agentName: string
  ): Promise<Memory | null> {
    const projectHash = this.hashProjectPath(projectPath);
    const memoryPath = path.join(
      this.basePath,
      "projects",
      projectHash,
      "agents",
      `${agentName}.json`
    );

    if (!fs.existsSync(memoryPath)) {
      return null;
    }

    return JSON.parse(fs.readFileSync(memoryPath, "utf-8"));
  }

  // 保存 Agent 记忆
  async saveAgentMemory(agentName: string, entry: MemoryEntry) {
    const memory = await this.loadAgentMemory(agentName);
    memory.entries.push(entry);
    memory.updated_at = new Date().toISOString();

    // 限制记忆数量（保留最重要的）
    if (memory.entries.length > 1000) {
      memory.entries = this.pruneMemories(memory.entries, 800);
    }

    const memoryPath = path.join(
      this.basePath,
      "agents",
      agentName,
      "long-term.json"
    );

    fs.mkdirSync(path.dirname(memoryPath), { recursive: true });
    fs.writeFileSync(memoryPath, JSON.stringify(memory, null, 2));
  }

  // 保存项目记忆
  async saveProjectMemory(
    projectPath: string,
    agentName: string,
    entry: MemoryEntry
  ) {
    const projectHash = this.hashProjectPath(projectPath);
    
    // 确保 meta.json 存在
    await this.ensureProjectMeta(projectPath, projectHash);

    const memoryPath = path.join(
      this.basePath,
      "projects",
      projectHash,
      "agents",
      `${agentName}.json`
    );

    let memory: Memory;
    if (fs.existsSync(memoryPath)) {
      memory = JSON.parse(fs.readFileSync(memoryPath, "utf-8"));
    } else {
      memory = this.createEmptyMemory(agentName, projectPath);
    }

    memory.entries.push(entry);
    memory.updated_at = new Date().toISOString();

    // 限制项目记忆数量
    if (memory.entries.length > 500) {
      memory.entries = this.pruneMemories(memory.entries, 400);
    }

    fs.mkdirSync(path.dirname(memoryPath), { recursive: true });
    fs.writeFileSync(memoryPath, JSON.stringify(memory, null, 2));
  }

  // 记录访问
  async recordAccess(memoryId: string) {
    // 在所有记忆文件中查找并更新访问记录
    // 实现省略...
  }

  // 按标签删除
  async removeByTag(
    scope: "agent" | "project",
    agentName: string,
    tag: string,
    projectPath?: string
  ) {
    // 实现省略...
  }

  // 裁剪记忆（保留重要的）
  private pruneMemories(
    entries: MemoryEntry[],
    targetCount: number
  ): MemoryEntry[] {
    // 计算重要性分数
    const scored = entries.map(entry => {
      const recencyScore = 1 / (
        (Date.now() - new Date(entry.last_accessed).getTime()) / (1000 * 60 * 60 * 24) + 1
      );
      const accessScore = Math.log(entry.accessed_count + 1);
      const confidenceScore = entry.metadata.confidence;

      return {
        entry,
        score: recencyScore * 0.3 + accessScore * 0.3 + confidenceScore * 0.4
      };
    });

    // 按分数排序，保留前 N 个
    scored.sort((a, b) => b.score - a.score);
    return scored.slice(0, targetCount).map(s => s.entry);
  }

  private hashProjectPath(projectPath: string): string {
    return crypto
      .createHash("md5")
      .update(path.resolve(projectPath))
      .digest("hex")
      .substring(0, 16);
  }

  private createEmptyMemory(
    agentName: string,
    projectPath?: string
  ): Memory {
    return {
      agent_name: agentName,
      project_path: projectPath,
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
      entries: []
    };
  }

  private async ensureProjectMeta(projectPath: string, projectHash: string) {
    const metaPath = path.join(
      this.basePath,
      "projects",
      projectHash,
      "meta.json"
    );

    if (!fs.existsSync(metaPath)) {
      const meta = {
        project_path: path.resolve(projectPath),
        created_at: new Date().toISOString(),
        last_accessed: new Date().toISOString(),
        agents_used: []
      };

      fs.mkdirSync(path.dirname(metaPath), { recursive: true });
      fs.writeFileSync(metaPath, JSON.stringify(meta, null, 2));
    }
  }
}
```

### Memory Tools 实现

```typescript
// src/runtime/tools/memory-read.ts
export class MemoryReadTool implements BuiltinTool {
  name = "memory_read";

  async execute(args: {
    scope: "agent" | "project" | "all";
    tags?: string[];
    types?: MemoryType[];
    limit?: number;
    min_confidence?: number;
  }, context: ExecutionContext): Promise<MemoryEntry[]> {
    const memoryManager = new MemoryManager();
    let memories: MemoryEntry[] = [];

    // 1. 读取 Agent 长期记忆
    if (args.scope === "agent" || args.scope === "all") {
      const agentMemory = await memoryManager.loadAgentMemory(
        context.agent.identity.name
      );
      memories.push(...agentMemory.entries);
    }

    // 2. 读取项目记忆
    if (args.scope === "project" || args.scope === "all") {
      const projectMemory = await memoryManager.loadProjectMemory(
        context.cwd,
        context.agent.identity.name
      );
      if (projectMemory) {
        memories.push(...projectMemory.entries);
      }
    }

    // 3. 按 tags 过滤
    if (args.tags && args.tags.length > 0) {
      memories = memories.filter(m =>
        m.metadata.tags.some(tag => args.tags!.includes(tag))
      );
    }

    // 4. 按 types 过滤
    if (args.types && args.types.length > 0) {
      memories = memories.filter(m => args.types!.includes(m.type));
    }

    // 5. 按置信度过滤
    const minConfidence = args.min_confidence || 0.5;
    memories = memories.filter(m => m.metadata.confidence >= minConfidence);

    // 6. 按重要性排序
    memories.sort((a, b) => {
      const scoreA = this.calculateScore(a);
      const scoreB = this.calculateScore(b);
      return scoreB - scoreA;
    });

    // 7. 限制数量
    const limit = args.limit || 10;
    const result = memories.slice(0, limit);

    // 8. 更新访问记录
    for (const memory of result) {
      memory.accessed_count++;
      memory.last_accessed = new Date().toISOString();
      await memoryManager.recordAccess(memory.id);
    }

    return result;
  }

  private calculateScore(memory: MemoryEntry): number {
    const recency = Date.now() - new Date(memory.last_accessed).getTime();
    const recencyScore = 1 / (recency / (1000 * 60 * 60 * 24) + 1); // 天数
    const accessScore = Math.log(memory.accessed_count + 1);
    const confidenceScore = memory.metadata.confidence;

    return recencyScore * 0.3 + accessScore * 0.3 + confidenceScore * 0.4;
  }
}
```

```typescript
// src/runtime/tools/memory-write.ts
export class MemoryWriteTool implements BuiltinTool {
  name = "memory_write";

  async execute(args: {
    scope: "agent" | "project";
    type: MemoryType;
    content: string;
    tags?: string[];
    confidence?: number;
    replace_tag?: string;
  }, context: ExecutionContext): Promise<{ id: string; replaced?: string }> {
    const memoryManager = new MemoryManager();

    // 如果指定替换，先删除旧的
    let replacedId: string | undefined;
    if (args.replace_tag) {
      replacedId = await memoryManager.removeByTag(
        args.scope,
        context.agent.identity.name,
        args.replace_tag,
        args.scope === "project" ? context.cwd : undefined
      );
    }

    // 创建新记忆
    const entry: MemoryEntry = {
      id: `mem_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      type: args.type,
      content: args.content,
      metadata: {
        confidence: args.confidence || 0.7,
        source: "self",
        tags: args.tags || []
      },
      created_at: new Date().toISOString(),
      accessed_count: 0,
      last_accessed: new Date().toISOString()
    };

    // 保存
    if (args.scope === "agent") {
      await memoryManager.saveAgentMemory(
        context.agent.identity.name,
        entry
      );
    } else {
      await memoryManager.saveProjectMemory(
        context.cwd,
        context.agent.identity.name,
        entry
      );
    }

    return {
      id: entry.id,
      replaced: replacedId
    };
  }
}
```

---

## 7. CLI 命令

### 查看记忆

```bash
# 查看 Agent 长期记忆
agent-deploy memory list code-reviewer

# 输出:
# Agent: code-reviewer
# Total memories: 15
# 
# 1. [preference] 用户喜欢简洁的审查报告... (confidence: 0.95)
#    Tags: report-style, user-preference
#    Accessed: 15 times, Last: 2026-06-07 14:20
# 
# 2. [pattern] async/await 函数缺少 try-catch... (confidence: 0.88)
#    Tags: code-pattern, error-handling
#    Accessed: 8 times, Last: 2026-06-07 15:10

# 查看项目记忆
agent-deploy memory list code-reviewer --project ./my-project

# 按标签过滤
agent-deploy memory list code-reviewer --tags user-preference,report-style

# 查看详细内容
agent-deploy memory show mem_1733567890_abc
```

### 清除记忆

```bash
# 清除 Agent 所有记忆（需确认）
agent-deploy memory clear code-reviewer

# 清除项目记忆
agent-deploy memory clear code-reviewer --project ./my-project

# 清除特定标签
agent-deploy memory clear code-reviewer --tags outdated

# 清除低置信度记忆
agent-deploy memory clear code-reviewer --confidence-below 0.5

# 清除旧记忆
agent-deploy memory clear code-reviewer --older-than 2026-01-01
```

### 导出/导入记忆

```bash
# 导出 Agent 记忆
agent-deploy memory export code-reviewer > code-reviewer-memories.json

# 导出项目记忆
agent-deploy memory export code-reviewer --project ./my-project > project-memories.json

# 导入记忆
agent-deploy memory import code-reviewer < code-reviewer-memories.json

# 导入时合并（不覆盖现有）
agent-deploy memory import code-reviewer --merge < memories.json
```

### 统计信息

```bash
# 显示记忆统计
agent-deploy memory stats

# 输出:
# Memory Statistics
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 
# Agent Memories:
#   code-reviewer: 15 entries (last updated: 2 hours ago)
#   file-summarizer: 8 entries (last updated: 1 day ago)
#   tapd-task-manager: 5 entries (last updated: 3 days ago)
# 
# Project Memories:
#   /home/user/project-1: 3 agents, 12 entries
#   /home/user/project-2: 2 agents, 7 entries
# 
# Total: 47 memory entries
# Disk usage: 156 KB
```

---

## 8. 最佳实践

### ✅ DO - 推荐做法

**1. 明确记忆范围**
```yaml
# ✅ Good - Agent 级记忆用于通用知识
- tool: memory_write
  args:
    scope: "agent"
    type: "pattern"
    content: "异步函数要加 try-catch"

# ✅ Good - 项目级记忆用于特定上下文
- tool: memory_write
  args:
    scope: "project"
    type: "context"
    content: "这个项目使用 TypeScript strict mode"
```

**2. 使用合适的置信度**
```yaml
# ✅ Good - 用户明确反馈 → 高置信度
- tool: memory_write
  args:
    confidence: 0.95
    source: "user"

# ✅ Good - 自我学习 → 中等置信度
- tool: memory_write
  args:
    confidence: 0.7
    source: "self"
```

**3. 添加有用的标签**
```yaml
# ✅ Good - 多个相关标签
- tool: memory_write
  args:
    tags: ["typescript", "type-safety", "best-practice"]
```

**4. 定期清理**
```yaml
# 在 pipeline 开始时清理低质量记忆
- step: cleanup_old_memories
  tool: memory_forget
  args:
    older_than: "{{ 6_months_ago }}"
    confidence_below: 0.5
  on_fail: skip
```

### ❌ DON'T - 避免做法

**1. 不要存储敏感信息**
```yaml
# ❌ Bad - 不要存储密码、API Key
- tool: memory_write
  args:
    content: "TAPD_API_KEY=sk-xxxxx"  # 不要这样！

# ✅ Good - 只记忆使用模式
- tool: memory_write
  args:
    content: "用户配置了 TAPD 集成"
```

**2. 不要无限累积**
```yaml
# ❌ Bad - 每次执行都添加记忆
- tool: memory_write
  args:
    content: "执行了一次审查"  # 会无限增长

# ✅ Good - 只记忆有价值的信息
- tool: memory_write
  args:
    content: "发现新的代码模式..."
    replace_tag: "code-pattern-xyz"  # 替换而不是累积
```

**3. 不要存储大文件内容**
```yaml
# ❌ Bad - 存储完整代码
- tool: memory_write
  args:
    content: "{{ entire_file_content }}"  # 太大

# ✅ Good - 存储摘要或模式
- tool: memory_write
  args:
    content: "文件使用了 React Hooks 模式"
```

---

## 9. 自提升示例

### 学习用户偏好

```yaml
# 用户第一次使用
pipeline:
  - step: review
    tool: llm_chat
    args:
      prompt: "Review code..."
    output: review

# 用户反馈："太冗长了，简洁点"

  - step: learn_preference
    tool: memory_write
    args:
      scope: "agent"
      type: "preference"
      content: "用户喜欢简洁的报告，每个问题一句话"
      tags: ["user-preference", "report-style"]
      confidence: 0.9

# 下次运行时
pipeline:
  - step: recall
    tool: memory_read
    args:
      tags: ["user-preference"]
    output: preferences

  - step: review
    tool: llm_chat
    args:
      system_prompt: |
        Preferences: {{ steps.recall.output }}
        Keep reports concise.
    output: review  # 现在会生成简洁的报告
```

### 记住常见错误

```yaml
# 第一次遗漏了检查
pipeline:
  - step: review
    output: review  # 遗漏了 React Hooks 依赖问题

# 用户指出："你忘了检查 useEffect 依赖数组"

  - step: learn_error
    tool: memory_write
    args:
      scope: "project"
      type: "error"
      content: "之前遗漏了 React Hooks 依赖检查，需要重点关注"
      tags: ["react", "hooks", "past-error"]
      confidence: 0.92

# 下次运行
pipeline:
  - step: recall_errors
    tool: memory_read
    args:
      tags: ["past-error"]
    output: past_errors

  - step: review
    tool: llm_chat
    args:
      system_prompt: |
        Important: Learn from past mistakes:
        {{ steps.recall_errors.output }}
```

---

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)
- [Builtin Tools 规范](./builtin-tools.md)

---

**Memory System - Agent 自我提升的关键** 🧠

