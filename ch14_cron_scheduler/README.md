# Ch14: Cron Scheduler — 独立守护线程与队列处理器

## 概述

Ch14 引入了**定时任务调度系统**，通过独立的守护线程实现基于 cron 表达式的任务调度。核心设计采用四层架构，解耦调度器、队列、处理器和消费者，支持持久化存储和多线程安全。

### 四层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      四层 Cron 调度架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: Scheduler                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  cron_scheduler_loop (守护线程)                           │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 每秒轮询时间                                        │   │  │
│  │  │ 遍历 scheduled_jobs                                │   │  │
│  │  │ cron_matches(cron_expr, now) → 匹配则写入 cron_queue│   │  │
│  │  │ _last_fired 防止同一分钟重复触发                    │   │  │
│  │  └───────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  Layer 2: Queue                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  cron_queue: list[CronJob]  (线程安全队列)                │  │
│  │  cron_lock: threading.Lock                              │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ scheduler 写入 → queue_processor 读取 → 清空        │   │  │
│  │  └───────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  Layer 3: Queue Processor                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  queue_processor_loop (守护线程)                         │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 0.2s 轮询 has_cron_queue()                        │   │  │
│  │  │ agent_lock.acquire(blocking=False) 非阻塞获取      │   │  │
│  │  │ 获取成功 → run_agent_turn_locked()                 │   │  │
│  │  │ 获取失败 → 等待下次轮询                             │   │  │
│  │  └───────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  Layer 4: Consumer                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  agent_loop 中 consume_cron_queue()                     │  │
│  │  ┌───────────────────────────────────────────────────┐   │  │
│  │  │ 消费 cron_queue 中的任务                           │   │  │
│  │  │ 注入为 user message: "[Scheduled] {prompt}"       │   │  │
│  │  │ LLM 处理并执行                                     │   │  │
│  │  └───────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## CronJob 数据模型

### CronJob Dataclass

```python
@dataclass
class CronJob:
    id: str        # 唯一标识符
    cron: str      # 5字段 cron 表达式，如 "0 9 * * *"
    prompt: str    # 触发时注入的消息
    recurring: bool  # True = 重复执行，False = 一次性
    durable: bool    # True = 持久化到磁盘，False = 会话级
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | str | `cron_{6位随机数}`，保证唯一性 |
| `cron` | str | 标准 5 字段 cron 表达式 |
| `prompt` | str | 触发时作为用户消息注入 |
| `recurring` | bool | 是否重复执行 |
| `durable` | bool | 是否持久化到 `.scheduled_tasks.json` |

### 持久化存储

```python
DURABLE_PATH = WORKDIR / ".scheduled_tasks.json"

def save_durable_jobs():
    durable = [asdict(j) for j in scheduled_jobs.values() if j.durable]
    DURABLE_PATH.write_text(json.dumps(durable, indent=2))

def load_durable_jobs():
    if not DURABLE_PATH.exists():
        return
    jobs = json.loads(DURABLE_PATH.read_text())
    for j in jobs:
        job = CronJob(**j)
        err = validate_cron(job.cron)
        if err:
            print(f"[cron] skipping invalid job {job.id}: {err}")
            continue
        scheduled_jobs[job.id] = job
```

**设计特点**：
- 启动时自动加载持久化任务
- 验证失败的任务跳过但不阻断加载
- `schedule_job` / `cancel_job` 时自动保存

## Cron 表达式匹配

### 标准 5 字段格式

```
┌──────────────────────────────────────────────────────────────┐
│                    Cron 表达式格式                          │
├──────────────────────────────────────────────────────────────┤
│                                                            │
│  格式：minute hour day-of-month month day-of-week          │
│                                                            │
│  示例："0 9 * * *"  → 每天早上 9:00                        │
│        "0 */2 * * *" → 每 2 小时                          │
│        "0 9 * * 1-5" → 工作日早上 9:00                    │
│                                                            │
│  字段范围：                                                 │
│  ┌─────────┬─────────┬──────────────┬───────┬────────────┐ │
│  │ minute  │ hour    │ day-of-month │ month │ day-of-week│ │
│  │ 0-59    │ 0-23    │ 1-31         │ 1-12  │ 0-6        │ │
│  │         │         │              │       │ (0=Sunday) │ │
│  └─────────┴─────────┴──────────────┴───────┴────────────┘ │
│                                                            │
└──────────────────────────────────────────────────────────────┘
```

### cron_matches — DOM/DOW OR 语义

```python
def cron_matches(cron_expr: str, dt: datetime) -> bool:
    minute, hour, dom, month, dow = fields
    
    m = _cron_field_matches(minute, dt.minute)
    h = _cron_field_matches(hour, dt.hour)
    dom_ok = _cron_field_matches(dom, dt.day)
    month_ok = _cron_field_matches(month, dt.month)
    dow_ok = _cron_field_matches(dow, dow_val)
    
    if not (m and h and month_ok):
        return False
    
    # DOM 和 DOW 使用 OR 语义
    dom_unconstrained = dom == "*"
    dow_unconstrained = dow == "*"
    if dom_unconstrained and dow_unconstrained:
        return True
    if dom_unconstrained:
        return dow_ok
    if dow_unconstrained:
        return dom_ok
    return dom_ok or dow_ok  # 两者都约束时，任一匹配即可
```

**DOM/DOW OR 语义示例**：

| 表达式 | 含义 |
|--------|------|
| `"0 9 * * 1-5"` | 工作日（周一到周五）早上 9:00 |
| `"0 9 1-5 * *"` | 每月 1-5 号早上 9:00 |
| `"0 9 1-5 * 1-5"` | 每月 1-5 号**或**工作日早上 9:00（任一条件满足即可） |

### _last_fired — 分钟级去重

```python
_last_fired: dict[str, str] = {}  # job_id → "YYYY-MM-DD HH:MM"

def cron_scheduler_loop():
    while True:
        time.sleep(1)
        now = datetime.now()
        minute_marker = now.strftime("%Y-%m-%d %H:%M")
        
        with cron_lock:
            for job in list(scheduled_jobs.values()):
                if cron_matches(job.cron, now):
                    if _last_fired.get(job.id) != minute_marker:
                        cron_queue.append(job)
                        _last_fired[job.id] = minute_marker
```

**设计原因**：
- scheduler 每秒轮询一次
- 同一个 cron 表达式可能在同一分钟内多次匹配
- 使用 `"YYYY-MM-DD HH:MM"` 作为标记，确保每分钟只触发一次

### _cron_field_matches — 字段匹配规则

```python
def _cron_field_matches(field: str, value: int) -> bool:
    if field == "*":              return True          # 通配符
    if field.startswith("*/"):    return value % step == 0  # 步长
    if "," in field:              return any(...)     # 枚举
    if "-" in field:              return lo <= value <= hi  # 范围
    return value == int(field)    # 精确匹配
```

**支持的表达式**：

| 格式 | 示例 | 含义 |
|------|------|------|
| `*` | `*` | 任意值 |
| `*/N` | `*/2` | 每 N 个单位 |
| `N` | `5` | 精确值 |
| `N-M` | `1-5` | 范围 |
| `N,M,K` | `1,3,5` | 枚举 |

## 线程安全设计

### 两把锁的职责

```python
cron_lock = threading.Lock()    # 保护 cron 相关共享状态
agent_lock = threading.Lock()   # 保护 agent turn 执行
```

| 锁 | 保护的资源 | 使用场景 |
|----|-----------|---------|
| `cron_lock` | `scheduled_jobs`、`cron_queue`、`_last_fired` | scheduler 写入、queue_processor 读取、schedule_job/cancel_job |
| `agent_lock` | `session_history`、`session_context` | 用户输入、queue_processor 自动执行 |

### queue_processor 的非阻塞获取

```python
def queue_processor_loop():
    while True:
        time.sleep(0.2)
        if not has_cron_queue():
            continue
        # 非阻塞获取，避免与用户输入冲突
        if not agent_lock.acquire(blocking=False):
            continue
        try:
            if not has_cron_queue():
                continue
            run_agent_turn_locked()
        finally:
            agent_lock.release()
```

**设计要点**：
- 使用 `acquire(blocking=False)` 非阻塞获取
- 如果 agent 正在处理用户输入，自动跳过等待下次轮询
- 双重检查 `has_cron_queue()`：获取锁后再次确认
- `finally` 确保锁总是被释放

## scheduler_loop — 独立守护线程

```python
def cron_scheduler_loop():
    while True:
        time.sleep(1)
        now = datetime.now()
        minute_marker = now.strftime("%Y-%m-%d %H:%M")
        
        with cron_lock:
            for job in list(scheduled_jobs.values()):
                try:
                    if cron_matches(job.cron, now):
                        if _last_fired.get(job.id) != minute_marker:
                            cron_queue.append(job)
                            _last_fired[job.id] = minute_marker
                        if not job.recurring:
                            scheduled_jobs.pop(job.id, None)
                            if job.durable:
                                save_durable_jobs()
                except Exception as e:
                    print(f"[cron error] {job.id}: {e}")
```

**关键设计**：

1. **Error Isolation**：单个 job 异常被捕获，不影响其他 job 和调度线程
2. **Recurring 管理**：一次性任务触发后自动从 `scheduled_jobs` 移除
3. **Durable 同步**：移除持久化任务时自动保存

## agent_loop 中的集成

### Layer 4: 消费队列

```python
def agent_loop(messages: list, context: dict) -> dict:
    while True:
        # Layer 4: 消费触发的 cron 任务
        fired = consume_cron_queue()
        for job in fired:
            messages.append({"role": "user",
                             "content": f"[Scheduled] {job.prompt}"})
            print(f"[inject cron] {job.prompt[:50]}")
        
        # LLM 调用...
```

**注入格式**：`[Scheduled] {job.prompt}`

### 用户输入的锁保护

```python
if __name__ == "__main__":
    threading.Thread(target=queue_processor_loop, daemon=True).start()
    
    while True:
        query = input("s14 >> ")
        with agent_lock:
            run_agent_turn_locked(query)
```

用户输入和 queue_processor 通过 `agent_lock` 互斥，确保同一时间只有一个 agent turn 在执行。

## 新增工具

| 工具 | 功能 | 参数 |
|------|------|------|
| `schedule_cron` | 调度定时任务 | `cron`(必填)、`prompt`(必填)、`recurring`、`durable` |
| `list_crons` | 列出所有定时任务 | 无 |
| `cancel_cron` | 取消定时任务 | `job_id`(必填) |

### schedule_cron 工具定义

```python
{"name": "schedule_cron",
 "description": "Schedule a cron job. cron is 5-field: min hour dom month dow.",
 "input_schema": {"type": "object",
                  "properties": {
                      "cron": {"type": "string",
                               "description": "5-field cron expression"},
                      "prompt": {"type": "string",
                                 "description": "Message to inject when fired"},
                      "recurring": {"type": "boolean",
                                    "description": "True=recurring, False=one-shot"},
                      "durable": {"type": "boolean",
                                  "description": "True=persist to disk"}},
                  "required": ["cron", "prompt"]}}
```

## 与前序章节的关系

| 章节 | 核心功能 | 与 ch14 的关系 |
|------|---------|---------------|
| ch13 | 后台任务系统 | ch14 继承了后台任务执行机制 |
| ch12 | 文件持久化任务系统 | ch14 继承了任务系统 |
| ch10 | 动态 System Prompt | ch14 继承了 prompt 组装机制 |

### 启动流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        启动流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. load_durable_jobs()  ← 加载持久化定时任务                    │
│  2. cron_scheduler_loop() 启动守护线程                          │
│  3. queue_processor_loop() 启动守护线程（main 中）               │
│  4. 用户输入循环等待                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 实践练习

### 练习1：创建定时任务

1. 运行 `python -m ch14_cron_scheduler.code`
2. 输入："创建一个定时任务，每分钟触发一次，消息内容是 '检查系统状态'"
3. 使用 `list_crons` 查看任务列表
4. 等待一分钟，观察 `[cron fire]` 和 `[inject cron]` 日志
5. 使用 `cancel_cron` 取消任务

### 练习2：测试持久化

1. 创建一个持久化定时任务
2. 退出程序
3. 查看 `.scheduled_tasks.json` 文件内容
4. 重新运行程序，使用 `list_crons` 验证任务是否自动加载

### 练习3：理解 DOM/DOW OR 语义

1. 创建任务：`schedule_cron("0 * * * 1-5", "工作日提醒")`
2. 创建任务：`schedule_cron("0 * 1-5 * *", "日期提醒")`
3. 创建任务：`schedule_cron("0 * 1-5 * 1-5", "组合提醒")`
4. 在周末测试这些任务，观察触发行为

### 练习4：测试一次性任务

1. 创建一次性任务：`schedule_cron("* * * * *", "一次性任务", recurring=false)`
2. 等待触发，观察是否只触发一次
3. 使用 `list_crons` 验证任务是否被自动移除

### 练习5：测试队列处理器

1. 创建一个定时任务，设置为当前时间的下一分钟触发
2. 在等待期间输入其他问题，验证用户输入和定时任务不会冲突
3. 观察 `[queue processor] delivering scheduled work` 日志
