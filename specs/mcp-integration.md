# MCP Integration 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

MCP (Model Context Protocol) 集成让 Agent 能够调用外部工具和服务。在 Agent Protocol v3 中，MCP 配置可以**打包在 Agent 内部**一起发布，用户下载 Agent 时自动获得 MCP 集成能力。

### 核心理念

✅ **Agent 是完整的包** - MCP 配置随 Agent 一起发布  
✅ **开箱即用** - 用户只需配置 API Key，无需手动安装 MCP servers  
✅ **可移植** - Agent 在任何环境都能运行（本地 / 平台）

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

## 参考

- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)

---

**MCP Integration - Agent 自带 MCP 配置，开箱即用** 🔌
