# Ch12: Task System — 文件持久化任务依赖图

## 概述

Ch12 引入了**文件持久化的任务系统**，支持任务创建、认领、完成以及基于 `blockedBy` 的依赖管理。与 ch05 的内存中任务列表不同，ch12 的任务系统支持跨会话持久化和复杂的依赖关系。

### 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Task System 架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   创建任务    │    │    认领任务   │    │   完成任务    │      │
│  │ create_task  │    │ claim_task   │    │complete_task │      │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    .tasks/ 目录                          │   │
│  │  task_1700000000_1234.json  ← 每个任务一个文件           │   │
│  │  task_1700000001_5678.json  ← JSON 格式持久化           │   │
│  │  task_1700000002_9012.json  ← 支持跨会话保存             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   依赖关系图 (DAG)                        │   │
│  │                                                          │   │
│  │    task_A (completed)                                    │   │
│  │         │                                                │   │
│  │         ▼                                                │   │
│  │    task_B (in_progress)  ── blockedBy ──→ task_A        │   │
│  │         │                                                │   │
│  │         ▼                                                │   │
│  │    task_C (pending)      ── blockedBy ──→ task_B        │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Task 数据模型

### Task Dataclass

```python
@dataclass
class Task:
    id: str                  # 唯一标识符
    subject: str             # 任务主题（简短描述）
    description: str         # 详细描述
    status: str              # pending | in_progress | completed
    owner: str | None        # 任务所有者（多 agent 场景）
    blockedBy: list[str]     # 依赖的任务 ID 列表
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | str | `task_{时间戳}_{4位随机数}`，保证唯一性 |
| `subject` | str | 简短主题，如 "设计数据库架构" |
| `description` | str | 可选的详细描述 |
| `status` | str | 三态：`pending`、`in_progress`、`completed` |
| `owner` | str\|None | 任务所有者名称，预留多 agent 协作 |
| `blockedBy` | list[str] | 依赖的任务 ID 列表，构成 DAG |

### 任务 ID 生成策略

```python
id=f"task_{int(time.time())}_{random.randint(0, 9999):04d}"
```

**设计原因**：
- 时间戳确保大致的创建顺序
- 4 位随机数避免同一秒内创建的任务 ID 冲突
- 固定格式便于文件系统存储和检索

## 任务生命周期

### 状态转换图

```
                    ┌─────────────────────────────────────────┐
                    │              Task 生命周期               │
                    ├─────────────────────────────────────────┤
                    │                                         │
                    │      ┌───────────┐                      │
                    │      │  pending   │                      │
                    │      └─────┬─────┘                      │
                    │            │                             │
                    │      can_start?                         │
                    │            │                             │
                    │    ┌───────┴───────┐                    │
                    │    │ Yes           │ No                  │
                    │    ▼               ▼                    │
                    │  claim_task   保持 pending              │
                    │    │               (阻塞中)              │
                    │    ▼                                    │
                    │  ┌───────────┐                          │
                    │  │in_progress│                          │
                    │  └─────┬─────┘                          │
                    │        │                                │
                    │  complete_task                          │
                    │        │                                │
                    │        ▼                                │
                    │  ┌───────────┐                          │
                    │  │ completed │                          │
                    │  └───────────┘                          │
                    │        │                                │
                    │        ▼                                │
                    │  检查下游任务并报告解锁                   │
                    │                                         │
                    └─────────────────────────────────────────┘
```

### 状态转换规则

| 转换 | 条件 | 函数 |
|------|------|------|
| `pending` → `in_progress` | 所有 `blockedBy` 任务已完成 | `claim_task()` |
| `in_progress` → `completed` | 当前任务执行完毕 | `complete_task()` |
| `completed` | 终态，不再变化 | - |

## 依赖管理

### can_start — 依赖检查

```python
def can_start(task_id: str) -> bool:
    task = load_task(task_id)
    for dep_id in task.blockedBy:
        if not _task_path(dep_id).exists():
            return False  # 缺失的依赖视为 blocked
        if load_task(dep_id).status != "completed":
            return False
    return True
```

**防御性设计**：缺失的依赖 ID 视为阻塞状态，而非忽略。这避免了因任务删除或 ID 错误导致的意外执行。

### complete_task — 解锁下游任务

```python
def complete_task(task_id: str) -> str:
    task = load_task(task_id)
    task.status = "completed"
    save_task(task)
    
    # 查找所有被当前任务解锁的下游任务
    unblocked = [t.subject for t in list_tasks()
                 if t.status == "pending" and t.blockedBy and can_start(t.id)]
    
    msg = f"Completed {task.id} ({task.subject})"
    if unblocked:
        msg += f"\nUnblocked: {', '.join(unblocked)}"
    return msg
```

**工作原理**：
1. 将当前任务标记为 `completed`
2. 遍历所有任务，查找 `pending` 状态且 `blockedBy` 非空的任务
3. 对每个符合条件的任务调用 `can_start()` 检查是否已解锁
4. 报告所有被解锁的下游任务

### 依赖图示例

```
任务创建顺序：
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  create_task("设计数据库", blockedBy=[])         → task_A    │
│  create_task("编写 API", blockedBy=["task_A"])   → task_B    │
│  create_task("编写前端", blockedBy=["task_B"])   → task_C    │
│                                                              │
│  依赖链：task_A → task_B → task_C                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

任务完成顺序：
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  claim_task("task_A")    → task_A: in_progress               │
│  complete_task("task_A") → task_A: completed                 │
│                            Unblocked: 编写 API               │
│                                                              │
│  claim_task("task_B")    → task_B: in_progress               │
│  complete_task("task_B") → task_B: completed                 │
│                            Unblocked: 编写前端               │
│                                                              │
│  claim_task("task_C")    → task_C: in_progress               │
│  complete_task("task_C") → task_C: completed                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## 文件持久化

### 存储结构

```
.workdir/
└── .tasks/
    ├── task_1700000000_1234.json
    ├── task_1700000001_5678.json
    └── task_1700000002_9012.json
```

每个任务单独存储为 JSON 文件，便于独立读写和版本控制。

### 序列化与反序列化

```python
def save_task(task: Task):
    _task_path(task.id).write_text(json.dumps(asdict(task), indent=2))

def load_task(task_id: str) -> Task:
    return Task(**json.loads(_task_path(task_id).read_text()))
```

**使用 dataclass 的优势**：
- `asdict()` 自动转换为字典，无需手动编写序列化逻辑
- `**` 解包自动创建对象，无需手动编写反序列化逻辑
- 类型安全，字段类型在编译时检查

## 任务工具

### 新增工具列表

| 工具 | 功能 | 参数 |
|------|------|------|
| `create_task` | 创建新任务 | `subject`(必填)、`description`、`blockedBy` |
| `list_tasks` | 列出所有任务 | 无 |
| `get_task` | 获取任务详情 | `task_id` |
| `claim_task` | 认领任务 | `task_id` |
| `complete_task` | 完成任务 | `task_id` |

### list_tasks 输出格式

```
○ task_1700000000_1234: 设计数据库 [pending]
● task_1700000001_5678: 编写 API [in_progress] [agent]
✓ task_1700000002_9012: 编写前端 [completed] [agent] (blockedBy: task_1700000001_5678)
```

**图标说明**：
- `○`：pending（待处理）
- `●`：in_progress（进行中）
- `✓`：completed（已完成）

## 与 ch05 todo_write 的对比

| 维度 | ch05 todo_write | ch12 Task System |
|------|-----------------|------------------|
| **存储方式** | 内存中列表 | 文件系统持久化 |
| **生命周期** | 单次会话 | 跨会话 |
| **依赖关系** | 无 | 支持 `blockedBy` DAG |
| **状态管理** | pending/in_progress/completed | 相同三态 |
| **所有者** | 无 | 支持 owner 字段 |
| **完成机制** | 手动更新状态 | 自动检查依赖解锁 |
| **适用场景** | 简单任务规划 | 复杂项目管理 |

## 多 Agent 预留设计

`owner` 字段为多 Agent 协作场景预留：

```python
def claim_task(task_id: str, owner: str = "agent") -> str:
    task = load_task(task_id)
    task.owner = owner
    task.status = "in_progress"
    save_task(task)
```

当前默认 owner 为 `"agent"`，在多 Agent 场景中可以传递不同的 Agent 名称，实现任务分配和协作。

## 与 ch11 的关系

代码注释明确说明：

```python
"""
Note: Teaching code keeps a basic agent loop to stay focused on the task
system. S11's full error recovery (RecoveryState, backoff, escalation,
reactive compact, fallback model) is omitted — in real CC, tasks.ts and
withRetry are independent layers that compose naturally.
"""
```

**设计哲学**：任务系统和错误恢复是两个独立的层，可以自然组合。在真实的 Claude Code 中，这两个功能是分别实现的，可以按需组合使用。

## 实践练习

### 练习1：创建任务依赖链

1. 运行 `python -m ch12_task_system.code`
2. 输入："创建三个任务：设计数据库、编写 API（依赖设计数据库）、编写前端（依赖编写 API）"
3. 使用 `list_tasks` 查看任务状态
4. 尝试认领编写前端任务，观察是否被阻塞
5. 依次完成设计数据库、编写 API、编写前端，观察解锁机制

### 练习2：测试缺失依赖

1. 创建一个任务 `task_X`，设置 `blockedBy=["non_existent_task"]`
2. 尝试认领 `task_X`，观察错误信息
3. 创建 `non_existent_task` 并完成它
4. 再次尝试认领 `task_X`

### 练习3：理解文件持久化

1. 创建几个任务
2. 退出程序
3. 查看 `.tasks/` 目录下的 JSON 文件
4. 重新运行程序
5. 使用 `list_tasks` 验证任务是否持久化

### 练习4：扩展任务系统

尝试添加新功能：

```python
def delete_task(task_id: str) -> str:
    """删除任务（注意：可能导致依赖该任务的其他任务永远无法完成）"""
    path = _task_path(task_id)
    if not path.exists():
        return f"Error: Task {task_id} not found"
    path.unlink()
    return f"Deleted {task_id}"
```

添加对应的工具定义和 handler，测试删除功能。
