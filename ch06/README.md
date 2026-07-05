# Ch06: Subagent — 子代理与上下文隔离

## 快速开始

```bash
conda activate learnclaude
python -m ch06.code
```

测试 prompt：`请创建一个完整的 Python 项目，包含多个模块和测试文件，然后总结项目结构`

---

## 核心概念：子代理系统

ch06 在 ch05 的基础上，引入了 **Subagent（子代理）系统**，实现了任务分解和上下文隔离。

### 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Subagent 系统架构                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Parent Agent (主代理)                         │   │
│  │  SYSTEM: "For complex sub-problems, use the task tool..."        │   │
│  │  TOOLS: bash, read_file, write_file, edit_file, glob,            │   │
│  │          todo_write, task  ← NEW: 用于创建子代理                   │   │
│  │  messages: [完整对话历史]                                          │   │
│  └───────────────────────────┬──────────────────────────────────────┘   │
│                              │                                          │
│                              ▼ task("创建子任务")                        │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Subagent (子代理)                             │   │
│  │  SUB_SYSTEM: "Complete the task, then return a concise summary"  │   │
│  │  SUB_TOOLS: bash, read_file, write_file, edit_file, glob        │   │
│  │  (NO task tool — 防止递归创建子代理)                               │   │
│  │  messages: [{"role": "user", "content": "子任务描述"}] ← 新鲜上下文│   │
│  │                              │                                      │
│  │                              ▼ 执行子任务 (最多 30 轮)               │
│  │                              │                                      │
│  │                              ▼ 返回总结                              │
│  └───────────────────────────┬──────────────────────────────────────┘   │
│                              │                                          │
│                              ▼ 仅返回总结文本                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Parent Agent (继续)                           │   │
│  │  子代理的中间结果被丢弃，只保留最终总结                            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Ch06 新增特性

| 特性 | 说明 | 代码位置 |
|------|------|---------|
| `task` 工具 | 创建子代理处理复杂子任务 | [第 259-263 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L259-L263) |
| `spawn_subagent()` | 子代理创建和执行函数 | [第 156-196 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L156-L196) |
| `SUB_SYSTEM` | 子代理专属的 SYSTEM prompt | [第 47-51 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L47-L51) |
| `SUB_TOOLS` | 子代理可用的工具列表（无 task） | [第 269-280 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L269-L280) |
| `SUB_HANDLERS` | 子代理的工具分发映射 | [第 283-286 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L283-L286) |
| `extract_text()` | 从消息内容中提取文本的辅助函数 | [第 150-154 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L150-L154) |
| 安全限制 | 子代理最多执行 30 轮 | [第 160 行](file:///e:/Agents/learn_claude_code/ch06/code.py#L160) |

---

## 代码解析

### 1. SYSTEM Prompt 更新

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}."
    "Before starting any multi-step task, use todo_write to plan your steps."
    "Update status as you go.Use bash to solve tasks. Act, don't explain."
    "For complex sub-problems, use the task tool to spawn a subagent."  # ← 新增
    "当前环境是windows环境下。"
)
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L39-L45)

**关键变化**：新增 "For complex sub-problems, use the task tool to spawn a subagent"，引导主代理在遇到复杂子问题时创建子代理。

### 2. 子代理 SYSTEM Prompt

```python
SUB_SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Complete the task you were given, then return a concise summary. "
    "Do not delegate further."
)
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L47-L51)

**设计意图**：

| 指令 | 作用 |
|------|------|
| `"Complete the task you were given"` | 明确子代理的目标是完成分配的任务 |
| `"return a concise summary"` | 要求返回简洁的总结，而非详细的中间步骤 |
| `"Do not delegate further"` | 禁止子代理再创建子代理（虽然工具列表中已经没有 task） |

### 3. task 工具定义

```python
TOOLS.append({
    "name": "task",
    "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
    "input_schema": {"type": "object", "properties": {"description": {"type": "string"}}, "required": ["description"]},
})
TOOL_HANDLERS["task"] = spawn_subagent
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L259-L264)

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `description` | string | 子任务的描述，会作为子代理的初始用户输入 |

**动态添加**：使用 `TOOLS.append()` 在运行时动态添加工具，而非在定义时硬编码。

### 4. 子代理工具列表

```python
SUB_TOOLS = [
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    {"name": "read_file", "description": "Read file contents.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]}},
    {"name": "write_file", "description": "Write content to a file.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    {"name": "edit_file", "description": "Replace exact text in a file once.",
     "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}}, "required": ["path", "old_text", "new_text"]}},
    {"name": "glob", "description": "Find files matching a glob pattern.",
     "input_schema": {"type": "object", "properties": {"pattern": {"type": "string"}}, "required": ["pattern"]}},
]
# NO "task" tool — prevent recursive spawning
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L269-L281)

**关键区别**：子代理的工具列表中**没有 `task` 工具**，这是防止递归创建子代理的关键安全措施。

### 5. 子代理创建函数

```python
def spawn_subagent(description: str) -> str:
    """Spawn a subagent with fresh messages[], return summary only."""
    print(f"\n\033[35m[Subagent spawned]\033[0m")
    messages = [{"role": "user", "content": description}]  # fresh context
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=SUB_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                blocked = trigger_hooks("PreToolUse", block)
                if blocked:
                    results.append({"type": "tool_result", "tool_use_id": block.id,
                                    "content": str(blocked)})
                    continue
                handler = SUB_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown: {block.name}"
                trigger_hooks("PostToolUse", block, output)
                print(f"  \033[90m[sub] {block.name}: {str(output)[:100]}\033[0m")
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": output})
        messages.append({"role": "user", "content": results})
    result = extract_text(messages[-1]["content"])
    if not result:
        for msg in reversed(messages):
            if msg["role"] == "assistant":
                result = extract_text(msg["content"])
                if result:
                    break
        if not result:
            result = "Subagent stopped after 30 turns without final answer."
    print(f"\033[35m[Subagent done]\033[0m")
    return result
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L156-L196)

**核心流程**：

```
┌─────────────────────────────────────────────────────────────┐
│                    spawn_subagent() 流程                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 创建新鲜消息上下文                                        │
│     messages = [{"role": "user", "content": description}]   │
│                                                             │
│  2. 最多执行 30 轮循环                                       │
│     for _ in range(30):                                     │
│         ├─► 调用 LLM (使用 SUB_SYSTEM 和 SUB_TOOLS)          │
│         ├─► 如果 stop_reason != "tool_use" → 退出循环        │
│         └─► 执行工具调用，收集结果                            │
│                                                             │
│  3. 提取最终总结                                             │
│     ├─► 优先取最后一条消息的文本内容                          │
│     ├─► 如果为空，向前查找 assistant 的文本                  │
│     └─► 如果都为空，返回默认消息                              │
│                                                             │
│  4. 返回总结（丢弃所有中间消息）                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**安全机制**：

| 机制 | 说明 |
|------|------|
| **最多 30 轮** | 防止子代理无限循环 |
| **无 task 工具** | 防止递归创建子代理 |
| **权限钩子** | 子代理也会触发 `permission_hook`，危险操作仍需用户确认 |
| **新鲜上下文** | 子代理只看到自己的任务描述，不受主代理历史影响 |

### 6. 文本提取辅助函数

```python
def extract_text(content) -> str:
    """Extract text from message content blocks."""
    if not isinstance(content, list):
        return str(content)
    return "\n".join(getattr(b, "text", "") for b in content if getattr(b, "type", None) == "text")
```

[代码位置](file:///e:/Agents/learn_claude_code/ch06/code.py#L150-L154)

**功能**：从消息内容中提取纯文本，用于获取子代理的最终总结。

**处理逻辑**：
- 如果 content 不是列表，直接转为字符串
- 如果是列表，遍历所有 block，提取 `type == "text"` 的内容
- 使用 `getattr()` 兼容字典和 SDK 对象两种格式

### 7. 主代理循环

主代理的 `agent_loop()` 与 ch05 基本相同，但 `TOOLS` 列表中新增了 `task` 工具：

```python
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown: {block.name}"
```

当主代理调用 `task` 工具时，`spawn_subagent()` 被执行，返回子代理的总结作为工具结果。

---

## 完整工作流程示例

以"创建一个包含多个模块的 Python 项目"为例：

### 第一步：用户输入

```
请创建一个完整的 Python 项目，包含：
1. 主程序模块
2. 工具函数模块
3. 测试文件
4. README 文档
然后总结项目结构。
```

### 第二步：主代理规划

主代理使用 `todo_write` 创建任务列表，并决定使用 `task` 工具处理子任务：

```
[HOOK] todo_write([{'content': '创建项目目录结构', 'status': 'pending'}, {'content': '编写主程序模块', 'status': 'pending'}, {'content': '编写工具函数模块', 'status': 'pending'}, {'content': '编写测试文件', 'status': 'pending'}, {'content': '创建 README', 'status': 'pending'}, {'content': '总结项目结构', 'status': 'pending'}])

## Current Tasks
  [ ] 创建项目目录结构
  [ ] 编写主程序模块
  [ ] 编写工具函数模块
  [ ] 编写测试文件
  [ ] 创建 README
  [ ] 总结项目结构
```

### 第三步：主代理创建子代理

主代理调用 `task` 工具，将"创建项目"的子任务交给子代理：

```
[HOOK] task(['创建一个 Python 项目，包含主程序模块、工具函数模块和测试文件'])

[Subagent spawned]
  [sub] bash: mkdir myproject
  [sub] write_file: print("Hello from main")
  [sub] write_file: def add(a, b): return a + b
  [sub] write_file: import unittest...
[Subagent done]
```

### 第四步：子代理返回总结

子代理完成任务后，返回简洁的总结：

```
已创建项目结构：
- myproject/
  - main.py (主程序)
  - utils.py (工具函数)
  - tests/test_utils.py (测试文件)
```

### 第五步：主代理继续

主代理收到子代理的总结，更新任务状态，并继续完成剩余任务：

```
[HOOK] todo_write([{'content': '创建项目目录结构', 'status': 'completed'}, {'content': '编写主程序模块', 'status': 'completed'}, {'content': '编写工具函数模块', 'status': 'completed'}, {'content': '编写测试文件', 'status': 'completed'}, {'content': '创建 README', 'status': 'in_progress'}, {'content': '总结项目结构', 'status': 'pending'}])

[HOOK] write_file(['myproject/README.md', '# My Project\n...'])
[HOOK] todo_write([...])

项目已完成！结构如下：
- myproject/
  - main.py
  - utils.py
  - tests/test_utils.py
  - README.md
```

---

## 扩展思考

### 子代理设计原则

1. **上下文隔离**：子代理有独立的消息历史，避免主代理的长上下文干扰
2. **职责明确**：子代理专注于完成单一子任务，返回总结而非详细过程
3. **防止递归**：子代理没有 `task` 工具，避免无限嵌套
4. **安全限制**：最多 30 轮执行，防止无限循环
5. **权限继承**：子代理共享主代理的权限钩子，安全策略一致

### 设计模式分析

**组合模式（Composite Pattern）**：
- 主代理和子代理都是代理，具有相同的接口（工具调用能力）
- 主代理可以包含子代理
- 子代理完成后返回结果给主代理

**工作者模式（Worker Pattern）**：
- 主代理负责任务分解和协调
- 子代理负责执行具体子任务
- 结果汇总后返回给主代理

### 可能的改进方向

1. **子代理类型**：
   - 创建不同类型的子代理（如代码编写、测试、文档）
   - 每个类型有专属的 SYSTEM prompt 和工具集

2. **子代理池**：
   - 预创建多个子代理实例
   - 支持并发执行多个子任务
   - 负载均衡

3. **上下文传递**：
   - 选择性地传递部分主代理上下文给子代理
   - 支持子代理间的信息共享

4. **子代理管理**：
   - 子代理执行状态监控
   - 子代理超时处理
   - 子代理错误恢复

5. **结果增强**：
   - 返回结构化结果（JSON）而非纯文本
   - 支持子代理返回文件列表、代码片段等

---

## 实践练习

### 练习 1：测试基础功能

使用以下 prompt 测试子代理功能：

```
请创建一个 Python 脚本计算斐波那契数列，然后创建测试文件验证正确性
```

观察主代理是否会调用 `task` 工具创建子代理。

### 练习 2：复杂任务分解

使用以下 prompt 测试任务分解能力：

```
请完成以下任务：
1. 创建一个名为 calculator 的 Python 包
2. 实现加减乘除四个基本运算
3. 为每个运算编写单元测试
4. 创建一个命令行接口
5. 生成项目文档
```

观察主代理如何分解任务，是否会使用子代理处理部分子任务。

### 练习 3：子代理限制测试

使用以下 prompt 测试安全限制：

```
请编写一个无限循环的 Python 脚本并运行它
```

观察权限钩子是否会阻止危险命令，以及子代理是否有 30 轮限制。