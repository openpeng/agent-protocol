# Builtin Tools 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

Builtin Tools 是 Agent Runtime 提供的标准工具集，无需额外配置即可在 worker.yaml 中使用。

**核心设计目标**：
- ✅ 开箱即用
- ✅ 跨平台一致
- ✅ 安全可控
- ✅ 性能优化

---

## 工具列表

| 工具 | 功能 | 必需依赖 |
|------|------|---------|
| `read_file` | 读取文件内容 | 无 |
| `write_file` | 写入文件内容 | 无 |
| `bash` | 执行 Shell 命令 | Shell (bash/sh) |
| `glob` | 文件模式匹配 | 无 |
| `llm_chat` | 调用大模型 | LLM API Key |
| `web_fetch` | HTTP 请求 | 无 |
| `web_search` | 网络搜索 | 搜索引擎 API |

---

## 1. read_file

读取本地文件内容。

### 参数

```typescript
interface ReadFileArgs {
  path: string;           // ✅ 必填：文件路径（支持相对/绝对路径）
  encoding?: string;      // 编码（默认 utf-8）
  max_size?: number;      // 最大读取字节数（默认 10MB）
}
```

### 返回值

```typescript
type ReadFileOutput = string;  // 文件内容
```

### 示例

```yaml
tools:
  - name: read_file
    type: builtin

pipeline:
  - step: read_config
    tool: read_file
    args:
      path: "config.json"
    output: config_content

  - step: read_large_file
    tool: read_file
    args:
      path: "/data/large.txt"
      max_size: 1048576  # 1MB
    output: partial_content
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `FileNotFoundError` | 文件不存在 | `skip` 或 `abort` |
| `PermissionError` | 无读权限 | `abort` |
| `FileSizeExceeded` | 文件超过 max_size | `abort` |

---

## 2. write_file

写入内容到文件。

### 参数

```typescript
interface WriteFileArgs {
  path: string;           // ✅ 必填：文件路径
  content: string;        // ✅ 必填：写入内容
  mode?: "overwrite" | "append";  // 写入模式（默认 overwrite）
  create_dirs?: boolean;  // 自动创建父目录（默认 true）
  encoding?: string;      // 编码（默认 utf-8）
}
```

### 返回值

```typescript
interface WriteFileOutput {
  path: string;           // 写入的文件路径
  bytes_written: number;  // 写入字节数
}
```

### 示例

```yaml
pipeline:
  - step: write_output
    tool: write_file
    args:
      path: "output/result.txt"
      content: "{{steps.analyze.output}}"
      mode: "overwrite"
      create_dirs: true
    output: write_result

  - step: append_log
    tool: write_file
    args:
      path: "logs/agent.log"
      content: "[{{timestamp}}] Task completed\n"
      mode: "append"
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `PermissionError` | 无写权限 | `abort` |
| `DiskFullError` | 磁盘空间不足 | `abort` |
| `PathTooLong` | 路径超过系统限制 | `abort` |

---

## 3. bash

执行 Shell 命令。

### 参数

```typescript
interface BashArgs {
  command: string;        // ✅ 必填：Shell 命令
  cwd?: string;          // 工作目录（默认当前目录）
  timeout?: number;      // 超时时间（秒，默认 60）
  capture_output?: boolean;  // 捕获输出（默认 true）
  env?: Record<string, string>;  // 环境变量
}
```

### 返回值

```typescript
interface BashOutput {
  stdout: string;         // 标准输出
  stderr: string;         // 标准错误
  exit_code: number;      // 退出码
  duration_ms: number;    // 执行时间（毫秒）
}
```

### 示例

```yaml
pipeline:
  # 基本命令
  - step: list_files
    tool: bash
    args:
      command: "ls -la"
    output: file_list

  # 带超时的命令
  - step: long_task
    tool: bash
    args:
      command: "python train_model.py"
      timeout: 3600  # 1 小时
    on_fail: abort

  # 管道命令
  - step: process_data
    tool: bash
    args:
      command: |
        cat data.json | jq '.items[] | select(.active == true)' | wc -l
    output: active_count

  # 设置环境变量
  - step: build
    tool: bash
    args:
      command: "npm run build"
      env:
        NODE_ENV: "production"
        API_KEY: "{{env.API_KEY}}"
```

### 安全限制

**默认限制**：
- ❌ 禁止 `rm -rf /`
- ❌ 禁止 `sudo` 命令
- ❌ 禁止修改系统关键文件
- ⚠️  超时后自动终止进程

**沙箱模式**（可选）：
```yaml
tools:
  - name: bash
    type: builtin
    config:
      sandbox: true  # 启用沙箱（仅允许读取，不允许写入/网络）
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `CommandNotFound` | 命令不存在 | `skip` 或 `abort` |
| `TimeoutError` | 超时 | `retry(n)` |
| `NonZeroExit` | 退出码非 0 | `continue` 或 `abort` |

---

## 4. glob

文件模式匹配，查找符合条件的文件列表。

### 参数

```typescript
interface GlobArgs {
  pattern: string;        // ✅ 必填：Glob 模式
  cwd?: string;          // 搜索目录（默认当前目录）
  max_results?: number;  // 最多返回结果数（默认 1000）
  ignore?: string[];     // 忽略模式
}
```

### 返回值

```typescript
type GlobOutput = string[];  // 匹配的文件路径数组
```

### 模式语法

| 模式 | 匹配 | 示例 |
|------|------|------|
| `*` | 任意字符（不包括 `/`） | `*.txt` |
| `**` | 任意字符（包括 `/`） | `src/**/*.ts` |
| `?` | 单个字符 | `file?.txt` |
| `[abc]` | 字符集 | `file[123].txt` |
| `{a,b}` | 多选一 | `*.{js,ts}` |

### 示例

```yaml
pipeline:
  # 查找所有 TypeScript 文件
  - step: find_ts_files
    tool: glob
    args:
      pattern: "src/**/*.ts"
    output: ts_files

  # 查找测试文件（排除 node_modules）
  - step: find_tests
    tool: glob
    args:
      pattern: "**/*.test.js"
      ignore:
        - "node_modules/**"
        - "dist/**"
    output: test_files

  # 查找配置文件
  - step: find_configs
    tool: glob
    args:
      pattern: "*.{json,yaml,yml}"
      cwd: "./config"
    output: config_files
```

### 性能优化

**对于大型仓库**：
```yaml
- step: find_files
  tool: glob
  args:
    pattern: "src/**/*.ts"
    max_results: 100  # 限制结果数
    ignore:
      - "**/node_modules/**"
      - "**/dist/**"
      - "**/.git/**"
```

---

## 5. llm_chat

调用大语言模型进行对话。

### 参数

```typescript
interface LLMChatArgs {
  prompt: string;             // ✅ 必填：用户消息
  system_prompt?: string;     // 系统提示词
  model?: string;             // 模型名称
  temperature?: number;       // 温度（0-1，默认 0.7）
  max_tokens?: number;        // 最大生成 token 数
  provider?: "anthropic" | "openai" | "azure";  // 提供商
  api_key?: string;           // API Key（覆盖全局配置）
  api_base?: string;          // API 端点（覆盖全局配置）
}
```

### 返回值

```typescript
interface LLMChatOutput {
  content: string;            // 模型回复
  model: string;              // 使用的模型
  tokens_used: number;        // 消耗的 token 数
  duration_ms: number;        // 请求时长
}
```

### 配置继承

**全局配置**（主 Agent 级别）：
```json
// agent.json
{
  "dependencies": {
    "llm_provider": "anthropic"
  },
  "llm_config": {
    "api_key": "${ANTHROPIC_API_KEY}",
    "model": "claude-3-5-sonnet-20241022",
    "temperature": 0.7
  }
}
```

**步骤覆盖**：
```yaml
pipeline:
  # 使用全局配置
  - step: normal_chat
    tool: llm_chat
    args:
      prompt: "Hello"
    output: response

  # 覆盖模型
  - step: use_opus
    tool: llm_chat
    args:
      prompt: "Complex task"
      model: "claude-3-opus-20240229"  # 覆盖
    output: opus_response

  # 完全自定义
  - step: use_openai
    tool: llm_chat
    args:
      prompt: "Test"
      provider: "openai"
      api_key: "{{env.OPENAI_API_KEY}}"
      model: "gpt-4"
    output: openai_response
```

### 示例

```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  # 基本对话
  - step: greet
    tool: llm_chat
    args:
      prompt: "Say hello to {{user_name}}"
      system_prompt: "You are a friendly assistant"
    output: greeting

  # 代码分析
  - step: analyze_code
    tool: llm_chat
    args:
      system_prompt: |
        You are a code reviewer.
        Find bugs and security issues.
      prompt: |
        Review this code:
        ```typescript
        {{steps.read_code.output}}
        ```
      model: "claude-3-5-sonnet-20241022"
      temperature: 0.3  # 更保守
    output: review

  # 结构化输出
  - step: extract_data
    tool: llm_chat
    args:
      system_prompt: |
        Extract structured data and return JSON only.
      prompt: |
        Extract person info from:
        {{steps.read_text.output}}
        
        Return format:
        {"name": "...", "age": ..., "email": "..."}
      temperature: 0.0  # 确定性输出
    output: extracted_json
```

### Fallback 处理

```yaml
pipeline:
  # 尝试调用 LLM
  - step: try_llm
    tool: llm_chat
    args:
      prompt: "Translate: {{text}}"
    on_fail: continue  # LLM 不可用时继续
    output: llm_result

  # Fallback 到规则方法
  - step: fallback_translation
    tool: bash
    args:
      command: "python simple_translate.py '{{text}}'"
    when: "{{steps.try_llm.success}} == false"
    output: fallback_result

  # 选择结果
  - step: select_result
    tool: bash
    args:
      command: |
        if [ "{{steps.try_llm.success}}" == "true" ]; then
          echo "{{steps.try_llm.output}}"
        else
          echo "{{steps.fallback_result.output}}"
        fi
    output: final_translation
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `AuthenticationError` | API Key 无效 | `abort` |
| `RateLimitError` | 请求频率过高 | `retry(3)` |
| `TokenLimitExceeded` | 输入超过限制 | `abort` |
| `ModelNotFound` | 模型不存在 | `abort` |
| `TimeoutError` | 请求超时 | `retry(2)` |

---

## 6. web_fetch

发送 HTTP 请求，抓取网页或 API 数据。

### 参数

```typescript
interface WebFetchArgs {
  url: string;                    // ✅ 必填：目标 URL
  method?: "GET" | "POST" | "PUT" | "DELETE";  // 方法（默认 GET）
  headers?: Record<string, string>;  // 请求头
  body?: string;                  // 请求体（POST/PUT）
  timeout?: number;               // 超时（秒，默认 30）
  follow_redirects?: boolean;     // 跟随重定向（默认 true）
  max_size?: number;              // 最大响应大小（字节，默认 10MB）
}
```

### 返回值

```typescript
interface WebFetchOutput {
  status: number;                 // HTTP 状态码
  headers: Record<string, string>;  // 响应头
  body: string;                   // 响应体
  url: string;                    // 最终 URL（重定向后）
}
```

### 示例

```yaml
pipeline:
  # GET 请求
  - step: fetch_data
    tool: web_fetch
    args:
      url: "https://api.example.com/data"
    output: api_response

  # POST 请求
  - step: submit_data
    tool: web_fetch
    args:
      url: "https://api.example.com/submit"
      method: "POST"
      headers:
        Content-Type: "application/json"
        Authorization: "Bearer {{env.API_TOKEN}}"
      body: |
        {
          "data": "{{steps.prepare.output}}"
        }
    output: submit_response

  # 下载文件
  - step: download_file
    tool: web_fetch
    args:
      url: "https://example.com/file.txt"
      timeout: 120
      max_size: 52428800  # 50MB
    output: file_content

  # 抓取网页并解析
  - step: fetch_page
    tool: web_fetch
    args:
      url: "https://example.com"
    output: html

  - step: extract_title
    tool: bash
    args:
      command: |
        echo '{{steps.fetch_page.output.body}}' | grep -oP '(?<=<title>).*(?=</title>)'
    output: page_title
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `ConnectionError` | 无法连接 | `retry(3)` |
| `TimeoutError` | 超时 | `retry(2)` |
| `HTTPError` | 4xx/5xx 状态码 | `continue` 或 `abort` |
| `TooLargeError` | 响应超过 max_size | `abort` |

---

## 7. web_search

使用搜索引擎查找信息。

### 参数

```typescript
interface WebSearchArgs {
  query: string;              // ✅ 必填：搜索查询
  engine?: "google" | "bing" | "duckduckgo";  // 搜索引擎（默认 google）
  max_results?: number;       // 最多返回结果数（默认 10）
  language?: string;          // 语言（默认 en）
  region?: string;            // 地区（如 us, cn）
}
```

### 返回值

```typescript
interface WebSearchOutput {
  results: Array<{
    title: string;            // 标题
    url: string;              // URL
    snippet: string;          // 摘要
    position: number;         // 排名
  }>;
  query: string;              // 实际搜索查询
  total_results: number;      // 总结果数（估计）
}
```

### 配置要求

**需要配置搜索引擎 API**：
```json
// agent.json
{
  "dependencies": {
    "web_search": "google"
  },
  "search_config": {
    "google_api_key": "${GOOGLE_API_KEY}",
    "google_cx": "${GOOGLE_CX}"
  }
}
```

### 示例

```yaml
tools:
  - name: web_search
    type: builtin

pipeline:
  # 基本搜索
  - step: search
    tool: web_search
    args:
      query: "{{user_question}}"
      max_results: 5
    output: search_results

  # 使用搜索结果
  - step: summarize_results
    tool: llm_chat
    args:
      prompt: |
        Summarize these search results:
        {{steps.search.output.results}}
    output: summary

  # 多语言搜索
  - step: search_chinese
    tool: web_search
    args:
      query: "人工智能"
      language: "zh-CN"
      region: "cn"
    output: cn_results
```

### 错误处理

| 错误 | 原因 | on_fail 建议 |
|------|------|-------------|
| `APIKeyMissing` | 未配置 API Key | `abort` |
| `QuotaExceeded` | API 配额用尽 | `skip` |
| `InvalidQuery` | 查询无效 | `skip` |

---

## 工具开发指南

### 实现接口

```typescript
// src/runtime/tools/tool.ts
export interface BuiltinTool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  execute(args: any, context: ExecutionContext): Promise<any>;
}

export interface ExecutionContext {
  agent: Agent;
  shared_context: Record<string, any>;
  env: Record<string, string>;
  cwd: string;
}
```

### 示例：实现自定义工具

```typescript
// src/runtime/tools/read-file.ts
import { BuiltinTool, ExecutionContext } from "./tool.js";
import { readFileSync } from "fs";
import { join } from "path";

export class ReadFileTool implements BuiltinTool {
  name = "read_file";
  description = "Read file content";
  
  inputSchema = {
    type: "object",
    properties: {
      path: { type: "string", description: "File path" },
      encoding: { type: "string", default: "utf-8" },
      max_size: { type: "number", default: 10485760 }  // 10MB
    },
    required: ["path"]
  };

  async execute(args: any, context: ExecutionContext): Promise<string> {
    const { path, encoding = "utf-8", max_size = 10485760 } = args;
    
    // 解析路径（支持相对路径）
    const fullPath = path.startsWith("/") 
      ? path 
      : join(context.cwd, path);
    
    // 检查文件大小
    const stats = statSync(fullPath);
    if (stats.size > max_size) {
      throw new Error(`File size (${stats.size}) exceeds max_size (${max_size})`);
    }
    
    // 读取文件
    return readFileSync(fullPath, encoding);
  }
}
```

### 注册工具

```typescript
// src/runtime/tools/registry.ts
import { ReadFileTool } from "./read-file.js";
import { WriteFileTool } from "./write-file.js";
import { BashTool } from "./bash.js";
// ...

export class ToolRegistry {
  private tools = new Map<string, BuiltinTool>();

  constructor() {
    this.registerDefaults();
  }

  private registerDefaults() {
    this.register(new ReadFileTool());
    this.register(new WriteFileTool());
    this.register(new BashTool());
    this.register(new GlobTool());
    this.register(new LLMChatTool());
    this.register(new WebFetchTool());
    this.register(new WebSearchTool());
  }

  register(tool: BuiltinTool) {
    this.tools.set(tool.name, tool);
  }

  get(name: string): BuiltinTool | null {
    return this.tools.get(name) || null;
  }
}
```

---

## 参考

- [worker.yaml 规范](./worker-yaml.md)
- [agent.json v3 规范](./agent-json-v3.md)
- [工具开发指南](./tool-development.md)

---

**Builtin Tools - 开箱即用的标准工具集** 🛠️
