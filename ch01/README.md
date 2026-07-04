# Ch01: 构建基础代码代理

## 快速开始

```bash
conda create -n learnclaude python=3.11 -y
conda activate learnclaude
pip install -r requirements.txt
copy .env.example .env
python -m ch01.code
```

运行后输入：`Create a file called hello.py that prints "Hello, World!"`

---

## 代码解析

### 1. 环境配置与初始化

```python
import os
from anthropic import Anthropic
from dotenv import load_dotenv
import subprocess

load_dotenv(override=True)

client = Anthropic(base_url=os.getenv("ANTHROPIC_BASE_URL"))
MODEL = os.environ["MODEL_ID"]
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."
```

**关键点说明：**
- `load_dotenv(override=True)`：从 `.env` 文件加载环境变量，覆盖系统已有的同名变量
- `Anthropic` 客户端初始化时使用了自定义的 `base_url`，支持通过环境变量配置代理或自定义端点
- `SYSTEM` 提示词告知模型当前工作目录，使其能够在正确的上下文中执行命令

### 2. 工具定义

```python
TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

**工具定义规范：**
- `name`：工具名称，模型通过此名称调用工具
- `description`：工具功能描述，帮助模型理解何时应该使用此工具
- `input_schema`：输入参数的 JSON Schema 定义，指定参数类型和必填项

### 3. 工具执行函数

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
    except (FileNotFoundError, OSError) as e:
        return f"Error: {e}"
```

**安全机制：**
- **危险命令过滤**：阻止 `rm -rf /`、`sudo`、`shutdown` 等危险操作
- **超时控制**：命令执行最长等待 120 秒，防止无限阻塞
- **输出截断**：限制输出长度为 50000 字符，避免超出 API 限制

**执行细节：**
- 使用 `shell=True` 允许执行复杂命令
- `cwd=os.getcwd()` 确保命令在当前工作目录执行
- `capture_output=True` 同时捕获标准输出和标准错误

### 4. 核心代理循环

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
                print(f"\033[33m$ {block.input['command']}\033[0m")
                output = run_bash(block.input["command"])
                print(output[:200])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

**工作流程：**

```
┌─────────────────────────────────────────────────────────────┐
│                    代理循环流程                              │
├─────────────────────────────────────────────────────────────┤
│  1. 用户输入 → 追加到 messages 列表                          │
│                    ↓                                        │
│  2. 调用 LLM API，传入 messages 和 TOOLS                    │
│                    ↓                                        │
│  3. 检查 stop_reason                                        │
│     ├─ "tool_use" → 执行工具 → 收集结果 → 返回步骤 2         │
│     └─ 其他 → 返回最终结果                                   │
└─────────────────────────────────────────────────────────────┘
```

**关键步骤：**
1. 将消息和工具定义发送给 LLM
2. 检查 `stop_reason`：
   - 如果是 `"tool_use"`，执行模型要求的工具
   - 如果不是，循环结束，返回最终回答
   - stop_reason可选值：end_turn（正常结束）、max_tokens（达到 Token 上限）、tool_use（工具调用）。
3. 执行工具时，收集结果并以 `tool_result` 格式追加到消息历史
4. 继续循环，让模型根据工具执行结果继续推理

### 5. 主程序入口

```python
if __name__ == "__main__":
    print("s01: Agent Loop")
    print("输入问题，回车发送。输入 q 退出。\n")
    history = []
    while True:
        try:
            query = input("\033[36ms01 >> \033[0m")
        except (EOFError, KeyboardInterrupt):
            break
        if query.lower() == "q":
            break
        history.append({"role": "user", "content": query})
        agent_loop(history)
        response_content = history[-1]["content"]
        if isinstance(response_content, list):
            for block in response_content:
                if getattr(block, "type", None) == "text":
                    print(block.text)
        print()
```

**交互逻辑：**
- 提供命令行交互界面，提示符为 `s01 >>`
- 支持连续对话，`history` 列表保存完整对话历史
- 输入 `q` 或按 `Ctrl+C` 退出程序
- 输出时只显示文本内容，过滤掉工具调用块

---

## 扩展思考

### 可能的改进方向

1. **增加更多工具**：
   - 文件读写工具（`read_file`, `write_file`）
   - 代码编辑工具（`edit_file`）
   - 网络请求工具（`http_request`）

2. **增强安全机制**：
   - 命令白名单机制
   - 目录访问限制
   - 命令执行权限控制

3. **优化用户体验**：
   - 添加命令历史记录
   - 支持快捷键操作
   - 增加输出格式化

4. **提升推理能力**：
   - 添加长期记忆存储
   - 实现任务规划和分解
   - 增加错误恢复机制

### 核心设计模式

这段代码展示了 **ReAct（Reasoning + Acting）** 模式的核心思想：

1. **推理**：模型根据问题决定下一步行动
2. **行动**：执行工具获取外部信息
3. **反馈**：将工具执行结果反馈给模型
4. **循环**：重复直到问题解决

这种模式使 LLM 能够与外部环境交互，完成超出其内置知识范围的任务。