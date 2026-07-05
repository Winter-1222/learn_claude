# Ch11: Error Recovery — 三条恢复路径与指数退避

## 概述

Ch11 引入了**多层次错误恢复系统**，为 LLM 调用的各种异常情况提供自动恢复机制。核心设计包含三条恢复路径和一个统一的状态追踪器。

### 三条恢复路径

```
┌─────────────────────────────────────────────────────────────────────┐
│                        LLM 调用错误恢复流程                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌─────────────────┐                              │
│                    │   LLM Call      │                              │
│                    └────────┬────────┘                              │
│                             │                                       │
│              ┌──────────────┼──────────────┐                        │
│              │              │              │                        │
│         [成功]         [stop_reason]    [异常]                       │
│              │              │              │                        │
│              │         max_tokens?    ┌────┴────┐                   │
│              │              │         │ 429/529 │                   │
│              │              │         │ 其他异常 │                   │
│              │              ↓         └────┬────┘                   │
│              │    ┌─────────────────┐      │                        │
│              │    │   Path 1:       │      ↓                        │
│              │    │   Token 升级    │  ┌─────────────────┐          │
│              │    │   8K → 64K      │  │   Path 3:       │          │
│              │    │   续写提示(3次)  │  │   指数退避      │          │
│              │    └─────────────────┘  │   429: 简单重试   │          │
│              │                         │   529: 切换模型   │          │
│              │                         └────────┬────────┘          │
│              │                                  │                    │
│              ↓                                  ↓                    │
│        正常处理                        with_retry 内部循环             │
│        tool_use/完成                   非瞬态错误 re-raise             │
│                                             │                        │
│                                             ↓                        │
│                                    ┌─────────────────┐               │
│                                    │   Path 2:       │               │
│                                    │   Prompt Too    │               │
│                                    │   Long          │               │
│                                    │   紧急压缩(1次)  │               │
│                                    └─────────────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## RecoveryState 状态机

`RecoveryState` 类追踪整个循环中的恢复尝试状态：

```python
class RecoveryState:
    def __init__(self):
        self.has_escalated = False           # 是否已升级到 64K
        self.recovery_count = 0               # 续写重试次数
        self.consecutive_529 = 0              # 连续 529 错误计数
        self.has_attempted_reactive_compact = False  # 是否尝试过紧急压缩
        self.current_model = PRIMARY_MODEL    # 当前使用的模型
```

### 状态转换图

```
                    ┌─────────────────────────────────────────────────┐
                    │          RecoveryState 状态转换                  │
                    ├─────────────────────────────────────────────────┤
                    │                                                 │
                    │  has_escalated:                                  │
                    │    False ──[首次 max_tokens]──→ True (永久)      │
                    │                                                 │
                    │  recovery_count:                                 │
                    │    0 ──[64K仍截断]──→ 1 ──[仍截断]──→ 2 ──→ 3  │
                    │                                        │        │
                    │                                        ↓        │
                    │                                    [停止恢复]   │
                    │                                                 │
                    │  consecutive_529:                               │
                    │    0 ──[529]──→ 1 ──[529]──→ 2 ──[529]──→ 3   │
                    │                                        │        │
                    │                                        ↓        │
                    │                              [切换 fallback]     │
                    │                              consecutive_529=0   │
                    │                                                 │
                    │  has_attempted_reactive_compact:                │
                    │    False ──[prompt_too_long]──→ True (永久)     │
                    │                                                 │
                    │  current_model:                                 │
                    │    PRIMARY_MODEL ──[529x3]──→ FALLBACK_MODEL    │
                    │                                                 │
                    └─────────────────────────────────────────────────┘
```

## Path 1: max_tokens — Token 升级与续写

### 两步策略

当 LLM 响应因 `max_tokens` 被截断时，系统采用两步恢复策略：

```python
if response.stop_reason == "max_tokens":
    # 第一步：首次升级，不追加截断输出，直接用 64K 重试
    if not state.has_escalated:
        max_tokens = ESCALATED_MAX_TOKENS
        state.has_escalated = True
        continue
    
    # 第二步：64K 仍不够，追加截断输出 + 续写提示
    messages.append({"role": "assistant", "content": response.content})
    if state.recovery_count < MAX_RECOVERY_RETRIES:
        messages.append({"role": "user", "content": CONTINUATION_PROMPT})
        state.recovery_count += 1
        continue
```

### 设计原因

| 步骤 | 行为 | 原因 |
|------|------|------|
| 第一步 | 不追加输出，直接重试 | 8K 截断可能只是临时问题，64K 可能一次完成 |
| 第二步 | 追加输出 + 续写提示 | 64K 仍截断说明内容确实很长，需要继续生成 |
| 续写提示 | "Resume directly — no apology, no recap" | 避免重复，直接接在截断处继续 |

### CONTINUATION_PROMPT

```python
CONTINUATION_PROMPT = (
    "Output token limit hit. Resume directly — "
    "no apology, no recap. Pick up mid-thought."
)
```

**关键指令**：
- "Resume directly"：直接继续，不要道歉
- "no recap"：不要重复之前的内容
- "Pick up mid-thought"：从中断处继续思考

## Path 2: prompt_too_long — 紧急压缩

当 API 返回 prompt 过长错误时，触发紧急压缩：

```python
if is_prompt_too_long_error(e):
    if not state.has_attempted_reactive_compact:
        messages[:] = reactive_compact(messages)
        state.has_attempted_reactive_compact = True
        continue
    # 压缩后仍过长，放弃恢复
```

### is_prompt_too_long_error 判断逻辑

```python
def is_prompt_too_long_error(e: Exception) -> bool:
    msg = str(e).lower()
    return (("prompt" in msg and "long" in msg)
            or "prompt_is_too_long" in msg
            or "context_length_exceeded" in msg
            or "max_context_window" in msg)
```

覆盖多种错误消息格式，确保正确识别。

### reactive_compact 简化版

```python
def reactive_compact(messages: list) -> list:
    tail = messages[-5:]
    return [{"role": "user",
             "content": "[Reactive compact] Earlier conversation trimmed. "
                        "Continue from where you left off."}, *tail]
```

**与 ch08 的区别**：

| 维度 | ch08 reactive_compact | ch11 reactive_compact |
|------|----------------------|----------------------|
| **压缩方式** | LLM 生成摘要 | 简单保留尾部 5 条消息 |
| **复杂度** | 需要额外 API 调用 | 纯内存操作，零延迟 |
| **适用场景** | 教学版（展示 LLM 压缩） | 生产版（快速恢复） |
| **设计原因** | 展示上下文压缩技术 | ch08/ch09 已覆盖，此处简化 |

## Path 3: 429/529 — 指数退避

### with_retry 分层设计

`with_retry` 只处理瞬态错误（429/529），非瞬态错误 re-raise 给外层：

```python
def with_retry(fn, state: RecoveryState):
    for attempt in range(MAX_RETRIES):
        try:
            result = fn()
            state.consecutive_529 = 0  # 成功后重置
            return result
        except Exception as e:
            # 429/529 → 指数退避重试
            # 其他错误 → re-raise 给外层处理
            raise
```

**分层设计哲学**：

| 层级 | 处理范围 | 策略 |
|------|---------|------|
| **with_retry 内部** | 429/529 瞬态错误 | 指数退避，最多 10 次 |
| **外层 try/except** | prompt_too_long、其他异常 | 紧急压缩、错误提示 |

### 429 Rate Limit（速率限制）

```python
if "ratelimit" in name.lower() or "429" in msg:
    delay = retry_delay(attempt)
    print(f"  [429 rate limit] retry {attempt+1}/{MAX_RETRIES}, wait {delay:.1f}s")
    time.sleep(delay)
    continue
```

### 529 Overloaded（服务过载）

```python
if "overloaded" in name.lower() or "529" in msg:
    state.consecutive_529 += 1
    if state.consecutive_529 >= MAX_CONSECUTIVE_529:
        if FALLBACK_MODEL:
            state.current_model = FALLBACK_MODEL
            state.consecutive_529 = 0
            print(f"  [529 x{MAX_CONSECUTIVE_529}] switching to {FALLBACK_MODEL}")
    delay = retry_delay(attempt)
    print(f"  [529 overloaded] retry {attempt+1}/{MAX_RETRIES}, wait {delay:.1f}s")
    time.sleep(delay)
    continue
```

**Fallback Model 机制**：
- 连续 3 次 529 后切换到备用模型
- 需要在 `.env` 中配置 `FALLBACK_MODEL_ID`
- 切换后重置 `consecutive_529` 计数器

### 指数退避算法

```python
def retry_delay(attempt, retry_after=None):
    if retry_after:
        return retry_after  # Retry-After 优先
    base = min(BASE_DELAY_MS * (2 ** attempt), 32000) / 1000
    jitter = random.uniform(0, base * 0.25)
    return base + jitter
```

**算法解析**：

| 参数 | 值 | 说明 |
|------|-----|------|
| `BASE_DELAY_MS` | 500 | 初始延迟 0.5 秒 |
| 退避公式 | `500 * 2^attempt` | 指数增长 |
| 最大延迟 | 32000 ms = 32 秒 | 防止无限等待 |
| Jitter | 0 ~ 25% | 随机抖动，避免重试风暴 |
| `Retry-After` | 优先使用 | 如果 API 返回此头，直接使用 |

**退避时间表**：

| 重试次数 | 基础延迟 | 最大总延迟（含 jitter） |
|---------|---------|----------------------|
| 1 | 0.5s | ~0.625s |
| 2 | 1.0s | ~1.25s |
| 3 | 2.0s | ~2.5s |
| 4 | 4.0s | ~5.0s |
| 5 | 8.0s | ~10.0s |
| 6+ | 32.0s | ~40.0s |

## 配置参数

### 环境变量

```env
MODEL_ID=claude-3-sonnet-20240229        # 主模型
FALLBACK_MODEL_ID=claude-3-haiku-20240307  # 备用模型（529 时切换）
```

### 常量定义

```python
ESCALATED_MAX_TOKENS = 64000     # 升级后的 token 上限
DEFAULT_MAX_TOKENS = 8000        # 默认 token 上限
MAX_RECOVERY_RETRIES = 3         # 续写最大重试次数
MAX_RETRIES = 10                 # 指数退避最大重试次数
BASE_DELAY_MS = 500              # 退避基础延迟（毫秒）
MAX_CONSECUTIVE_529 = 3          # 连续 529 后切换模型
```

## 与前序章节的关系

| 章节 | 核心功能 | 与 ch11 的关系 |
|------|---------|---------------|
| ch08 | 四层上下文压缩管道 | ch11 的 reactive_compact 简化版基于 ch08 的概念 |
| ch10 | 动态 System Prompt 组装 | ch11 继承了 prompt 组装和缓存机制 |
| ch11 | 错误恢复系统 | **新增**三条恢复路径和指数退避 |

### agent_loop 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    ch11 agent_loop 完整流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户输入                                                        │
│     ↓                                                           │
│  agent_loop(messages, context)                                   │
│     ↓                                                           │
│  system = get_system_prompt(context)  ← ch10 机制                │
│  state = RecoveryState()  ← 初始化状态机                         │
│  max_tokens = DEFAULT_MAX_TOKENS                                 │
│     ↓                                                           │
│  ┌─ while True ──────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  with_retry(LLM Call, state)                              │  │
│  │     ├── 429 → 指数退避重试 (内部)                          │  │
│  │     ├── 529 → 指数退避 + 可能切换模型 (内部)               │  │
│  │     ├── 其他异常 → re-raise → 外层处理                     │  │
│  │     └── 成功 → response                                   │  │
│  │                                                           │  │
│  │  [外层 except]                                            │  │
│  │     ├── prompt_too_long → reactive_compact (1次)          │  │
│  │     └── 其他 → 错误提示 + 返回                             │  │
│  │                                                           │  │
│  │  response.stop_reason == "max_tokens"?                    │  │
│  │     ├── 否 → 追加响应，检查 tool_use                       │  │
│  │     └── 是 → Path 1 恢复                                  │  │
│  │           ├── 未升级 → 8K→64K，不追加，重试               │  │
│  │           └── 已升级 → 追加输出 + 续写提示 (最多3次)       │  │
│  │                                                           │  │
│  │  tool_use?                                                │  │
│  │     ├── 否 → 返回                                         │  │
│  │     └── 是 → 执行工具                                     │  │
│  │           └── 更新 context → 重新组装 prompt               │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 实践练习

### 练习1：模拟 max_tokens 恢复

1. 运行 `python -m ch11_error_recovery.code`
2. 输入："请详细介绍 Python 的 asyncio 模块，包括事件循环、协程、任务、Future、信号量等概念，每个概念都要给出代码示例"
3. 观察控制台输出的 `[max_tokens] escalating` 和 `[max_tokens] continuation` 日志

### 练习2：测试 529 Fallback 机制

1. 在 `.env` 中配置 `FALLBACK_MODEL_ID`：
```env
MODEL_ID=claude-3-sonnet-20240229
FALLBACK_MODEL_ID=claude-3-haiku-20240307
```

2. 修改代码，模拟 529 错误：
```python
# 在 client.messages.create 调用前添加
if random.random() < 0.5:
    raise Exception("529 Service Unavailable")
```

3. 运行程序，观察连续 3 次 529 后是否切换到 fallback 模型

### 练习3：理解指数退避

在 `retry_delay` 函数中添加调试打印：

```python
def retry_delay(attempt, retry_after=None):
    base = min(BASE_DELAY_MS * (2 ** attempt), 32000) / 1000
    jitter = random.uniform(0, base * 0.25)
    print(f"  [debug] attempt={attempt}, base={base:.2f}s, jitter={jitter:.2f}s")
    return base + jitter
```

观察每次重试的延迟变化。

### 练习4：扩展恢复路径

尝试添加新的恢复路径：

```python
# 在 with_retry 的 except 块中添加
if "timeout" in name.lower() or "timedout" in msg:
    delay = retry_delay(attempt)
    print(f"  [timeout] retry {attempt+1}/{MAX_RETRIES}, wait {delay:.1f}s")
    time.sleep(delay)
    continue
```

模拟网络超时错误，测试新的恢复路径。
