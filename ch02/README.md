# Ch02: 扩展工具集与安全机制

## 快速开始

```bash
conda activate learnclaude
python -m ch02.code
```

测试 prompt：`请阅读ch01\README.md,然后给我一个总结。`

---

## 代码解析

### 1. 环境配置

```python
import os
from anthropic import Anthropic
from dotenv import load_dotenv
import subprocess
from pathlib import Path

load_dotenv(override=True)

client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
WORKDIR = Path.cwd()
SYSTEM = f"You are a coding agent at {WORKDIR}. Use bash to solve tasks. Act, don't explain.当前环境是windows环境下。"
```

**改进点：**
- 使用 `Path` 对象管理工作目录，提供更安全的路径操作
- SYSTEM 提示词明确指出当前是 Windows 环境，帮助模型选择正确的命令

### 2. 工具定义

ch02 在 ch01 的 `bash` 工具基础上新增了 4 个文件操作工具：

#### 工具列表

| 工具名称 | 功能 | 必填参数 | 可选参数 |
|---------|------|---------|---------|
| `bash` | 执行 shell 命令 | `command` (str) | - |
| `read_file` | 读取文件内容 | `path` (str) | `limit` (int) |
| `write_file` | 写入文件内容 | `path`, `content` (str) | - |
| `edit_file` | 替换文件中的文本 | `path`, `old_text`, `new_text` (str) | - |
| `glob` | 查找匹配的文件 | `pattern` (str) | - |

#### read_file 工具

```python
{
    "name": "read_file",
    "description": "Read file contents.",
    "input_schema": {
        "type": "object",
        "properties": {"path": {"type": "string"}, "limit": {"type": "integer"}},
        "required": ["path"]
    }
}
```

**参数说明：**
- `path`：文件路径（必填）
- `limit`：返回的最大行数（可选）

#### write_file 工具

```python
{
    "name": "write_file",
    "description": "Write content to a file.",
    "input_schema": {
        "type": "object",
        "properties": {"path": {"type": "string"}, "content": {"type": "string"}},
        "required": ["path", "content"]
    }
}
```

**参数说明：**
- `path`：目标文件路径（必填）
- `content`：要写入的内容（必填）

#### edit_file 工具

```python
{
    "name": "edit_file",
    "description": "Replace exact text in a file once.",
    "input_schema": {
        "type": "object",
        "properties": {"path": {"type": "string"}, "old_text": {"type": "string"}, "new_text": {"type": "string"}},
        "required": ["path", "old_text", "new_text"]
    }
}
```

**参数说明：**
- `path`：目标文件路径（必填）
- `old_text`：要被替换的文本（必填）
- `new_text`：新文本（必填）

#### glob 工具

```python
{
    "name": "glob",
    "description": "Find files matching a glob pattern.",
    "input_schema": {
        "type": "object",
        "properties": {"pattern": {"type": "string"}},
        "required": ["pattern"]
    }
}
```

**参数说明：**
- `pattern`：glob 匹配模式（必填），如 `*.py`、`**/*.md`

### 3. 安全路径检查

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

**安全机制说明：**

```
┌─────────────────────────────────────────────────────────────┐
│                    路径安全检查流程                          │
├─────────────────────────────────────────────────────────────┤
│  输入路径: "../../etc/passwd"                               │
│                    ↓                                        │
│  WORKDIR / p: 解析为绝对路径                                │
│                    ↓                                        │
│  .resolve(): 解析所有符号链接和上级目录                      │
│                    ↓                                        │
│  is_relative_to(WORKDIR): 检查是否在工作目录内              │
│                    ↓                                        │
│  ✓ 在目录内 → 返回安全路径                                   │
│  ✗ 不在目录内 → 抛出 ValueError                             │
└─────────────────────────────────────────────────────────────┘
```

**防护效果：**
- 防止路径遍历攻击（如 `../../etc/passwd`）
- 确保所有文件操作都限制在工作目录内
- 保护系统敏感文件不被访问或修改

### 4. 工具执行函数

#### run_read

```python
def run_read(path: str, limit: int | None = None) -> str:
    try:
        lines = safe_path(path).read_text().splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        return "\n".join(lines)
    except Exception as e:
        return f"Error: {e}"
```

**功能特点：**
- 使用 `safe_path` 确保路径安全
- 支持 `limit` 参数限制返回行数
- 超过限制时显示省略信息

#### run_write

```python
def run_write(path: str, content: str) -> str:
    try:
        file_path = safe_path(path)
        file_path.parent.mkdir(parents=True, exist_ok=True)
        file_path.write_text(content)
        return f"Wrote {len(content)} bytes to {path}"
    except Exception as e:
        return f"Error: {e}"
```

**功能特点：**
- 自动创建不存在的父目录
- 返回写入的字节数作为确认信息

#### run_edit

```python
def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        file_path = safe_path(path)
        text = file_path.read_text()
        if old_text not in text:
            return f"Error: text not found in {path}"
        file_path.write_text(text.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

**功能特点：**
- 使用 `text.replace(..., 1)` 只替换第一次出现的文本
- 替换前验证 `old_text` 是否存在于文件中

#### run_glob

```python
def run_glob(pattern: str) -> str:
    import glob as g
    try:
        results = []
        for match in g.glob(pattern, root_dir=WORKDIR):
            if (WORKDIR / match).resolve().is_relative_to(WORKDIR):
                results.append(match)
        return "\n".join(results) if results else "(no matches)"
    except Exception as e:
        return f"Error: {e}"
```

**功能特点：**
- 使用 `root_dir=WORKDIR` 限制搜索范围
- 对每个匹配结果进行安全路径检查

### 5. 工具分发映射

```python
TOOL_HANDLERS = {
    "bash": run_bash, "read_file": run_read, "write_file": run_write,
    "edit_file": run_edit, "glob": run_glob,
}
```

**设计模式：策略模式（Strategy Pattern）**

这是 ch02 的核心改进，将工具名称映射到对应的处理函数：

```
┌─────────────────────────────────────────────────────────────┐
│                    工具分发流程                              │
├─────────────────────────────────────────────────────────────┤
│  LLM 返回工具调用: {"name": "read_file", "input": {...}}    │
│                    ↓                                        │
│  TOOL_HANDLERS["read_file"] → 获取 run_read 函数            │
│                    ↓                                        │
│  run_read(**block.input) → 执行工具                          │
│                    ↓                                        │
│  返回结果给 LLM                                             │
└─────────────────────────────────────────────────────────────┘
```

**优势：**
- **扩展性**：添加新工具只需定义工具描述和处理函数，然后添加到字典中
- **解耦**：工具定义和执行逻辑分离
- **简洁**：agent_loop 中只需一行代码 `TOOL_HANDLERS[block.name](**block.input)`

### 6. 代理循环改进

```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"\033[33m> {block.name}\033[0m")
                if block.name == "bash":
                    print(f"\033[33m$ {block.input['command']}\033[0m")
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown: {block.name}"
                print(output[:200])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

**与 ch01 的对比：**

| 版本 | 工具执行方式 | 代码行数 | 扩展性 |
|------|-------------|---------|--------|
| ch01 | 硬编码调用 `run_bash` | 固定 | 差 |
| ch02 | 通过 `TOOL_HANDLERS` 映射 | 动态 | 好 |

---

## 扩展思考

### 工具设计原则

1. **单一职责**：每个工具只做一件事
2. **安全优先**：所有文件操作都经过 `safe_path` 检查
3. **清晰的输入输出**：定义明确的参数和返回值格式
4. **容错性**：捕获异常并返回友好的错误信息

### 可能的改进方向

1. **添加更多工具**：
   - `list_dir`：列出目录内容
   - `delete_file`：删除文件（需谨慎实现）
   - `http_request`：发送 HTTP 请求
   - `search_code`：搜索代码内容

2. **增强安全机制**：
   - 命令白名单/黑名单
   - 文件大小限制
   - 操作日志记录

3. **优化用户体验**：
   - 彩色输出区分工具类型
   - 进度条显示长时间操作
   - 支持批量操作

4. **提升推理能力**：
   - 添加工具优先级排序
   - 实现工具选择策略
   - 添加工具使用历史记录

---

## 实践练习

尝试使用以下 prompt 测试代码：

1. `列出当前目录下所有的 .py 文件`
2. `创建一个名为 test.py 的文件，内容为打印 'Hello World'`
3. `读取 ch01/code.py 的前 20 行`
4. `修改 test.py，将 'Hello World' 改为 'Hello Agent'`