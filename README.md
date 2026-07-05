# Learn Claude Code — 智能代理开发学习路径

一个循序渐进的智能代理开发学习项目，从基础的 agent loop 开始，逐步构建功能完善的编码代理系统。

## 快速开始

```bash
# 创建并激活虚拟环境
conda create -n learnclaude python=3.11 -y
conda activate learnclaude

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp .env.example .env
# 编辑 .env，填入你的 API 密钥和模型 ID

# 运行第一个章节
python -m ch01_agent_loop.code
```

---

## 学习路线图

### 章节概览

| 章节 | 名称 | 主题 | 核心功能 |
|------|------|------|---------|
| [ch01](file:///e:/Agents/learn_claude_code/ch01_agent_loop/README.md) | Agent Loop | 基础代理循环 | ReAct 模式、LLM 调用、工具执行 |
| [ch02](file:///e:/Agents/learn_claude_code/ch02_tool_use/README.md) | Tool Use | 多工具支持 | 文件读写、命令执行、工具分发 |
| [ch03](file:///e:/Agents/learn_claude_code/ch03_permission/README.md) | Permission | 权限系统 | 三闸门权限管道、危险命令拦截 |
| [ch04](file:///e:/Agents/learn_claude_code/ch04_hooks/README.md) | Hooks | 钩子系统 | 事件驱动架构、解耦扩展逻辑 |
| [ch05](file:///e:/Agents/learn_claude_code/ch05_todo_write/README.md) | Todo Write | 任务规划 | 任务列表管理、Nag 提醒机制 |
| [ch06](file:///e:/Agents/learn_claude_code/ch06_subagent/README.md) | Subagent | 子代理系统 | 任务分解、上下文隔离、递归限制 |
| [ch07](file:///e:/Agents/learn_claude_code/ch07_skill_loading/README.md) | Skill Loading | 技能加载 | 两级知识注入、技能注册表 |
| [ch08](file:///e:/Agents/learn_claude_code/ch08_context_compact/README.md) | Context Compact | 上下文压缩 | 四层压缩管道、大结果持久化 |
| [ch09](file:///e:/Agents/learn_claude_code/ch09_memory/README.md) | Memory | 记忆系统 | 跨会话持久化、索引检索、自动合并 |
| [ch10](file:///e:/Agents/learn_claude_code/ch10_system_prompt/README.md) | System Prompt | 动态 Prompt | 运行时组装、确定性缓存、状态驱动 |
| [ch11](file:///e:/Agents/learn_claude_code/ch11_error_recovery/README.md) | Error Recovery | 错误恢复 | 三条恢复路径、指数退避、Fallback 模型 |
| [ch12](file:///e:/Agents/learn_claude_code/ch12_task_system/README.md) | Task System | 任务系统 | 文件持久化、依赖图、生命周期管理 |
| [ch13](file:///e:/Agents/learn_claude_code/ch13_background_tasks/README.md) | Background Tasks | 后台任务 | 线程异步执行、通知注入、线程安全 |
| [ch14](file:///e:/Agents/learn_claude_code/ch14_cron_scheduler/README.md) | Cron Scheduler | 定时调度 | 四层架构、cron 表达式、队列处理器 |

### 学习路径建议

```
第 1 周：基础概念
    ├── ch01: 理解 ReAct 模式和代理循环
    └── ch02: 掌握多工具定义和执行

第 2 周：安全与扩展
    ├── ch03: 实现权限系统
    └── ch04: 理解钩子系统和解耦设计

第 3 周：智能增强
    ├── ch05: 添加任务规划功能
    └── ch06: 实现子代理系统
```
第 4 周：高级功能
    ├── ch07: 技能加载系统
    └── ch08: 上下文压缩管道

第 5 周：持久化与优化
    ├── ch09: 记忆系统（跨会话知识）
    └── ch10: 动态 Prompt（运行时组装）

第 6 周：鲁棒性增强
    └── ch11: 错误恢复（三条路径、指数退避）

第 7 周：任务管理
    └── ch12: 任务系统（依赖图、持久化）

第 8 周：异步执行
    └── ch13: 后台任务（线程异步、通知注入）

第 9 周：定时调度
    └── ch14: Cron 调度器（四层架构、cron 表达式）
```

---

## 项目架构

```
learn_claude_code/
├── ch01_agent_loop/          # 基础代理循环
├── ch02_tool_use/            # 多工具支持
├── ch03_permission/          # 权限系统
├── ch04_hooks/               # 钩子系统
├── ch05_todo_write/          # 任务规划
├── ch06_subagent/            # 子代理系统
├── ch07_skill_loading/       # 技能加载
├── ch08_context_compact/     # 上下文压缩
├── ch09_memory/              # 记忆系统
├── ch10_system_prompt/       # 动态 Prompt
├── ch11_error_recovery/      # 错误恢复
├── ch12_task_system/         # 任务系统
├── ch13_background_tasks/    # 后台任务
├── ch14_cron_scheduler/      # Cron 调度器
├── skills/                   # 技能文件目录
│   ├── agent-builder/        # 代理构建技能
│   ├── code-review/          # 代码审查技能
│   ├── mcp-builder/          # MCP 服务器构建技能
│   └── pdf/                  # PDF 处理技能
├── .env.example              # 环境变量模板
├── .gitignore               # Git 忽略配置
├── requirements.txt         # Python 依赖
└── README.md               # 项目说明
```

---

## 核心设计模式

### ReAct 模式（Reasoning + Acting）

每个章节的代理都遵循 ReAct 模式：

```
用户输入
    │
    ▼
LLM 推理（思考应该做什么）
    │
    ▼
工具调用（执行操作）
    │
    ▼
工具结果反馈
    │
    ▼
循环直到完成
```

### 渐进式开发

每个章节在前面章节的基础上添加新功能：

| 章节 | 新增功能 | 保留功能 |
|------|---------|---------|
| ch01 | agent loop | - |
| ch02 | 多工具、工具分发 | agent loop |
| ch03 | 权限检查 | agent loop、工具 |
| ch04 | 钩子系统 | 权限、工具、loop |
| ch05 | todo_write、Nag 提醒 | 钩子、权限、工具 |
| ch06 | task 工具、子代理 | todo、钩子、权限 |
| ch07 | load_skill、技能注册表 | 子代理、todo、钩子 |
| ch08 | 四层压缩管道 | 技能、子代理、todo |
| ch09 | 记忆系统、索引检索、自动合并 | 压缩管道、技能、子代理 |
| ch10 | 动态 Prompt、确定性缓存 | 记忆系统、压缩管道、技能 |
| ch11 | 三条恢复路径、指数退避、Fallback | 动态 Prompt、记忆系统、压缩管道 |
| ch12 | 文件持久化任务系统、依赖图、生命周期 | 动态 Prompt、记忆系统 |
| ch13 | 后台任务、线程异步执行、通知注入 | 任务系统、动态 Prompt、记忆系统 |
| ch14 | Cron 调度器、四层架构、cron 表达式匹配 | 后台任务、任务系统、动态 Prompt |

---

## 环境配置

### .env 文件

```env
# API 配置
ANTHROPIC_BASE_URL=https://api.anthropic.com/v1
MODEL_ID=claude-3-sonnet-20240229

# 其他配置（根据章节需求）
# API_KEY=your-api-key
```

### 依赖安装

```bash
pip install -r requirements.txt
```

依赖列表：
- `anthropic` — Anthropic API SDK
- `python-dotenv` — 环境变量加载
- `pyyaml` — YAML 解析（ch07+）

---

## 运行方式

### 方式一：模块方式（推荐）

```bash
python -m ch01_agent_loop.code
python -m ch02_tool_use.code
python -m ch03_permission.code
# ...
```

### 方式二：直接运行

```bash
cd ch01_agent_loop
python code.py
```

### 方式三：使用 conda run

```bash
conda run -n learnclaude python -m ch01_agent_loop.code
```

---

## 实践建议

1. **按顺序学习**：每个章节建立在前一个章节的基础上，建议按 ch01 → ch08 的顺序学习
2. **运行代码**：每个章节都有测试 prompt，运行代码并观察输出
3. **修改代码**：尝试修改配置参数、添加新工具、扩展功能
4. **阅读 README**：每个章节的 README 包含详细的代码解析和扩展思考
5. **完成练习**：每个章节的 README 末尾都有实践练习，帮助巩固知识

---

## 扩展资源

- [Anthropic API Documentation](https://docs.anthropic.com/claude/docs)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Agent Building Guide](https://github.com/anthropics/anthropic-sdk-python)

---

## 许可证

MIT License