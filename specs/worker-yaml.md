# worker.yaml Pipeline 规范

**版本**: 3.0.0  
**状态**: Draft  
**最后更新**: 2026-06-07

---

## 概述

`worker.yaml` 定义了 Agent 的执行流程（Pipeline），包括：
- 工具声明
- 步骤编排
- 数据流转
- 条件执行
- 错误处理

它是 Agent Protocol 的**核心运行时规范**。

---

## 完整结构

```yaml
# 工具声明
tools:
  - name: tool_name
    type: builtin | custom
    config: {}

# 共享上下文（可选）
shared_context:
  key: value

# 执行流程
pipeline:
  - step: step_name
    tool: tool_name
    args:
      param: value
    output: variable_name
    on_fail: abort | skip | continue | retry(n)
    when: condition_expression
```

---

## 1. tools 工具声明

### 语法

```yaml
tools:
  - name: string          # ✅ 必填：工具名称
    type: builtin|custom  # ✅ 必填：工具类型
    config: {}            # 可选：工具配置
```

### 工具类型

#### builtin - 内置工具

由运行时提供，无需额外配置。

```yaml
tools:
  - name: read_file
    type: builtin
  
  - name: write_file
    type: builtin
  
  - name: bash
    type: builtin
  
  - name: llm_chat
    type: builtin
```

**内置工具列表**（详见 [builtin-tools.md](./builtin-tools.md)）：
- `read_file` - 读取文件
- `write_file` - 写入文件
- `bash` - 执行 Shell 命令
- `glob` - 文件模式匹配
- `web_fetch` - HTTP 请求
- `web_search` - 网络搜索
- `llm_chat` - 调用大模型

#### custom - 自定义工具

需要额外配置或脚本实现。

```yaml
tools:
  - name: my_parser
    type: custom
    config:
      script: ./libs/parser.py
      args: ["--format", "json"]
```

---

## 2. shared_context 共享上下文

### 语法

```yaml
shared_context:
  key1: value1
  key2: value2
```

### 用途

在所有 pipeline 步骤间共享的上下文变量。

```yaml
shared_context:
  project_dir: /home/user/project
  output_dir: /tmp/agent-output
  max_retries: 3

pipeline:
  - step: read
    tool: read_file
    args:
      path: "{{shared_context.project_dir}}/README.md"
  
  - step: write
    tool: write_file
    args:
      path: "{{shared_context.output_dir}}/summary.txt"
      content: "{{steps.analyze.output}}"
```

---

## 3. pipeline 执行流程

### 步骤语法

```yaml
pipeline:
  - step: string              # ✅ 必填：步骤唯一名称
    tool: string              # ✅ 必填：使用的工具名
    args: {}                  # ✅ 必填：工具参数
    output: string            # 可选：输出变量名
    on_fail: string           # 可选：失败策略
    when: string              # 可选：执行条件
```

### 字段详解

#### step - 步骤名称

**规则**：
- 唯一性：同一 pipeline 内不能重复
- 格式：`snake_case` 推荐
- 用途：被其他步骤引用（`{{steps.step_name.output}}`）

```yaml
pipeline:
  - step: read_config      # ✅ Good
  - step: parseData        # ⚠️  可以，但推荐 parse_data
  - step: 读取配置         # ❌ Bad - 避免中文
```

#### tool - 工具名称

指向 `tools` 中声明的工具。

```yaml
tools:
  - name: llm_chat
    type: builtin

pipeline:
  - step: analyze
    tool: llm_chat  # 引用上面声明的工具
```

#### args - 工具参数

传递给工具的参数，支持**模板变量**。

```yaml
pipeline:
  - step: greet
    tool: llm_chat
    args:
      prompt: "Say hello to {{user_name}}"
      system_prompt: "You are friendly"
      model: "claude-3-5-sonnet"
```

#### output - 输出变量

将步骤的输出保存到变量，供后续步骤使用。

```yaml
pipeline:
  - step: read
    tool: read_file
    args:
      path: "data.txt"
    output: raw_content  # 保存输出

  - step: analyze
    tool: llm_chat
    args:
      prompt: "Analyze: {{steps.read.output}}"  # 引用上一步输出
    output: analysis
```

**引用方式**：
- `{{steps.step_name.output}}` - 引用某步骤的输出
- `{{steps.read.output}}` - 简写形式
- `{{steps.read.raw_content}}` - 如果工具返回对象，可访问字段

#### on_fail - 失败策略

定义步骤失败时的行为。

```yaml
pipeline:
  - step: try_llm
    tool: llm_chat
    args:
      prompt: "Process data"
    on_fail: continue  # LLM 失败时继续执行

  - step: fallback
    tool: bash
    args:
      command: "python fallback.py"
    when: "{{steps.try_llm.success}} == false"
```

**策略类型**：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `abort` | 立即中止整个 pipeline（默认） | 关键步骤失败 |
| `skip` | 跳过此步骤，继续执行 | 可选步骤 |
| `continue` | 记录失败但继续，`success=false` | 需要 fallback |
| `retry(n)` | 重试 n 次，仍失败则 abort | 临时错误（网络） |

**示例**：

```yaml
# 示例 1: 关键步骤必须成功
- step: read_config
  tool: read_file
  args:
    path: "config.json"
  on_fail: abort  # 默认，可省略

# 示例 2: 可选步骤
- step: read_cache
  tool: read_file
  args:
    path: ".cache/data.json"
  on_fail: skip  # 缓存不存在也没关系

# 示例 3: Fallback 模式
- step: try_llm
  tool: llm_chat
  args:
    prompt: "Translate: {{text}}"
  on_fail: continue  # 失败了继续

- step: fallback_translation
  tool: bash
  args:
    command: "python simple_translate.py"
  when: "{{steps.try_llm.success}} == false"

# 示例 4: 重试
- step: fetch_data
  tool: web_fetch
  args:
    url: "https://api.example.com/data"
  on_fail: retry(3)  # 重试 3 次
```

#### when - 条件执行

根据条件决定是否执行此步骤。

```yaml
pipeline:
  - step: check_file
    tool: bash
    args:
      command: "test -f data.txt && echo exists || echo missing"
    output: file_status

  - step: download
    tool: web_fetch
    args:
      url: "https://example.com/data.txt"
      output: "data.txt"
    when: "{{steps.check_file.output}} == 'missing'"  # 仅当文件不存在时下载
```

**条件表达式语法**：

```yaml
# 1. 布尔比较
when: "{{steps.prev.success}} == true"
when: "{{steps.prev.success}} == false"

# 2. 字符串比较
when: "{{steps.check.output}} == 'found'"
when: "{{steps.check.output}} != 'error'"

# 3. 数字比较
when: "{{steps.count.output}} > 0"
when: "{{steps.score.output}} >= 80"

# 4. 存在性检查
when: "{{file_path}} != ''"
when: "{{optional_param}}"  # 非空即为 true

# 5. 逻辑组合（高级，v3.1+）
when: "{{a}} == true && {{b}} > 0"
when: "{{a}} == 'yes' || {{b}} == 'yes'"
```

---

## 4. 模板变量系统

### 变量来源

| 来源 | 语法 | 示例 |
|------|------|------|
| 运行时参数 | `{{var}}` | `{{file_path}}` |
| 步骤输出 | `{{steps.name.output}}` | `{{steps.read.output}}` |
| 共享上下文 | `{{shared_context.key}}` | `{{shared_context.project_dir}}` |
| 环境变量 | `{{env.VAR}}` | `{{env.HOME}}` |

### 使用示例

```yaml
# 运行时参数
pipeline:
  - step: greet
    tool: llm_chat
    args:
      prompt: "Hello {{user_name}}"  # 运行时传入

# 步骤输出
  - step: read
    tool: read_file
    args:
      path: "{{file_path}}"
    output: content
  
  - step: summarize
    tool: llm_chat
    args:
      prompt: "Summarize: {{steps.read.output}}"  # 引用上一步

# 共享上下文
shared_context:
  max_length: 500

pipeline:
  - step: truncate
    tool: bash
    args:
      command: "head -c {{shared_context.max_length}} input.txt"

# 环境变量
  - step: check_home
    tool: bash
    args:
      command: "ls {{env.HOME}}"
```

### 嵌套访问

如果工具返回对象，可以访问嵌套字段：

```yaml
pipeline:
  - step: parse_json
    tool: bash
    args:
      command: "cat config.json"
    output: config  # 假设返回 {"name": "app", "version": "1.0"}

  - step: use_config
    tool: llm_chat
    args:
      prompt: "The app name is {{steps.parse_json.output.name}}"
      # 或者
      prompt: "Version: {{steps.parse_json.config.version}}"
```

---

## 5. 完整示例

### 示例 1: 文件摘要生成器

```yaml
tools:
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin
  - name: write_file
    type: builtin

shared_context:
  max_summary_length: 500

pipeline:
  # 步骤 1: 读取文件
  - step: read_input
    tool: read_file
    args:
      path: "{{file_path}}"
    output: raw_content
    on_fail: abort

  # 步骤 2: 检查内容长度
  - step: check_length
    tool: bash
    args:
      command: "echo '{{steps.read_input.output}}' | wc -c"
    output: content_length

  # 步骤 3: 生成摘要（仅当内容超过阈值）
  - step: generate_summary
    tool: llm_chat
    args:
      system_prompt: |
        You are a summarization expert.
        Generate concise summaries with key points.
      prompt: |
        Summarize the following text in {{shared_context.max_summary_length}} characters:
        
        {{steps.read_input.output}}
      model: "claude-3-5-sonnet"
    output: summary
    when: "{{steps.check_length.output}} > 1000"
    on_fail: continue

  # 步骤 4: Fallback（如果 LLM 失败）
  - step: fallback_summary
    tool: bash
    args:
      command: "echo '{{steps.read_input.output}}' | head -c {{shared_context.max_summary_length}}"
    output: summary
    when: "{{steps.generate_summary.success}} == false"

  # 步骤 5: 写入输出
  - step: write_output
    tool: write_file
    args:
      path: "{{output_path}}"
      content: |
        # Summary
        
        {{steps.generate_summary.output}}
        
        ---
        Generated at: {{timestamp}}
    on_fail: abort
```

**运行**：
```bash
agent-deploy run ./file-summarizer \
  --args file_path=/data/report.txt \
  --args output_path=/tmp/summary.md \
  --args timestamp="2026-06-07 10:30"
```

### 示例 2: 多步骤代码审查

```yaml
tools:
  - name: glob
    type: builtin
  - name: read_file
    type: builtin
  - name: llm_chat
    type: builtin
  - name: write_file
    type: builtin

shared_context:
  review_aspects:
    - security
    - performance
    - maintainability

pipeline:
  # 步骤 1: 查找所有 TypeScript 文件
  - step: find_files
    tool: glob
    args:
      pattern: "src/**/*.ts"
    output: file_list

  # 步骤 2: 读取文件内容（假设批量处理）
  - step: read_files
    tool: bash
    args:
      command: |
        for file in {{steps.find_files.output}}; do
          cat "$file"
        done
    output: all_code

  # 步骤 3: 安全审查
  - step: security_review
    tool: llm_chat
    args:
      system_prompt: "You are a security expert. Find vulnerabilities."
      prompt: |
        Review this code for security issues:
        {{steps.all_code.output}}
      model: "claude-3-5-sonnet"
    output: security_report
    on_fail: continue

  # 步骤 4: 性能分析
  - step: performance_review
    tool: llm_chat
    args:
      system_prompt: "You are a performance expert. Find bottlenecks."
      prompt: |
        Analyze performance issues:
        {{steps.all_code.output}}
    output: performance_report
    on_fail: continue

  # 步骤 5: 生成报告
  - step: generate_report
    tool: write_file
    args:
      path: "review-report.md"
      content: |
        # Code Review Report
        
        ## Security
        {{steps.security_review.output}}
        
        ## Performance
        {{steps.performance_review.output}}
```

### 示例 3: 条件分支与重试

```yaml
tools:
  - name: web_fetch
    type: builtin
  - name: bash
    type: builtin
  - name: llm_chat
    type: builtin

pipeline:
  # 步骤 1: 尝试从 API 获取数据
  - step: fetch_from_api
    tool: web_fetch
    args:
      url: "https://api.example.com/data"
      method: GET
    output: api_data
    on_fail: retry(3)  # 重试 3 次

  # 步骤 2: Fallback 到本地缓存
  - step: load_cache
    tool: bash
    args:
      command: "cat .cache/data.json"
    output: cache_data
    when: "{{steps.fetch_from_api.success}} == false"
    on_fail: skip

  # 步骤 3: 选择数据源
  - step: select_data
    tool: bash
    args:
      command: |
        if [ "{{steps.fetch_from_api.success}}" == "true" ]; then
          echo "{{steps.fetch_from_api.output}}"
        else
          echo "{{steps.load_cache.output}}"
        fi
    output: final_data

  # 步骤 4: 处理数据
  - step: process
    tool: llm_chat
    args:
      prompt: "Process this data: {{steps.select_data.output}}"
    output: result
```

---

## 6. 高级特性

### 循环处理（未来版本）

```yaml
# v3.1+ 规划中
pipeline:
  - step: find_files
    tool: glob
    args:
      pattern: "*.txt"
    output: files

  - step: process_each
    tool: llm_chat
    for_each: "{{steps.find_files.output}}"  # 遍历数组
    args:
      prompt: "Summarize: {{item}}"  # item 为当前元素
    output: summaries  # 输出为数组
```

### 并行执行（未来版本）

```yaml
# v3.1+ 规划中
pipeline:
  - parallel:
      - step: task_a
        tool: llm_chat
        args:
          prompt: "Task A"
      
      - step: task_b
        tool: llm_chat
        args:
          prompt: "Task B"
    
    output: results  # [task_a_result, task_b_result]
```

---

## 7. 最佳实践

### ✅ DO - 推荐做法

**1. 命名清晰**：
```yaml
# ✅ Good
- step: read_config
- step: validate_input
- step: generate_summary

# ❌ Bad
- step: step1
- step: do_stuff
- step: s
```

**2. 错误处理**：
```yaml
# ✅ Good - 关键步骤 abort
- step: read_config
  tool: read_file
  args:
    path: "config.json"
  on_fail: abort

# ✅ Good - 可选步骤 skip
- step: load_cache
  tool: read_file
  args:
    path: ".cache"
  on_fail: skip
```

**3. Fallback 机制**：
```yaml
# ✅ Good
- step: try_llm
  tool: llm_chat
  args:
    prompt: "Translate: {{text}}"
  on_fail: continue

- step: fallback
  tool: bash
  args:
    command: "python simple_translate.py"
  when: "{{steps.try_llm.success}} == false"
```

**4. 输出命名**：
```yaml
# ✅ Good - 语义化命名
- step: read
  output: raw_content

- step: analyze
  output: analysis_result

# ❌ Bad
- step: read
  output: out1

- step: analyze
  output: result
```

### ❌ DON'T - 避免做法

**1. 避免循环依赖**：
```yaml
# ❌ Bad - 无限循环
- step: a
  args:
    input: "{{steps.b.output}}"
  output: a_out

- step: b
  args:
    input: "{{steps.a.output}}"  # 循环依赖！
  output: b_out
```

**2. 避免过深嵌套**：
```yaml
# ❌ Bad - 难以维护
{{steps.a.output.data.items[0].value}}

# ✅ Good - 添加中间步骤
- step: extract_value
  tool: bash
  args:
    command: "echo '{{steps.a.output}}' | jq '.data.items[0].value'"
  output: value
```

**3. 避免硬编码**：
```yaml
# ❌ Bad
- step: process
  args:
    api_key: "sk-1234567890"  # 硬编码密钥！

# ✅ Good
- step: process
  args:
    api_key: "{{env.API_KEY}}"  # 从环境变量读取
```

---

## 8. 错误处理

### 工具执行失败

```yaml
pipeline:
  - step: risky_operation
    tool: web_fetch
    args:
      url: "{{api_url}}"
    on_fail: retry(3)
    output: data
```

**运行时行为**：
1. 第 1 次失败 → 重试
2. 第 2 次失败 → 重试
3. 第 3 次失败 → 重试
4. 第 4 次失败 → abort（中止整个 pipeline）

### 验证错误

```yaml
# 语法错误：缺少必填字段
- step: invalid
  # tool: llm_chat  # 缺少 tool 字段
  args:
    prompt: "test"
```

**运行时错误**：
```
ValidationError: Step 'invalid' missing required field 'tool'
```

---

## 9. 调试技巧

### 启用详细日志

```bash
agent-deploy run ./my-agent --verbose
```

输出：
```
[INFO] Starting pipeline execution
[INFO] Step 'read_input' executing with tool 'read_file'
[DEBUG] Args: {"path": "/data/input.txt"}
[INFO] Step 'read_input' completed in 12ms
[DEBUG] Output: "File content here..."
[INFO] Step 'analyze' executing with tool 'llm_chat'
...
```

### 检查步骤状态

```yaml
pipeline:
  - step: debug_status
    tool: bash
    args:
      command: |
        echo "Previous step success: {{steps.prev.success}}"
        echo "Previous step output: {{steps.prev.output}}"
```

---

## 参考

- [agent.json v3 规范](./agent-json-v3.md)
- [Builtin Tools 规范](./builtin-tools.md)
- [模板变量系统](./template-variables.md)

---

**worker.yaml - 声明式 Pipeline 编排** 🔧
