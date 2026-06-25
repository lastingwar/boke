---
title: "Hermes Agent v0.16 Kanban Swarm 深度解析：多智能体协作架构"
date: 2026-06-25
tags: ["hermes-agent", "multi-agent", "kanban", "orchestration", "swarm", "architecture"]
description: "深入剖析 Hermes Agent v0.16 Kanban Swarm 的设计哲学——一个不引入第二调度器、以 SQLite 持久化为核心的多智能体协作系统，以及它如何区别于 LangGraph、AutoGen、CrewAI 等框架。"
author: "Nous Research"
---

# Hermes Agent v0.16 Kanban Swarm 深度解析：多智能体协作架构

> **导读**：Kanban Swarm 是 Hermes Agent 内置的一套基于 SQLite 看板的多智能体协作系统。本文从架构设计、核心机制、实战案例三个维度拆解，并与 LangGraph、AutoGen、CrewAI 做深度对比。

## 目录

- [引言：当你的 Agent 需要一支团队](#引言当你需要一支团队)
- [什么是 Kanban Swarm？](#什么是-kanban-swarm)
- [架构四层：Board → Dispatcher → Worker → Orchestrator](#架构四层board--dispatcher--worker--orchestrator)
  - [第一层：Board（存储层）](#第一层board存储层)
  - [第二层：Dispatcher（调度层）](#第二层dispatcher调度层)
  - [第三层：Worker（执行层）](#第三层worker执行层)
  - [第四层：Orchestrator（编排层）](#第四层orchestrator编排层)
- [核心机制详解](#核心机制详解)
  - [依赖链与 Fan-in/Fan-out](#依赖链与-fan-infan-out)
  - [Workspace 隔离策略](#workspace-隔离策略)
  - [Heartbeat 与故障恢复](#heartbeat-与故障恢复)
  - [Goal Mode 循环](#goal-mode-循环)
  - [Blackboard 协议](#blackboard-协议)
- [实战案例：被自己编排的博客流水线](#实战案例被自己编排的博客流水线)
- [生态对比：Kanban Swarm vs LangGraph / AutoGen / CrewAI](#生态对比kanban-swarm-vs-langgraph--autogen--crewai)
- [不是银弹：Kanban Swarm 的适用边界](#不是银弹kanban-swarm-的适用边界)
- [未来展望](#未来展望)

---

## 引言：当你的 Agent 需要一支团队

你让一个 Agent 去写一篇技术博客。它调研了 8 个来源，整理了 16KB 的研究笔记……然后上下文窗口炸了。你换了个办法：扔给三个 Agent 分别搞调研、写作、审核，结果发现——怎么通信？谁等谁？中间那个崩了怎么办？输出怎么汇总？

这不是科幻设定，而是多智能体编排（multi-agent orchestration）的真实痛点。现有框架给了你两条路：要么用 LangGraph 手写状态图——灵活但每加一个 Agent 都是 Python 工程；要么用 CrewAI 定义角色——直白但一旦流程分叉就抓瞎。更关键的是，这些方案几乎都是**无状态的**——Agent 崩溃、重启或上下文被压缩后，进度归零。

Nous Research 在 Hermes Agent v0.16 中给出了一个不同的答案：**Kanban Swarm**。它只有一个核心论断——所有协调状态都应该持久化在数据库中，且不引入第二个调度器。这篇文章会从架构设计、核心机制、实战案例三个维度拆解它，并与 LangGraph、AutoGen、CrewAI 做对比，帮你在选型时做出判断。

## 什么是 Kanban Swarm？

Kanban Swarm 不是一个独立的框架，而是 Hermes Agent 内置的一套**基于 SQLite 看板的多智能体协作系统**。它把「看板」（Kanban Board）这个概念从人类项目管理搬到了 Agent 协作域——每个任务是 SQLite 中的一行，每个交接（handoff）是任何人都可读写的数据行，每个 Worker 是一个完整的 OS 进程，拥有自己的身份、工具集和记忆。

它在 v0.14（2026-05-16，"The Tenacity Release"）首次引入，v0.15（2026-05-28，"The Velocity Release"）加入 Swarm 拓扑和自动分解，v0.16（2026-06-05，"The Surface Release"）完善了 goal mode、文件附件和并发控制。

## 架构四层：Board → Dispatcher → Worker → Orchestrator

理解 Kanban Swarm 最快的方式是沿着一个任务的生命周期走一遍：

```
triage → todo → ready → running → done / blocked / archived
```

### 第一层：Board（存储层）

Board 是一个独立的 SQLite 数据库（默认 `~/.hermes/kanban.db`），隔离整个项目的任务队列。多个 Board 可以并存——一个用于博客流水线，一个用于代码审查，互不干扰。

```bash
# 创建和切换 Board
hermes kanban boards create blog-pipeline --name "博客流水线" --icon 📝
hermes kanban boards switch blog-pipeline
```

**为什么重要**：SQLite 意味着无需部署 Redis、Postgres 或任何外部依赖。重启后状态毫发无损。这是 Kanban 区别于所有内存态框架的根基。

### 第二层：Dispatcher（调度层）

Dispatcher 是一个嵌入 Gateway 进程的调度循环（默认每 60 秒执行一次），负责三件事：

1. **回收 stale claims**——Worker 超时（默认 4 小时无 heartbeat）后，将任务重新放回 `ready` 队列，不计入失败次数
2. **依赖提升**——当所有父任务完成时，自动将子任务从 `todo` 提升到 `ready`
3. **原子化领取与派生**——领取 `ready` 任务，spawn 对应 profile 的 Worker 进程

没有单独的调度守护进程，没有消息队列。所有协调状态就是 SQLite 中不断流转的行。

### 第三层：Worker（执行层）

Worker 是被 Dispatcher spawn 的完整 Hermes Agent 进程。它通过环境变量 `HERMES_KANBAN_TASK` 获得任务 ID，自动注入 `kanban_*` 工具集：

| 工具 | 用途 |
|------|------|
| `kanban_show()` | 读取任务详情，包括父任务 handoff（summary + metadata） |
| `kanban_list()` | 按条件列出任务（按 profile、状态、tenant 过滤） |
| `kanban_complete(summary, metadata)` | 完成任务，输出结构化交接信息 |
| `kanban_block(reason)` | 遇到无法决策的问题时请求人工介入 |
| `kanban_unblock(task_id)` | 将 blocked 任务移回 ready 队列 |
| `kanban_heartbeat(note)` | 长任务期间（>1h）的存活信号 |
| `kanban_create(title, assignee)` | 创建子任务 |
| `kanban_link(parent_id, child_id)` | 建立依赖关系 |

Worker 不需要知道自己的任务 ID——`kanban_show()` 无参数调用默认读取当前任务。完成任务时，`kanban_complete` 写入的 `summary` 和 `metadata` 会成为下游 Worker 读到的上下文。

```python
# Worker 视角：完成任务
kanban_complete(
    summary="已分析 auth 模块，发现 3 个安全问题：JWT 未设置过期、密码未加盐、refresh token 可重放",
    metadata={
        "changed_files": ["audit/auth-report.md"],
        "findings": ["CWE-613", "CWE-759", "CWE-290"],
        "severity": "high"
    }
)
```

**为什么重要**：`metadata` 是机器可读的结构化交接——下游 Worker 可以解析它做条件判断，而不仅仅依赖人类可读的 `summary`。这是实现可靠自动化工作流的关键。

### 第四层：Orchestrator（编排层）

Orchestrator 是顶层协调者。它不执行具体任务，而是通过 `kanban_create` 分解目标、通过 `kanban_link` 建立依赖图。最极致的用例是 Swarm 拓扑：

```bash
hermes kanban swarm "Audit our API surface for security regressions" \
  --worker researcher:"Scan endpoints and dependencies" \
  --worker coder:"Check auth middleware implementation" \
  --worker coder:"Review rate limiting and input validation" \
  --verifier reviewer \
  --synthesizer writer
```

一条命令，生成如下拓扑：

```
         ┌─ Worker 1 (researcher) ─┐
         │                         │
 Swarm ──┼─ Worker 2 (coder) ──────┼── Verifier (reviewer) ── Synthesizer (writer)
 Root    │                         │
         └─ Worker 3 (coder) ──────┘
```

三个 Worker 并行运行，全部完成后 Verifier 审查（必须设置 `{"gate": "pass"}`），最终由 Synthesizer 汇总输出。注意：**整个 Swarm 模块仅 279 行 Python 代码**——它不需要第二个调度器，直接复用现有的 Kanban 内核。

## 核心机制详解

### 依赖链与 Fan-in/Fan-out

Kanban 的依赖链实现了一个简洁但强大的编排原语：

```bash
# Fan-out：一个父任务派生多个并行子任务
SCHEMA=$(hermes kanban create "Design auth schema" --assignee backend-dev --json | jq -r .id)
hermes kanban create "Implement auth API" --assignee backend-dev --parent $SCHEMA
hermes kanban create "Write auth tests" --assignee qa-dev --parent $SCHEMA
```

当 `SCHEMA` 完成时，两个子任务同时提升到 `ready`——这就是 Fan-out。Fan-in 同理：一个任务可以设置多个父任务，只有**全部**完成后才被激活。没有 DAG 引擎，没有工作流 DSL，只有 SQLite 的 parent→child 边。

### Workspace 隔离策略

每个 Worker 都在独立的 workspace 中运行，三种类型覆盖不同场景：

| 类型 | 生命周期 | 适用场景 |
|------|----------|----------|
| **scratch**（默认） | 任务完成后**删除** | 临时计算、一次性分析 |
| **dir:\<path\>** | 永久保留 | 共享代码库、持续集成的项目目录 |
| **worktree** | 永久保留（Git worktree） | 并行编辑同一仓库的不同分支 |

此外，Kanban 还提供了 **Tenant 隔离**机制：在同一 Board 内通过 `--tenant` 标签对任务进行逻辑分组。Tenant 隔离 workspace 路径（不同 tenant 的任务写入独立的目录）和 memory key 空间，确保多个并行流水线（如同步进行博客 A 和代码审查 B）不会互相污染上下文。这在多项目共享一个 Board 时尤为有用。

对于代码生成任务，`worktree` 是最佳选择——每个 Worker 获得一个隔离的 Git worktree，避免并发修改冲突。

### Heartbeat 与故障恢复

这是 Kanban 最被低估但最关键的设计。Worker 在长操作（>1h）期间定期调用 `kanban_heartbeat(note="analyzing large codebase")`。Dispatcher 每 60 秒扫描一次：如果某个运行中的任务超过 4 小时没有 heartbeat，自动回收为 `ready` 状态并重新排队。回收**不计入失败次数**——它只是假设 Worker 进程崩溃了，而不是任务本身失败了。

这解决了多智能体系统中一个棘手的问题：某个 Worker 静默死亡后，整个流水线不会永远卡在 `running`。

### Goal Mode 循环

v0.16 引入的 Goal Mode 允许 Worker 进入自动重试循环。每次 turn 后，一个辅助 judge 检查是否达成目标。未达成且 budget 未耗尽时，Worker 原地继续；达成后自动 block 等待人工审核。这对需要多轮迭代的开放任务（如「重构 auth 模块并覆盖率达到 90%」）特别有效。

### Blackboard 协议

并行 Worker 之间通过看板注释交换状态——这就是 Blackboard 协议。Worker A 写入：

```
kanban_comment(task_id="t_swarm_root", body='{"key": "endpoints_found", "value": ["/login", "/register", "/refresh"]}')
```

Worker B 读取 `kanban_show()` 的 comment thread 后解析 JSON，获取所有对等 Worker 的阶段性输出。不需要 RPC，不需要共享内存，所有状态都在 SQLite 中。

## 实战案例：被自己编排的博客流水线

这篇文章本身就是一个 Kanban Swarm 的真实产物。它的工作流是：

```
Orchestrator 创建流程
  ↓
调研 Worker（researcher profile）
  → 搜索官方文档、GitHub releases、社区博客
  → 输出 research.md（345 行，16.6 KB）
  ↓
写作 Worker（writer profile）
  → 通过 kanban_show() 读取 researcher 的 summary + metadata
  → 根据结构化建议撰写本文
  ↓
审核 Worker（reviewer profile，verifier gate）
  → 检查技术准确性、代码示例可运行性
  ↓
完成
```

每一步的交接都是 `kanban_complete` 中写入的结构化数据，而非靠提示词传递。如果调研阶段失败，Dispatcher 自动回收并重新分配——整个流水线不会在凌晨 3 点等你手动重启。

## 生态对比：Kanban Swarm vs LangGraph / AutoGen / CrewAI

| 维度 | Kanban Swarm | LangGraph | AutoGen | CrewAI |
|------|-------------|-----------|---------|--------|
| **核心理念** | SQLite 看板状态机 | Python 显式状态图 | 对话式 Agent 协作 | 模拟人类团队 |
| **持久化** | ✅ 原生 SQLite | ❌ 需外部存储 | ❌ 内存态 | ❌ 内存态 |
| **崩溃恢复** | ✅ Dispatcher 自动回收 | ❌ 需自行实现 | ❌ | ❌ |
| **人类审查** | ✅ Verifier gate / unblock | ⚠️ 手动实现 | ⚠️ 手动实现 | ⚠️ 手动实现 |
| **审计追溯** | ✅ SQLite 永久保存 | ❌ | ❌ | ❌ |
| **代码量（核心）** | 279 行 | 框架级（数万行） | 框架级 | 框架级 |
| **学习曲线** | 低（复用看板概念） | 高（状态图理论） | 中 | 低 |

Kanban Swarm 的独特之处在于三点：

1. **不引入第二调度器**：所有协调状态就是 SQLite 的行，Dispatcher、Dashboard、CLI、通知器都在同一数据模型上工作。
2. **持久化不是附加功能，而是设计前提**：Agent 崩溃、重启、被 SIGTERM 杀掉——状态都在。OpenAI Swarm 在复杂度升高时精度从 84% 跌至 0%，很大程度上因为它无状态的 handoff 设计。（来源：magnus919.com 对比评测）
3. **人类与 Agent 在同一界面上协作**：同一块看板、同一条注释线程。你可以在 Dashboard 上直接 unblock 一个被卡住的任务，也可以写一条注释给下游 Worker。

## 不是银弹：Kanban Swarm 的适用边界

Kanban Swarm 最适合的场景：有明确步骤依赖的流水线任务（调研→分析→输出）、需要人类审查门控的工作流、跨多个 profile 的持久化协作。

**不适合**的场景：需要毫秒级响应的实时 Agent 对话（用 `delegate_task`）、不需要跨 session 状态的单次分析、逻辑分支极度复杂的动态路由（LangGraph 的显式状态图更合适）。

## 未来展望

从 v0.14 到 v0.16 的演进节奏（三周一个大版本）来看，Kanban 的迭代方向清晰：更多自动化（如 triage 阶段的自动分解器）、更丰富的拓扑原语（条件分支、循环）、更深的多 Board 协作。`/swarm` 命令的三阶段渐进设计（全自动 → CLI 手动 → 完全自主）说明 Nous 的愿景是用自然语言一句话驱动整个多 Agent 团队。

对于已经在用 Hermes Agent 的开发者来说，Kanban 值得花一个下午的时间上手——它把 `delegate_task` 的「函数调用」思维升级为「工作队列」思维，而这种思维转变，是构建可靠多智能体系统的第一步。

---

*本文基于 Hermes Agent v0.16 官方文档及社区分析撰写。完整调研笔记见上游 research.md（345 行，8 个参考来源）。*
