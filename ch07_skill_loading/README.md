# Ch07: Skill Loading — 两级按需知识注入

## 快速开始

```bash
conda activate learnclaude
python -m ch07_skill_loading.code
```

测试 prompt：`请使用 code-review 技能来审查 ch06/code.py 的代码`

---

## 核心概念：两级技能加载系统

ch07 在 ch06 的基础上，引入了 **Skill Loading（技能加载）系统**，实现了两级按需知识注入机制。

### 系统架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       两级技能加载系统架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Layer 1: 低成本层（始终存在）                                     │   │
│  │                                                                   │   │
│  │  启动时扫描 skills/ 目录                                          │   │
│  │       │                                                           │   │
│  │       ▼                                                           │   │
│  │  构建技能目录（名称 + 一行描述）                                     │   │
│  │       │                                                           │   │
│  │       ▼                                                           │   │
│  │  注入到 SYSTEM Prompt                                             │   │
│  │  "Skills available: agent-builder, code-review, mcp-builder, pdf"│   │
│  │                                                                   │   │
│  │  成本：~100 tokens/技能（非常低）                                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Layer 2: 高成本层（按需加载）                                     │   │
│  │                                                                   │   │
│  │  代理调用 load_skill("code-review")                               │   │
│  │       │                                                           │   │
│  │       ▼                                                           │   │
│  │  从 SKILL_REGISTRY 查找技能                                        │   │
│  │       │                                                           │   │
│  │       ▼                                                           │   │
│  │  返回完整的 SKILL.md 内容                                         │   │
│  │       │                                                           │   │
│  │       ▼                                                           │   │
│  │  通过 tool_result 注入到对话上下文                                  │   │
│  │                                                                   │   │
│  │  成本：~2000 tokens/技能（较高）                                   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  skills/ 目录结构：                                                    │
│    agent-builder/SKILL.md                                             │
│    code-review/SKILL.md                                               │
│    mcp-builder/SKILL.md                                               │
│    pdf/SKILL.md                                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Ch07 新增特性

| 特性 | 说明 | 代码位置 |
|------|------|---------|
| `SKILLS_DIR` | 技能目录路径配置 | [第 36 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L36) |
| `SKILL_REGISTRY` | 技能注册表（名称→元数据+内容） | [第 54 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L54) |
| `_parse_frontmatter()` | 解析 YAML frontmatter | [第 40-51 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L40-L51) |
| `_scan_skills()` | 扫描技能目录并构建注册表 | [第 56-71 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L56-L71) |
| `list_skills()` | 列出所有技能（名称+描述） | [第 73-77 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L73-L77) |
| `build_system()` | 构建包含技能目录的 SYSTEM prompt | [第 80-92 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L80-L92) |
| `load_skill()` | 加载完整技能内容 | [第 248-253 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L248-L253) |
| `load_skill` 工具 | 运行时加载技能内容的工具 | [第 316-323 行](file:///e:/Agents/learn_claude_code/ch07/code.py#L316-L323) |

---

## 代码解析

### 1. 技能目录配置

```python
SKILLS_DIR = WORKDIR / "skills"
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L36)

**配置说明**：技能文件存储在项目根目录下的 `skills/` 文件夹中。

### 2. YAML Frontmatter 解析

```python
def _parse_frontmatter(text: str) -> tuple[dict, str]:
    """Parse YAML frontmatter from SKILL.md. Returns (meta, body)."""
    if not text.startswith("---"):
        return {}, text
    parts = text.split("---", 2)
    if len(parts) < 3:
        return {}, text
    try:
        meta = yaml.safe_load(parts[1]) or {}
    except yaml.YAMLError:
        meta = {}
    return meta, parts[2].strip()
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L40-L51)

**功能**：从 SKILL.md 文件中解析 YAML frontmatter 元数据。

**SKILL.md 文件格式**：

```yaml
---
name: "code-review"
description: "代码审查技能：分析代码质量、安全性和最佳实践"
---

# Code Review 技能

## 使用指南

1. 阅读待审查的代码
2. 检查代码风格和规范
3. 识别潜在的安全漏洞
4. 提供改进建议

## 检查清单

- [ ] 变量命名是否清晰
- [ ] 函数是否单一职责
- [ ] 是否有足够的注释
- [ ] 错误处理是否完善
```

**解析结果**：

| 返回值 | 内容 |
|--------|------|
| `meta` | `{"name": "code-review", "description": "代码审查技能：分析代码质量、安全性和最佳实践"}` |
| `body` | 去除 frontmatter 后的正文内容 |

### 3. 技能注册表

```python
SKILL_REGISTRY: dict[str, dict] = {}

def _scan_skills():
    """Scan skills/ dir, populate SKILL_REGISTRY with name/description/content."""
    if not SKILLS_DIR.exists():
        return
    for d in sorted(SKILLS_DIR.iterdir()):
        if not d.is_dir():
            continue
        manifest = d / "SKILL.md"
        if manifest.exists():
            raw = manifest.read_text()
            meta, body = _parse_frontmatter(raw)
            name = meta.get("name", d.name)
            desc = meta.get("description", raw.split("\n")[0].lstrip("#").strip())
            SKILL_REGISTRY[name] = {"name": name, "description": desc, "content": raw}

_scan_skills()
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L54-L71)

**扫描流程**：

```
skills/
    ├── agent-builder/
    │   └── SKILL.md  → 解析 → SKILL_REGISTRY["agent-builder"]
    ├── code-review/
    │   └── SKILL.md  → 解析 → SKILL_REGISTRY["code-review"]
    ├── mcp-builder/
    │   └── SKILL.md  → 解析 → SKILL_REGISTRY["mcp-builder"]
    └── pdf/
        └── SKILL.md  → 解析 → SKILL_REGISTRY["pdf"]
```

**注册表结构**：

```python
{
    "code-review": {
        "name": "code-review",
        "description": "代码审查技能",
        "content": "---\nname: \"code-review\"\n...（完整内容）"
    }
}
```

### 4. 技能列表生成

```python
def list_skills() -> str:
    """List all skills (name + one-line description)."""
    if not SKILL_REGISTRY:
        return "(no skills found)"
    return "\n".join(f"- **{s['name']}**: {s['description']}" for s in SKILL_REGISTRY.values())
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L73-L77)

**输出示例**：

```
- **agent-builder**: 创建智能代理的技能
- **code-review**: 代码审查技能
- **mcp-builder**: MCP 服务器构建技能
- **pdf**: PDF 处理技能
```

### 5. SYSTEM Prompt 构建

```python
def build_system() -> str:
    """Build SYSTEM prompt with skill catalog injected at startup."""
    catalog = list_skills()
    return (
        f"You are a coding agent at {WORKDIR}. "
        f"Skills available:\n{catalog}\n"
        "Use load_skill to get full details when needed."
        "Before starting any multi-step task, use todo_write to plan your steps."
        "Update status as you go.Use bash to solve tasks. Act, don't explain."
        "For complex sub-problems, use the task tool to spawn a subagent."
        "当前环境是windows环境下。"
    )
SYSTEM = build_system()
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L80-L92)

**生成的 SYSTEM prompt 示例**：

```
You are a coding agent at E:\Agents\learn_claude_code. 
Skills available:
- **agent-builder**: 创建智能代理的技能
- **code-review**: 代码审查技能
- **mcp-builder**: MCP 服务器构建技能
- **pdf**: PDF 处理技能
Use load_skill to get full details when needed.
Before starting any multi-step task, use todo_write to plan your steps.
...
```

**设计意图**：

| 部分 | 作用 |
|------|------|
| `"Skills available:"` | 告诉代理有哪些技能可用 |
| `"Use load_skill to get full details"` | 引导代理在需要时加载完整内容 |
| 只包含名称和描述 | 控制 token 消耗，避免 SYSTEM prompt 过大 |

### 6. 技能加载函数

```python
def load_skill(name: str) -> str:
    """Load full skill content. Lookup via registry — no path traversal."""
    skill = SKILL_REGISTRY.get(name)
    if not skill:
        return f"Skill not found: {name}"
    return skill["content"]
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L248-L253)

**安全设计**：

| 机制 | 说明 |
|------|------|
| **注册表查找** | 通过技能名称从 `SKILL_REGISTRY` 查找，而非直接读取文件 |
| **无路径遍历** | 不接受文件路径参数，防止路径遍历攻击 |
| **预先扫描** | 技能在启动时就已扫描并验证，运行时不会读取新文件 |

### 7. load_skill 工具定义

```python
{
    "name": "load_skill", 
    "description": "Load the full content of a skill by name.",
    "input_schema": {
        "type": "object", 
        "properties": {"name": {"type": "string"}}, 
        "required": ["name"]}
},
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L316-L323)

**参数说明**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | string | 技能名称，如 `"code-review"` |

### 8. 工具分发映射

```python
TOOL_HANDLERS = {
    "bash": run_bash, "read_file": run_read, "write_file": run_write,
    "edit_file": run_edit, "glob": run_glob, "todo_write": run_todo_write,
    "task": spawn_subagent, "load_skill": load_skill,  # ← 新增
}
```

[代码位置](file:///e:/Agents/learn_claude_code/ch07/code.py#L325-L329)

---

## 完整工作流程示例

以"使用 code-review 技能审查代码"为例：

### 第一步：系统启动

```python
_scan_skills()  # 扫描 skills/ 目录
SYSTEM = build_system()  # 构建包含技能目录的 SYSTEM prompt
```

生成的 SYSTEM prompt：

```
You are a coding agent at E:\Agents\learn_claude_code. 
Skills available:
- **agent-builder**: 创建智能代理的技能
- **code-review**: 代码审查技能
- **mcp-builder**: MCP 服务器构建技能
- **pdf**: PDF 处理技能
Use load_skill to get full details when needed.
...
```

### 第二步：用户输入

```
请使用 code-review 技能来审查 ch06/code.py 的代码
```

### 第三步：代理加载技能

代理识别到需要使用 `code-review` 技能，调用 `load_skill` 工具：

```
[HOOK] load_skill(['code-review'])
```

工具返回完整的 SKILL.md 内容，作为 `tool_result` 注入到对话中。

### 第四步：代理执行审查

代理使用加载的技能知识，执行代码审查：

```
[HOOK] read_file(['ch06/code.py'])
[HOOK] bash(['python -m pylint ch06/code.py'])
```

### 第五步：返回审查结果

代理根据技能指南，输出结构化的审查报告。

---

## 扩展思考

### 两级加载的设计优势

| 层级 | 内容 | Token 成本 | 何时使用 |
|------|------|-----------|---------|
| Layer 1 | 名称 + 一行描述 | ~100 tokens/技能 | 始终存在，告诉代理有哪些技能 |
| Layer 2 | 完整 SKILL.md | ~2000 tokens/技能 | 按需加载，代理需要时才注入 |

**优势**：

1. **节省 Token**：不常用的技能不会占用 SYSTEM prompt 的空间
2. **灵活扩展**：可以添加大量技能而不影响性能
3. **按需获取**：代理只在需要时才加载完整内容
4. **知识隔离**：不同技能之间相互独立

### SKILL.md 编写指南

一个好的技能文件应该包含：

1. **YAML Frontmatter**：
   - `name`: 技能名称（用于 `load_skill` 调用）
   - `description`: 一行描述（显示在技能目录中）

2. **技能内容**：
   - 使用指南：如何使用这个技能
   - 最佳实践：遵循的规则和原则
   - 检查清单：可操作的检查点
   - 示例：具体的使用示例

### 可能的改进方向

1. **技能版本管理**：
   - 支持技能的多个版本
   - 版本回退机制
   - 技能更新通知

2. **技能依赖**：
   - 技能之间的依赖关系
   - 自动加载依赖技能
   - 技能冲突检测

3. **技能搜索**：
   - 根据关键词搜索技能
   - 技能推荐系统
   - 技能评分和评价

4. **技能缓存**：
   - 缓存已加载的技能内容
   - 智能缓存策略（LRU、LFU）
   - 缓存失效机制

5. **技能热更新**：
   - 在运行时动态添加技能
   - 技能文件监控
   - 无需重启即可更新技能

---

## 实践练习

### 练习 1：测试基础功能

使用以下 prompt 测试技能加载功能：

```
请使用 code-review 技能来审查 ch01/code.py 的代码
```

观察代理是否会：
1. 调用 `load_skill("code-review")`
2. 读取目标文件
3. 根据技能指南输出审查报告

### 练习 2：创建自定义技能

创建一个新的技能文件 `skills/my-skill/SKILL.md`：

```yaml
---
name: "my-skill"
description: "我的自定义技能：创建 Python CLI 工具"
---

# My Skill: Python CLI 工具创建

## 使用指南

1. 创建项目目录结构
2. 编写主程序（使用 argparse）
3. 添加命令行参数
4. 创建 setup.py 或 pyproject.toml
5. 测试 CLI 功能

## 最佳实践

- 使用 argparse 或 click 库
- 提供清晰的帮助信息
- 支持 -h/--help 参数
- 错误处理和用户友好的提示
```

然后测试：

```
请使用 my-skill 技能创建一个命令行工具，用于计算文件行数
```

### 练习 3：技能组合使用

使用以下 prompt 测试多个技能的组合：

```
请创建一个完整的代理项目，使用 agent-builder 技能设计架构，然后使用 code-review 技能审查代码质量
```

观察代理是否会依次加载和使用多个技能。