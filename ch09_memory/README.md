# Ch09: Memory System — 跨会话持久化知识

## 概述

Ch09 引入了**持久化记忆系统**，使 Agent 能够跨会话保留用户偏好、项目事实和参考信息。与 Ch08 解决单次会话内的上下文压缩不同，Ch09 关注的是**跨会话的知识积累与复用**。

### 核心数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                      agent_loop 中的记忆流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│   │ 1. Load Index│───→│ 2. Select    │───→│ 3. Inject    │    │
│   │   (MEMORY.md)│    │ Relevant     │    │ Content      │    │
│   │   (name+desc)│    │ Memories     │    │              │    │
│   └──────────────┘    └──────────────┘    └──────────────┘    │
│         │                   │                    │            │
│         ↓                   ↓                    ↓            │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│   │ SYSTEM prompt│    │ LLM semantic │    │ User turn    │    │
│   │ (始终注入)    │    │ + Keyword    │    │ (条件注入)    │    │
│   └──────────────┘    └──────────────┘    └──────────────┘    │
│                                                                 │
│                           ↓                                    │
│                   ┌──────────────┐                             │
│                   │ 4. Compress  │ ← s08 四层压缩管道            │
│                   │ Pipeline     │                             │
│                   └──────────────┘                             │
│                           ↓                                    │
│                   ┌──────────────┐                             │
│                   │   LLM Call   │                             │
│                   └──────────────┘                             │
│                           ↓                                    │
│                   ┌──────────────┐                             │
│                   │ 5. Extract   │ ← 从 pre_compress 快照提取    │
│                   │ + Consolidate│ ← 达到阈值自动合并             │
│                   └──────────────┘                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 存储层设计

### 目录结构

```
.memory/
├── MEMORY.md          ← 索引文件（≤200行，仅包含 name + description）
├── user-preference-tabs.md  ← 个体记忆文件
├── project-facts.md         ← 项目事实
├── feedback-guidance.md     ← 反馈指导
└── reference-external.md    ← 外部参考
```

### 记忆文件格式（YAML Frontmatter）

每个记忆文件采用标准 Markdown + YAML Frontmatter 格式：

```markdown
---
name: user-preference-tabs
description: 用户偏好使用制表符而非空格进行代码缩进
type: user
---

用户明确要求代码中使用制表符(Tab)进行缩进，而非空格。
这个偏好适用于所有项目文件。
```

**字段说明**：
- `name`：短标识符，使用 kebab-case 格式
- `description`：一行摘要，用于索引查询和相关性匹配
- `type`：记忆类型，取值范围见下文

### 四种记忆类型

| 类型 | 含义 | 示例 |
|------|------|------|
| `user` | 用户偏好 | 代码风格、工具偏好、工作习惯 |
| `feedback` | 反馈指导 | 用户纠正、改进建议 |
| `project` | 项目事实 | 技术栈、架构决策、约束条件 |
| `reference` | 外部指针 | 文档链接、API 参考 |

### 索引重建机制

每次写入记忆文件后，`_rebuild_index()` 函数会自动重建 `MEMORY.md`：

```python
def _rebuild_index():
    lines = []
    for f in sorted(MEMORY_DIR.glob("*.md")):
        if f.name == "MEMORY.md": continue
        meta, body = _parse_frontmatter(raw)
        name = meta.get("name", f.stem)
        desc = meta.get("description", body.split("\n")[0][:80])
        lines.append(f"- [{name}]({f.name}) — {desc}")
    MEMORY_INDEX.write_text("\n".join(lines) + "\n")
```

**设计原因**：
- 索引文件轻量（仅 name + description），每次都注入 SYSTEM prompt 成本低
- 个体文件按需加载，避免浪费 token
- 自动重建保证索引始终与实际文件同步

## 检索层设计

### select_relevant_memories — 两阶段相关性选择

`select_relevant_memories()` 函数负责从所有记忆文件中选择与当前对话相关的内容：

```python
def select_relevant_memories(messages: list, max_items: int = 5) -> list[str]:
    # 阶段1: LLM 语义选择（优先）
    prompt = "Given the recent conversation... select indices of relevant memories..."
    response = client.messages.create(...)
    # 解析 JSON 数组 [0, 3]
    
    # 阶段2: 关键词匹配回退（LLM 失败时）
    keywords = [w.lower() for w in recent.split() if len(w) > 3]
    for f in files:
        text = (f["name"] + " " + f["description"]).lower()
        if any(kw in text for kw in keywords):
            selected.append(f["filename"])
```

**工作流程**：

```
┌─────────────────────────────────────────────────────────┐
│              select_relevant_memories 流程               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 收集最近3条用户消息作为上下文                          │
│           ↓                                             │
│  2. 构建记忆目录（name + description）                    │
│           ↓                                             │
│  3. LLM 语义选择                                         │
│     ├── 成功 → 返回选中索引                               │
│     └── 失败 → 阶段4                                     │
│           ↓                                             │
│  4. 关键词匹配回退（词长>3的单词）                         │
│           ↓                                             │
│  5. 返回选中的文件名列表（最多5个）                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**关键参数**：
- `max_items=5`：限制每次最多加载5条记忆，防止上下文膨胀

### load_memories — 注入相关记忆

```python
def load_memories(messages: list) -> str:
    selected_files = select_relevant_memories(messages)
    if not selected_files: return ""
    
    parts = ["<relevant_memories>"]
    for filename in selected_files:
        content = read_memory_file(filename)
        if content: parts.append(content)
    parts.append("</relevant_memories>")
    return "\n\n".join(parts)
```

记忆内容被包裹在 `<relevant_memories>` 标签中，注入到当前用户消息的开头。

## 提取层设计

### extract_memories — 从对话中提取新记忆

每次对话结束（非 tool_use 停止）时，从原始对话中提取新记忆：

```python
def extract_memories(messages: list):
    # 收集最近10条消息的文本内容
    dialogue_parts = []
    for msg in messages[-10:]:
        role = msg.get("role", "?")
        content = msg.get("content", "")
        # ... 提取文本 ...
    
    # 将已有记忆传给 LLM 进行去重
    existing_desc = "\n".join(f"- {m['name']}: {m['description']}" for m in existing)
    
    # LLM 提取新记忆
    prompt = "Extract user preferences, constraints, or project facts..."
    items = json.loads(match.group())
    
    # 写入新记忆文件
    for mem in items:
        write_memory_file(name, mem_type, desc, body)
```

**关键设计**：
- 使用 `pre_compress` 快照（压缩前的完整消息）进行提取，确保信息完整性
- 去重机制：将已有记忆列表传给 LLM，让它判断是否已有覆盖
- 返回格式：JSON 数组 `[{name, type, description, body}, ...]`

### pre_compress 快照的重要性

在 agent_loop 中，压缩前的消息被保存为 `pre_compress`：

```python
pre_compress = [m if isinstance(m, dict) else {"role": m.get("role",""),
    "content": str(m.get("content",""))} for m in messages]
```

**原因**：
- 压缩后的消息可能丢失工具调用细节、完整输出等
- 提取记忆需要完整的对话上下文才能准确识别用户偏好
- `pre_compress` 是压缩前的完整备份，保证提取质量

## 合并层设计

### consolidate_memories — 记忆合并与清理

当记忆文件数量达到 `CONSOLIDATE_THRESHOLD=10` 时自动触发：

```python
def consolidate_memories():
    files = list_memory_files()
    if len(files) < CONSOLIDATE_THRESHOLD: return  # 未达阈值，跳过
    
    # 将所有记忆内容传给 LLM 进行合并
    prompt = "Consolidate the following memory files. Rules:\n" \
             "1. Merge duplicates into one\n" \
             "2. Remove outdated/contradicted memories\n" \
             "3. Keep the total under 30 memories\n" \
             "4. Preserve important user preferences above all"
    
    # 删除所有旧文件（保留 MEMORY.md）
    for f in MEMORY_DIR.glob("*.md"):
        if f.name != "MEMORY.md": f.unlink()
    
    # 写入合并后的新记忆
    for mem in items:
        write_memory_file(name, mem_type, desc, body)
```

**合并规则**：
1. **去重**：将重复或高度相似的记忆合并为一条
2. **删除过时**：移除已失效或被后续对话否定的记忆
3. **数量限制**：总记忆数不超过30条
4. **优先级**：用户偏好优先保留

## 关键设计决策

### 为什么索引只放 name + description？

- **成本低**：索引轻量，每次都注入 SYSTEM prompt 不会显著增加 token 消耗
- **按需加载**：只有被判断为相关的记忆才会加载完整内容
- **分层设计**：两层加载策略（索引 → 完整内容）平衡了可用性和效率

### 为什么用 pre_compress 快照提取记忆？

- **信息完整性**：压缩后的消息可能丢失关键细节（如完整的工具输出）
- **提取准确性**：记忆提取需要完整上下文才能准确识别用户偏好和项目事实
- **独立于压缩**：即使压缩策略变化，记忆提取仍然可靠

### 为什么记忆注入在 user turn 而非 system？

- **灵活性**：不同对话轮次可能需要不同的相关记忆
- **时效性**：用户最近的问题决定了哪些记忆相关
- **可控性**：可以精确控制注入位置和内容

**memory_turn 定位机制**：

代码通过 `memory_turn` 索引精确追踪原始用户消息的位置：

```python
# 记录原始用户消息的索引
memory_turn = len(messages) - 1 if messages and isinstance(messages[-1].get("content"), str) else None

# 在 LLM 调用时，将记忆拼接到该用户消息前面
request_messages[memory_turn] = {
    **messages[memory_turn],
    "content": memories_content + "\n\n" + messages[memory_turn]["content"],
}
```

**工作原理**：
1. 在 agent_loop 开始时，记录当前用户消息的索引位置
2. 即使经过压缩后消息列表发生变化，`memory_turn` 仍然指向原始位置
3. 在构造 LLM 请求时，复制消息列表并将记忆内容拼接到正确的用户消息前面
4. 这样确保记忆始终与当前用户问题关联，而非全局注入 SYSTEM

### Ch08 vs Ch09：互补而非替代

| 维度 | Ch08 Context Compact | Ch09 Memory |
|------|----------------------|-------------|
| **解决问题** | 单次会话内上下文超限 | 跨会话知识持久化 |
| **时间范围** | 当前会话 | 所有历史会话 |
| **存储方式** | 临时压缩（运行时） | 持久化文件（磁盘） |
| **核心机制** | 四层压缩管道 | 索引 + 检索 + 提取 + 合并 |
| **token 策略** | 减少单次调用消耗 | 按需加载相关内容 |
| **熔断器** | 有（连续失败3次停止） | 无（本章未包含） |

## 实践练习

### 练习1：体验记忆持久化

1. 运行 `python -m ch09_memory.code`
2. 输入："我喜欢用 Python 编写代码，代码风格要符合 PEP8 规范"
3. 输入："q" 退出
4. 重新运行程序
5. 输入："帮我写一段代码"
6. 观察系统是否记住了你的偏好

### 练习2：手动添加记忆

1. 在 `.memory/` 目录下创建文件 `project-tech-stack.md`：
```markdown
---
name: project-tech-stack
description: 项目使用 Python 3.11 和 Anthropic API
type: project
---

本项目技术栈：
- Python 3.11+
- Anthropic Claude API
- 使用 conda 管理虚拟环境
- 所有代码遵循 PEP8 规范
```

2. 运行程序，输入与技术栈相关的问题，观察系统是否加载了该记忆

### 练习3：测试记忆合并

1. 连续添加超过10条记忆（通过多次对话让系统自动提取）
2. 观察控制台输出的 `[Memory: consolidated X → Y memories]` 日志
3. 检查 `.memory/` 目录下的文件数量变化

### 练习4：理解 pre_compress 快照

在 `agent_loop` 中添加调试打印：

```python
print(f"Pre-compress: {len(pre_compress)} messages")
print(f"Post-compress: {len(messages)} messages")
```

观察压缩前后消息数量的变化，理解为什么需要从 `pre_compress` 提取记忆。
