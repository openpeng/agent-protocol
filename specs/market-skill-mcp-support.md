# Market Skills & MCP 支持规范

**版本**: 1.0.0
**状态**: Proposal
**最后更新**: 2026-06-17

---

## 概述

当前 agent-market 市场服务在注册/发布 Agent 时，**仅提取 agent.json 的基础元数据字段**（identity、tags、dependencies 等），未解析和存储 `skills` 与 `mcp_servers` 信息。这导致市场无法支持按 skills/MCP 能力进行搜索、发现和展示。

agent-builder 构建器项目已广泛使用 `skills` 和 `mcp_servers` 字段来定义 Agent 的能力边界，市场需同步扩充以完整支持 Agent Protocol v3 规范。

### 目标

✅ **Skills 可发现** — 市场 API 返回 Agent 包含的 Skills 列表
✅ **MCP 可发现** — 市场 API 返回 Agent 依赖的 MCP Servers 配置
✅ **按能力搜索** — 支持按 Skill 名称、MCP Server 名称筛选 Agent
✅ **向后兼容** — 不影响现有 API 和数据库结构
✅ **零破坏性变更** — 已有 Agent 注册/下载流程不受影响

---

## 1. 当前状态分析

### 1.1 数据库现状

当前 `agents` 表共 28 列，**无任何 skills 或 mcp 相关字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `json_content` | TEXT | 完整 agent.json 的 JSON 文本（唯一包含 skills/mcp 的列） |
| `type` | TEXT | Agent 类型：agent、subagent、skill、workflow |
| `tags` | TEXT | JSON 数组（如 `["tapd", "mcp"]`） |
| `dependencies` | TEXT | JSON 对象（如 `{"llm_provider": "anthropic"}`） |

**问题**：`json_content` 是全文存储，无法用 SQL 高效查询 skills 或 mcp_servers。

### 1.2 API 现状

| 端点 | Skills/MCP 支持 |
|------|-----------------|
| `POST /api/v1/agents` | ❌ 上传时不解析 skills/mcp |
| `GET /api/v1/agents/{id}` | ❌ 响应无 skills/mcp 字段 |
| `GET /api/v1/agents` | ❌ 不支持按 skill/mcp 筛选 |
| `GET /api/v1/discover` | ❌ 不暴露 skills/mcp 能力 |

### 1.3 验证现状

`verify.py` 的 `verify_package()` 仅验证：
- agent.json 存在与 JSON 合法性
- identity 必填字段
- v2 instructions 长度 >= 50 字符
- subagent 引用完整性

**未验证**：skill type 一致性、MCP config_path 可达性、重复 skill/mcp name。

---

## 2. 数据模型

### 2.1 agent.json 中的两种 Skills 格式

市场上实际存在两种 skills 定义方式，均需支持：

**格式 A — Agent Protocol v3 标准**（subagent + `type: "skill"`）：

```json
{
  "subagents": [
    { "name": "orchestrator", "path": "orchestrator.yaml" },
    { "name": "text-summarizer", "path": "skills/text-summarizer/worker.yaml", "type": "skill" },
    { "name": "translator", "path": "skills/translator/worker.yaml", "type": "skill" }
  ]
}
```

Skills 作为 Agent 目录内的子目录打包发布。

**格式 B — agent-builder 平铺格式**（agent.json 顶层 `skills` 数组）：

```json
{
  "skills": [
    {
      "name": "wecom-message",
      "display_name": "企微消息助手",
      "version": "1.0.0",
      "description": "企业微信消息管理助手",
      "icon": "💬",
      "category": "企业协作",
      "parameters": {
        "msgType": "text",
        "toType": "user"
      }
    }
  ]
}
```

参数由 agent-builder 前端表单驱动，Skill 作为独立子 Agent 在运行时可用。

#### 格式 C — skills/*.yaml 文件（Runtime 文件系统加载）

```
my-agent/
├── agent.json
├── worker.yaml
└── skills/
    ├── text-summarizer.yaml      # Skill 1: WorkerYaml pipeline
    └── translator.yaml           # Skill 2: WorkerYaml pipeline
```

**来源**：Agent 包内 `skills/*.yaml` 文件。Runtime 的 `SkillLoader.loadSkills()` 扫描此目录，文件名（不含扩展名）作为 Skill 名称。每个 `.yaml` 文件是完整的 `WorkerYaml` Pipeline 定义。

> **注**：三种格式共存时按 Skill name 去重合并，市场提取时不做取舍。当前 agent-builder 主要使用格式 B，格式 A 为 v3 规范推荐，格式 C 尚未在现有 Agent 中使用。

### 2.2 MCP 配置的三种格式

市场需同时支持三种 MCP 定义来源：

#### 格式 A — agent.json 顶层 `mcp_servers` 数组（agent-builder 使用）

```json
{
  "mcp_servers": [
    {
      "name": "wecom-api",
      "description": "企业微信开放平台 API",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-wecom"],
      "env": {
        "WECOM_CORPID": "${CORPID}",
        "WECOM_CORPSECRET": "${CORPSECRET}"
      }
    }
  ]
}
```

**来源**：agent-builder 前端表单配置，直接写入 agent.json。

#### 格式 B — mcp/config.json 文件（Runtime 文件系统加载）

```json
{
  "mcpServers": {
    "aliyun-log": {
      "type": "stdio",
      "command": "npx",
      "args": ["--registry=https://registry.npmmirror.com", "-y", "@openpeng/alilog-mcp"],
      "env": {
        "CRED_SOURCE": "consul",
        "CONSUL_URL": "https://dev-consul.gaodunwangxiao.com"
      }
    }
  }
}
```

**来源**：Agent 包内 `mcp/config.json` 文件（Claude Desktop 兼容格式）。Runtime 的 `MCPToolLoader.loadConfig()` 在包解压后加载。

#### 格式 C — Agent Protocol v3 `mcp` 字段

```json
{
  "mcp": {
    "config_path": "./mcp/servers.json",
    "required_servers": [
      {
        "name": "tapd",
        "description": "TAPD project management",
        "package": "@openpeng/mcp-tapd",
        "version": "^1.0.0",
        "tools": ["tapd_create_story", "tapd_list_stories"],
        "required_env": ["TAPD_API_KEY"]
      }
    ]
  }
}
```

**来源**：Agent Protocol v3 规范定义的标准格式，MCP 依赖声明在 agent.json 的 `mcp` 字段。

> **注**：三种格式共存时按 server name 去重合并，市场提取时不做取舍。

### 2.3 数据库扩展方案

对 JSON 列的 LIKE 查询在大数据量下效率低下。采用 **独立表 + 关联表** 设计，将 Skill 和 MCP Server 提升为市场一级实体。

**新增 4 张表**：

```sql
-- Skill 定义表（市场级实体）
CREATE TABLE IF NOT EXISTS skills (
    id TEXT PRIMARY KEY,                   -- qualified name: "agent-name/skill-name"
    original_name TEXT NOT NULL,           -- 原始 name（不含前缀）
    display_name TEXT NOT NULL DEFAULT '',
    description TEXT NOT NULL DEFAULT '',
    version TEXT NOT NULL DEFAULT '',
    category TEXT NOT NULL DEFAULT '',
    icon TEXT NOT NULL DEFAULT '',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- MCP Server 定义表（市场级实体）
CREATE TABLE IF NOT EXISTS mcp_servers (
    id TEXT PRIMARY KEY,                   -- qualified name: "agent-name/server-name"
    original_name TEXT NOT NULL,           -- 原始 name（不含前缀）
    description TEXT NOT NULL DEFAULT '',
    command TEXT NOT NULL DEFAULT '',
    args TEXT NOT NULL DEFAULT '[]',       -- JSON array
    package TEXT NOT NULL DEFAULT '',
    tools TEXT NOT NULL DEFAULT '[]',      -- JSON array
    required_env TEXT NOT NULL DEFAULT '[]', -- JSON array
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Agent-Skill 关联表
CREATE TABLE IF NOT EXISTS agent_skills (
    agent_id TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    skill_id TEXT NOT NULL REFERENCES skills(id) ON DELETE CASCADE,
    PRIMARY KEY (agent_id, skill_id)
);

-- Agent-MCP 关联表
CREATE TABLE IF NOT EXISTS agent_mcp_servers (
    agent_id TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    mcp_server_id TEXT NOT NULL REFERENCES mcp_servers(id) ON DELETE CASCADE,
    PRIMARY KEY (agent_id, mcp_server_id)
);

CREATE INDEX IF NOT EXISTS idx_skills_original_name ON skills(original_name);
CREATE INDEX IF NOT EXISTS idx_mcp_servers_original_name ON mcp_servers(original_name);
CREATE INDEX IF NOT EXISTS idx_agent_skills_skill_id ON agent_skills(skill_id);
CREATE INDEX IF NOT EXISTS idx_agent_mcp_servers_server_id ON agent_mcp_servers(mcp_server_id);
```

**表关系图**：

```
agents ──< agent_skills >── skills
agents ──< agent_mcp_servers >── mcp_servers
```

多对多关系：一个 Agent 可包含多个 Skill，一个 Skill 可被多个 Agent 复用。

**设计原则**：
- **独立实体**：Skill 和 MCP Server 可脱离 Agent 独立注册、查询
- **关联追踪**：通过 junction 表精准定位 Skill/MCP 的使用方
- **级联删除**：Agent 删除时自动清理关联关系；Skill/MCP 被所有 Agent 解除关联后可手动清理
- **高效查询**：`WHERE skill_id = ?` 走索引，比 `LIKE '%"name"%'` 效率高

### 2.4 命名策略：Agent 命名空间前缀

不同 Agent 可能包含同名的 skill 或 MCP server（如多个 Agent 都有自己的 `text-summarizer` skill）。为避免冲突，市场存储时采用 **`agent-name/skill-name`** 命名空间前缀：

```
原始 name:  wecom-message
存储 name:  wecom-assistant/wecom-message
```

这样每个 skill/MCP server 在全市场范围内具有唯一标识，同时保留所属 Agent 的上下文。

### 2.5 存储格式

**skills_info** — 归一化后的技能列表（Agent 维度）：

```json
[
  {
    "name": "wecom-assistant/wecom-message",
    "agent": "wecom-assistant",
    "display_name": "企微消息助手",
    "description": "企业微信消息管理助手",
    "version": "1.0.0",
    "category": "企业协作",
    "icon": "💬"
  },
  {
    "name": "content-processor/text-summarizer",
    "agent": "content-processor",
    "display_name": "文本摘要",
    "description": "Summarizes text into concise bullet points",
    "version": "1.0.0",
    "category": "nlp"
  }
]
```

归一化规则：
1. 如果 agent.json 有顶层 `skills` 数组 → 直接提取，name 加上 `{agent_name}/` 前缀
2. 如果 subagents 中有 `type: "skill"` 的项 → 同上处理
3. 如果包内有 `skills/*.yaml` 文件 → 扫描文件名，加上 `{agent_name}/` 前缀
4. **同一 Agent 内**多种格式共存时按原始 name 去重

**mcp_info** — 归一化后的 MCP 依赖列表（Agent 维度）：

```json
[
  {
    "name": "wecom-assistant/wecom-api",
    "agent": "wecom-assistant",
    "description": "企业微信开放平台 API",
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-wecom"],
    "required_env": ["WECOM_CORPID", "WECOM_CORPSECRET"]
  },
  {
    "name": "tapd-task-manager/tapd",
    "agent": "tapd-task-manager",
    "description": "TAPD project management",
    "package": "@openpeng/mcp-tapd",
    "tools": ["tapd_create_story", "tapd_list_stories"],
    "required_env": ["TAPD_API_KEY"]
  }
]
```

归一化规则：
1. 如果 agent.json 有 `mcp_servers` 数组 → 直接提取，name 加上 `{agent_name}/` 前缀
2. 如果 agent.json 有 `mcp.required_servers` → 同上处理
3. 如果包内有 `mcp/config.json` → 读取 mcpServers，name 加上前缀
4. **同一 Agent 内**多种格式共存时按原始 name 去重

---

## 3. Skill & MCP 作为市场一级公民

### 3.1 设计理念

借鉴主流包管理平台的设计：

| 平台 | 类比 |
|------|------|
| **npm** | 包 → 依赖 (dependencies)，也可独立搜索 `npm search` |
| **PyPI** | 包 → classifiers，可按分类独立浏览 |
| **Docker Hub** | 镜像 → layers/tags，可按标签独立搜索 |
| **VS Code Marketplace** | Extension → contributes.commands，可浏览命令 |

**Agent Market 方案**：Skills 和 MCP Servers 作为独立实体，有自己的注册表，通过关联表追踪与 Agent 的关系。

### 3.2 独立注册 Skill（非 Agent 上传渠道）

除从 agent.json 提取外，支持通过 API 独立注册 Skill/MCP Server：

```bash
# 独立注册一个 skill
curl -X POST /api/v1/skills \
  -H "Authorization: Bearer $API_KEY" \
  -d '{
    "id": "standalone/text-summarizer",
    "display_name": "文本摘要",
    "description": "Summarizes text into concise bullet points",
    "version": "1.0.0",
    "category": "nlp"
  }'
```

**使用场景**：
- 尚未绑定到具体 Agent 的 skill/MCP 原型
- 第三方 MCP server 注册到市场供 Agent 引用
- 市场管理员预置的基础能力库

### 3.3 数据库查询（替换 JSON 聚合）

**查询全市场所有 Skills**：

```sql
SELECT s.*, GROUP_CONCAT(a_s.agent_id) as agent_ids
FROM skills s
LEFT JOIN agent_skills a_s ON s.id = a_s.skill_id
GROUP BY s.id
ORDER BY s.display_name
```

**查询某 Skill 被哪些 Agent 使用**：

```sql
SELECT a.* FROM agents a
JOIN agent_skills a_s ON a.id = a_s.agent_id
WHERE a_s.skill_id = ?
```

**查询某 Agent 包含的所有 Skills**：

```sql
SELECT s.* FROM skills s
JOIN agent_skills a_s ON s.id = a_s.skill_id
WHERE a_s.agent_id = ?
```

> **效率对比**：索引 JOIN 查询比 `json_each()` 全表扫描快一个数量级，1000+ Agent 场景下优势明显。

### 3.4 市场级 API 端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/api/v1/skills` | 全市场 Skills 列表（支持 q/category 筛选） |
| `GET` | `/api/v1/skills/{id}` | Skill 详情 + 关联 Agent 列表 |
| `POST` | `/api/v1/skills` | 独立注册 Skill（需 publisher 认证） |
| `DELETE` | `/api/v1/skills/{id}` | 删除 Skill（需 admin 认证） |
| `GET` | `/api/v1/mcp-servers` | 全市场 MCP Servers 列表（支持 q 筛选） |
| `GET` | `/api/v1/mcp-servers/{id}` | MCP Server 详情 + 关联 Agent 列表 |
| `POST` | `/api/v1/mcp-servers` | 独立注册 MCP Server（需 publisher 认证） |
| `DELETE` | `/api/v1/mcp-servers/{id}` | 删除 MCP Server（需 admin 认证） |

**`GET /api/v1/skills` 响应**：

```json
{
  "total": 8,
  "skills": [
    {
      "id": "wecom-assistant/wecom-message",
      "original_name": "wecom-message",
      "display_name": "企微消息助手",
      "description": "企业微信消息管理助手",
      "category": "企业协作",
      "agent_count": 1
    },
    {
      "id": "content-processor/text-summarizer",
      "original_name": "text-summarizer",
      "display_name": "文本摘要",
      "description": "Summarizes text into concise bullet points",
      "category": "nlp",
      "agent_count": 3
    }
  ]
}
```

查询参数：`?q=`（搜索描述/名称）、`?category=`（分类筛选）、`?page=` / `?page_size=`

**`GET /api/v1/skills/{id}` 响应**：

```json
{
  "id": "content-processor/text-summarizer",
  "original_name": "text-summarizer",
  "display_name": "文本摘要",
  "description": "Summarizes text into concise bullet points",
  "version": "2.1.0",
  "category": "nlp",
  "agents": [
    { "id": "content-processor", "name": "Content Processor", "version": "1.0.0" },
    { "id": "daily-report", "name": "日报生成器", "version": "2.1.0" }
  ]
}
```

### 3.5 Agent 上传时的同步逻辑

Agent 注册/更新时，提取的 skill/MCP 信息自动同步到独立表：

```
上传 Agent 包
  ├── extract_skills_info()  → 提取 skill 列表
  ├── sync_skills(skills)     → INSERT OR REPLACE INTO skills
  │   └── INSERT OR REPLACE INTO agent_skills (agent_id, skill_id)
  ├── extract_mcp_info()      → 提取 MCP 列表
  └── sync_mcp_servers(mcp)   → INSERT OR REPLACE INTO mcp_servers
      └── INSERT OR REPLACE INTO agent_mcp_servers (agent_id, mcp_server_id)
```

**关键规则**：
1. Skill/MCP 表使用 `INSERT OR REPLACE` 保证幂等性
2. 关联表**先清除旧关联**再写入新关联（Agent 更新可能增减 skill）
3. 级联删除：Agent 删除时 `ON DELETE CASCADE` 自动清理关联

> **废弃 agents 表的 `skills_info` / `mcp_info` JSON 列**。之前提案的 ALTER TABLE 方案撤回，改用独立表设计。existing agents 表中的此列无需回填（新方案不依赖它）。

---

## 4. 代码变更

### 4.1 数据库层 (`database.py`)

**initialize() 迁移** — 在现有 `CREATE TABLE` 块中追加 4 张新表：

```python
async def initialize(self):
    # ... 现有 agents / ratings / downloads / api_keys 建表语句 ...
    await self._conn.executescript("""
        -- 新增：Skill 定义表
        CREATE TABLE IF NOT EXISTS skills (
            id TEXT PRIMARY KEY,
            original_name TEXT NOT NULL,
            display_name TEXT NOT NULL DEFAULT '',
            description TEXT NOT NULL DEFAULT '',
            version TEXT NOT NULL DEFAULT '',
            category TEXT NOT NULL DEFAULT '',
            icon TEXT NOT NULL DEFAULT '',
            created_at TEXT NOT NULL DEFAULT (datetime('now')),
            updated_at TEXT NOT NULL DEFAULT (datetime('now'))
        );
        -- 新增：MCP Server 定义表
        CREATE TABLE IF NOT EXISTS mcp_servers (
            id TEXT PRIMARY KEY,
            original_name TEXT NOT NULL,
            description TEXT NOT NULL DEFAULT '',
            command TEXT NOT NULL DEFAULT '',
            args TEXT NOT NULL DEFAULT '[]',
            package TEXT NOT NULL DEFAULT '',
            tools TEXT NOT NULL DEFAULT '[]',
            required_env TEXT NOT NULL DEFAULT '[]',
            created_at TEXT NOT NULL DEFAULT (datetime('now')),
            updated_at TEXT NOT NULL DEFAULT (datetime('now'))
        );
        -- 新增：Agent-Skill 关联表
        CREATE TABLE IF NOT EXISTS agent_skills (
            agent_id TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
            skill_id TEXT NOT NULL REFERENCES skills(id) ON DELETE CASCADE,
            PRIMARY KEY (agent_id, skill_id)
        );
        -- 新增：Agent-MCP 关联表
        CREATE TABLE IF NOT EXISTS agent_mcp_servers (
            agent_id TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
            mcp_server_id TEXT NOT NULL REFERENCES mcp_servers(id) ON DELETE CASCADE,
            PRIMARY KEY (agent_id, mcp_server_id)
        );
        CREATE INDEX IF NOT EXISTS idx_skills_original_name ON skills(original_name);
        CREATE INDEX IF NOT EXISTS idx_mcp_servers_original_name ON mcp_servers(original_name);
        CREATE INDEX IF NOT EXISTS idx_agent_skills_skill_id ON agent_skills(skill_id);
        CREATE INDEX IF NOT EXISTS idx_agent_mcp_servers_server_id ON agent_mcp_servers(mcp_server_id);
    """)
```

**新增数据库方法**：

```python
# ---- Skill CRUD ----

async def upsert_skill(self, skill_data: dict):
    """INSERT OR REPLACE 技能定义"""
    now = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
    await self._conn.execute(
        """INSERT OR REPLACE INTO skills
           (id, original_name, display_name, description, version, category, icon, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?)""",
        (skill_data["id"], skill_data["original_name"], skill_data.get("display_name", ""),
         skill_data.get("description", ""), skill_data.get("version", ""),
         skill_data.get("category", ""), skill_data.get("icon", ""), now)
    )
    await self._conn.commit()

async def get_skill(self, skill_id: str) -> dict | None:
    cursor = await self._conn.execute("SELECT * FROM skills WHERE id=?", (skill_id,))
    row = await cursor.fetchone()
    return dict(row) if row else None

async def list_skills(self, q="", category="", page=1, page_size=20):
    conditions, params = [], []
    if q:
        conditions.append("(original_name LIKE ? OR display_name LIKE ? OR description LIKE ?)")
        like = f"%{q}%"
        params.extend([like, like, like])
    if category:
        conditions.append("category=?")
        params.append(category)
    where = " AND ".join(conditions) if conditions else "1=1"
    cursor = await self._conn.execute(
        f"""SELECT s.*, COUNT(a_s.agent_id) as agent_count
            FROM skills s
            LEFT JOIN agent_skills a_s ON s.id = a_s.skill_id
            WHERE {where} GROUP BY s.id ORDER BY s.display_name LIMIT ? OFFSET ?""",
        params + [page_size, (page-1)*page_size]
    )
    rows = await cursor.fetchall()
    cursor = await self._conn.execute(
        f"SELECT COUNT(*) FROM skills WHERE {where}", params
    )
    total = (await cursor.fetchone())[0]
    return total, [dict(r) for r in rows]

async def delete_skill(self, skill_id: str) -> bool:
    cursor = await self._conn.execute("DELETE FROM skills WHERE id=?", (skill_id,))
    await self._conn.commit()
    return cursor.rowcount > 0

# ---- MCP Server CRUD ----

async def upsert_mcp_server(self, mcp_data: dict):
    """INSERT OR REPLACE MCP 服务定义"""
    now = datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%SZ")
    await self._conn.execute(
        """INSERT OR REPLACE INTO mcp_servers
           (id, original_name, description, command, args, package, tools, required_env, updated_at)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
        (mcp_data["id"], mcp_data["original_name"], mcp_data.get("description", ""),
         mcp_data.get("command", ""), json.dumps(mcp_data.get("args", [])),
         mcp_data.get("package", ""), json.dumps(mcp_data.get("tools", [])),
         json.dumps(mcp_data.get("required_env", [])), now)
    )
    await self._conn.commit()

async def list_mcp_servers(self, q="", page=1, page_size=20):
    # 类似 list_skills，按 mcp_servers + agent_mcp_servers 聚合
    ...

async def delete_mcp_server(self, mcp_id: str) -> bool:
    ...

# ---- 关联表操作 ----

async def sync_agent_skills(self, agent_id: str, skill_ids: list[str]):
    """替换 Agent 的 skill 关联关系（先删后插）"""
    await self._conn.execute("DELETE FROM agent_skills WHERE agent_id=?", (agent_id,))
    for skill_id in skill_ids:
        await self._conn.execute(
            "INSERT OR IGNORE INTO agent_skills (agent_id, skill_id) VALUES (?, ?)",
            (agent_id, skill_id)
        )
    await self._conn.commit()

async def sync_agent_mcp_servers(self, agent_id: str, mcp_ids: list[str]):
    """替换 Agent 的 MCP server 关联关系（先删后插）"""
    await self._conn.execute("DELETE FROM agent_mcp_servers WHERE agent_id=?", (agent_id,))
    for mcp_id in mcp_ids:
        await self._conn.execute(
            "INSERT OR IGNORE INTO agent_mcp_servers (agent_id, mcp_server_id) VALUES (?, ?)",
            (agent_id, mcp_id)
        )
    await self._conn.commit()

async def get_agent_skills(self, agent_id: str) -> list[dict]:
    """获取 Agent 的所有 Skills（通过关联表 JOIN）"""
    cursor = await self._conn.execute(
        "SELECT s.* FROM skills s JOIN agent_skills a_s ON s.id = a_s.skill_id WHERE a_s.agent_id=?",
        (agent_id,)
    )
    return [dict(r) for r in await cursor.fetchall()]

async def get_agent_mcp_servers(self, agent_id: str) -> list[dict]:
    cursor = await self._conn.execute(
        "SELECT m.* FROM mcp_servers m JOIN agent_mcp_servers a_m ON m.id = a_m.mcp_server_id WHERE a_m.agent_id=?",
        (agent_id,)
    )
    return [dict(r) for r in await cursor.fetchall()]

async def get_skill_agents(self, skill_id: str) -> list[dict]:
    """查询使用某 Skill 的所有 Agent"""
    cursor = await self._conn.execute(
        "SELECT a.id, a.name, a.version FROM agents a JOIN agent_skills a_s ON a.id = a_s.agent_id WHERE a_s.skill_id=?",
        (skill_id,)
    )
    return [dict(r) for r in await cursor.fetchall()]

async def get_mcp_server_agents(self, mcp_id: str) -> list[dict]:
    cursor = await self._conn.execute(
        "SELECT a.id, a.name, a.version FROM agents a JOIN agent_mcp_servers a_m ON a.id = a_m.agent_id WHERE a_m.mcp_server_id=?",
        (mcp_id,)
    )
    return [dict(r) for r in await cursor.fetchall()]
```

### 4.2 Pydantic 模型 (`models.py`)

**新增模型**：

```python
class SkillInfo(BaseModel):
    id: str                  # qualified id: "agent-name/skill-name"
    original_name: str = ""  # 原始 name（不含 agent 前缀）
    display_name: str = ""
    description: str = ""
    version: str = ""
    category: str = ""
    icon: str = ""

class MCPInfo(BaseModel):
    id: str                  # qualified id: "agent-name/server-name"  
    original_name: str = ""  # 原始 name
    description: str = ""
    command: str = ""
    args: list[str] = Field(default_factory=list)
    package: str = ""
    tools: list[str] = Field(default_factory=list)
    required_env: list[str] = Field(default_factory=list)
```

**扩展现有模型**：

```python
class AgentResponse(BaseModel):
    # ... 现有字段 ...
    skills_info: list[SkillInfo] = Field(default_factory=list)
    mcp_info: list[MCPInfo] = Field(default_factory=list)

class AgentListItem(BaseModel):
    # ... 现有字段 ...
    skill_count: int = 0        # Skills 数量（概要显示）
    mcp_server_count: int = 0   # MCP Servers 数量（概要显示）

# ---- 市场级 Skill/MCP 列表模型（新增） ----

class SkillMarketItem(BaseModel):
    id: str
    original_name: str
    display_name: str
    description: str
    category: str
    agent_count: int = 0    # 使用该 Skill 的 Agent 数量

class SkillMarketListResponse(BaseModel):
    total: int = 0
    page: int = 1
    page_size: int = 20
    skills: list[SkillMarketItem] = Field(default_factory=list)

class SkillDetailResponse(BaseModel):
    id: str
    original_name: str
    display_name: str
    description: str
    version: str = ""
    category: str = ""
    agents: list[dict] = Field(default_factory=list)  # [{"id","name","version"},...]

class MCPServerMarketItem(BaseModel):
    id: str
    original_name: str
    description: str
    command: str = ""
    agent_count: int = 0

class MCPServerMarketListResponse(BaseModel):
    total: int = 0
    page: int = 1
    page_size: int = 20
    servers: list[MCPServerMarketItem] = Field(default_factory=list)

class MCPServerDetailResponse(BaseModel):
    id: str
    original_name: str
    description: str
    command: str = ""
    args: list[str] = Field(default_factory=list)
    required_env: list[str] = Field(default_factory=list)
    agents: list[dict] = Field(default_factory=list)
```

### 4.3 服务端 API (`server.py`)

**注册流程增强** — 在 `register_agent()` 中，提取 metadata 后新增同步逻辑：

```python
# 解析 skills 和 mcp 信息（传入提取目录以支持文件系统格式）
skills = _extract_skills_info(metadata, extract_dir)
mcp_list = _extract_mcp_info(metadata, extract_dir)

# 同步到独立表
for skill in skills:
    await db.upsert_skill(skill)
skill_ids = [s["id"] for s in skills]
await db.sync_agent_skills(agent_id, skill_ids)

for mcp in mcp_list:
    await db.upsert_mcp_server(mcp)
mcp_ids = [m["id"] for m in mcp_list]
await db.sync_agent_mcp_servers(agent_id, mcp_ids)
```

**获取 Agent 详情时** — 从关联表读取 skills/mcp：

```python
agent["skills_info"] = await db.get_agent_skills(agent_id)
agent["mcp_info"] = await db.get_agent_mcp_servers(agent_id)
```

**搜索 Agent 时筛选** — 使用 JOIN 而非 LIKE：

```python
async def list_agents(self, ..., skill=None, mcp=None):
    # 按 Skill 筛选：JOIN agent_skills 表
    if skill:
        conditions.append("a.id IN (SELECT agent_id FROM agent_skills WHERE skill_id LIKE ?)")
        params.append(f"%/{skill}")
    # 按 MCP server 筛选：JOIN agent_mcp_servers 表
    if mcp:
        conditions.append("a.id IN (SELECT agent_id FROM agent_mcp_servers WHERE mcp_server_id LIKE ?)")
        params.append(f"%/{mcp}")
```

**新增 Skill/MCP 管理路由**：

```python
# ---- Skills 市场级 API ----

@app.get("/api/v1/skills", response_model=SkillMarketListResponse)
async def list_skills(q="", category="", page=1, page_size=20):
    total, items = await get_db().list_skills(q, category, page, page_size)
    return SkillMarketListResponse(total=total, page=page, page_size=page_size,
                                   skills=[SkillMarketItem(**i) for i in items])

@app.get("/api/v1/skills/{skill_id}", response_model=SkillDetailResponse)
async def get_skill_detail(skill_id: str):
    skill = await get_db().get_skill(skill_id)
    if not skill:
        raise HTTPException(404, f"Skill '{skill_id}' 未找到")
    agents = await get_db().get_skill_agents(skill_id)
    return SkillDetailResponse(**skill, agents=agents)

@app.post("/api/v1/skills", status_code=201)
async def register_skill(data: dict, auth=Depends(verify_publisher)):
    await get_db().upsert_skill(data)
    return {"ok": True, "id": data["id"]}

@app.delete("/api/v1/skills/{skill_id}", status_code=204)
async def delete_skill(skill_id: str, auth=Depends(verify_admin)):
    if not await get_db().delete_skill(skill_id):
        raise HTTPException(404, f"Skill '{skill_id}' 未找到")

# ---- MCP Servers 市场级 API（同理） ----
# GET  /api/v1/mcp-servers
# GET  /api/v1/mcp-servers/{server_id}
# POST /api/v1/mcp-servers
# DELETE /api/v1/mcp-servers/{server_id}
```

### 4.4 包验证增强 (`verify.py`)

新增校验规则：

```python
# ---- Skills 校验 ----
skills = config.get("skills", [])
for i, skill in enumerate(skills):
    if not skill.get("name"):
        errors.append(f"skills[{i}] 缺少 name")
    elif not re.match(r'^[a-zA-Z0-9_\-]+$', skill["name"]):
        errors.append(f"skills[{i}].name 格式无效: {skill['name']}")

# ---- MCP Servers 校验 (agent.json 中的声明式配置) ----
mcp_config = config.get("mcp", {})
mcp_servers = config.get("mcp_servers", [])
all_mcp = (mcp_config.get("required_servers", []) or []) + mcp_servers
mcp_names = []
for i, srv in enumerate(all_mcp):
    if not srv.get("name"):
        errors.append(f"mcp_servers[{i}] 缺少 name")
    else:
        if srv["name"] in mcp_names:
            errors.append(f"mcp_servers 名称重复: {srv['name']}")
        mcp_names.append(srv["name"])

# ---- mcp/config.json 校验 (文件系统中的 MCP 配置) ----
mcp_config_file = pkg_path / "mcp" / "config.json"
if mcp_config_file.exists():
    try:
        mcp_cfg = json.loads(mcp_config_file.read_text(encoding="utf-8"))
        servers = mcp_cfg.get("mcpServers", {})
        for name, srv in servers.items():
            if not isinstance(srv, dict):
                errors.append(f"mcp/config.json 中 server '{name}' 格式无效")
            elif not srv.get("command"):
                errors.append(f"mcp/config.json 中 server '{name}' 缺少 command")
    except json.JSONDecodeError:
        errors.append("mcp/config.json 不是有效的 JSON")

# ---- Subagent Skill 类型一致性 ----
for sa in config.get("subagents", []):
    if sa.get("type") == "skill" and config.get("type") != "agent":
        pass  # 仅记录 warning，不作为 error
```

### 4.5 提取工具函数（新增 `src/market/skills_mcp.py`）

```python
def _extract_skills_info(metadata: dict, extract_dir: Path = None) -> list[dict]:
    """从 agent.json 和文件系统归一化提取 skills 信息"""
    agent_name = metadata.get("identity", {}).get("name", "")
    skills = []

    # 格式 B: agent.json 顶层 skills 数组（agent-builder 使用）
    for skill in metadata.get("skills", []):
        raw_name = skill.get("name", "")
        skills.append({
            "id": f"{agent_name}/{raw_name}" if agent_name else raw_name,
            "original_name": raw_name,
            "display_name": skill.get("display_name", raw_name),
            "description": skill.get("description", ""),
            "version": skill.get("version", ""),
            "category": skill.get("category", ""),
            "icon": skill.get("icon", ""),
        })

    # 格式 A: subagents 中 type: "skill" 的项（v3 规范）
    seen_names = {s["original_name"] for s in skills}
    for sa in metadata.get("subagents", []):
        if sa.get("type") == "skill":
            raw_name = sa.get("name", "")
            if raw_name and raw_name not in seen_names:
                skills.append({
                    "id": f"{agent_name}/{raw_name}",
                    "original_name": raw_name,
                    "display_name": sa.get("display_name", raw_name),
                    "description": sa.get("description", ""),
                    "version": metadata.get("identity", {}).get("version", ""),
                    "category": "",
                    "icon": "",
                })
                seen_names.add(raw_name)

    # 格式 C: skills/*.yaml 文件（Runtime 文件系统加载）
    if extract_dir:
        skills_dir = extract_dir / "skills"
        if skills_dir.is_dir():
            for yaml_file in skills_dir.glob("*.yaml"):
                raw_name = yaml_file.stem
                if raw_name not in seen_names:
                    skills.append({
                        "id": f"{agent_name}/{raw_name}",
                        "original_name": raw_name,
                        "display_name": raw_name,
                        "description": f"Skill from {yaml_file.name}",
                        "version": metadata.get("identity", {}).get("version", ""),
                        "category": "",
                        "icon": "",
                    })
                    seen_names.add(raw_name)

    return skills


def _extract_mcp_info(metadata: dict, extract_dir: Path = None) -> list[dict]:
    """从 agent.json 和文件系统归一化提取 MCP 信息"""
    agent_name = metadata.get("identity", {}).get("name", "")
    mcp_list = []

    # 格式 A1: agent.json 顶层 mcp_servers 数组（agent-builder 使用）
    for srv in metadata.get("mcp_servers", []):
        raw_name = srv.get("name", "")
        mcp_list.append({
            "id": f"{agent_name}/{raw_name}" if agent_name else raw_name,
            "original_name": raw_name,
            "description": srv.get("description", ""),
            "command": srv.get("command", ""),
            "args": srv.get("args", []),
            "package": srv.get("package", ""),
            "tools": srv.get("tools", []),
            "required_env": list(srv.get("env", {}).keys()) if isinstance(srv.get("env"), dict) else [],
        })

    seen_names = {m["original_name"] for m in mcp_list}

    # 格式 A2: mcp.required_servers（v3 规范）
    mcp_config = metadata.get("mcp", {})
    for srv in mcp_config.get("required_servers", []):
        raw_name = srv.get("name", "")
        if raw_name and raw_name not in seen_names:
            mcp_list.append({
                "id": f"{agent_name}/{raw_name}",
                "original_name": raw_name,
                "description": srv.get("description", ""),
                "command": "",
                "args": [],
                "package": srv.get("package", ""),
                "tools": srv.get("tools", []),
                "required_env": srv.get("required_env", []),
            })
            seen_names.add(raw_name)

    # 格式 B: mcp/config.json 文件（Runtime 文件系统加载，Claude Desktop 格式）
    if extract_dir:
        mcp_config_paths = [
            extract_dir / "mcp" / "config.json",
            extract_dir / "mcp" / "servers.json",
        ]
        for cfg_path in mcp_config_paths:
            if cfg_path.is_file():
                try:
                    with open(cfg_path, encoding="utf-8") as f:
                        mcp_cfg = json.load(f)
                    for raw_name, srv_cfg in mcp_cfg.get("mcpServers", {}).items():
                        if raw_name not in seen_names:
                            mcp_list.append({
                                "id": f"{agent_name}/{raw_name}",
                                "original_name": raw_name,
                                "description": srv_cfg.get("description", ""),
                                "command": srv_cfg.get("command", ""),
                                "args": srv_cfg.get("args", []),
                                "package": "",
                                "tools": [],
                                "required_env": list(srv_cfg.get("env", {}).keys()) if isinstance(srv_cfg.get("env"), dict) else [],
                            })
                            seen_names.add(raw_name)
                except (json.JSONDecodeError, OSError):
                    pass  # 解析失败则忽略

    return mcp_list
```

---

## 5. API 变更摘要

### 5.1 新增端点

| 方法 | 路径 | 认证 | 说明 |
|------|------|------|------|
| `GET` | `/api/v1/skills` | — | 全市场 Skills 列表（q/category/page/page_size） |
| `GET` | `/api/v1/skills/{id}` | — | Skill 详情 + 关联 Agent 列表 |
| `POST` | `/api/v1/skills` | publisher | 独立注册 Skill |
| `DELETE` | `/api/v1/skills/{id}` | admin | 删除 Skill |
| `GET` | `/api/v1/mcp-servers` | — | 全市场 MCP Servers 列表（q/page/page_size） |
| `GET` | `/api/v1/mcp-servers/{id}` | — | MCP Server 详情 + 关联 Agent 列表 |
| `POST` | `/api/v1/mcp-servers` | publisher | 独立注册 MCP Server |
| `DELETE` | `/api/v1/mcp-servers/{id}` | admin | 删除 MCP Server |

### 5.2 现有端点扩展参数

| 端点 | 参数 | 类型 | 示例 |
|------|------|------|------|
| `GET /api/v1/agents` | `skill` | string | `?skill=text-summarizer` |
| `GET /api/v1/agents` | `mcp` | string | `?mcp=tapd` |
| `GET /api/v1/discover` | `skill` | string | `?skill=text-summarizer` |
| `GET /api/v1/discover` | `mcp_server` | string | `?mcp_server=tapd` |
| `GET /api/v1/discover` | `has_skill` | string | `?has_skill=true` |
| `GET /api/v1/discover` | `has_mcp` | string | `?has_mcp=true` |

### 5.3 响应字段扩展

**`GET /api/v1/agents/{id}` (AgentResponse)**：

```json
{
  "id": "wecom-assistant",
  "name": "企微智能助手",
  "...": "...",
  "skills_info": [
    {
      "id": "wecom-assistant/wecom-message",
      "original_name": "wecom-message",
      "display_name": "企微消息助手",
      "description": "企业微信消息管理助手",
      "version": "1.0.0",
      "category": "企业协作",
      "icon": "💬"
    }
  ],
  "mcp_info": [
    {
      "id": "wecom-assistant/wecom-api",
      "original_name": "wecom-api",
      "description": "企业微信开放平台 API",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-wecom"],
      "required_env": ["WECOM_CORPID", "WECOM_CORPSECRET"]
    }
  ]
}
```

**`GET /api/v1/discover`**：

```json
{
  "total": 42,
  "agents": [
    {
      "id": "wecom-assistant",
      "name": "企微智能助手",
      "version": "1.0.0",
      "display_name": "企微智能助手",
      "description": "基于企业微信生态的智能办公助手",
      "category": "企业协作",
      "type": "agent",
      "tags": ["企业微信", "办公自动化"],
      "skills": [
        { "name": "wecom-message", "display_name": "企微消息助手" },
        { "name": "wecom-calendar", "display_name": "企微日程管理" }
      ],
      "mcp_servers": [
        { "name": "wecom-api", "description": "企业微信开放平台 API" }
      ]
    }
  ]
}
```

---

## 6. 实施步骤

| 步骤 | 变更文件 | 内容 |
|------|---------|------|
| **Step 1** | `database.py` | 新增 `skills`、`mcp_servers`、`agent_skills`、`agent_mcp_servers` 4 张表；新增 upsert/list/get/delete + sync/get_agent 关联方法 |
| **Step 2** | `models.py` | 新增 `SkillInfo`、`MCPInfo`、`SkillMarketItem`、`MCPServerMarketItem` 及 List/Detail Response 模型 |
| **Step 3** | `skills_mcp.py` (新) | 实现 `_extract_skills_info()` / `_extract_mcp_info()` 提取函数（返回 `id` / `original_name`） |
| **Step 4** | `server.py` | Agent 注册时调用 upsert_skill + sync_agent_skills；新增 8 个 skills/mcp-servers REST 路由；搜索改用 JOIN |
| **Step 5** | `verify.py` | 增加 skills/mcp_servers/mcp-config.json 格式校验 |
| **Step 6** | `discovery.py` | 发现端点增加 `skill` / `mcp_server` / `has_skill` / `has_mcp` 筛选 |
| **Step 7** | `tests/` | skills/mcp 提取单元测试；独立注册 CRUD 测试；关联表完整性测试；Agent 上传同步测试 |

### 测试清单

- [ ] 4 张新表创建幂等（CREATE TABLE IF NOT EXISTS 重复执行不报错）
- [ ] 格式 A（v3 subagent skill）提取正确，id 含 agent 前缀
- [ ] 格式 B（agent-builder skills 数组）提取正确，id 含 agent 前缀
- [ ] 格式 C（skills/*.yaml 文件系统）提取正确
- [ ] 单 Agent 内同名 skill 去重正确
- [ ] 不同 Agent 的同名 skill 不冲突（前缀不同 → 独立记录）
- [ ] 格式 A1（mcp_servers 数组）提取 + upsert 正确
- [ ] 格式 A2（mcp.required_servers v3 格式）提取 + upsert 正确
- [ ] 格式 B（mcp/config.json）提取 + upsert 正确
- [ ] Agent 上传时 skill/MCP 自动同步到独立表
- [ ] Agent 更新时关联表正确重建（先删后插）
- [ ] Agent 删除时关联表 ON DELETE CASCADE 自动清理
- [ ] `POST /api/v1/skills` 独立注册 Skill 成功
- [ ] `DELETE /api/v1/skills/{id}` 删除 Skill（无 Agent 关联时）
- [ ] `GET /api/v1/skills` 返回全市场列表（含 agent_count）
- [ ] `GET /api/v1/skills/{id}` 返回详情 + 关联 Agent 列表
- [ ] `GET /api/v1/mcp-servers` / `{id}` 正确
- [ ] `?skill=text-summarizer` JOIN 筛选 Agent 返回正确结果
- [ ] `?mcp=tapd` JOIN 筛选 Agent 返回正确结果
- [ ] 独立注册的 Skill 与 Agent 提取的 Skill 共存正确
- [ ] 无 skills/mcp 的旧 Agent 不影响主流程
- [ ] 上传/下载流程不受影响（回归测试）

---

## 7. 向后兼容性

| 场景 | 兼容性 |
|------|--------|
| 现有 Agent 包上传 | ✅ 自动提取并同步到新表，不影响上传流程 |
| 现有 API 响应 | ✅ 新增字段在 JSON 中追加，调用方忽略未知字段 |
| 数据库迁移 | ✅ CREATE TABLE IF NOT EXISTS 幂等，不影响现有表 |
| 无 skills/mcp 的旧 Agent | ✅ skills/mcp 表无记录，agent_skills 关联为空 |
| 新 Agent 包含 skills/mcp | ✅ 自动 INSERT OR REPLACE 到独立表 + 建立关联 |
| agents 表结构 | ✅ 不变，无需 ALTER TABLE |
| 独立注册 skill/mcp | ✅ POST /skills 和 POST /mcp-servers 新端点 |
| 搜索/筛选 | ✅ JOIN 替代 LIKE，新增可选参数 |

---

## 8. 开放问题

1. **Qualified Name 分隔符**：使用 `/` (如 `wecom-assistant/wecom-message`) 还是 `::` (如 `wecom-assistant::wecom-message`)？
   - **建议**：使用 `/`，与 Docker image、npm scope 等主流平台一致。

2. **Skill/MCP 跨 Agent 复用**：如果两个 Agent 的 skill 语义相同（如都是 `text-summarizer`），`id` 中含不同 agent 前缀导致存储为不同记录，是否可以建立"等价关系"？
   - **建议**：Phase 1 不处理语义级去重，每个 agent 前缀的记录独立。后续可加 `canonical_id` 字段做归一化。

3. **孤立数据清理**：当 Skill/MCP Server 不再被任何 Agent 使用时（关联表中无记录），是否自动删除该 Skill/MCP 记录？
   - **建议**：Phase 1 保留（可能被独立注册或后续复用），后续可加定时清理任务或手动 GC API。

4. **Skills 版本管理**：Agent subagent skill 的 version 应取自 skill 自身的 agent.json 还是父 Agent 的版本号？
   - **建议**：优先从 skill 自身的 agent.json 读取，回退到父 Agent 版本号。

5. **MCP 敏感信息**：独立注册 MCP Server 时，`env` 中不应包含实际 secret。是否需要在上传时校验？
   - **建议**：校验不以 `${` 开头的环境变量值（可能是硬编码 secret）。

6. **独立注册的权限**：`POST /api/v1/skills` 应允许 publisher 注册还是仅 admin？
   - **建议**：publisher 可注册，admin 可删除。与 Agent 注册的权限模型一致。

---

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [Skill System ��范](./skill-system.md)
- [MCP Integration 规范](./mcp-integration.md)
- [Agent Discovery Protocol](./discovery.md)
- [wecom-assistant agent.json 示例](../../agent-builder/agents/wecom-assistant/agent.json)

---

**Market Skills & MCP 支持 — 市场具备完整的 Agent 能力发现** 🧩🔌
