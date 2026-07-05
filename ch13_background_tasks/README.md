# Ch13: Background Tasks — 线程异步执行与通知注入

## 概述

Ch13 引入了**后台任务系统**，通过线程实现工具的异步执行，使 Agent 能够在等待长时间操作（如安装、编译、测试）的同时继续响应用户。核心机制包括：线程调度、线程安全存储、通知注入。

### 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      后台任务系统架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │   主线程 (UI)     │      │   工作线程 (BG)   │                │
│  └────────┬─────────┘      └────────┬─────────┘                │
│           │                         │                          │
│           ▼                         ▼                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              共享状态 (带 background_lock)                 │  │
│  │                                                          │  │
│  │  background_tasks: {bg_id → {tool_use_id, command, status}}│  │
│  │  background_results: {bg_id → output}                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                         │                          │
│           │ start_background_task   │ worker()                 │
│           │ ○ 创建任务记录           │ ● 执行工具               │
│           │ ○ 启动守护线程           │ ● 写入结果               │
│           │ ○ 返回 bg_id            │                          │
│           │                         │                          │
│           ▼                         ▼                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              collect_background_results()                │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ 1. 读取 completed 状态的任务                      │  │  │
│  │  │ 2. 弹出结果，生成 <task_notification>             │  │  │
│  │  │ 3. 注入到下一轮 user message                      │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 后台任务决策机制

### should_run_background — 两级决策

```python
def should_run_background(tool_name: str, tool_input: dict) -> bool:
    if tool_input.get("run_in_background"):
        return True
    return is_slow_operation(tool_name, tool_input)
```

**决策优先级**：

| 优先级 | 条件 | 来源 |
|--------|------|------|
| 1 | `run_in_background=True` | LLM 显式请求 |
| 2 | 慢操作启发式匹配 | `is_slow_operation()` |

### is_slow_operation — 慢操作启发式

```python
def is_slow_operation(tool_name: str, tool_input: dict) -> bool:
    if tool_name != "bash":
        return False
    cmd = tool_input.get("command", "").lower()
    slow_keywords = ["install", "build", "test", "deploy", "compile",
                     "docker build", "pip install", "npm install",
                     "cargo build", "pytest", "make"]
    return any(kw in cmd for kw in slow_keywords)
```

**慢操作关键词**：

| 类别 | 关键词 |
|------|--------|
| 包管理 | `pip install`, `npm install` |
| 构建 | `build`, `compile`, `make`, `cargo build`, `docker build` |
| 测试 | `test`, `pytest` |
| 部署 | `deploy` |

### run_in_background 参数说明

```python
# bash 工具的 input_schema 添加了 run_in_background 参数
{"name": "bash", "description": "Run a shell command.",
 "input_schema": {"type": "object",
                  "properties": {
                      "command": {"type": "string"},
                      "run_in_background": {"type": "boolean"}},
                  "required": ["command"]}}
```

**重要设计细节**：`run_in_background` 参数存在于工具 schema 中，但实际由 `agent_loop` 的 `should_run_background` 判断分发，`run_bash` 函数本身不处理此参数：

```python
def run_bash(command: str, run_in_background: bool = False) -> str:
    # run_in_background is handled by agent_loop dispatch, not here
    ...
```

**原因**：后台执行的调度逻辑应该在 agent_loop 层面统一处理，而非分散在各个工具 handler 中。

## 线程安全设计

### 共享状态与锁

```python
background_tasks: dict[str, dict] = {}   # bg_id → {tool_use_id, command, status}
background_results: dict[str, str] = {}   # bg_id → output
background_lock = threading.Lock()
```

### 锁的使用场景

| 操作 | 位置 | 锁保护 |
|------|------|--------|
| 创建后台任务记录 | `start_background_task()` | `background_tasks` 写入 |
| 写入执行结果 | `worker()` 线程 | `background_tasks` 状态更新 + `background_results` 写入 |
| 读取完成任务 | `collect_background_results()` | `background_tasks` 读取 |
| 弹出结果 | `collect_background_results()` | `background_tasks` + `background_results` 删除 |

### worker 线程实现

```python
def start_background_task(block) -> str:
    global _bg_counter
    _bg_counter += 1
    bg_id = f"bg_{_bg_counter:04d}"
    cmd = block.input.get("command", block.name)

    def worker():
        result = execute_tool(block)
        with background_lock:
            background_tasks[bg_id]["status"] = "completed"
            background_results[bg_id] = result

    with background_lock:
        background_tasks[bg_id] = {
            "tool_use_id": block.id,
            "command": cmd,
            "status": "running",
        }
    thread = threading.Thread(target=worker, daemon=True)
    thread.start()
    return bg_id
```

**关键设计**：
- 使用 `daemon=True` 创建守护线程，主线程退出时自动终止
- `worker()` 函数捕获闭包中的 `block` 和 `bg_id`
- 所有对共享字典的操作都在 `with background_lock:` 保护下

## 通知注入机制

### 完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                 agent_loop 后台任务流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LLM 返回 tool_use                                           │
│           │                                                     │
│           ▼                                                     │
│  2. should_run_background(tool_name, input)?                    │
│           │                                                     │
│     ┌─────┴─────┐                                               │
│     │           │                                               │
│    Yes         No                                               │
│     │           │                                               │
│     ▼           ▼                                               │
│  start_       execute_tool()                                    │
│  background_  立即返回结果                                      │
│  task()                                                        │
│     │                                                           │
│     ▼                                                           │
│  返回占位符                                                      │
│  "[Background task bg_0001 started]"                            │
│           │                                                     │
│           ▼                                                     │
│  3. collect_background_results()                                │
│     ┌─────────────────────────────────────┐                    │
│     │ 读取 background_tasks 中 status=    │                    │
│     │ "completed" 的任务                  │                    │
│     │ 弹出结果，生成 <task_notification>  │                    │
│     └─────────────────────────────────────┘                    │
│           │                                                     │
│           ▼                                                     │
│  4. 注入到 user_content                                         │
│     user_content = [tool_results] + [bg_notifications]          │
│     messages.append({"role": "user", "content": user_content})  │
│           │                                                     │
│           ▼                                                     │
│  5. 下一轮 LLM 调用，模型看到通知                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### <task_notification> 格式

```python
notifications.append(
    f"<task_notification>\n"
    f"  <task_id>{bg_id}</task_id>\n"
    f"  <status>completed</status>\n"
    f"  <command>{task['command']}</command>\n"
    f"  <summary>{summary}</summary>\n"
    f"</task_notification>")
```

**字段说明**：

| 字段 | 说明 |
|------|------|
| `task_id` | 后台任务 ID（如 `bg_0001`） |
| `status` | 任务状态（当前只有 `completed`） |
| `command` | 执行的命令 |
| `summary` | 输出摘要（截断到 200 字符） |

### 与普通 tool_result 的区别

| 维度 | 普通 tool_result | 后台任务 |
|------|-----------------|---------|
| 执行方式 | 同步阻塞 | 异步非阻塞 |
| 结果返回时机 | 立即返回 | 完成后通过通知注入 |
| tool_use_id | 重用原始 block.id | **不重用**，通过 task_notification 传递 |
| 输出长度 | 完整输出（最多 50000 字符） | 摘要截断到 200 字符 |
| 上下文影响 | 立即消耗 token | 延迟消耗，节省等待时间 |

### 输出截断策略

```python
summary = output[:200] if len(output) > 200 else output
```

**设计原因**：
- 后台任务通常产生大量输出（如编译日志）
- 完整输出会占用过多上下文 token
- 摘要足以让模型判断任务是否成功
- 完整输出在 `background_results` 中被 pop 后丢弃

## agent_loop 中的集成

### 工具执行分发

```python
for block in response.content:
    if block.type != "tool_use":
        continue

    if should_run_background(block.name, block.input):
        bg_id = start_background_task(block)
        results.append({"type": "tool_result",
                        "tool_use_id": block.id,
                        "content": f"[Background task {bg_id} started]..."})
    else:
        output = execute_tool(block)
        results.append({"type": "tool_result",
                        "tool_use_id": block.id,
                        "content": output})
```

### 通知注入

```python
user_content = list(results)
bg_notifications = collect_background_results()
if bg_notifications:
    for notif in bg_notifications:
        user_content.append({"type": "text", "text": notif})
messages.append({"role": "user", "content": user_content})
```

**关键设计**：工具结果和后台通知在同一条 user message 中一起注入，确保模型在下一轮调用时能同时看到所有信息。

## 与前序章节的关系

| 章节 | 核心功能 | 与 ch13 的关系 |
|------|---------|---------------|
| ch12 | 文件持久化任务系统 | ch13 继承了任务系统，后台任务是任务执行的一种方式 |
| ch10 | 动态 System Prompt | ch13 继承了 prompt 组装和缓存机制 |
| ch09 | 记忆系统 | ch13 继承了记忆加载机制 |

### 简化说明

代码注释明确说明：

```python
"""
Note: Teaching code keeps a basic agent loop to stay focused on the task
system. S11's full error recovery (RecoveryState, backoff, escalation,
reactive compact, fallback model) is omitted.
"""
```

**设计哲学**：后台任务系统和错误恢复是独立的层，可以按需组合。

## 实践练习

### 练习1：体验后台任务

1. 运行 `python -m ch13_background_tasks.code`
2. 输入："安装 requests 包并查看安装结果"
3. 观察控制台输出的 `[background] dispatched` 日志
4. 输入任意内容触发下一轮，观察 `[background done]` 和 `[inject]` 日志

### 练习2：测试显式后台请求

1. 运行程序
2. 输入："执行命令 'sleep 5'，并在后台运行"
3. 观察是否显式设置 `run_in_background=True`
4. 快速输入下一个问题，验证后台任务执行期间可以继续交互

### 练习3：理解线程安全

在 `start_background_task` 和 `worker` 中添加调试打印：

```python
def start_background_task(block) -> str:
    ...
    print(f"  [debug] Created bg task {bg_id}, lock acquired: {background_lock.locked()}")
    ...

def worker():
    ...
    print(f"  [debug] Worker {bg_id} completed, writing result")
    ...
```

观察锁的获取和释放时机。

### 练习4：扩展慢操作关键词

尝试添加新的慢操作关键词：

```python
slow_keywords = [
    # ... 现有关键词 ...
    "git clone", "git pull", "rsync", "wget", "curl",
    "terraform apply", "kubectl apply", "helm install"
]
```

测试这些命令是否被正确识别为慢操作并在后台执行。
