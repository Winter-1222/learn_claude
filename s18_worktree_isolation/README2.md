# s18: Worktree Isolation — 深入理解文件级隔离机制

## 概述

s18 引入了 **git worktree** 实现多 Agent 之间的文件级隔离。每个 Agent 可以在独立的工作目录中执行任务，互不干扰。这是解决多 Agent 协作时文件冲突问题的核心机制。

## 什么是 Git Worktree？

### Git Worktree vs Git Branch

| 维度 | Git Branch | Git Worktree |
|------|-----------|--------------|
| **工作目录** | 共享同一个工作目录 | **每个 worktree 有独立的工作目录** |
| **分支切换** | 会改变工作目录内容 | **不影响其他 worktree** |
| **文件系统** | 共享所有文件 | **完全隔离** |
| **仓库数据** | 共享 `.git` | **共享 `.git`** |
| **适用场景** | 单任务开发 | **多任务并行开发** |

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Git Worktree 架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Main Repository (/)                                            │
│  ├── .git/               ← 所有 worktree 共享的仓库数据          │
│  ├── .tasks/             ← 任务定义（共享）                      │
│  ├── .worktrees/         ← worktree 根目录                      │
│  │   ├── auth/           ← worktree: auth                       │
│  │   │   ├── .git/       ← 指向主仓库的 git 链接                  │
│  │   │   ├── src/        ← 独立的源代码目录                      │
│  │   │   └── ...                                               │
│  │   │                                                          │
│  │   ├── ui/             ← worktree: ui                         │
│  │   │   ├── .git/                                              │
│  │   │   ├── src/                                               │
│  │   │   └── ...                                               │
│  │   │                                                          │
│  │   └── events.jsonl    ← 事件日志（共享）                      │
│  └── ...                                                       │
│                                                                 │
│  Branch 关系:                                                   │
│    main ← HEAD                                                  │
│    wt/auth  ← .worktrees/auth 的分支                            │
│    wt/ui    ← .worktrees/ui 的分支                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 核心命令

```bash
# 创建 worktree（当前代码实现）
git worktree add .worktrees/{name} -b wt/{name} HEAD

# 删除 worktree
git worktree remove .worktrees/{name} --force

# 删除关联分支
git branch -D wt/{name}
```

## Task-Worktree 绑定机制

### 完整生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                  Task-Worktree 绑定生命周期                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. create_worktree(name, task_id)                              │
│     ├── validate_worktree_name(name)                           │
│     ├── git worktree add .worktrees/{name} -b wt/{name}        │
│     └── bind_task_to_worktree(task_id, name)  ← 写入 task.worktree│
│                                                                 │
│  2. spawn_teammate(name, role, prompt)                          │
│     └── 创建线程，初始化 wt_ctx = {"path": None}                │
│                                                                 │
│  3. teammate: claim_task(task_id)                               │
│     ├── 成功后读取 task.worktree                                │
│     └── wt_ctx["path"] = ".worktrees/{name}"  ← 切换工作目录     │
│                                                                 │
│  4. teammate: 执行工具操作                                       │
│     ├── _run_bash(command) → run_bash(command, cwd=_wt_cwd())   │
│     ├── _run_read(path)     → run_read(path, cwd=_wt_cwd())     │
│     └── _run_write(path, content) → run_write(path, content, cwd=_wt_cwd())│
│                                                                 │
│  5. teammate: complete_task(task_id)                            │
│     └── wt_ctx["path"] = None  ← 清空工作目录                    │
│                                                                 │
│  6. lead: remove_worktree(name) 或 keep_worktree(name)          │
│     ├── remove: git worktree remove + branch -D                 │
│     └── keep: 保留分支供人工审查                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### wt_ctx 闭包机制

这是实现隔离的核心代码路径：

```python
def run():
    wt_ctx = {"path": None}  # 闭包变量，追踪当前 worktree 路径

    def _wt_cwd() -> Path | None:
        p = wt_ctx["path"]
        return Path(p) if p else None

    def _run_bash(command: str) -> str:
        return run_bash(command, cwd=_wt_cwd())  # 动态获取 cwd

    def _run_read(path: str) -> str:
        return run_read(path, cwd=_wt_cwd())

    def _run_write(path: str, content: str) -> str:
        return run_write(path, content, cwd=_wt_cwd())
```

**工作原理**：
1. `wt_ctx` 是一个字典，作为闭包变量存在于 teammate 线程中
2. `_wt_cwd()` 函数动态返回当前 worktree 路径
3. 所有工具操作（bash、read、write）都通过 `_wt_cwd()` 获取工作目录
4. 当 teammate claim 一个绑定了 worktree 的任务时，`wt_ctx["path"]` 被设置为 worktree 路径
5. 当 task 完成时，`wt_ctx["path"]` 被清空

### claim_task 时的工作目录切换

```python
def _run_claim_task(task_id: str):
    result = claim_task(task_id, owner=name)
    if "Claimed" in result:
        task = load_task(task_id)
        if task.worktree:
            wt_ctx["path"] = str(WORKTREES_DIR / task.worktree)
        else:
            wt_ctx["path"] = None
    return result
```

**关键逻辑**：只有当 claim 成功且任务绑定了 worktree 时，才切换工作目录。

## Worktree 管理

### 创建工作树

```python
def create_worktree(name: str, task_id: str = "") -> str:
    err = validate_worktree_name(name)
    if err:
        return f"Error: {err}"
    path = WORKTREES_DIR / name
    if path.exists():
        return f"Worktree '{name}' already exists at {path}"
    ok, result = run_git(["worktree", "add", str(path), "-b", f"wt/{name}", "HEAD"])
    if not ok:
        return f"Git error: {result}"
    if task_id:
        bind_task_to_worktree(task_id, name)
    log_event("create", name, task_id)
    return f"Worktree '{name}' created at {path}"
```

**流程**：验证名称 → 检查是否已存在 → git worktree add → 绑定任务（可选）→ 记录事件

### 删除工作树

```python
def remove_worktree(name: str, discard_changes: bool = False) -> str:
    err = validate_worktree_name(name)
    if err:
        return err
    path = WORKTREES_DIR / name
    if not path.exists():
        return f"Worktree '{name}' not found"
    if not discard_changes:
        files, commits = _count_worktree_changes(path)
        if files > 0 or commits > 0:
            return (f"Worktree '{name}' has {files} uncommitted file(s) "
                    f"and {commits} unpushed commit(s). "
                    "Use discard_changes=true to force removal.")
    ok1, _ = run_git(["worktree", "remove", str(path), "--force"])
    run_git(["branch", "-D", f"wt/{name}"])
    log_event("remove", name)
    return f"Worktree '{name}' removed"
```

**三种终结策略**：

| 策略 | 操作 | 适用场景 |
|------|------|---------|
| `remove_worktree(name)` | 安全删除，有未提交变更则拒绝 | 正常完成的任务 |
| `remove_worktree(name, discard_changes=true)` | 强制删除，丢弃所有变更 | 确认不需要的任务 |
| `keep_worktree(name)` | 保留分支供人工审查 | 需要审查的任务 |

### 保留工作树

```python
def keep_worktree(name: str) -> str:
    err = validate_worktree_name(name)
    if err:
        return err
    log_event("keep", name)
    return f"Worktree '{name}' kept for review (branch: wt/{name})"
```

保留 worktree 不会删除文件和分支，供后续人工审查。

## 安全机制

### 工作树名称验证

```python
VALID_WT_NAME = re.compile(r'^[A-Za-z0-9._-]{1,64}$')

def validate_worktree_name(name: str) -> str | None:
    if not name:
        return "Worktree name cannot be empty"
    if name == "." or name == "..":
        return f"'{name}' is not a valid worktree name"
    if not VALID_WT_NAME.match(name):
        return (f"Invalid worktree name '{name}': "
                "only letters, digits, dots, underscores, dashes (1-64 chars)")
    return None
```

**安全保障**：
- 拒绝空名称
- 拒绝 `.` 和 `..`（防止路径遍历攻击）
- 只允许字母、数字、点、下划线、破折号
- 长度限制 1-64 字符

### 变更计数检查

```python
def _count_worktree_changes(path: Path) -> tuple[int, int]:
    r1 = subprocess.run(["git", "status", "--porcelain"],
                        cwd=path, capture_output=True, text=True, timeout=10)
    files = len([l for l in r1.stdout.strip().splitlines() if l.strip()])
    
    r2 = subprocess.run(["git", "log", "@{push}..HEAD", "--oneline"],
                        cwd=path, capture_output=True, text=True, timeout=10)
    commits = len([l for l in r2.stdout.strip().splitlines() if l.strip()])
    
    return files, commits
```

**检查内容**：
- `files`：未提交的文件数量
- `commits`：未推送的提交数量

**保护机制**：如果两者都大于 0，remove_worktree 会拒绝删除，除非显式指定 `discard_changes=true`。

### 路径安全

```python
def safe_path(p: str, cwd: Path = None) -> Path:
    base = cwd or WORKDIR
    path = (base / p).resolve()
    if not path.is_relative_to(base):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

所有文件操作都通过 `safe_path` 验证，确保不会访问工作目录之外的文件。

## 事件日志

```python
def log_event(event_type: str, worktree_name: str, task_id: str = ""):
    event = {"type": event_type, "worktree": worktree_name,
             "task_id": task_id, "ts": time.time()}
    events_file = WORKTREES_DIR / "events.jsonl"
    with open(events_file, "a") as f:
        f.write(json.dumps(event) + "\n")
```

**事件类型**：

| 类型 | 含义 |
|------|------|
| `create` | 创建 worktree |
| `remove` | 删除 worktree |
| `keep` | 保留 worktree |

**用途**：提供审计追踪，便于排查问题和分析工作流。

## 新增工具

### Lead 工具（3个）

| 工具 | 功能 | 参数 |
|------|------|------|
| `create_worktree` | 创建 worktree 并可选绑定任务 | `name`(必填)、`task_id`(可选) |
| `remove_worktree` | 删除 worktree（安全检查） | `name`(必填)、`discard_changes`(可选) |
| `keep_worktree` | 保留 worktree 供审查 | `name`(必填) |

### Teammate 工具增强

Teammate 的工具操作会自动使用 worktree 作为工作目录：

```python
# teammate 内部的工具定义
sub_tools = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "list_tasks", ...},
    {"name": "claim_task", ...},
    {"name": "complete_task", ...},
]
```

当 teammate claim 一个绑定了 worktree 的任务后，所有 bash、read_file、write_file 操作都会在该 worktree 目录下执行。

## 完整工作流示例

### 场景：并行开发两个功能

```
1. Lead 创建任务和工作树
   ├── create_task("实现用户认证模块", id=task_001)
   ├── create_worktree("auth", task_001)
   ├── create_task("实现 UI 界面", id=task_002)
   └── create_worktree("ui", task_002)

2. Lead 派遣队友
   ├── spawn_teammate("alice", "backend developer", "实现用户认证")
   └── spawn_teammate("bob", "frontend developer", "实现 UI 界面")

3. Teammate 认领任务
   ├── alice: claim_task(task_001) → wt_ctx["path"] = ".worktrees/auth"
   └── bob: claim_task(task_002) → wt_ctx["path"] = ".worktrees/ui"

4. Teammate 并行工作（完全隔离）
   ├── alice 在 .worktrees/auth 中开发认证模块
   └── bob 在 .worktrees/ui 中开发 UI 界面

5. Teammate 完成任务
   ├── alice: complete_task(task_001) → wt_ctx["path"] = None
   └── bob: complete_task(task_002) → wt_ctx["path"] = None

6. Lead 清理工作树
   ├── remove_worktree("auth")
   └── remove_worktree("ui")
```

## 与前序章节的关系

| 章节 | 核心功能 | 与 s18 的关系 |
|------|---------|---------------|
| s12 | 文件持久化任务系统 | Task dataclass 增加了 `worktree` 字段 |
| s15 | MessageBus 消息总线 | 用于 teammate 和 lead 之间的通信 |
| s16 | 协议状态管理 | 用于 shutdown 和 plan_approval 协议 |
| s17 | 自主 Agent 循环 | Teammate 继承了 idle_poll 和任务扫描 |

## 关键设计决策

### 为什么用 git worktree 而非普通目录？

| 原因 | 说明 |
|------|------|
| **版本控制** | 每个 worktree 有独立分支，可以单独提交和推送 |
| **代码共享** | 共享 `.git` 仓库，避免重复存储 |
| **分支隔离** | 每个 worktree 可以 checkout 不同分支 |
| **协作友好** | 完成后可以合并回主分支 |

### 为什么 wt_ctx 要用字典而不是直接变量？

```python
# 正确：字典是可变对象，可以在闭包中修改
wt_ctx = {"path": None}
wt_ctx["path"] = new_path

# 错误：普通变量不可变，闭包中无法重新绑定
wt_path = None
wt_path = new_path  # 这会创建新的局部变量
```

这是 Python 闭包的一个常见陷阱。使用字典可以绕过这个限制。

### 为什么 task.worktree 在 bind 时不自动 claim？

```python
def bind_task_to_worktree(task_id: str, worktree_name: str):
    task = load_task(task_id)
    task.worktree = worktree_name
    save_task(task)
    # 注意：这里没有改变 task.status
```

**设计原因**：
- 创建 worktree 和 claim task 是两个独立操作
- 可能由不同 Agent 执行（lead 创建，teammate claim）
- 保持状态一致性，避免意外的状态变化

## 实践练习

### 练习1：体验 worktree 隔离

1. 运行 `python s18_worktree_isolation/code.py`
2. 创建两个任务和对应的 worktree：
   - `create_task("开发 API", id=task_api)`
   - `create_worktree("api", task_api)`
   - `create_task("开发测试", id=task_test)`
   - `create_worktree("test", task_test)`
3. 派遣两个队友分别认领任务
4. 观察队友是否在各自的 worktree 目录下工作

### 练习2：测试安全检查

1. 创建 worktree 并写入文件
2. 不提交更改，尝试删除 worktree
3. 观察是否被拒绝
4. 使用 `discard_changes=true` 强制删除

### 练习3：理解 wt_ctx 闭包

在 teammate 的 `run()` 函数中添加调试打印：

```python
def _run_bash(command: str) -> str:
    print(f"  [debug] bash in cwd: {_wt_cwd()}")
    return run_bash(command, cwd=_wt_cwd())
```

观察队友 claim 任务前后的工作目录变化。

### 练习4：测试事件日志

1. 创建、删除、保留 worktree
2. 查看 `.worktrees/events.jsonl` 文件
3. 验证事件是否正确记录

### 练习5：手动操作 worktree

1. 查看 worktree 列表：`git worktree list`
2. 进入 worktree：`cd .worktrees/auth`
3. 查看分支：`git branch`
4. 删除 worktree：`git worktree remove .worktrees/auth`
