# Changelog

All notable changes to the Agent Protocol specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

## Upcoming in v3.1

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
| 3.0.x | ✅ Current | - | Full support, active development |
| 2.0.x | ⚠️ Maintenance | 2027-06-01 | Auto-converted to v3, no new features |
| 1.0.x | ❌ Deprecated | 2026-12-31 | Migrate to v2 or v3 |

---

## Migration Timeline

- **2026-06-07**: v3.0.0 released
- **2026-09-01**: v2 enters maintenance mode (security fixes only)
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
