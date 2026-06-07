# agent.json v3 ↔ OpenAI Agents SDK 兼容分析

**状态**: Research  
**日期**: 2026-06-07

---

## 映射关系

| agent.json v3 | OpenAI Agents SDK | 兼容度 |
|---------------|-------------------|--------|
| `identity.name` | `Agent.name` | ✅ 直接映射 |
| `instructions` | `Agent.instructions` | ✅ 直接映射 |
| `identity.description` | `Agent.description` | ✅ 直接映射 |
| `entry.main_subagent` | (无直接等价) | ⚠️ v3 独有概念 |
| `subagents` | `Agent.handoffs` | ⚠️ 语义相近 |
| `dependencies.llm_provider` | `Agent.model` | ⚠️ 需转换 |
| `worker.yaml pipeline` | (无等价) | ❌ pipeline 是 v3 独有 |

### 详细分析

#### 直接兼容的字段

```json
// agent.json v3
{
  "identity": { "name": "reviewer", "description": "Code reviewer" },
  "instructions": { "source": "inline", "content": "Review the code..." }
}

// → OpenAI Agents SDK
Agent.create({
  name: "reviewer",
  description: "Code reviewer",
  instructions: "Review the code..."
})
```

#### Handoffs ↔ Subagents

OpenAI Agents SDK 的 `handoffs` 允许 Agent 将控制权交给其他 Agent。
agent.json v3 的 `subagents` + `invoke_agent` 实现类似功能。

```json
// agent.json v3
{
  "subagents": [
    { "name": "reviewer", "path": "reviewer.yaml" },
    { "name": "formatter", "path": "formatter.yaml" }
  ]
}

// → OpenAI Agents SDK
Agent.create({
  name: "orchestrator",
  handoffs: [reviewerAgent, formatterAgent]
})
```

#### 不兼容的概念

- **Pipeline**: agent.json v3 的 YAML Pipeline 在 OpenAI Agents SDK 中没有直接等价物。需要在 bridge 层将 YAML steps 编译为 Agent tool calls。
- **Market**: Agent Market 的分发机制在 OpenAI 生态中没有等价物。

---

## 建议的 Bridge 方向

1. **agent.json → OpenAI Agents**: 提取 `identity` + `instructions` 作为 Agent 定义，`subagents` 转为 `handoffs`，`worker.yaml` 跳过（使用内置 tool 等价物）
2. **OpenAI Agents → agent.json**: 提取 Agent 定义生成 agent.json，`handoffs` 转为 `subagents`

**实现优先级**: 低（研究阶段，Market 生态成熟后评估）
