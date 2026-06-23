# MCP Integration 规范

**版本**: 3.1.0
**状态**: Draft
**最后更新**: 2026-06-23

---

## 概述

MCP (Model Context Protocol) 集成让 Agent 能够调用外部工具和服务。在 Agent Protocol v3 中，MCP 配置可以**打包在 Agent 内部**一起发布，用户下载 Agent 时自动获得 MCP 集成能力。

**v3.1 扩展**：新增**市场引用模式**，MCP Server 可独立打包发布到云端市场，Agent 通过 `ref` 引用而非内联打包，运行时自动解析、下载、缓存。支持内联、引用、混合三种模式。

### 核心理念

✅ **Agent 是完整的包** - MCP 配置随 Agent 一起发布
✅ **开箱即用** - 用户只需配置 API Key，无需手动安装 MCP servers
✅ **可移植** - Agent 在任何环境都能运行（本地 / 平台）
✅ **按需引用** - MCP Server 可独立发布到市场，Agent 通过引用按需加载（v3.1 新增）
✅ **缓存复用** - 引用的 MCP Server 自动缓存到本地，避免重复下载（v3.1 新增）

---

## 1. Agent 包含 MCP 配置

### Agent 目录结构

```
my-agent/
├── agent.json              # 声明需要的 MCP servers
├── worker.yaml             # 使用 MCP tools 的 pipeline
├── mcp/                    # MCP 配置（打包在 Agent 内）
│   ├── servers.json        # MCP server 配置
│   └── README.md           # 用户配置说明
└── README.md
```

### agent.json 声明

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "tapd-task-manager",
    "version": "1.0.0",
    "description": "Manages TAPD tasks using MCP tools"
  },
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml"
    }
  ],
  "mcp": {
    "config_path": "./mcp/servers.json",
    "required_servers": [
      {
        "name": "tapd",
        "description": "TAPD project management MCP server",
        "package": "@openpeng/mcp-tapd",
        "version": "^1.0.0",
        "tools": [
          "tapd_create_story",
          "tapd_list_stories",
          "tapd_update_story"
        ],
        "required_env": [
          "TAPD_API_KEY",
          "TAPD_WORKSPACE_ID"
        ],
        "optional": false
      }
    ]
  }
}
```

**关键字段**:
- `mcp.config_path` - MCP 配置文件路径（相对于 agent.json）
- `mcp.required_servers[].package` - NPM 包名（用于自动安装）
- `mcp.required_servers[].version` - 版本约束
- `mcp.required_servers[].required_env` - 必需的环境变量（用户需配置）

### mcp/servers.json 配置

```json
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd@^1.0.0"],
      "env": {
        "TAPD_API_KEY": "${TAPD_API_KEY}",
        "TAPD_WORKSPACE_ID": "${TAPD_WORKSPACE_ID}"
      }
    }
  }
}
```

**环境变量引用**: `${VAR_NAME}` 从用户环境读取

### mcp/README.md 用户说明

```markdown
# MCP 配置说明

本 Agent 需要以下 MCP servers：

## TAPD Server

**用途**: 创建和管理 TAPD 任务

**配置步骤**:

1. 获取 TAPD API Key:
   - 登录 TAPD → 个人设置 → API
   - 生成新的 API Token

2. 设置环境变量:
   ```bash
   export TAPD_API_KEY=your_api_key_here
   export TAPD_WORKSPACE_ID=12345
   ```

3. 运行 Agent:
   ```bash
   agent-deploy run ./tapd-task-manager
   ```

**首次运行**: MCP server 会自动下载（使用 npx）
```

---

## 2. worker.yaml 中使用 MCP Tools

### 声明 MCP Tool

```yaml
tools:
  - name: llm_chat
    type: builtin
  
  # MCP tool 声明
  - name: tapd_create_story
    type: mcp
    server: tapd  # 对应 agent.json 中的 server name

pipeline:
  - step: analyze_requirement
    tool: llm_chat
    args:
      system_prompt: "Extract story details from user input."
      prompt: "{{user_input}}"
    output: story_details

  # 使用 MCP tool
  - step: create_story
    tool: tapd_create_story
    args:
      workspace_id: "{{workspace_id}}"
      name: "{{steps.analyze_requirement.output.title}}"
      description: "{{steps.analyze_requirement.output.description}}"
    output: created_story
    on_fail: abort  # MCP 调用失败 → 终止

  - step: report
    tool: llm_chat
    args:
      prompt: |
        Story created:
        - ID: {{steps.create_story.output.id}}
        - URL: {{steps.create_story.output.url}}
    output: result
```

---

## 3. 发布和使用流程

### 发布 Agent (含 MCP 配置)

```bash
# Agent 作者打包
cd my-agent/

# 目录结构:
# my-agent/
# ├── agent.json (声明 MCP servers)
# ├── worker.yaml
# ├── mcp/
# │   ├── servers.json (MCP 配置)
# │   └── README.md (用户说明)
# └── README.md

# 发布到 Market
agent-deploy publish .

# 上传内容:
# ✅ agent.json
# ✅ worker.yaml
# ✅ mcp/servers.json (MCP 配置一起打包)
# ✅ mcp/README.md
```

### 用户下载和使用

```bash
# 用户下载 Agent
agent-deploy download tapd-task-manager

# 下载后目录:
# tapd-task-manager/
# ├── agent.json
# ├── worker.yaml
# ├── mcp/
# │   ├── servers.json  ← MCP 配置已包含
# │   └── README.md     ← 配置说明
# └── README.md

# 1. 查看 MCP 配置要求
cat tapd-task-manager/mcp/README.md

# 2. 配置环境变量
export TAPD_API_KEY=your_key
export TAPD_WORKSPACE_ID=12345

# 3. 运行 Agent (首次会自动安装 MCP server)
agent-deploy run ./tapd-task-manager \
  --args user_input="Create a login page"

# 输出:
# 🔌 Installing MCP server: tapd (@openpeng/mcp-tapd@^1.0.0)
# ✅ MCP server ready: tapd (3 tools available)
# 🚀 Running agent: tapd-task-manager
# ✅ Story created: #67890
```

---

## 4. Runtime 实现

### MCP Loader

```typescript
// src/runtime/mcp/loader.ts
export class MCPLoader {
  async loadFromAgent(agent: Agent): Promise<Map<string, MCPTool>> {
    const tools = new Map<string, MCPTool>();
    
    if (!agent.mcp) {
      return tools;
    }

    // 1. 读取 Agent 内置的 MCP 配置
    const configPath = path.resolve(
      agent.basePath,
      agent.mcp.config_path || "./mcp/servers.json"
    );
    const mcpConfig = JSON.parse(fs.readFileSync(configPath, "utf-8"));

    // 2. 验证环境变量
    this.validateEnvironment(agent.mcp.required_servers);

    // 3. 为每个 server 创建 client
    for (const serverDef of agent.mcp.required_servers) {
      const serverConfig = mcpConfig.servers[serverDef.name];
      if (!serverConfig) {
        throw new Error(`MCP server config not found: ${serverDef.name}`);
      }

      // 4. 启动 MCP server
      const client = await this.startServer(serverConfig);

      // 5. 列举 tools
      const availableTools = await client.listTools();

      // 6. 注册到工具系统
      for (const toolName of serverDef.tools) {
        const toolDef = availableTools.find(t => t.name === toolName);
        if (!toolDef) {
          if (!serverDef.optional) {
            throw new Error(`Required MCP tool not found: ${toolName}`);
          }
          continue;
        }

        const mcpTool = new MCPTool(client, toolDef);
        tools.set(toolName, mcpTool);
      }
    }

    return tools;
  }

  private validateEnvironment(servers: MCPServerDefinition[]) {
    const missing: string[] = [];

    for (const server of servers) {
      for (const envVar of server.required_env || []) {
        if (!process.env[envVar]) {
          missing.push(`${server.name}: ${envVar}`);
        }
      }
    }

    if (missing.length > 0) {
      throw new Error(
        `Missing required environment variables:\n` +
        missing.map(m => `  - ${m}`).join("\n") +
        `\n\nPlease see mcp/README.md for setup instructions.`
      );
    }
  }

  private async startServer(config: MCPServerConfig): Promise<MCPClient> {
    // 替换环境变量
    const env = this.resolveEnv(config.env);

    // 启动 MCP server process
    const client = new MCPClient({
      command: config.command,
      args: config.args,
      env
    });

    await client.connect();
    return client;
  }

  private resolveEnv(
    env: Record<string, string>
  ): Record<string, string> {
    const resolved: Record<string, string> = {};

    for (const [key, value] of Object.entries(env)) {
      // ${VAR_NAME} → process.env.VAR_NAME
      resolved[key] = value.replace(
        /\$\{(\w+)\}/g,
        (_, varName) => process.env[varName] || ""
      );
    }

    return resolved;
  }
}
```

### MCP Tool 类型

```typescript
// src/runtime/tools/mcp-tool.ts
export class MCPTool implements BuiltinTool {
  name: string;
  type = "mcp";
  server: string;
  
  constructor(
    private client: MCPClient,
    private toolDef: MCPToolDefinition
  ) {
    this.name = toolDef.name;
    this.server = toolDef.server;
  }

  async execute(args: any, context: ExecutionContext): Promise<any> {
    try {
      const result = await this.client.callTool(this.name, args);
      return result;
    } catch (error) {
      if (error instanceof MCPConnectionError) {
        throw new ToolError(
          `MCP server '${this.server}' not available. ` +
          `Check that required environment variables are set.`,
          "MCP_SERVER_UNAVAILABLE"
        );
      }
      throw error;
    }
  }
}
```

---

## 5. Deploy 模式（部署到平台）

### 场景

Agent 部署到支持 MCP 的平台（如 Claude Code），利用平台配置的 MCP servers。

### Deploy 行为

```bash
agent-deploy deploy ./tapd-task-manager -t claude_code
```

**生成的 `.claude/commands/tapd-task-manager.md`**:

```markdown
# /tapd-task-manager

You are a TAPD task manager.

## Required MCP Servers

This agent requires the following MCP servers to be configured in your Claude Code settings:

- **tapd** (@openpeng/mcp-tapd)
  - Tools: tapd_create_story, tapd_list_stories, tapd_update_story
  - Env: TAPD_API_KEY, TAPD_WORKSPACE_ID

## Workflow

When the user provides task requirements:

1. Use llm_chat to analyze and extract story details
2. Use mcp__tapd__tapd_create_story to create the story
3. Report the created story ID and URL

## Available MCP Tools

- mcp__tapd__tapd_create_story
- mcp__tapd__tapd_list_stories
- mcp__tapd__tapd_update_story
```

**用户体验**:
1. Agent 部署后，Claude Code 提示用户需要配置 MCP servers
2. 用户在 Claude Code 设置中添加 TAPD MCP server
3. 配置环境变量后，Agent 可使用 MCP tools

---

## 6. 完整示例

### Agent: TAPD Task Manager

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "tapd-task-manager",
    "version": "1.0.0",
    "display_name": "TAPD Task Manager",
    "description": "AI-powered TAPD task management",
    "author": "Agent Protocol Team",
    "tags": ["tapd", "project-management", "mcp"]
  },
  "entry": {
    "main_subagent": "worker"
  },
  "subagents": [
    {
      "name": "worker",
      "path": "worker.yaml"
    }
  ],
  "dependencies": {
    "llm_provider": "anthropic"
  },
  "mcp": {
    "config_path": "./mcp/servers.json",
    "required_servers": [
      {
        "name": "tapd",
        "description": "TAPD project management tools",
        "package": "@openpeng/mcp-tapd",
        "version": "^1.0.0",
        "tools": [
          "tapd_create_story",
          "tapd_list_stories",
          "tapd_update_story"
        ],
        "required_env": [
          "TAPD_API_KEY",
          "TAPD_WORKSPACE_ID"
        ],
        "optional": false
      }
    ]
  }
}
```

**mcp/servers.json**:
```json
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd@^1.0.0"],
      "env": {
        "TAPD_API_KEY": "${TAPD_API_KEY}",
        "TAPD_WORKSPACE_ID": "${TAPD_WORKSPACE_ID}"
      }
    }
  }
}
```

**worker.yaml**:
```yaml
tools:
  - name: llm_chat
    type: builtin
  - name: tapd_create_story
    type: mcp
    server: tapd
  - name: tapd_list_stories
    type: mcp
    server: tapd

shared_context:
  workspace_id: "{{workspace_id}}"

pipeline:
  # Step 1: 分析用户输入
  - step: analyze
    tool: llm_chat
    args:
      system_prompt: |
        You are a Business Analyst.
        Extract story details from user input.
        Return JSON: {
          "action": "create" | "list",
          "story_name": "...",
          "description": "..."
        }
      prompt: "{{user_input}}"
      temperature: 0.3
    output: analysis

  # Step 2: 列出现有 stories
  - step: list_stories
    tool: tapd_list_stories
    args:
      workspace_id: "{{shared_context.workspace_id}}"
      status: "planning"
    output: existing_stories
    when: "{{steps.analyze.output.action}} == 'list'"
    on_fail: skip

  # Step 3: 创建新 story
  - step: create_story
    tool: tapd_create_story
    args:
      workspace_id: "{{shared_context.workspace_id}}"
      name: "{{steps.analyze.output.story_name}}"
      description: "{{steps.analyze.output.description}}"
    output: created_story
    when: "{{steps.analyze.output.action}} == 'create'"
    on_fail: abort

  # Step 4: 生成报告
  - step: report
    tool: llm_chat
    args:
      prompt: |
        Generate user-friendly report:
        
        {% if steps.create_story.output %}
        Created story:
        - ID: {{steps.create_story.output.id}}
        - Name: {{steps.create_story.output.name}}
        - URL: {{steps.create_story.output.url}}
        {% endif %}
        
        {% if steps.list_stories.output %}
        Found {{steps.list_stories.output.length}} stories.
        {% endif %}
    output: result
```

### 使用流程

**发布 Agent**:
```bash
cd tapd-task-manager/
agent-deploy publish .

# 上传内容包括:
# - agent.json (含 MCP 声明)
# - worker.yaml
# - mcp/servers.json (MCP 配置)
# - mcp/README.md (用户说明)
```

**用户使用**:
```bash
# 1. 下载
agent-deploy download tapd-task-manager

# 2. 查看配置说明
cat tapd-task-manager/mcp/README.md

# 3. 配置环境变量
export TAPD_API_KEY=your_api_key
export TAPD_WORKSPACE_ID=12345

# 4. 运行
agent-deploy run ./tapd-task-manager \
  --args workspace_id=12345 \
  --args user_input="Create a login page with OAuth"

# 输出:
# 🔌 Loading MCP configuration from agent...
# 📦 Installing MCP server: tapd (@openpeng/mcp-tapd@^1.0.0)...
# ✅ MCP server ready: tapd (3 tools available)
# 🚀 Running agent: tapd-task-manager
# 
# [分析用户输入...]
# [调用 tapd_create_story...]
# 
# ✅ Story created successfully!
# - ID: #67890
# - Name: 用户登录页面 with OAuth
# - URL: https://tapd.cn/12345/stories/view/67890
```

---

## 7. 最佳实践

### ✅ DO - 推荐做法

**1. MCP 配置打包在 Agent 内**:
```
my-agent/
├── agent.json
├── mcp/
│   ├── servers.json  ← 打包 MCP 配置
│   └── README.md     ← 用户说明
```

**2. 使用 NPM 包安装 MCP servers**:
```json
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd@^1.0.0"]  // ✅ 自动安装
    }
  }
}
```

**3. 清晰的环境变量说明**:
```markdown
# mcp/README.md

## 必需环境变量

- `TAPD_API_KEY`: TAPD API Token
  - 获取方式: TAPD → 个人设置 → API
- `TAPD_WORKSPACE_ID`: 工作空间 ID
  - 获取方式: 从 TAPD URL 复制
```

**4. 优雅的错误处理**:
```yaml
- step: create_story
  tool: tapd_create_story
  args: {...}
  on_fail: abort  # ✅ MCP 失败 → 明确终止
```

### ❌ DON'T - 避免做法

**1. 不要依赖全局 MCP 配置**:
```json
// ❌ Bad - 依赖用户全局配置
{
  "mcp": {
    "use_global_config": true  // 用户可能没配置
  }
}

// ✅ Good - Agent 自带配置
{
  "mcp": {
    "config_path": "./mcp/servers.json"
  }
}
```

**2. 不要硬编码 API Key**:
```json
// ❌ Bad - 硬编码
{
  "env": {
    "TAPD_API_KEY": "real_api_key_here"  // 安全风险
  }
}

// ✅ Good - 环境变量
{
  "env": {
    "TAPD_API_KEY": "${TAPD_API_KEY}"  // 从用户环境读取
  }
}
```

**3. 不要假设 MCP server 已安装**:
```yaml
# ❌ Bad - 没有 fallback
- step: must_use_mcp
  tool: tapd_create_story
  on_fail: abort

# ✅ Good - 提供 fallback 或清晰的错误提示
- step: try_mcp
  tool: tapd_create_story
  on_fail: skip

- step: manual_instruction
  tool: llm_chat
  args:
    prompt: "MCP tool unavailable. Please manually create story..."
  when: "{{steps.try_mcp.success}} == false"
```

---

## 8. Kimi WebBridge — HTTP JSON MCP（非标准 stdio）

### 概述

**Kimi WebBridge** 是一种特殊的 MCP 集成模式：它不依赖 stdio 子进程，而是通过 **HTTP JSON API** 与本地运行的浏览器扩展 daemon 通信。

这意味着：
- ✅ **不需要 shell / subprocess 权限**
- ✅ **不依赖 npx / node / python**
- ✅ **通过浏览器扩展 + 本地 daemon 工作**
- ✅ **在任何受限环境中都能运行**

### 架构

```
LLM Agent (Python/Node)
    │
    │  POST http://127.0.0.1:10086/command
    │  {"action": "navigate", "args": {"url": "..."}}
    ▼
Kimi WebBridge Daemon (端口 10086)
    │
    │  Chrome DevTools Protocol
    ▼
浏览器扩展 (Chrome/Edge)
    │
    │  真正执行浏览器操作
    ▼
用户浏览器中的网页
```

### 工作原理

1. 用户在 Chrome/Edge 安装 **Kimi WebBridge 浏览器扩展**
2. 扩展激活后，本地 daemon 监听 `http://127.0.0.1:10086`
3. Agent 通过 HTTP POST `/command` 发送操作指令
4. Daemon 通过 CDP（Chrome DevTools Protocol）驱动浏览器执行

### agent.json 声明

```json
{
  "schema_version": "2.0",
  "identity": {
    "name": "kimi-webbridge-operator",
    "display_name": "🌉 Kimi WebBridge 浏览器助手",
    "description": "通过 Kimi WebBridge 浏览器扩展与网页交互"
  },
  "mcp_servers": [
    {
      "name": "kimi-webbridge",
      "command": "npx",
      "args": ["@kimi/webbridge-mcp"]
    }
  ],
  "capabilities": [
    {"type": "tool_call", "name": "browser_navigate"},
    {"type": "tool_call", "name": "browser_click"},
    {"type": "tool_call", "name": "browser_snapshot"},
    {"type": "tool_call", "name": "browser_evaluate"},
    {"type": "tool_call", "name": "browser_screenshot"},
    {"type": "tool_call", "name": "browser_fill"}
  ]
}
```

> **注意**：`command: "npx"` 和 `args` 是声明性的描述，实际运行时 Python/Node 运行时识别 `name` 含 `webbridge` 后会走 HTTP JSON 路径，**不会真正启动 npx 子进程**。

### 工具命名约定（重要）

| 层 | 工具名示例 | 说明 |
|---|---|---|
| agent.json capabilities | `browser_navigate` | 市场 agent.json 中的声明名 |
| 运行时 schema | `webbridge_navigate` | LLM 看到的工具名 |
| WebBridge daemon action | `navigate` | 底层 HTTP API 的 action |

运行时**同时注册两套名字**（`webbridge_*` + `browser_*`），LLM 用哪个都能匹配。

### HTTP API 接口

- **健康检查**: `GET http://127.0.0.1:10086/status`
  - 返回: `{"running": true, "extension_connected": true, "version": "v1.10.0"}`
- **执行命令**: `POST http://127.0.0.1:10086/command`
  - Body: `{"action": "<action_name>", "args": {...}}`
  - 成功: `{"ok": true, "data": {...}}`
  - 失败: `{"ok": false, "error": {"code": "...", "message": "..."}}`

### 可用工具（12 个 WebBridge actions，运行时注册为 24 个 schema）

> 注意：以下工具名是 **LLM 工具 schema 命名**。`webbridge_*` 和 `browser_*` 各注册一份，底层 action 见第三列。

| webbridge_ / browser_ | daemon action | 说明 |
|---|---|---|
| `webbridge_navigate` / `browser_navigate` | `navigate` | 打开 URL |
| `webbridge_snapshot` / `browser_snapshot` | `snapshot` | 获取可访问性树 |
| `webbridge_click` / `browser_click` | `click` | 点击元素（CSS selector） |
| `webbridge_fill` / `browser_fill` | `fill` | 填写表单（selector + value） |
| `webbridge_type` / `browser_type` | `key_type` | 在焦点元素中输入文本 |
| `webbridge_keys` / `browser_keys` | `send_keys` | 发送按键（enter/tab/escape/arrow 等） |
| `webbridge_evaluate` / `browser_evaluate` | `evaluate` | 执行 JavaScript |
| `webbridge_screenshot` / `browser_screenshot` | `screenshot` | 截图 |
| `webbridge_pdf` / `browser_pdf` | `save_as_pdf` | 保存 PDF |
| `webbridge_list_tabs` / `browser_list_tabs` | `list_tabs` | 列出 tabs |
| `webbridge_find_tab` / `browser_find_tab` | `find_tab` | 查找并切换 tab |
| `webbridge_close_tab` / `browser_close_tab` | `close_tab` | 关闭当前 tab |

### 部署检查清单

部署含 WebBridge 的 Agent 前，确认用户已：

1. 在 Chrome/Edge 安装 [Kimi WebBridge 扩展](https://chrome.google.com/webstore)
2. 扩展已启用且浏览器 daemon 运行中
3. `netstat -ano | findstr 10086` 显示 LISTENING
4. 扩展图标显示"已连接"状态

### 排错

| 症状 | 原因 | 解决 |
|---|---|---|
| WebBridge ✗ 无法连接 | 扩展未启用 | 在 Chrome 扩展页面启用 Kimi WebBridge |
| running=true 但扩展不响应 | 扩展需要重新加载 | 刷新扩展页面，重启浏览器 |
| HTTP 403/401 | token 不正确 | 设置 `WEBBRIDGE_TOKEN` 环境变量 |
| 工具调用后页面没变化 | 页面尚未加载完成 | 在 snapshot 前加 `time.sleep(1)` 等待 |

---

## 9. 市场引用模式（v3.1 扩展）

### 概述

v3.1 在原有**内联打包**基础上，新增**市场引用模式**：MCP Server 可独立打包发布到云端市场，Agent 通过 `ref` 引用而非内联完整配置。运行时自动解析引用、下载依赖、缓存复用。

### 三种使用模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **内联打包**（现有） | MCP 配置随 Agent 一起发布，`mcp/` 目录打包在 Agent 内 | 自定义 MCP、私有部署、离线使用 |
| **市场引用**（新增） | Agent 通过 `ref` 引用市场中的 MCP Server，运行时自动下载 | 公共 MCP Server、多 Agent 共享、版本管理 |
| **混合**（推荐） | 部分内联 + 部分引用，灵活组合 | 大多数生产场景 |

```
模式 A: 内联打包（现有）          模式 B: 市场引用（新增）           模式 C: 混合（推荐）
┌─────────────────────┐          ┌─────────────────────┐          ┌─────────────────────┐
│ my-agent/           │          │ my-agent/           │          │ my-agent/           │
│ ├── agent.json      │          │ ├── agent.json      │          │ ├── agent.json      │
│ │   └── mcp: {      │          │ │   └── mcp_servers:│          │ │   └── mcp_servers:│
│ │       config_path │          │ │       [{ref,      │          │ │       [{ref, ...},│
│ │     }             │          │ │        version,   │          │ │        {name, ...}│
│ │                   │          │ │        market_url}]│          │ │       ]           │
│ └── mcp/            │          │ └── (无 mcp/ 目录)  │          │ ├── mcp/            │
│     └── servers.json │          │                     │          │ │   └── local-mcp/   │
│         (完整配置)   │          │ 运行时自动下载:      │          │     └── servers.json│
└─────────────────────┘          │ ~/.agent-hub/cache/ │          └─────────────────────┘
     自包含但臃肿                    └── mcp-servers/     │          灵活可控
                                  └─────────────────────┘
                                       精简但依赖网络
```

### MCPRef 引用格式

agent.json 中 `mcp_servers` 数组元素新增引用字段：

```typescript
interface MCPRef {
  // 模式 A: 内联完整定义（向后兼容）
  name?: string;
  description?: string;
  command?: string;
  args?: string[];
  env?: Record<string, string>;

  // 模式 B: 市场引用（新增）
  ref?: string;           // 引用标识: "tapd" 或 "openpeng/tapd"
  version?: string;       // 版本约束: "^1.0.0", ">=2.0.0", "*"
  market_url?: string;    // 市场地址: "https://market.aitboy.cn"
  source?: "inline" | "market" | "local";  // 来源类型

  // 运行时覆盖（可选）
  env_override?: Record<string, string>;  // 覆盖引用的 MCP 环境变量
}
```

**解析规则**:
- 如果 `ref` 存在 → 市场引用模式
- 如果 `name` 存在且无 `ref` → 内联模式（向后兼容）
- `source` 显式声明时以 `source` 为准

### 版本约束语法

| 语法 | 含义 | 示例 |
|------|------|------|
| `1.0.0` | 精确版本 | 只匹配 1.0.0 |
| `^1.0.0` | 兼容版本 | 匹配 1.x.x，不匹配 2.0.0 |
| `~1.0.0` | 近似版本 | 匹配 1.0.x，不匹配 1.1.0 |
| `>=1.0.0` | 大于等于 | 匹配 1.0.0 及以上 |
| `*` | 任意版本 | 匹配最新版本 |

### MCP Server 独立包规范

MCP Server 可独立打包发布到市场，结构如下：

```
tapd-mcp-server/
├── mcp-server.json     # MCP Server 元数据（必需）
├── mcp-config.json     # MCP 配置（Claude Desktop 兼容格式，必需）
├── README.md           # 配置说明（必需）
└── scripts/
    └── install.sh      # 安装脚本（可选）
```

#### mcp-server.json 格式

```json
{
  "schema_version": "1.0.0",
  "identity": {
    "name": "tapd",
    "version": "1.0.0",
    "display_name": "TAPD MCP Server",
    "description": "TAPD project management MCP server",
    "author": "OpenPeng",
    "package": "@openpeng/mcp-tapd"
  },
  "config": {
    "source": "file",
    "file": "mcp-config.json"
  },
  "tools": [
    "tapd_create_story",
    "tapd_list_stories",
    "tapd_update_story"
  ],
  "required_env": [
    "TAPD_API_KEY",
    "TAPD_WORKSPACE_ID"
  ]
}
```

#### mcp-config.json 格式

```json
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd@^1.0.0"],
      "env": {
        "TAPD_API_KEY": "${TAPD_API_KEY}",
        "TAPD_WORKSPACE_ID": "${TAPD_WORKSPACE_ID}"
      }
    }
  }
}
```

### 运行时引用解析

Agent 启动时，运行时自动解析 MCP 引用：

```
Agent 启动
  │
  ▼
读取 agent.json mcp_servers 数组
  │
  ├── 内联定义（无 ref）→ 直接使用现有流程（§4 MCPLoader）
  │
  └── 引用定义（有 ref）→ 解析引用
                                │
                                ▼
                          检查本地缓存
                          ~/.agent-hub/cache/mcp-servers/{ref}@{version}/
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                缓存命中                缓存未命中
                    │                       │
                    ▼                       ▼
                直接使用                从市场下载
                加载 mcp-config.json    解压到缓存目录
                应用 env_override       验证 mcp-server.json
                    │                       │
                    └───────────┬───────────┘
                                ▼
                          合并到 Agent MCP 配置
                          启动 MCP Server 进程
```

### 缓存策略

```
~/.agent-hub/cache/
├── mcp-servers/
│   ├── tapd@1.0.0/
│   │   ├── mcp-server.json
│   │   ├── mcp-config.json
│   │   └── README.md
│   ├── kimi-webbridge@2.1.0/
│   │   ├── mcp-server.json
│   │   ├── mcp-config.json
│   │   └── README.md
│   └── ...
└── index.json          # 缓存索引: {ref: {version, path, downloaded_at, etag}}
```

**缓存规则**:
- 按 `ref@resolved_version` 目录存储
- 下载时记录 `etag`，启动时检查是否需要更新
- `version: "*"` 或 `^x.x.x` 时，每日检查一次最新版本
- 缓存清理: `agent-deploy cache clean --unused-for 30d`

### env_override：运行时环境变量覆盖

引用的 MCP Server 可通过 `env_override` 在运行时覆盖环境变量，无需修改原始 MCP 配置：

```json
{
  "mcp_servers": [
    {
      "ref": "tapd",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market",
      "env_override": {
        "TAPD_WORKSPACE_ID": "12345",
        "TAPD_API_KEY": "${MY_CUSTOM_TAPD_KEY}"
      }
    }
  ]
}
```

**覆盖规则**:
- `env_override` 中的值会**合并**到 MCP Server 原始 `env` 配置中
- 同名变量以 `env_override` 为准
- 支持 `${VAR_NAME}` 语法从运行时环境读取

### CLI 命令

```bash
# 打包 MCP Server
agent-deploy mcp pack <path> [-o, --output <file>]

# 上传 MCP Server 到市场
agent-deploy mcp upload <path> [-m, --market <url>] [-k, --api-key <key>] [-f, --force]

# 从市场下载 MCP Server
agent-deploy mcp download <ref> [-v, --version <ver>] [-o, --output <dir>]

# 列出已缓存的 MCP Servers
agent-deploy mcp list --cached

# 缓存管理
agent-deploy cache status
agent-deploy cache clean [--all] [--unused-for <days>]
agent-deploy cache update [--agent <path>] [--dry-run]
```

### agent.json 引用示例

#### 纯引用模式

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "tapd-task-manager",
    "version": "1.0.0",
    "description": "Manages TAPD tasks using market-referenced MCP"
  },
  "entry": {"main_subagent": "worker"},
  "subagents": [
    {"name": "worker", "path": "worker.yaml"}
  ],
  "mcp_servers": [
    {
      "ref": "tapd",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market",
      "env_override": {
        "TAPD_WORKSPACE_ID": "${TAPD_WORKSPACE_ID}"
      }
    }
  ]
}
```

#### 混合模式（推荐）

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "hybrid-agent",
    "version": "1.0.0",
    "description": "Mixed inline and referenced MCP servers"
  },
  "mcp_servers": [
    {
      "ref": "tapd",
      "version": "^1.0.0",
      "market_url": "https://market.aitboy.cn",
      "source": "market"
    },
    {
      "name": "kimi-webbridge",
      "command": "npx",
      "args": ["@kimi/webbridge-mcp"],
      "source": "inline"
    }
  ]
}
```

### 向后兼容性

| 场景 | 兼容性 |
|------|--------|
| 现有内联 MCP 配置（`mcp/` 目录） | 完全兼容，无 `ref` 字段时按内联处理 |
| 现有 `mcp_servers` 数组（内联定义） | 完全兼容，无 `ref` 字段时按内联处理 |
| 现有 Agent 包下载 | 兼容，下载内容不变 |
| 新引用模式 Agent | 需要运行时支持引用解析（agent-compose 升级后） |

---

## 参考

- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)

---

**MCP Integration - Agent 自带 MCP 配置，开箱即用** 🔌
