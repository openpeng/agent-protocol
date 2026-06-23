# Changelog

All notable changes to the Agent Protocol specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [3.1.0] - 2026-06-23

### Added

**Skill & MCP Independent Packaging**:
- Skill 独立打包规范 — `skill.json` + `SKILL.md` + `scripts/` + `templates/`
- MCP Server 独立打包规范 — `mcp-server.json` + `mcp-config.json`
- CLI 打包命令：`agent-deploy skill pack`、`agent-deploy mcp pack`
- CLI 上传/下载命令：`agent-deploy skill upload/download`、`agent-deploy mcp upload/download`

**Market Reference Mechanism**:
- Agent `skills` 数组支持 `ref` + `version` + `market_url` 市场引用模式
- Agent `mcp_servers` 数组支持 `ref` + `version` + `market_url` 市场引用模式
- 混合模式：同一 Agent 中内联定义与市场引用共存
- 市场端点：`POST /skills/upload`、`GET /skills/{id}/download`、`POST /mcp-servers/upload`、`GET /mcp-servers/{id}/download`

**Version Constraints**:
- 语义化版本约束语法：`^1.0.0`、`~1.0.0`、`>=1.0.0`、`*`、精确版本
- 运行时版本解析器：查询市场可用版本，按约束匹配并自动降级

**Runtime Resolver & Cache Management**:
- 运行时引用解析器：Agent 启动时自动解析 `skills`/`mcp_servers` 引用
- 本地缓存目录：`~/.agent-hub/cache/skills/`、`~/.agent-hub/cache/mcp-servers/`
- 缓存索引：`index.json` 记录版本、路径、下载时间、etag
- CLI 缓存管理：`agent-deploy cache status`、`agent-deploy cache clean`、`agent-deploy cache update`

**Migration Tool**:
- `agent-deploy migrate --from 3.0 --to 3.1` 自动迁移工具
- 将 `skills/*/agent.json`（`type: "skill"`）转换为 `skills/*/skill.json`
- 将 `subagents` 中 `type: "skill"` 迁移至顶层 `skills` 数组

**Specifications**:
- `SPEC_skill_mcp_reference.md` — Skill & MCP 独立打包与引用机制完整 SPEC
- `specs/market-skill-mcp-support.md` — 市场 Skill & MCP 支持规范
- `specs/skill-system.md` — 更新至 v3.1（`skill.json` 为 Skill 唯一格式）
- `specs/mcp-integration.md` — 更新至 v3.1（引用模式、混合模式）

### Changed

**Breaking Changes**:
- Skill 统一使用 `skill.json` 定义，不再使用 `agent.json` + `type: "skill"`
- `schema_version` 取值更新为 `"3.1"`（仍兼容 `"3.0"`）

**Backward Compatible**:
- 运行时仍支持解析 v3.0 的 `subagents type: "skill"` 格式（发出弃用警告）
- 现有内联 `skills` 数组完全兼容（无 `ref` 字段时按内联处理）
- 现有内联 MCP 配置完全兼容（无 `ref` 字段时按内联处理）

### Deprecated

- `subagents` 中 `type: "skill"`（迁移至顶层 `skills` 数组）
- Skill 使用 `agent.json` 定义（迁移至 `skill.json`）

---

## [3.0.0] - 2026-06-07

### Added

**Core Features**:
- ✅ Pipeline workflow system (worker.yaml) — 串行 + invoke_parallel 并行执行
- ✅ Builtin tools system (9 工具: llm_chat, read_file, write_file, bash, glob, web_fetch, web_search, invoke_agent, list_agents)
- ✅ Subagent composition mechanism (invoke + invoke_parallel)
- ✅ Dynamic Overrides (Phase 8: instructions/skills/MCP shared_context/trusted/cwd/env)
- ✅ Entry point declaration
- ✅ Dependencies declaration + DFS 循环检测
- ✅ Template variable system ({{var}}, {{steps.x.output}}, {{shared.key}})
- ✅ Conditional execution (when clauses with ==/!=/>/< operators)
- ✅ Error handling strategies (on_fail: abort/skip/continue + retry with exponential backoff)
- ✅ Shared context across pipeline steps
- ✅ Agent cache (manifest + semver)

**Specifications**:
- agent.json v3 complete specification
- worker.yaml pipeline specification
- Builtin tools specification
- JSON Schemas for agent.json and worker.yaml

**Examples**:
- minimal-agent - Basic hello world example
- file-summarizer - Complete file processing example
- multi-subagent - Advanced orchestrator pattern

**Documentation**:
- Migration guide (v2 → v3)
- Complete API reference
- Best practices guide

### Changed

**Breaking Changes**:
- `schema_version` now required (must be "3.0")
- `entry` field now required (defines entry point)
- `subagents` field now required (at least one subagent)

**Backward Compatible**:
- `instructions` field still supported (v2 compatibility)
- `capabilities` field still supported (v2 compatibility)
- `compatibility` field still supported (v2 compatibility)

### Deprecated

- Static `instructions` field (use worker.yaml pipeline instead)
- Implicit dependencies (use explicit `dependencies` declaration)

---

## [2.0.0] - 2026-05-15

### Added

- Standardized `identity` structure
- `instructions` field with format/source options
- Support for external instruction files
- `capabilities` array
- `compatibility` object

### Changed

- Unified metadata structure
- Consistent naming conventions

### Deprecated

- Flat structure in agent.json (use `identity` instead)

---

## [1.0.0] - 2026-04-01

### Added

- Initial specification
- SKILL.md format support
- Basic agent metadata

---

## Upcoming in v3.2

### Planned Features

- ⏳ Loop constructs (for_each over arrays)
- ⏳ Pipeline debugging tools
- ⏳ Performance profiling & monitoring
- ⏳ Agent lifecycle management (promote/demote workflow)
- ⏳ Multi-version agent support (install specific versions)
- ⏳ Agent health check & timeout controls
- ⏳ Template variable scoping improvements

---

## Version Support

| Version | Status | Support Until | Notes |
|---------|--------|---------------|-------|
| 3.1.x | ✅ Current | - | Full support, active development |
| 3.0.x | ⚠️ Maintenance | 2027-06-01 | Auto-converted to v3.1, no new features |
| 2.0.x | ⚠️ Maintenance | 2027-06-01 | Auto-converted to v3, no new features |
| 1.0.x | ❌ Deprecated | 2026-12-31 | Migrate to v2 or v3 |

---

## Migration Timeline

- **2026-06-23**: v3.1.0 released (Skill/MCP market references, independent packaging)
- **2026-06-07**: v3.0.0 released
- **2026-09-01**: v3.0 enters maintenance mode (security fixes only)
- **2026-09-01**: v2 enters maintenance mode (security fixes only)
- **2027-06-01**: v3.0 support ends
- **2027-06-01**: v2 support ends
- **2026-12-31**: v1 support ends

---

## How to Upgrade

### From v2 to v3

See [migration-v2-to-v3.md](./compatibility/migration-v2-to-v3.md)

Quick steps:
1. Update `schema_version` to "3.0"
2. Add `entry` and `subagents` fields
3. Create worker.yaml from instructions
4. Test with `agent-deploy run`

### From v1 to v3

1. First migrate v1 → v2 (see v2 migration guide)
2. Then migrate v2 → v3 (see above)

Or use automated tool:
```bash
agent-deploy migrate ./my-v1-agent --target v3
```

---

## Community

- **Discussions**: https://github.com/agent-protocol/discussions
- **Issues**: https://github.com/agent-protocol/issues
- **Pull Requests**: https://github.com/agent-protocol/pulls

---

**Agent Protocol - Building the future of AI agents** 🚀
