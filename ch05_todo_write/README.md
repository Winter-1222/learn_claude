# Ch05: TodoWrite — 规划工具与任务追踪

## 快速开始

```bash
conda activate learnclaude
python -m ch05_todo_write.code
```

测试 prompt：`请在ch05目录下创建一个新的文件夹，其中包含 Hello World 脚本和 README 文件`

---

## 核心概念：规划驱动的工作流

ch05 在 ch04 的 Hook 系统基础上，引入了 **TodoWrite 规划工具** 和 **Nag 提醒机制**，实现了"先规划后执行"的工作流程。

### 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Ch05 系统架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────┐     ┌──────────────────────┐     ┌──────────────────┐   │
│  │  User    │ ──► │  SYSTEM Prompt       │ ──► │  LLM             │   │
│  │  Prompt  │     │  "plan before execute"│     │                  │   │
│  └──────────┘     └──────────────────────┘     └────────┬─────────┘   │
│                                                         │             │
│                                                         ▼             │
│                                              ┌──────────────────────┐ │
│                                              │  todo_write 工具     │ │
│                                              │  (创建任务列表)       │ │
│                                              └──────────┬───────────┘ │
│                                                         │             │
│                                                         ▼             │
│                                              ┌──────────────────────┐ │
│                                              │  CURRENT_TODOS       │ │
│                                              │  (内存中的任务状态)    │ │
│                                              └──────────┬───────────┘ │
│                                                         │             │
│                                                         ▼             │
│                                              ┌──────────────────────┐ │
│                                              │  rounds_since_todo  │ │
│                                              │  (计数器: 3轮未更新)  │ │
│                                              └──────────┬───────────┘ │
│                                                         │             │
│                                              ┌──────────▼───────────┐ │
│                                              │  <reminder> 注入     │ │
│                                              │  "Update your todos"│ │
│                                              └──────────┬───────────┘ │
│                                                         │             │
│                                                         ▼             │
│                                              ┌──────────────────────┐ │
│                                              │  TOOL_HANDLERS       │ │
│                                              │  bash/read/write/... │ │
│                                              └──────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Ch05 新增特性

| 特性 | 说明 | 代码位置 |
|------|------|---------|
| `todo_write` 工具 | 创建和管理任务列表 | [第 185-193 行](file:///e:/Agents/learn_claude_code/ch05/code.py#L185-L193) |
| `run_todo_write()` | 任务列表执行函数 | [第 130-141 行](file:///e:/Agents/learn_claude_code/ch05/code.py#L130-L141) |
| `_normalize_todos()` | 任务数据验证和规范化 | [第 110-128 行](file:///e:/Agents/learn_claude_code/ch05/code.py#L110-L128) |
| `rounds_since_todo` | 任务更新计数器 | [第 301 行](file:///e:/Agents/learn_claude_code/ch05/code.py#L301) |
| Nag 提醒 | 3 轮未更新任务时注入提醒 | [第 307-310 行](file:///e:/Agents/learn_claude_code/ch05/code.py#L307-L310) |

---

## 代码解析

### 1. SYSTEM Prompt 规划引导

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}."
    "Before starting any multi-step task, use todo_write to plan your steps."
    "Update status as you go.Use bash to solve tasks. Act, don't explain."
    "当前环境是windows环境下。"
)
```

**关键变化**：新增 "Before starting any multi-step task, use todo_write to plan your steps" 引导，告诉模型在执行多步骤任务前先使用 `todo_write` 工具进行规划。

### 2. todo_write 工具定义

```python
{
    "name": "todo_write", 
    "description": "Create and manage a task list for your current coding session.",
    "input_schema": {
        "type": "object", 
        "properties": {"todos": {"type": "array", "items": {"type": "object", "properties": {"content": {"type": "string"}, "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]}}, 
        "required": ["content", "status"]}}}, 
        "required": ["todos"]}},
}
```

**输入参数结构**：

```json
{
    "todos": [
        {"content": "任务描述", "status": "pending"},
        {"content": "任务描述", "status": "in_progress"},
        {"content": "任务描述", "status": "completed"}
    ]
}
```

**状态枚举**：

| 状态 | 含义 | 图标 |
|------|------|------|
| `pending` | 待处理 | 空格 |
| `in_progress` | 进行中 | ▸ (青色) |
| `completed` | 已完成 | ✓ (绿色) |

### 3. 任务数据验证

```python
def _normalize_todos(todos):
    if isinstance(todos, str):
        try:
            todos = json.loads(todos)
        except json.JSONDecodeError:
            try:
                todos = ast.literal_eval(todos)
            except (SyntaxError, ValueError):
                return None, "Error: todos must be a list or JSON array string"
    if not isinstance(todos, list):
        return None, "Error: todos must be a list"
    for i, t in enumerate(todos):
        if not isinstance(t, dict):
            return None, f"Error: todos[{i}] must be an object"
        if "content" not in t or "status" not in t:
            return None, f"Error: todos[{i}] missing 'content' or 'status'"
        if t["status"] not in ("pending", "in_progress", "completed"):
            return None, f"Error: todos[{i}] has invalid status '{t['status']}'"
    return todos, None
```

**验证流程**：

```
输入 todos
    │
    ├─► 是字符串？
    │       ├─► 尝试 json.loads()
    │       └─► 失败 → 尝试 ast.literal_eval()
    │
    ├─► 是列表？
    │       └─► 否 → 返回错误
    │
    ├─► 遍历每个任务
    │       ├─► 是字典？
    │       ├─► 包含 content 和 status？
    │       └─► status 是有效枚举值？
    │
    └─► 返回验证后的 todos 和错误信息
```

**支持的输入格式**：

| 输入类型 | 示例 |
|---------|------|
| JSON 字符串 | `'[{"content": "task1", "status": "pending"}]'` |
| Python 列表字符串 | `"[{'content': 'task1', 'status': 'pending'}]"` |
| Python 列表对象 | `[{"content": "task1", "status": "pending"}]` |

### 4. 任务列表执行函数

```python
CURRENT_TODOS = []

def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    todos, error = _normalize_todos(todos)
    if error:
        return error
    CURRENT_TODOS = todos
    lines = ["\n\033[33m## Current Tasks\033[0m"]
    for t in CURRENT_TODOS:
        icon = {"pending": " ", "in_progress": "\033[36m▸\033[0m", "completed": "\033[32m✓\033[0m"}[t["status"]]
        lines.append(f"  [{icon}] {t['content']}")
    print("\n".join(lines))
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

**功能特点**：

1. **全局状态管理**：使用 `CURRENT_TODOS` 全局变量存储当前任务列表
2. **数据验证**：调用 `_normalize_todos()` 确保数据格式正确
3. **彩色输出**：根据状态显示不同颜色的图标
4. **状态反馈**：返回更新的任务数量

**输出示例**：

```
## Current Tasks
  [ ] 创建项目目录
  [▸] 编写 Hello World 脚本
  [✓] 创建 README 文件
```

### 5. Nag 提醒机制

```python
rounds_since_todo = 0

def agent_loop(messages: list):
    global rounds_since_todo
    while True:
        # s05: nag reminder — inject if model hasn't updated todos for 3 rounds
        if rounds_since_todo >= 3 and messages:
            messages.append({"role": "user",
                             "content": "<reminder>Update your todos.</reminder>"})
            rounds_since_todo = 0
        # ... LLM 调用 ...
        
        rounds_since_todo += 1
        
        # ... 工具执行 ...
        if block.name == "todo_write":
            rounds_since_todo = 0
```

**工作原理**：

```
┌─────────────────────────────────────────────────────────────┐
│                    Nag 提醒机制流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  每轮循环开始                                                │
│       │                                                     │
│       ▼                                                     │
│  rounds_since_todo >= 3 ?                                   │
│       │                                                     │
│       ├─► 是 ──► 注入 <reminder> ──► 重置计数器 ──► LLM     │
│       │                                                     │
│       └─► 否 ──► 继续                                      │
│                                                             │
│  LLM 返回工具调用                                           │
│       │                                                     │
│       ▼                                                     │
│  rounds_since_todo += 1                                     │
│       │                                                     │
│       ▼                                                     │
│  执行工具                                                   │
│       │                                                     │
│       ├─► 是 todo_write ──► rounds_since_todo = 0           │
│       │                                                     │
│       └─► 其他工具 ──► 计数器保持不变                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**设计意图**：

- **防止遗忘**：如果模型连续 3 轮没有更新任务列表，自动提醒
- **保持节奏**：确保任务状态与实际进度同步
- **轻量级**：只注入简短的提醒，不干扰正常对话

### 6. 工具分发映射更新

```python
TOOL_HANDLERS = {
    "bash": run_bash, "read_file": run_read, "write_file": run_write,
    "edit_file": run_edit, "glob": run_glob, "todo_write": run_todo_write,
}
```

**新增**：`"todo_write": run_todo_write`，将新工具添加到分发映射中。

---

## 完整工作流程示例

以"创建一个 Python 项目"为例：

### 第一步：用户输入

```
请创建一个 Python 项目，包含 Hello World 脚本和 README 文件
```

### 第二步：模型规划

LLM 收到 SYSTEM prompt 中的规划引导，调用 `todo_write` 创建任务列表：

```
[HOOK] todo_write([{'content': '创建项目目录', 'status': 'pending'}, {'content': '编写 Hello World 脚本', 'status': 'pending'}, {'content': '创建 README 文件', 'status': 'pending'}])

## Current Tasks
  [ ] 创建项目目录
  [ ] 编写 Hello World 脚本
  [ ] 创建 README 文件
```

### 第三步：执行任务

模型依次执行每个任务，同时更新任务状态：

```
[HOOK] bash(['mkdir myproject'])
[HOOK] todo_write([{'content': '创建项目目录', 'status': 'completed'}, {'content': '编写 Hello World 脚本', 'status': 'in_progress'}, {'content': '创建 README 文件', 'status': 'pending'}])

## Current Tasks
  [✓] 创建项目目录
  [▸] 编写 Hello World 脚本
  [ ] 创建 README 文件

[HOOK] write_file(['myproject/hello.py', 'print("Hello World")'])
[HOOK] todo_write([{'content': '创建项目目录', 'status': 'completed'}, {'content': '编写 Hello World 脚本', 'status': 'completed'}, {'content': '创建 README 文件', 'status': 'in_progress'}])

## Current Tasks
  [✓] 创建项目目录
  [✓] 编写 Hello World 脚本
  [▸] 创建 README 文件

[HOOK] write_file(['myproject/README.md', '# My Project\n\nA simple Hello World project.'])
[HOOK] todo_write([{'content': '创建项目目录', 'status': 'completed'}, {'content': '编写 Hello World 脚本', 'status': 'completed'}, {'content': '创建 README 文件', 'status': 'completed'}])

## Current Tasks
  [✓] 创建项目目录
  [✓] 编写 Hello World 脚本
  [✓] 创建 README 文件
```

### 第四步：Nag 提醒触发场景

如果模型在执行过程中忘记更新任务列表：

```
rounds_since_todo = 1 → 执行 bash
rounds_since_todo = 2 → 执行 write_file  
rounds_since_todo = 3 → 注入 <reminder>Update your todos.</reminder>
```

---

## 扩展思考

### 规划工具设计原则

1. **独立于执行**：`todo_write` 只负责规划，不执行任何实际操作
2. **状态同步**：通过更新任务状态反映实际进度
3. **可视化**：使用图标和颜色直观展示任务状态
4. **轻量级**：不增加太多认知负担

### 可能的改进方向

1. **任务持久化**：
   - 将任务列表保存到文件
   - 支持跨会话恢复任务
   - 实现任务历史记录

2. **任务管理增强**：
   - 添加任务优先级
   - 支持子任务
   - 任务依赖关系管理
   - 任务截止时间提醒

3. **智能规划**：
   - 自动拆分复杂任务
   - 推荐任务执行顺序
   - 任务进度预测

4. **交互优化**：
   - 键盘快捷键操作任务
   - 任务过滤和搜索
   - 任务统计和报告

---

## 实践练习

### 练习 1：测试基础功能

使用以下 prompt 测试 `todo_write` 工具：

```
请创建一个简单的 Python 脚本，实现计算斐波那契数列的功能
```

观察模型是否：
1. 先调用 `todo_write` 创建任务列表
2. 执行任务时更新状态
3. 完成后所有任务标记为 `completed`

### 练习 2：触发 Nag 提醒

使用以下 prompt 测试提醒机制：

```
请列出当前目录下的所有文件，然后读取 ch01/README.md，再读取 ch02/README.md
```

观察模型连续执行 3 个工具后是否收到 `<reminder>` 提醒。

### 练习 3：复杂任务规划

使用以下 prompt 测试多步骤任务：

```
请在ch05目录下创建一个新的文件夹，其中创建一个完整的 Flask Web 应用，包含：1. 主页显示欢迎信息 2. /api/hello 接口返回 JSON 3. 使用 Jinja2 模板 4. 添加 CSS 样式 5. 创建 README 文档
```

观察模型是否能够合理规划和执行所有步骤。