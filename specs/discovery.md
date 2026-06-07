# Agent Discovery Protocol

**版本**: 1.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

Agent Discovery Protocol (ADP) 定义了外部系统和 AI 工具如何发现 Market 上可用的 Agent。

---

## 端点

```
GET /api/v1/discover?capability=<string>&category=<string>&format=<string>
```

### 查询参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `capability` | string | 按能力筛选（如 `code-review`, `test-generation`） |
| `category` | string | 按分类筛选（如 `development`, `security`） |
| `format` | string | 按支持的输出格式筛选（如 `codebuddy`, `cursor`） |

### 响应

```json
{
  "total": 42,
  "agents": [
    {
      "id": "agent-abc123",
      "name": "code-reviewer",
      "version": "1.0.0",
      "display_name": "Code Reviewer",
      "description": "Comprehensive code review agent",
      "author": "OpenPeng",
      "category": "development",
      "type": "agent",
      "tags": ["code", "review", "development"],
      "dependencies": {
        "llm_provider": "anthropic"
      },
      "rating": 4.5,
      "download_count": 156,
      "compatibility": {
        "cursor": true,
        "claude_code": true,
        "codebuddy": true
      }
    }
  ]
}
```

---

## 设计原则

1. **自描述**: 响应包含足够的元数据，消费者无需额外查询即可判断 Agent 是否适用
2. **兼容**: 遵循 RESTful 惯例，易于集成到任何 HTTP 客户端
3. **可过滤**: 支持按能力、分类、格式等维度筛选，减少传输量
4. **可扩展**: 字段集可随时间扩展，消费者应忽略未知字段
