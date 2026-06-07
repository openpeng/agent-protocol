# MCP Integration 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

MCP (Model Context Protocol) 集成允许 Agent 访问外部工具和服务。Agent Protocol v3 支持三种 MCP 集成方式：

1. **使用平台的 MCP** - Agent 部署到 Claude Code 等平台后，访问平台配置的 MCP servers
2. **声明 MCP 依赖** - Agent 声明需要的 MCP tools，运行时验证可用性
3. **作为 MCP Server** - Agent 本身打包为 MCP server，供其他 AI 工具调用

---

## 1. 使用平台 MCP (Deploy 模式)

### 场景

Agent 部署到支持 MCP 的平台（如 Claude Code），利用平台已配置的 MCP servers。

### agent.json 声明

```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "tapd-task-manager",
    "description": "Manages TAPD tasks using MCP tools"
  },
  "dependencies": {
    "mcp_tools": ["tapd"]
  },
  "mcp": {
    "required_servers": [
      {
        "name": "tapd",
        "tools": [
          "tapd_create_story",
          "tapd_list_stories",
          "tapd_update_story"
        ],
        "optional": false
      }
    ]
  }
}
```

### worker.yaml 使用

```yaml
tools:
  - name: llm_chat
    type: builtin
  - name: tapd_create_story
    type: mcp
    server: tapd

pipeline:
  - step: analyze_requirement
    tool: llm_chat
    args:
      system_prompt: "You are a BA. Extract story details from user input."
      prompt: "{{user_input}}"
    output: story_details

  - step: create_story
    tool: tapd_create_story
    args:
      workspace_id: "{{workspace_id}}"
      name: "{{steps.analyze_requirement.output.title}}"
      description: "{{steps.analyze_requirement.output.description}}"
    output: created_story

  - step: report
    tool: llm_chat
    args:
      prompt: |
        Story created successfully:
        ID: {{steps.create_story.output.id}}
        URL: {{steps.create_story.output.url}}
    output: result
```

### 运行时行为

**Deploy 模式** (agent-deploy deploy -t claude_code):
```markdown
# /tapd-task-manager

You are a TAPD task manager.

When the user provides task requirements, follow these steps:

1. Use llm_chat to analyze and extract story details
2. Use mcp__tapd__tapd_create_story to create the story
3. Report the created story ID and URL

Available MCP tools:
- mcp__tapd__tapd_create_story
- mcp__tapd__tapd_list_stories
- mcp__tapd__tapd_update_story
```

**平台执行**:
- Claude Code 读取 `.claude/commands/tapd-task-manager.md`
- 用户调用 `/tapd-task-manager`
- Claude 可以访问 `mcp__tapd__*` 工具（平台已配置）
- Agent instructions 引导 Claude 正确使用这些工具

---

## 2. 声明 MCP 依赖 (Run 模式)

### 场景

Agent 在本地运行（agent-deploy run），需要访问 MCP tools。

### MCP Tool 类型

```typescript
// src/runtime/tools/mcp-tool.ts
export class MCPTool implements BuiltinTool {
  name: string;  // 例如: "tapd_create_story"
  type = "mcp";
  server: string;  // 例如: "tapd"
  
  constructor(
    private mcpClient: MCPClient,
    private toolDef: MCPToolDefinition
  ) {
    this.name = toolDef.name;
    this.server = toolDef.server;
  }

  async execute(args: any, context: ExecutionContext): Promise<any> {
    // 1. 连接到 MCP server
    const client = await this.mcpClient.connect(this.server);
    
    // 2. 调用 MCP tool
    const result = await client.callTool(this.name, args);
    
    return result;
  }
}
```

### MCP Client 配置

**~/.agent-deploy/mcp-config.json**:
```json
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd"],
      "env": {
        "TAPD_API_KEY": "${TAPD_API_KEY}",
        "TAPD_WORKSPACE_ID": "${TAPD_WORKSPACE_ID}"
      }
    },
    "dify": {
      "command": "node",
      "args": ["/path/to/dify-mcp-server/dist/index.js"],
      "env": {
        "DIFY_API_KEY": "${DIFY_API_KEY}"
      }
    }
  }
}
```

### 运行时加载

```typescript
// src/runtime/mcp-loader.ts
export class MCPLoader {
  async loadMCPTools(agent: Agent): Promise<Map<string, MCPTool>> {
    const tools = new Map<string, MCPTool>();
    
    if (!agent.mcp?.required_servers) {
      return tools;
    }

    for (const serverDef of agent.mcp.required_servers) {
      // 1. 从配置读取 server 信息
      const serverConfig = this.loadServerConfig(serverDef.name);
      
      // 2. 启动 MCP server
      const client = await this.startMCPServer(serverConfig);
      
      // 3. 列举 tools
      const availableTools = await client.listTools();
      
      // 4. 验证所需 tools 是否可用
      for (const toolName of serverDef.tools) {
        const toolDef = availableTools.find(t => t.name === toolName);
        if (!toolDef) {
          if (!serverDef.optional) {
            throw new Error(`Required MCP tool not found: ${toolName}`);
          }
          continue;
        }
        
        // 5. 注册到工具系统
        const mcpTool = new MCPTool(client, toolDef);
        tools.set(toolName, mcpTool);
      }
    }
    
    return tools;
  }
}
```

### 使用示例

```bash
# 运行需要 MCP 的 Agent
agent-deploy run ./tapd-task-manager \
  --args workspace_id=12345 \
  --args user_input="Create a login page"

# 输出:
# 🔌 Connecting to MCP server: tapd
# ✅ MCP tools loaded: tapd_create_story, tapd_list_stories, tapd_update_story
# 🚀 Running agent: tapd-task-manager
# ✅ Story created: #67890
```

---

## 3. Agent 作为 MCP Server

### 场景

将 Agent 打包为 MCP server，供 Claude Desktop、Cursor 等工具调用。

### 打包配置

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "code-auditor",
    "description": "Comprehensive code audit"
  },
  "mcp_server": {
    "enabled": true,
    "tools": [
      {
        "name": "audit_code",
        "description": "Run comprehensive code audit",
        "inputSchema": {
          "type": "object",
          "properties": {
            "path": {
              "type": "string",
              "description": "Path to code directory"
            }
          },
          "required": ["path"]
        }
      }
    ]
  }
}
```

### MCP Server 包装器

```typescript
// src/mcp-server/wrapper.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

export class AgentMCPServer {
  private server: Server;
  
  constructor(private agent: Agent) {
    this.server = new Server(
      {
        name: agent.identity.name,
        version: agent.identity.version
      },
      { capabilities: { tools: {} } }
    );
    
    this.setupHandlers();
  }

  private setupHandlers() {
    // 1. List tools
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      return {
        tools: this.agent.mcp_server.tools.map(tool => ({
          name: tool.name,
          description: tool.description,
          inputSchema: tool.inputSchema
        }))
      };
    });

    // 2. Call tool
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;
      
      // 运行 Agent pipeline
      const result = await this.agent.run(args);
      
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(result, null, 2)
          }
        ]
      };
    });
  }

  async start() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}
```

### 生成 MCP Server

```bash
# 打包 Agent 为 MCP server
agent-deploy pack ./code-auditor --mcp

# 输出:
# ✅ Created: code-auditor-mcp/
#    ├── index.js (MCP server entry)
#    ├── agent.json
#    ├── worker.yaml
#    └── package.json

# 安装
cd code-auditor-mcp
npm install
npm link

# 配置到 Claude Desktop
# ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "code-auditor": {
      "command": "npx",
      "args": ["code-auditor-mcp"]
    }
  }
}
```

### 在 Claude Desktop 使用

```
User: Use code-auditor tool to audit ./my-project

Claude: [调用 MCP tool]
{
  "tool": "audit_code",
  "args": {
    "path": "./my-project"
  }
}

[Agent 运行 pipeline]
[返回审计报告]

Claude: Here's the audit report:
- Security: 3 issues found
- Performance: 5 bottlenecks
- Quality: 12 code smells
...
```

---

## 4. MCP Tools 发现与注册

### 自动发现

```typescript
// src/runtime/mcp-discovery.ts
export class MCPDiscovery {
  async discoverTools(): Promise<MCPServerInfo[]> {
    const servers: MCPServerInfo[] = [];
    
    // 1. 从配置文件读取
    const config = this.loadMCPConfig();
    servers.push(...config.servers);
    
    // 2. 从环境变量读取
    // MCP_SERVERS=tapd:npx @openpeng/mcp-tapd,dify:node ./dify-server.js
    const envServers = this.parseEnvServers();
    servers.push(...envServers);
    
    // 3. 从 Claude Desktop 配置读取（如果存在）
    const claudeConfig = this.loadClaudeDesktopConfig();
    if (claudeConfig) {
      servers.push(...claudeConfig.mcpServers);
    }
    
    return servers;
  }
}
```

### 工具注册

```typescript
// src/runtime/tools/registry.ts
export class ToolRegistry {
  private builtinTools = new Map<string, BuiltinTool>();
  private mcpTools = new Map<string, MCPTool>();

  async registerMCPTools(agent: Agent) {
    const mcpLoader = new MCPLoader();
    const tools = await mcpLoader.loadMCPTools(agent);
    
    for (const [name, tool] of tools) {
      this.mcpTools.set(name, tool);
    }
  }

  get(name: string): BuiltinTool | MCPTool | null {
    // 1. 先查找 builtin
    if (this.builtinTools.has(name)) {
      return this.builtinTools.get(name)!;
    }
    
    // 2. 再查找 MCP
    if (this.mcpTools.has(name)) {
      return this.mcpTools.get(name)!;
    }
    
    return null;
  }
}
```

---

## 5. worker.yaml 中使用 MCP Tools

### 声明方式

```yaml
tools:
  - name: llm_chat
    type: builtin
  
  # 方式 1: 声明 MCP tool（需要在 agent.json 中定义 server）
  - name: tapd_create_story
    type: mcp
    server: tapd
  
  # 方式 2: 自动加载（从 agent.json 的 mcp.required_servers）
  - name: dify_workflow
    type: mcp
    server: dify

pipeline:
  - step: create_story
    tool: tapd_create_story
    args:
      workspace_id: "{{workspace_id}}"
      name: "{{story_name}}"
    output: story
    on_fail: abort  # MCP 工具失败 → abort

  - step: run_workflow
    tool: dify_workflow
    args:
      workflow_id: "12345"
      inputs:
        data: "{{steps.create_story.output}}"
    output: workflow_result
    on_fail: retry(2)  # MCP 工具可以重试
```

### 错误处理

```typescript
// MCP 工具特殊错误处理
export class MCPTool implements BuiltinTool {
  async execute(args: any, context: ExecutionContext): Promise<any> {
    try {
      const result = await this.mcpClient.callTool(this.name, args);
      return result;
    } catch (error) {
      if (error instanceof MCPConnectionError) {
        throw new ToolError(
          `MCP server '${this.server}' not available. ` +
          `Make sure it's configured in ~/.agent-deploy/mcp-config.json`,
          "MCP_SERVER_UNAVAILABLE"
        );
      } else if (error instanceof MCPToolNotFoundError) {
        throw new ToolError(
          `MCP tool '${this.name}' not found on server '${this.server}'.`,
          "MCP_TOOL_NOT_FOUND"
        );
      }
      throw error;
    }
  }
}
```

---

## 6. 示例：完整 MCP 集成

### Agent: TAPD Task Manager

**agent.json**:
```json
{
  "schema_version": "3.0",
  "identity": {
    "name": "tapd-task-manager",
    "version": "1.0.0",
    "description": "Manages TAPD tasks with AI assistance",
    "author": "Agent Protocol Team"
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
    "llm_provider": "anthropic",
    "mcp_tools": ["tapd"]
  },
  "mcp": {
    "required_servers": [
      {
        "name": "tapd",
        "tools": [
          "tapd_create_story",
          "tapd_list_stories",
          "tapd_update_story",
          "tapd_get_story"
        ],
        "optional": false
      }
    ]
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
  - step: analyze_input
    tool: llm_chat
    args:
      system_prompt: |
        You are a Business Analyst.
        Extract story details from user requirements.
        Return JSON: {
          "action": "create" | "list" | "update",
          "story_name": "...",
          "description": "..."
        }
      prompt: "{{user_input}}"
      temperature: 0.3
    output: analysis

  # Step 2: 列出现有 stories（如果需要）
  - step: list_existing
    tool: tapd_list_stories
    args:
      workspace_id: "{{shared_context.workspace_id}}"
      status: "planning"
    output: existing_stories
    when: "{{steps.analyze_input.output.action}} == 'list'"
    on_fail: skip

  # Step 3: 创建新 story
  - step: create_story
    tool: tapd_create_story
    args:
      workspace_id: "{{shared_context.workspace_id}}"
      name: "{{steps.analyze_input.output.story_name}}"
      description: "{{steps.analyze_input.output.description}}"
    output: created_story
    when: "{{steps.analyze_input.output.action}} == 'create'"
    on_fail: abort

  # Step 4: 生成报告
  - step: generate_report
    tool: llm_chat
    args:
      prompt: |
        Generate a user-friendly report based on:
        Action: {{steps.analyze_input.output.action}}
        
        {% if steps.create_story.output %}
        Created story:
        - ID: {{steps.create_story.output.id}}
        - Name: {{steps.create_story.output.name}}
        - URL: {{steps.create_story.output.url}}
        {% endif %}
        
        {% if steps.list_existing.output %}
        Found {{steps.list_existing.output.length}} stories.
        {% endif %}
    output: report
```

### 使用

**本地运行**:
```bash
# 配置 MCP
cat > ~/.agent-deploy/mcp-config.json << 'EOF'
{
  "servers": {
    "tapd": {
      "command": "npx",
      "args": ["-y", "@openpeng/mcp-tapd"],
      "env": {
        "TAPD_API_KEY": "${TAPD_API_KEY}",
        "TAPD_WORKSPACE_ID": "12345"
      }
    }
  }
}
EOF

# 运行 Agent
agent-deploy run ./tapd-task-manager \
  --args workspace_id=12345 \
  --args user_input="Create a user login page with OAuth support"

# 输出:
# 🔌 Loading MCP servers...
# ✅ Connected to MCP server: tapd (4 tools available)
# 🚀 Running agent: tapd-task-manager
# 📝 Creating story: User login page with OAuth
# ✅ Story created: #67890
# 🔗 URL: https://tapd.cn/12345/stories/view/67890
```

**部署到 Claude Code**:
```bash
agent-deploy deploy ./tapd-task-manager -t claude_code

# 在 Claude Code 中使用:
# /tapd-task-manager workspace_id=12345 "Create a dashboard page"
```

---

## 7. MCP 配置管理

### 配置文件位置

```
~/.agent-deploy/
├── mcp-config.json          # 用户级 MCP 配置
└── cache/
    └── mcp-tools.json       # 缓存的 MCP tools 列表
```

### 配置格式

```json
{
  "servers": {
    "<server_name>": {
      "command": "string",       // 启动命令
      "args": ["string"],        // 命令参数
      "env": {                   // 环境变量
        "KEY": "value"
      },
      "timeout": 30000,          // 启动超时（ms）
      "retry": 3                 // 连接失败重试次数
    }
  },
  "cache": {
    "enabled": true,             // 启用工具列表缓存
    "ttl": 3600                  // 缓存 TTL（秒）
  }
}
```

### CLI 管理命令

```bash
# 列出可用 MCP servers
agent-deploy mcp list

# 测试 MCP server 连接
agent-deploy mcp test tapd

# 列出 server 的 tools
agent-deploy mcp tools tapd

# 清除缓存
agent-deploy mcp cache-clear
```

---

## 8. 最佳实践

### ✅ DO - 推荐做法

**1. 显式声明依赖**:
```json
{
  "dependencies": {
    "mcp_tools": ["tapd", "dify"]
  },
  "mcp": {
    "required_servers": [...]
  }
}
```

**2. 优雅降级**:
```yaml
- step: try_mcp_tool
  tool: tapd_create_story
  args: {...}
  on_fail: continue  # MCP 不可用时继续

- step: fallback
  tool: llm_chat
  args:
    prompt: "MCP tool unavailable. Please create story manually..."
  when: "{{steps.try_mcp_tool.success}} == false"
```

**3. 错误提示**:
```typescript
if (!mcpServerAvailable) {
  throw new Error(
    `MCP server 'tapd' not configured. ` +
    `Please run: agent-deploy mcp config tapd`
  );
}
```

### ❌ DON'T - 避免做法

**1. 硬编码 MCP server 路径**:
```yaml
# ❌ Bad
tools:
  - name: tapd_tool
    type: custom
    config:
      script: "/usr/local/bin/tapd-mcp-server"  # 硬编码路径

# ✅ Good
tools:
  - name: tapd_create_story
    type: mcp
    server: tapd  # 从配置读取
```

**2. 忽略 MCP 不可用的情况**:
```yaml
# ❌ Bad - 没有 fallback
- step: must_use_mcp
  tool: tapd_create_story
  on_fail: abort  # MCP 不可用 → 整个 Agent 失败

# ✅ Good - 有 fallback
- step: try_mcp
  tool: tapd_create_story
  on_fail: continue

- step: manual_fallback
  when: "{{steps.try_mcp.success}} == false"
```

---

## 参考

- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [agent.json v3 规范](./agent-json-v3.md)
- [worker.yaml 规范](./worker-yaml.md)
- [Builtin Tools 规范](./builtin-tools.md)

---

**MCP Integration - 让 Agent 访问外部工具和服务** 🔌
