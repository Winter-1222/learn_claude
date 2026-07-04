# Ch03: 三闸门权限系统

## 快速开始

```bash
conda activate learnclaude
python -m ch03.code
```

测试 prompt：`删除当前目录下所有 .txt 文件`

---

## 代码解析

### 1. 核心概念：三闸门权限管道

ch03 在 ch02 的基础上引入了**三道安全门**，在工具执行前进行权限检查：

```
┌─────────────────────────────────────────────────────────────────┐
│                    三闸门权限管道流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌────────┐ │
│  │ Tool     │ ──► │ Gate 1   │ ──► │ Gate 2   │ ──► │ Gate 3 │ ──► │ Execute │
│  │ Call     │     │ Deny List│     │ Rule     │     │ User   │     │         │
│  └──────────┘     └──────────┘     └───┬──────┘     └───┬────┘     └────────┘
│                                        │                │
│                                        v                v
│                                   (匹配规则?)      (用户拒绝?)
│                                        │                │
│                                        v                v
│                                   ┌────────┐      ┌────────┐
│                                   │ Ask    │      │ Block  │
│                                   │ User   │      │        │
│                                   └────────┘      └────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**三道门的作用：**

| 闸门 | 名称 | 作用 | 处理方式 |
|------|------|------|---------|
| Gate 1 | 硬拒绝列表 | 阻止绝对危险的命令 | **直接阻止**，无需用户确认 |
| Gate 2 | 规则匹配 | 检测上下文相关的风险操作 | **询问用户**，等待确认 |
| Gate 3 | 用户审批 | 最终决策权交给用户 | 用户选择允许或拒绝 |

### 2. Gate 1: 硬拒绝列表

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if=", "> /dev/sda"]

def check_deny_list(command: str) -> str | None:
    for pattern in DENY_LIST:
        if pattern in command:
            return f"Blocked: '{pattern}' is on the deny list"
    return None
```

**设计原理：**

- **绝对禁止**：这些命令无论在什么情况下都不应该执行
- **快速失败**：在权限检查的第一步就拦截，避免后续处理
- **返回原因**：明确告知用户为什么命令被阻止

**拒绝列表内容：**

| 命令模式 | 危险原因 |
|---------|---------|
| `rm -rf /` | 删除整个系统文件 |
| `sudo` | 获取管理员权限 |
| `shutdown` / `reboot` | 关闭或重启系统 |
| `mkfs` | 格式化磁盘 |
| `dd if=` | 底层磁盘写入 |
| `> /dev/sda` | 覆盖磁盘设备 |

### 3. Gate 2: 规则匹配

```python
PERMISSION_RULES = [
    {"tools": ["write_file", "edit_file"],
     "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
     "message": "Writing outside workspace"},
    {"tools": ["bash"],
     "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777", "del"]),
     "message": "Potentially destructive command"},
]

def check_rules(tool_name: str, args: dict) -> str | None:
    for rule in PERMISSION_RULES:
        if tool_name in rule["tools"] and rule["check"](args):
            return rule["message"]
    return None
```

**规则定义结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `tools` | list | 适用的工具名称列表 |
| `check` | function | 检查函数，返回 `True` 表示匹配规则 |
| `message` | str | 匹配规则时显示的提示信息 |

**内置规则：**

#### 规则一：文件操作范围限制

```python
{"tools": ["write_file", "edit_file"],
 "check": lambda args: not (WORKDIR / args.get("path", "")).resolve().is_relative_to(WORKDIR),
 "message": "Writing outside workspace"}
```

**作用**：防止将文件写入工作目录之外的位置。

**检查逻辑**：
1. 获取 `path` 参数
2. 解析为绝对路径
3. 检查是否在工作目录内
4. 如果不在，则触发规则

#### 规则二：潜在破坏性命令检测

```python
{"tools": ["bash"],
 "check": lambda args: any(kw in args.get("command", "") for kw in ["rm ", "> /etc/", "chmod 777", "del"]),
 "message": "Potentially destructive command"}
```

**作用**：检测可能造成数据丢失或安全风险的命令。

**检测关键词：**

| 关键词 | 风险原因 |
|--------|---------|
| `rm ` | 删除文件 |
| `> /etc/` | 覆盖系统配置文件 |
| `chmod 777` | 设置全局可读写权限 |
| `del` | Windows 删除命令 |

### 4. Gate 3: 用户审批

```python
def ask_user(tool_name: str, args: dict, reason: str) -> str:
    print(f"\n\033[33m⚠  {reason}\033[0m")
    print(f"   Tool: {tool_name}({args})")
    choice = input("   Allow? [y/N] ").strip().lower()
    return "allow" if choice in ("y", "yes") else "deny"
```

**交互流程：**

1. **显示警告**：用黄色高亮显示风险原因
2. **展示详情**：显示要执行的工具名称和参数
3. **等待输入**：用户输入 `y`/`yes` 允许，其他任何输入都视为拒绝
4. **返回结果**：返回 `"allow"` 或 `"deny"`

**设计原则：**

- **默认拒绝**：用户按回车或输入其他内容都视为拒绝，安全性优先
- **清晰提示**：明确告知用户风险是什么，以及将要执行什么操作
- **简洁交互**：只需输入 `y` 或 `n`，操作简单

### 5. 权限管道整合

```python
def check_permission(block) -> bool:
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            print(f"\n\033[31m⛔ {reason}\033[0m")
            return False
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False
    return True
```

**执行顺序：**

```
check_permission(block)
        │
        ├─► block.name == "bash" ?
        │       │
        │       └─► check_deny_list(command)
        │               │
        │               ├─► 匹配拒绝列表 ──► 打印红色警告 ──► return False
        │               │
        │               └─► 未匹配 ──► 继续
        │
        ├─► check_rules(tool_name, args)
        │       │
        │       ├─► 匹配规则 ──► ask_user()
        │       │                       │
        │       │                       ├─► 用户拒绝 ──► return False
        │       │                       │
        │       │                       └─► 用户允许 ──► 继续
        │       │
        │       └─► 未匹配规则 ──► 继续
        │
        └─► return True
```

### 6. 代理循环集成

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
            if block.type != "tool_use":
                continue

            print(f"\033[36m> {block.name}\033[0m")
            if block.name == "bash":
                    print(f"\033[33m$ {block.input['command']}\033[0m")
            # s03 change: run through permission pipeline before executing
            if not check_permission(block):
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": "Permission denied."})
                continue
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

**与 ch02 的差异：**

在工具执行前插入了权限检查：

```python
# ch02: 直接执行
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)

# ch03: 先检查权限再执行
if not check_permission(block):
    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": "Permission denied."})
    continue
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input)
```

---

## 扩展思考

### 权限系统设计原则

1. **分层防御**：多层安全措施，即使某一层失效，其他层仍能提供保护
2. **最小权限**：只允许必要的操作，拒绝一切不必要的风险
3. **透明性**：明确告知用户为什么操作被阻止或需要确认
4. **用户控制权**：最终决策权交给用户，系统只提供建议

### 可能的改进方向

1. **动态权限调整**：
   - 根据用户信任等级调整权限严格程度
   - 允许用户自定义拒绝列表和规则
   - 实现权限白名单机制

2. **更智能的规则引擎**：
   - 使用正则表达式进行更精确的命令匹配
   - 添加命令上下文分析（如 `rm` 是否带 `-rf` 参数）
   - 实现命令历史分析，检测异常模式

3. **审计日志**：
   - 记录所有权限检查结果
   - 保存被阻止的命令记录
   - 生成安全报告

4. **批量操作支持**：
   - 对一系列操作进行一次性审批
   - 支持超时自动拒绝
   - 实现操作预览功能

---

## 实践练习

尝试使用以下 prompt 测试权限系统：

1. `删除当前目录下所有 .txt 文件` — 应触发 Gate 2（潜在破坏性命令）
2. `创建文件到 C:\Windows\test.txt` — 应触发 Gate 2（写入工作区外）
3. `sudo rm -rf /` — 应触发 Gate 1（硬拒绝列表）
4. `读取 ch01/README.md` — 应直接通过所有检查