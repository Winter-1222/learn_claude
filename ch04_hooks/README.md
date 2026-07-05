# Ch04: Hook 系统 — 解耦扩展逻辑

## 快速开始

```bash
conda activate learnclaude
python -m ch04_hooks.code
```

测试 prompt：`请列出当前目录下所有文件，然后读取 ch01/README.md 的内容`

---

## 核心概念：Hook 系统

ch04 引入了 **Hook（钩子）系统**，将扩展逻辑从主循环中分离出来，实现了更好的模块化和解耦。

### Hook 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hook 系统工作流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户输入 ──► UserPromptSubmit ──► trigger_hooks() ──► LLM          │
│                                                         │          │
│                                                         ▼          │
│                                              ┌─────────────────┐   │
│                                              │ stop_reason?    │   │
│                                              ├─────────────────┤   │
│                                              │ No ──► Stop ──► │──► exit
│                                              │ Yes            │   │
│                                              └───────┬─────────┘   │
│                                                      │             │
│                              PreToolUse ──► trigger_hooks()        │
│                              │     │                               │
│                              │     ├─► permission_hook (权限检查)  │
│                              │     └─► log_hook (日志记录)         │
│                              ▼                                     │
│                         TOOL_HANDLERS[x] (执行工具)                 │
│                              │                                     │
│                              ▼                                     │
│                              PostToolUse ──► trigger_hooks()       │
│                                              │                     │
│                                              └─► large_output_hook │
│                                                      │             │
│                                              results ──► messages  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Hook 事件列表

| 事件名称 | 触发时机 | 传入参数 | 用途 |
|---------|---------|---------|------|
| `UserPromptSubmit` | 用户输入后，发送给 LLM 之前 | `query` (str) | 注入上下文、日志记录 |
| `PreToolUse` | 工具执行前 | `block` (工具调用对象) | 权限检查、日志记录、预处理 |
| `PostToolUse` | 工具执行后 | `block`, `output` | 输出处理、大输出警告、后处理 |
| `Stop` | 循环即将退出时 | `messages` | 总结报告、清理工作 |

---

## 代码解析

### 1. Hook 注册表与核心函数

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:  # teaching shortcut: block this tool call
            return result
    return None
```

**设计原理：**

- **注册表模式**：`HOOKS` 字典将事件名称映射到回调函数列表
- **注册机制**：`register_hook()` 向指定事件添加回调函数
- **触发机制**：`trigger_hooks()` 按顺序执行所有回调，支持**短路返回**

**短路返回特性：**

```
trigger_hooks("PreToolUse", block)
        │
        ├─► permission_hook(block)
        │       │
        │       ├─► 返回 None ──► 继续执行下一个 hook
        │       │
        │       └─► 返回 "Permission denied" ──► 立即返回，阻止工具执行
        │
        └─► log_hook(block)
```

### 2. UserPromptSubmit Hook

```python
def context_inject_hook(query: str):
    print(f"\033[90m[HOOK] UserPromptSubmit: working in {WORKDIR}\033[0m")
    return None
```

**功能**：在用户输入发送给 LLM 之前记录日志，显示当前工作目录。

**触发时机**：用户输入后，`agent_loop()` 开始之前。

### 3. PreToolUse Hooks

#### permission_hook（权限检查）

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]
DESTRUCTIVE = ["rm ", "> /etc/", "chmod 777", "del", "Remove-Item"]

def permission_hook(block):
    """PreToolUse: s03 check_permission() logic moved here."""
    if block.name == "bash":
        # Gate 1: Hard deny list
        for pattern in DENY_LIST:
            if pattern in block.input.get("command", ""):
                print(f"\n\033[31m⛔ Blocked: '{pattern}'\033[0m")
                return "Permission denied by deny list"
        # Gate 2 & 3: Destructive commands
        for kw in DESTRUCTIVE:
            if kw in block.input.get("command", ""):
                print(f"\n\033[33m⚠  Potentially destructive command\033[0m")
                print(f"   Tool: {block.name}({block.input})")
                choice = input("   Allow? [y/N] ").strip().lower()
                if choice not in ("y", "yes"):
                    return "Permission denied by user"
    # 文件操作范围检查
    if block.name in ("write_file", "edit_file"):
        path = block.input.get("path", "")
        if not (WORKDIR / path).resolve().is_relative_to(WORKDIR):
            print(f"\n\033[33m⚠  Writing outside workspace\033[0m")
            print(f"   Tool: {block.name}({block.input})")
            choice = input("   Allow? [y/N] ").strip().lower()
            if choice not in ("y", "yes"):
                return "Permission denied by user"
    return None
```

**与 ch03 的对比：**

| 版本 | 权限检查位置 | 代码耦合度 | 扩展性 |
|------|-------------|-----------|--------|
| ch03 | 直接写在 `agent_loop()` 中 | 高耦合 | 差 |
| ch04 | 封装为 `permission_hook`，通过 `PreToolUse` 触发 | 低耦合 | 好 |

#### log_hook（日志记录）

```python
def log_hook(block):
    """PreToolUse: log every tool call."""
    args_preview = str(list(block.input.values())[:2])[:60]
    print(f"\033[90m[HOOK] {block.name}({args_preview})\033[0m")
    return None
```

**功能**：记录每次工具调用的名称和参数预览。

**输出示例**：
```
[HOOK] read_file(['ch01/README.md'])
[HOOK] bash(['ls -la'])
```

### 4. PostToolUse Hook

```python
def large_output_hook(block, output):
    """PostToolUse: warn on large output."""
    if len(str(output)) > 100000:
        print(f"\033[33m[HOOK] ⚠ Large output from {block.name}: {len(str(output))} chars\033[0m")
    return None
```

**功能**：检测工具输出是否过大（超过 100,000 字符），如果是则发出警告。

**用途**：
- 防止输出过多导致内存问题
- 提醒用户可能需要分页查看或处理大型输出

### 5. Stop Hook

```python
def summary_hook(messages: list):
    tool_count = 0
    tool_names = []
    for m in messages:
        content = m.get("content")
        if not isinstance(content, list):
            continue
        for b in content:
            if isinstance(b, dict) and b.get("type") == "tool_result":
                tool_count += 1
            elif isinstance(b, dict) and b.get("type") == "tool_use":
                tool_names.append(b.get("name", "unknown"))
    tools_str = ", ".join(set(tool_names)) if tool_names else "none"
    print(f"\033[90m[HOOK] Stop: session used {tool_count} tool calls ({tools_str})\033[0m")
    return None
```

**功能**：在会话结束时输出总结报告，包括：
- 工具调用次数
- 使用过的工具名称（去重）

**输出示例**：
```
[HOOK] Stop: session used 3 tool calls (read_file, write_file, bash)
```

### 6. Hook 注册

```python
register_hook("UserPromptSubmit", context_inject_hook)
register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
register_hook("PostToolUse", large_output_hook)
register_hook("Stop", summary_hook)
```

**注册顺序的重要性：**

对于 `PreToolUse` 事件，钩子按注册顺序执行：
1. `permission_hook` — 先检查权限，如果被阻止则不会执行后续钩子
2. `log_hook` — 记录日志

这体现了**短路返回**的设计优势：权限检查失败时，日志钩子不会被执行。

### 7. 代理循环集成

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            force = trigger_hooks("Stop", messages)
            if force:
                messages.append({"role": "user", "content": force})
                continue
            return

        results = []
        for block in response.content:
            if block.type != "tool_use":
                continue

            # s04 change: hook replaces hard-coded check_permission()
            blocked = trigger_hooks("PreToolUse", block)
            if blocked:
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": blocked})
                continue

            handler = TOOL_HANDLERS.get(block.name)
            output = handler(**block.input) if handler else f"Unknown: {block.name}"
            # s04: post hook
            trigger_hooks("PostToolUse", block, output)
            results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

**与 ch03 的核心差异：**

```python
# ch03: 硬编码权限检查
if not check_permission(block):
    results.append({"type": "tool_result", ...})
    continue

# ch04: 通过钩子触发
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({"type": "tool_result", ...})
    continue
```

---

## 扩展思考

### Hook 系统的设计优势

1. **解耦**：扩展逻辑与主循环完全分离
2. **可组合**：多个 hook 可以叠加在同一个事件上
3. **可插拔**：可以随时添加或移除 hook，无需修改主循环代码
4. **可拦截**：hook 可以通过返回非 None 值来阻止后续操作

### 设计模式分析

**观察者模式（Observer Pattern）**：
- `HOOKS` 注册表是主题（Subject）
- 各 hook 函数是观察者（Observer）
- `trigger_hooks()` 是通知机制

**责任链模式（Chain of Responsibility）**：
- 多个 hook 串联执行
- 每个 hook 可以决定是否继续传递
- 通过短路返回实现拦截

### 可能的改进方向

1. **Hook 优先级**：为 hook 添加优先级，控制执行顺序
2. **条件 Hook**：根据条件动态决定是否触发某个 hook
3. **异步 Hook**：支持异步操作（如网络请求、文件写入）
4. **Hook 管理工具**：提供启用/禁用 hook 的 API
5. **错误处理**：为 hook 执行添加异常捕获

---

## 实践练习

### 练习 1：添加新 Hook

尝试添加一个 `PostToolUse` hook，记录工具执行耗时：

```python
import time

def timing_hook(block, output):
    """PostToolUse: record execution time."""
    # 提示：需要在 PreToolUse 时记录开始时间
    print(f"\033[90m[HOOK] {block.name} executed in X ms\033[0m")
    return None
```

### 练习 2：修改现有 Hook

修改 `context_inject_hook`，在用户输入中自动添加工作目录信息：

```python
def context_inject_hook(query: str):
    # 返回非 None 值会将结果添加到消息中
    return f"Working directory: {WORKDIR}\nUser query: {query}"
```

### 练习 3：测试完整流程

使用以下 prompt 测试所有 hook 是否正常工作：

1. `列出当前目录下所有文件` — 应触发 `log_hook` 和 `summary_hook`
2. `删除当前目录下所有 .txt 文件` — 应触发 `permission_hook`（需要用户确认）
3. `创建一个包含 100000 个字符的文件` — 应触发 `large_output_hook`
4. `读取 ch03/README.md` — 应触发 `log_hook` 和 `context_inject_hook`
Create a file called test.txt
