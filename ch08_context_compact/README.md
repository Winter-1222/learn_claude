# Ch08: Context Compact — 四层上下文压缩管道

## 快速开始

```bash
conda activate learnclaude
python -m ch08_context_compact.code
```

测试 prompt：`请执行多个连续的工具调用，观察上下文压缩效果`

---

## 核心概念：上下文压缩系统

ch08 在 ch07 的基础上，引入了 **四层上下文压缩管道**，用于在对话历史过长时自动压缩上下文，避免超过模型的 token 限制。

### 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      四层上下文压缩管道架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  messages[]                                                             │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    压缩管道执行流程                               │   │
│  ├──────────────────────────────────────────────────────────────────┤   │
│  │                                                                  │   │
│  │  L3: tool_result_budget  ─→ 大结果持久化到磁盘                    │   │
│  │          │                                                       │   │
│  │          ▼                                                       │   │
│  │  L1: snip_compact       ─→ 裁剪中间消息（保留头部3条+尾部）        │   │
│  │          │                                                       │   │
│  │          ▼                                                       │   │
│  │  L2: micro_compact      ─→ 用占位符替换旧的 tool_result          │   │
│  │          │                                                       │   │
│  │          ▼                                                       │   │
│  │  [token > threshold?]   ─→ 检查是否超过 CONTEXT_LIMIT            │   │
│  │          │                                                       │   │
│  │    ┌─────┴─────┐                                                 │   │
│  │    ▼           ▼                                                 │   │
│  │   No          Yes                                                │   │
│  │    │           │                                                 │   │
│  │    │           ▼                                                 │   │
│  │    │    L4: compact_history ─→ LLM 完整总结（1次 API 调用）       │   │
│  │    │           │                                                 │   │
│  │    └─────┬─────┘                                                 │   │
│  │          ▼                                                       │   │
│  │    LLM call                                                      │   │
│  │          │                                                       │   │
│  │    [prompt_too_long?] ─→ API 返回过长错误                         │   │
│  │          │                                                       │   │
│  │         Yes                                                      │   │
│  │          │                                                       │   │
│  │          ▼                                                       │   │
│  │    Emergency: reactive_compact ─→ 紧急压缩（保留尾部5条）          │   │
│  │                                                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  核心原则：便宜的压缩先执行，昂贵的压缩后执行                            │
│  执行顺序：budget → snip → micro → auto                                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 四层压缩策略

| 层级 | 名称 | 作用 | 成本 | 代码位置 |
|------|------|------|------|---------|
| L1 | `snip_compact` | 裁剪中间消息，保留头部3条+尾部 | 低（纯 Python） | [第 292-321 行](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L292-L321) |
| L2 | `micro_compact` | 用占位符替换旧的 tool_result | 低（纯 Python） | [第 341-359 行](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L341-L359) |
| L3 | `tool_result_budget` | 大结果持久化到磁盘 | 中（文件 I/O） | [第 369-383 行](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L369-L383) |
| L4 | `compact_history` | LLM 完整总结 | 高（1 次 API 调用） | [第 404-408 行](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L404-L408) |

### 配置参数

```python
CONTEXT_LIMIT = 50000      # 上下文大小限制（字符数）
KEEP_RECENT = 3            # 保留最近的 tool_result 数量
PERSIST_THRESHOLD = 30000  # 持久化阈值（超过此大小的结果存储到磁盘）
MAX_REACTIVE_RETRIES = 1   # 紧急压缩的最大重试次数
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L263-L265)

---

## 代码解析

### 1. 辅助函数

#### 消息大小估算

```python
def estimate_size(msgs): return len(str(msgs))
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L267)

**功能**：通过将消息序列化为字符串来估算其大小。

#### Block 类型判断

```python
def _block_type(block):
    return block.get("type") if isinstance(block, dict) else getattr(block, "type", None)
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L269-L270)

**功能**：兼容字典和 SDK 对象两种格式，获取 block 的类型。

#### 工具调用/结果消息检测

```python
def _message_has_tool_use(msg):
    if msg.get("role") != "assistant": return False
    content = msg.get("content")
    if not isinstance(content, list): return False
    return any(_block_type(block) == "tool_use" for block in content)

def _is_tool_result_message(msg):
    if msg.get("role") != "user": return False
    content = msg.get("content")
    if not isinstance(content, list): return False
    return any(isinstance(block, dict) and block.get("type") == "tool_result" for block in content)
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L273-L288)

**功能**：检测消息是否包含 tool_use 或 tool_result。

### 2. L1: snip_compact — 裁剪中间消息

```python
def snip_compact(messages, max_messages=50):
    if len(messages) <= max_messages: return messages
    keep_head, keep_tail = 3, max_messages - 3
    head_end, tail_start = keep_head, len(messages) - keep_tail
    
    # 边界保护：不拆分配对的 tool_use/tool_result
    if head_end > 0 and _message_has_tool_use(messages[head_end - 1]):
        while head_end < len(messages) and _is_tool_result_message(messages[head_end]):
            head_end += 1
    if (tail_start > 0 and tail_start < len(messages)
            and _is_tool_result_message(messages[tail_start])
            and _message_has_tool_use(messages[tail_start - 1])):
        tail_start -= 1
    
    if head_end >= tail_start: return messages
    snipped = tail_start - head_end
    return messages[:head_end] + [{"role": "user", "content": f"[snipped {snipped} messages]"}] + messages[tail_start:]
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L292-L321)

**工作原理**：

```
裁剪前（100条消息）：
[msg0, msg1, msg2, msg3, msg4, msg5, ..., msg95, msg96, msg97, msg98, msg99]

裁剪后（最多50条消息）：
[msg0, msg1, msg2, [snipped 50 messages], msg53, msg54, ..., msg99]
            ↑                         ↑                        ↑
         头部3条                   裁剪标记                  尾部47条
```

**边界保护机制**：确保不会在 tool_use 和 tool_result 配对中间进行裁剪。

### 3. L2: micro_compact — 替换旧的 tool_result

```python
def collect_tool_results(messages):
    """收集所有 tool_result 块，返回 (消息索引, 块索引, 块对象) 三元组列表"""
    blocks = []
    for mi, msg in enumerate(messages):
        if msg.get("role") != "user" or not isinstance(msg.get("content"), list):
            continue
        for bi, block in enumerate(msg["content"]):
            if isinstance(block, dict) and block.get("type") == "tool_result":
                blocks.append((mi, bi, block))
    return blocks

def micro_compact(messages):
    """压缩旧的 tool_result：保留最近 KEEP_RECENT 个，更早的大结果替换为占位符"""
    tool_results = collect_tool_results(messages)
    if len(tool_results) <= KEEP_RECENT: return messages
    
    # 只压缩前面的，保留最后 KEEP_RECENT 个完整结果
    for _, _, block in tool_results[:-KEEP_RECENT]:
        if len(block.get("content", "")) > 120:
            block["content"] = "[Earlier tool result compacted. Re-run if needed.]"
    
    return messages
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L326-L359)

**工作原理**：

```
原始状态（10个 tool_result，KEEP_RECENT=3）：
[tool_result_1, tool_result_2, ..., tool_result_8, tool_result_9, tool_result_10]

压缩后：
[[占位符], [占位符], ..., tool_result_8, tool_result_9, tool_result_10]
              ↑                                    ↑
         前7个被压缩                          后3个保持完整
```

**关键细节**：
- 只压缩内容超过 120 字符的结果（短结果不值得压缩）
- 原地修改消息列表，不创建副本

### 4. L3: tool_result_budget — 大结果持久化

```python
def persist_large_output(tool_use_id, output):
    """将大结果持久化到磁盘，返回预览和文件路径"""
    if len(output) <= PERSIST_THRESHOLD: return output
    TOOL_RESULTS_DIR.mkdir(parents=True, exist_ok=True)
    path = TOOL_RESULTS_DIR / f"{tool_use_id}.txt"
    if not path.exists(): path.write_text(output)
    return f"<persisted-output>\nFull output: {path}\nPreview:\n{output[:2000]}\n</persisted-output>"

def tool_result_budget(messages, max_bytes=200_000):
    """检查最后一条消息的 tool_result 总大小，超过限制时持久化大结果"""
    last = messages[-1] if messages else None
    if not last or last.get("role") != "user" or not isinstance(last.get("content"), list):
        return messages
    
    blocks = [(i, b) for i, b in enumerate(last["content"]) 
              if isinstance(b, dict) and b.get("type") == "tool_result"]
    total = sum(len(str(b.get("content", ""))) for _, b in blocks)
    
    if total <= max_bytes: return messages
    
    # 按大小降序排序，优先压缩最大的结果
    ranked = sorted(blocks, key=lambda p: len(str(p[1].get("content", ""))), reverse=True)
    for _, block in ranked:
        if total <= max_bytes: break
        content = str(block.get("content", ""))
        if len(content) <= PERSIST_THRESHOLD: continue  # 小结果不压缩
        tid = block.get("tool_use_id", "unknown")
        block["content"] = persist_large_output(tid, content)
        total = sum(len(str(b.get("content", ""))) for _, b in blocks)
    
    return messages
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L362-L383)

**工作原理**：

```
原始结果（假设总大小 500KB）：
[tool_result: "非常长的输出...", tool_result: "中等输出", tool_result: "短输出"]

持久化后：
[tool_result: "<persisted-output>\nFull output: .task_outputs/tool-results/xxx.txt\nPreview: ...\n</persisted-output>", 
 tool_result: "中等输出", 
 tool_result: "短输出"]
```

**关键细节**：
- 只处理最后一条消息中的 tool_result
- 按大小降序排序，优先压缩最大的结果
- 保留前 2000 字符作为预览
- 文件存储在 `.task_outputs/tool-results/` 目录

### 5. L4: compact_history — LLM 完整总结

```python
def write_transcript(messages):
    """将完整消息历史写入 transcript 文件"""
    TRANSCRIPT_DIR.mkdir(parents=True, exist_ok=True)
    path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with path.open("w") as f:
        for msg in messages: f.write(json.dumps(msg, default=str) + "\n")
    return path

def summarize_history(messages):
    """调用 LLM 对对话历史进行总结"""
    conversation = json.dumps(messages, default=str)[:80000]
    prompt = ("Summarize this coding-agent conversation so work can continue.\n"
              "Preserve: 1. current goal, 2. key findings/decisions, 3. files read/changed, "
              "4. remaining work, 5. user constraints.\nBe compact but concrete.\n\n" + conversation)
    response = client.messages.create(
        model=MODEL, messages=[{"role": "user", "content": prompt}], max_tokens=2000)
    return "\n".join(getattr(block, "text", "") 
                     for block in response.content 
                     if getattr(block, "type", None) == "text").strip() or "(empty summary)"

def compact_history(messages):
    """完整压缩：保存 transcript，用 LLM 总结替换历史"""
    transcript_path = write_transcript(messages)
    print(f"[transcript saved: {transcript_path}]")
    summary = summarize_history(messages)
    return [{"role": "user", "content": f"[Compacted]\n\n{summary}"}]
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L386-L408)

**工作原理**：

```
原始消息（100条）：
[msg0, msg1, msg2, ..., msg98, msg99]

压缩后：
[{"role": "user", "content": "[Compacted]\n\n当前目标：...\n关键发现：...\n文件变更：...\n剩余工作：..."}]
```

**总结内容要求**：
1. 当前目标（current goal）
2. 关键发现/决策（key findings/decisions）
3. 读取/修改的文件（files read/changed）
4. 剩余工作（remaining work）
5. 用户约束（user constraints）

### 6. Emergency: reactive_compact — 紧急压缩

```python
def reactive_compact(messages):
    """API 返回 prompt_too_long 时的紧急压缩"""
    transcript = write_transcript(messages)
    tail_start = max(0, len(messages) - 5)
    
    # 边界保护：确保不拆分 tool_use/tool_result 配对
    if (tail_start > 0 and tail_start < len(messages)
            and _is_tool_result_message(messages[tail_start])
            and _message_has_tool_use(messages[tail_start - 1])):
        tail_start -= 1
    
    summary = summarize_history(messages[:tail_start])
    return [{"role": "user", "content": f"[Reactive compact]\n\n{summary}"}, *messages[tail_start:]]
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L411-L419)

**工作原理**：

```
原始消息（100条）：
[msg0, msg1, ..., msg95, msg96, msg97, msg98, msg99]

紧急压缩后（保留最后5条）：
[{"role": "user", "content": "[Reactive compact]\n\n总结..."}, msg95, msg96, msg97, msg98, msg99]
                                              ↑                        ↑
                                        前95条的总结             最后5条保持完整
```

**关键细节**：
- 保留最后 5 条消息的完整内容
- 对前面的消息进行 LLM 总结
- 用于 API 返回 prompt_too_long 错误时的紧急处理

### 7. compact 工具

```python
{
    "name": "compact", 
    "description": "Summarize earlier conversation to free context space.",
    "input_schema": {
        "type": "object", 
        "properties": {"focus": {"type": "string"}}}
},
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L490-L496)

**功能**：允许代理手动触发上下文压缩。

### 8. agent_loop 集成

```python
def agent_loop(messages: list):
    reactive_retries = 0
    global rounds_since_todo
    while True:
        # s08: 三层预处理（低成本，零 API 调用）
        messages[:] = tool_result_budget(messages)    # L3: 持久化大结果
        messages[:] = snip_compact(messages)          # L1: 裁剪中间消息
        messages[:] = micro_compact(messages)         # L2: 替换旧结果
        
        # s08: 仍然超过阈值 → LLM 总结（1 次 API 调用）
        if estimate_size(messages) > CONTEXT_LIMIT:
            print("[auto compact]")
            messages[:] = compact_history(messages)
        
        try:
            response = client.messages.create(...)
            reactive_retries = 0
        except Exception as e:
            # s08: API 返回过长错误 → 紧急压缩
            if ("prompt_too_long" in str(e).lower() or "too many tokens" in str(e).lower()) and reactive_retries < MAX_REACTIVE_RETRIES:
                print("[reactive compact]")
                messages[:] = reactive_compact(messages)
                reactive_retries += 1
                continue
            raise
        
        # ... 工具执行逻辑 ...
        
        # s08: 代理主动调用 compact 工具
        if block.name == "compact":
            messages[:] = compact_history(messages)
            results.append({"type": "tool_result", "tool_use_id": block.id,
                            "content": "[Compacted. Conversation history has been summarized.]"})
            messages.append({"role": "user", "content": results})
            break
```

[代码位置](file:///e:/Agents/learn_claude_code/ch08_context_compact/code.py#L627-L709)

**压缩触发时机**：

| 时机 | 触发方式 | 压缩类型 |
|------|---------|---------|
| 每次循环开始 | 自动 | L3 → L1 → L2 |
| 大小超过 CONTEXT_LIMIT | 自动 | L4 |
| API 返回 prompt_too_long | 异常处理 | reactive_compact |
| 代理调用 compact 工具 | 手动 | L4 |

---

## 扩展思考

### 压缩策略的设计原则

1. **成本优先**：先执行低成本的压缩（纯 Python 操作），再执行高成本的压缩（API 调用）
2. **语义保持**：确保不拆分配对的 tool_use/tool_result，避免上下文语义断裂
3. **渐进压缩**：逐层压缩，每层只做最小必要的压缩
4. **紧急回退**：当所有主动压缩都不够时，使用紧急压缩作为最后手段
5. **可恢复性**：大结果持久化到磁盘，需要时可以重新读取

### 不同压缩方式的对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| snip_compact | 速度快，零成本 | 可能丢失重要上下文 | 消息数量过多时 |
| micro_compact | 保留工具调用记录 | 丢失详细结果 | tool_result 过多时 |
| tool_result_budget | 大幅减少 token | 需要文件 I/O | 结果过大时 |
| compact_history | 最高压缩率 | 成本高（API 调用） | 所有其他方式都不够时 |
| reactive_compact | 紧急情况下的最后手段 | 可能丢失大量上下文 | API 返回过长错误时 |

### 文件存储结构

```
.project_root/
├── .task_outputs/
│   └── tool-results/          # L3: 大结果持久化
│       ├── tool_use_id_1.txt
│       ├── tool_use_id_2.txt
│       └── ...
├── .transcripts/              # L4: 完整对话记录
│   ├── transcript_1234567890.jsonl
│   ├── transcript_1234567891.jsonl
│   └── ...
└── ch08_context_compact/
    └── code.py
```

### 可能的改进方向

1. **智能压缩策略**：
   - 根据对话内容自动选择最合适的压缩方式
   - 对不同类型的消息采用不同的压缩策略
   - 保留关键决策和重要结果，丢弃重复或冗余信息

2. **增量压缩**：
   - 只压缩新增的消息，不重复压缩已压缩的内容
   - 维护压缩状态，避免每次都从头开始

3. **上下文感知压缩**：
   - 分析消息之间的依赖关系
   - 保留因果链上的关键消息
   - 丢弃已经被后续操作覆盖的消息

4. **压缩效果监控**：
   - 记录每次压缩前后的 token 数量
   - 评估压缩对模型响应质量的影响
   - 动态调整压缩阈值

5. **结果检索**：
   - 提供工具让代理可以检索已压缩的结果
   - 支持按工具名称、时间、内容搜索
   - 快速定位和恢复历史结果

---

## 实践练习

### 练习 1：测试压缩管道

使用以下 prompt 测试完整的压缩管道：

```
请创建一个包含大量输出的任务，例如：
1. 创建多个文件，每个文件包含大量内容
2. 运行命令生成大量输出
3. 读取大文件内容
```

观察终端输出中是否出现 `[auto compact]`、`[reactive compact]` 等压缩提示。

### 练习 2：测试手动压缩

使用以下 prompt 测试手动触发压缩：

```
请执行一些操作，然后调用 compact 工具来压缩上下文
```

观察代理是否会调用 `compact` 工具，并查看压缩后的效果。

### 练习 3：分析压缩效果

在代码中添加日志，记录每次压缩前后的消息数量和大小：

```python
def agent_loop(messages: list):
    while True:
        print(f"[DEBUG] Before compaction: {len(messages)} messages, {estimate_size(messages)} chars")
        
        messages[:] = tool_result_budget(messages)
        messages[:] = snip_compact(messages)
        messages[:] = micro_compact(messages)
        
        print(f"[DEBUG] After L1-L3: {len(messages)} messages, {estimate_size(messages)} chars")
        
        if estimate_size(messages) > CONTEXT_LIMIT:
            messages[:] = compact_history(messages)
            print(f"[DEBUG] After L4: {len(messages)} messages, {estimate_size(messages)} chars")
        
        # ...
```

### 练习 4：自定义压缩策略

修改压缩参数，观察不同策略的效果：

```python
CONTEXT_LIMIT = 30000  # 降低阈值，触发更频繁的压缩
KEEP_RECENT = 5        # 保留更多的 tool_result
PERSIST_THRESHOLD = 10000  # 更小的结果也持久化
```