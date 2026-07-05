# Ch10: System Prompt — 运行时动态组装与缓存

## 概述

Ch10 引入了**动态 System Prompt 组装机制**，从之前章节的硬编码 `SYSTEM` 字符串转向运行时根据实际状态（context）动态构建，并加入确定性缓存避免重复计算。

### 核心架构变化

```
┌─────────────────────────────────────────────────────────────────┐
│                      System Prompt 演进                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ch01-ch08:                                                     │
│    SYSTEM = "You are a coding agent..."  ← 硬编码字符串         │
│                                                                 │
│  ch09:                                                          │
│    SYSTEM = build_system()  ← 函数构建，但只在循环开始时调用一次  │
│                                                                 │
│  ch10:                                                          │
│    PROMPT_SECTIONS = {"identity": "...", "tools": "...", ...}   │
│    context = update_context(...)  ← 从真实状态推导               │
│    system = get_system_prompt(context)  ← 动态组装 + 缓存        │
│    [每轮工具调用后重新评估 context]                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### agent_loop 中的动态更新流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   agent_loop 动态 Prompt 流程                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户输入                                                        │
│     ↓                                                           │
│  agent_loop(messages, context)                                   │
│     ↓                                                           │
│  get_system_prompt(context)                                      │
│     ├── cache hit → 返回缓存的 prompt                            │
│     └── cache miss → assemble_system_prompt(context)            │
│            ↓                                                    │
│     LLM Call                                                    │
│            ↓                                                    │
│     tool_use?                                                   │
│     ├── No → 返回                                                │
│     └── Yes → 执行工具                                           │
│            ↓                                                    │
│     update_context(context, messages)  ← 重新评估状态            │
│            ↓                                                    │
│     get_system_prompt(context)  ← 可能生成新的 prompt            │
│            ↓                                                    │
│     循环继续                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## PROMPT_SECTIONS 分区设计

### 分区结构

`PROMPT_SECTIONS` 将 System Prompt 拆分为多个独立的片段：

```python
PROMPT_SECTIONS = {
    "identity": "You are a coding agent. Act, don't explain.",
    "tools": "Available tools: bash, read_file, write_file.",
    "workspace": f"Working directory: {WORKDIR}",
    "memory": "Relevant memories are injected below when available.",
}
```

### 设计优势

| 优势 | 说明 |
|------|------|
| **模块化** | 每个片段独立维护，便于修改和扩展 |
| **按需组装** | 根据当前 context 选择性加载片段 |
| **可读性** | 清晰的语义分区，易于理解和维护 |
| **可测试性** | 可以单独测试每个片段的效果 |

### 两类片段

1. **始终加载**：`identity`、`tools`、`workspace` —— 基础配置，每轮都需要
2. **条件加载**：`memory` —— 仅当 `.memory/MEMORY.md` 存在且有内容时加载

## assemble_system_prompt 组装逻辑

```python
def assemble_system_prompt(context: dict) -> str:
    sections = []
    
    # 始终加载的基础片段
    sections.append(PROMPT_SECTIONS["identity"])
    sections.append(PROMPT_SECTIONS["tools"])
    sections.append(PROMPT_SECTIONS["workspace"])
    
    # 条件加载：memory
    memories = context.get("memories", "")
    if memories:
        sections.append(f"Relevant memories:\n{memories}")
    
    return "\n\n".join(sections)
```

**组装规则**：
- 基础片段按固定顺序加入
- 条件片段根据 `context` 中的实际状态决定是否加入
- 最终用两个换行符连接所有片段

## get_system_prompt 缓存机制

### 核心问题：为什么不用 Python 的 `hash()`？

```python
def get_system_prompt(context: dict) -> str:
    global _last_context_key, _last_prompt
    key = json.dumps(context, sort_keys=True, ensure_ascii=False, default=str)
    if key == _last_context_key and _last_prompt:
        print("  \033[90m[cache hit] system prompt unchanged\033[0m")
        return _last_prompt
    _last_context_key = key
    _last_prompt = assemble_system_prompt(context)
    print(f"  \033[32m[assembled] sections: {', '.join(loaded)}\033[0m")
    return _last_prompt
```

**选择 `json.dumps` 的原因**：

| 方法 | 问题 |
|------|------|
| `hash()` | Python 进程启动时随机化，相同对象在不同进程中哈希值不同 |
| `hash()` | 对嵌套字典和列表不可靠 |
| `repr()` | 字典键顺序不确定（Python 3.7+ 保证插入顺序，但序列化结果不稳定） |
| `json.dumps(sort_keys=True)` | **确定性**：相同内容产生相同字符串 |

### 缓存机制工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                   get_system_prompt 缓存流程                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  输入 context                                                │
│     ↓                                                        │
│  json.dumps(context, sort_keys=True) → key                   │
│     ↓                                                        │
│  key == _last_context_key?                                   │
│     ├── Yes → [cache hit] 返回 _last_prompt                  │
│     └── No → assemble_system_prompt(context)                 │
│            ↓                                                 │
│       更新 _last_context_key 和 _last_prompt                  │
│            ↓                                                 │
│       [assembled] 打印加载的 sections                         │
│            ↓                                                 │
│       返回新的 prompt                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 缓存级别说明

代码注释明确指出了缓存级别的限制：

```python
"""
This cache only avoids redundant string assembly within a process.
Real Claude Code additionally protects API-level prompt cache via
stable section ordering and SYSTEM_PROMPT_DYNAMIC_BOUNDARY.
"""
```

| 缓存级别 | 当前实现 | 真实 Claude Code |
|----------|----------|------------------|
| **进程内** | ✅ `_last_context_key` + `_last_prompt` | ✅ |
| **API 级别** | ❌ 未实现 | ✅ `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` |

**API 级别缓存**：当 System Prompt 的某些部分（如 memory）频繁变化时，通过标记动态边界，API 可以智能缓存不变的部分，减少重复计算。

## update_context 状态推导

`update_context` 从真实系统状态推导 context：

```python
def update_context(context: dict, messages: list) -> dict:
    memories = ""
    if MEMORY_INDEX.exists():
        content = MEMORY_INDEX.read_text().strip()
        if content:
            memories = content
    return {
        "enabled_tools": list(TOOL_HANDLERS.keys()),
        "workspace": str(WORKDIR),
        "memories": memories,
    }
```

**状态来源**：

| Context 字段 | 来源 | 更新时机 |
|--------------|------|----------|
| `enabled_tools` | `TOOL_HANDLERS.keys()` | 每次调用 |
| `workspace` | `WORKDIR` | 每次调用 |
| `memories` | `.memory/MEMORY.md` 文件内容 | 文件存在且非空时 |

**设计要点**：`memories` 是从真实文件状态读取的，而非基于关键词匹配。这意味着：
- 只有当 `.memory/MEMORY.md` 存在且有内容时，memory 片段才会被加载
- 不需要额外的 LLM 调用来选择相关记忆（与 ch09 的 `select_relevant_memories` 不同）

## 与 ch09 的关键区别

| 维度 | ch09 | ch10 |
|------|------|------|
| **System Prompt 构建时机** | 循环开始时构建一次 | 每轮工具调用后重建 |
| **Memory 加载方式** | LLM 语义选择 + 关键词回退 | 直接读取文件内容（真实状态） |
| **缓存机制** | 无 | `json.dumps` 确定性缓存 |
| **Context 驱动** | 基于对话内容 | 基于系统真实状态 |

### ch09 vs ch10 的 System Prompt 生命周期

```
ch09:
  ┌─────────────────────────────────────────────────────┐
  │ system = build_system()  ← 只在 agent_loop 开始时   │
  │                                                     │
  │  while True:                                        │
  │    LLM Call (使用同一个 system)                      │
  │    tool_use?                                        │
  │    └── Yes → 执行工具，system 不变                   │
  │                                                     │
  └─────────────────────────────────────────────────────┘

ch10:
  ┌─────────────────────────────────────────────────────┐
  │ system = get_system_prompt(context)                 │
  │                                                     │
  │  while True:                                        │
  │    LLM Call (使用当前 system)                        │
  │    tool_use?                                        │
  │    └── Yes → 执行工具                               │
  │              ↓                                      │
  │         context = update_context(...)               │
  │         system = get_system_prompt(context)  ← 更新  │
  │                                                     │
  └─────────────────────────────────────────────────────┘
```

## 关键设计决策

### 为什么每轮工具调用后都重新评估 context？

- **动态适应性**：工具执行可能改变系统状态（如创建新的记忆文件）
- **实时性**：确保 Prompt 始终反映最新的系统状态
- **灵活性**：可以根据执行结果动态调整 Prompt 内容

### 为什么 memories 直接读取文件内容而非 LLM 选择？

- **简单性**：减少一次 LLM 调用，降低成本
- **可靠性**：避免 LLM 选择错误的记忆
- **一致性**：基于真实文件状态，而非语义推断

### 为什么用 `json.dumps` 而非 `hash()`？

- **确定性**：`json.dumps(sort_keys=True)` 保证相同内容产生相同字符串
- **可调试性**：生成的 key 是可读的 JSON 字符串，便于调试
- **兼容性**：对嵌套字典和列表都能正确处理

## 实践练习

### 练习1：观察缓存命中与未命中

1. 运行 `python -m ch10_system_prompt.code`
2. 输入："列出当前目录内容"
3. 观察控制台输出的 `[assembled]` 和 `[cache hit]` 日志
4. 继续输入："创建一个 test.txt 文件"
5. 观察工具调用后 context 是否重新评估

### 练习2：测试 memory 条件加载

1. 在 `.memory/` 目录下创建 `MEMORY.md`：
```
- [user-preference]: 用户偏好使用 Python 编写代码
```

2. 运行程序，输入："帮我写一段代码"
3. 观察 `[assembled]` 日志是否包含 `memory`
4. 删除 `MEMORY.md`，重新运行
5. 观察 `[assembled]` 日志是否不再包含 `memory`

### 练习3：理解动态更新机制

在 `agent_loop` 的 `update_context` 调用后添加调试打印：

```python
print(f"Context changed: {context}")
```

观察每轮工具调用后 context 是否变化，以及何时触发新的 prompt 组装。

### 练习4：扩展 PROMPT_SECTIONS

尝试添加一个新的 prompt section：

```python
PROMPT_SECTIONS = {
    # ... 现有片段 ...
    "time": "",  # 将在 update_context 中动态填充
}
```

然后在 `update_context` 中添加时间信息：

```python
import datetime

def update_context(context: dict, messages: list) -> dict:
    # ... 现有代码 ...
    return {
        # ... 现有字段 ...
        "current_time": datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
    }
```

最后在 `assemble_system_prompt` 中条件加载这个片段，观察效果。
