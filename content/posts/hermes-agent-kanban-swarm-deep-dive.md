---
title: "当 AI Agent 学会「分工协作」：Hermes Agent Kanban Swarm 深度解析"
date: 2026-07-15T10:04:25+08:00
tags:
  - AI Agent
  - Hermes Agent
  - Kanban
  - Multi-Agent
  - 多智能体协作
author: "mai-zi208"
---

# 当 AI Agent 学会「分工协作」：Hermes Agent Kanban Swarm 深度解析

> 单 Agent 再聪明，也只是一颗大脑。真正的生产力爆发，来自多个专业 Agent 的协同作战。本文将带你深入 Hermes Agent v0.18 的 Kanban Swarm —— 一个用 278 行代码构建的多智能体协作引擎。

---

## 一、引言：多 Agent 协作的「最后一公里」

你有没有遇到过这种情况？

你让 AI Agent 做一个完整项目，它信誓旦旦地说「我能行」。半小时后，你发现它在 3000 行代码里反复重构同一个模块，却忘了还有三个子系统压根没动。Context 窗口炸了，逻辑互相污染，最后你不得不手动拆分成十几个小任务逐个喂养。

这不是 AI 不够聪明，而是**单 Agent 架构的天然瓶颈**：一个大脑、一个执行线程、一个上下文窗口。

Hermes Agent v0.18 给出的答案是 **Kanban Swarm** —— 一个轻量级的多 Agent 智能调度层。它不引入重量级微服务、不依赖外部队列、不要求 Kubernetes。它只是在你已有的 SQLite Kanban Board 上，加了一层仅 **278 行**的拓扑构造器，然后让不同 profile 的 Agent 像一支真正的团队那样分工、协作、交付。

**为什么这很重要？** 因为多 Agent 协作的难点从来不在于「能不能」，而在于状态持久化、故障恢复、依赖管理、以及跨 worker 通信。Kanban Swarm 用极简的设计同时解决了这四个问题。

---

## 二、Kanban Swarm 是什么？

先给一个直觉层面的定义：

**Kanban Swarm 是一套建立在 SQLite 任务板之上的多 Agent 协作协议。它的核心由四个角色构成：**

| 角色 | 职责 | 类比 |
|------|------|------|
| **Board**（面板） | SQLite 数据库，存任务、依赖、事件、评论 | 团队的 Jira/Linear |
| **Task**（任务） | 一个可被认领、执行、阻塞、完成的工作单元 | 一张 Jira Ticket |
| **Worker**（工人） | 被 Dispatcher 派发的独立 Agent 进程（`hermes -p <profile> chat`） | 一个团队成员 |
| **Dispatcher**（调度器） | 每 60 秒扫描板子，回收超时任务、晋升就绪任务、派发新任务 | 团队的 PM + SRE |

工作流很简单：创建带依赖关系的任务 → Dispatcher 自动派发给不同 profile 的 Worker → Worker 完成后写 handoff → 下游自动继续。

**与 `delegate_task` 的关键区别：**

如果你用过 Hermes 的 `delegate_task`，你可能会问：这不就是另一个 delegation 吗？不。它们解决的是两个完全不同的问题：

- `delegate_task` 是**会话级**的并行子任务 —— 进程退出即丢失，适合 5 分钟内的临时推理
- Kanban Swarm 是**持久化**的团队级工作流 —— 数据落 SQLite，跨会话存活，适合数小时甚至数天的协作

用一句话总结：**delegate_task 是你临时叫两个同事过来帮忙，Kanban Swarm 是一个有 Jira、有 PM、有 SRE 的正规团队。**

---

## 三、架构深度解析

### 3.1 Orchestrator → Specialist 分解模式

Kanban Swarm 的精髓在于「分解」。两种分解路径：

**路径 A：AI 辅助分解（`kanban_decompose.py`）**  
调用辅助 LLM 阅读任务内容和 profile 花名册，自动输出 JSON 格式的子任务图：

```json
{
  "fanout": true,
  "tasks": [
    {"title": "调研 Kanban Swarm 架构", "assignee": "researcher", "parents": []},
    {"title": "撰写技术博客", "assignee": "writer", "parents": [0]},
    {"title": "审查内容", "assignee": "reviewer", "parents": [1]}
  ]
}
```

规则很聪明：2-6 个任务最佳（不会拆成 20 个碎片）；`parents` 用 0-based index 引用兄弟任务；无 parent = 并行；有 parent = 串行等待。

**路径 B：手工 Swarm 拓扑（`kanban_swarm.py`）**  
```bash
hermes kanban swarm \
  "调研并撰写 Hermes 博客" \
  --worker researcher:技术调研 \
  --worker writer:撰写博客 \
  --verifier reviewer \
  --synthesizer publisher
```

> 注意：goal 是位置参数，直接放在命令后面，无需 `--goal` flag。多个 worker 通过重复 `--worker` 指定，没有 `--writer` 这类别名。

生成一个标准的**扇出-扇入**拓扑：多个 worker 并行执行 → Verifier 汇总 → Synthesizer 产出终稿。每个 worker 的 task body 自动注入 swarm context（root_id、goal、协作规则）。

**为什么重要？** 这解决了一个关键痛点——分解策略的选择。简单场景用 AI 自动分解，复杂拓扑用手工精确控制。两者共享同一套底层板子和 Dispatcher，不需要在「自动化」和「可控性」之间二选一。

### 3.2 依赖管理：串行、并行、扇入

依赖通过 `task_links` 表的一条简单记录就能表达：

```
task_links (parent_id, child_id) — PRIMARY KEY
```

三条规则就能描述所有拓扑：
- **无共同依赖** → 全部 `ready` → 并行执行
- **链式依赖**（A ← B ← C）→ 严格串行
- **扇入**（D ← A, B, C）→ A/B/C 全部 done 后 D 才就绪

Dispatcher 每 tick 调用 `recompute_ready()`：扫描所有 `todo` 任务，parent 全部 done → 晋升 `ready`。整个机制就是一张有向无环图（DAG），简洁、可靠、零魔法。

### 3.3 工作空间策略

不同任务需要不同级别的隔离。Kanban Swarm 提供了三种 workspace 类型：

| 类型 | 路径 | 适用场景 |
|------|------|---------|
| `scratch`（默认） | 临时目录，任务结束后清理 | 一次性调研、分析 |
| `dir` | 用户指定的绝对路径 | 共享工作区、多人协作 |
| `worktree` | git worktree，自动创建隔离分支 | 代码修改，避免并发冲突 |

最妙的是 **project-linked worktree**：当任务关联了 project，会自动创建确定性分支名 `<project-slug>/<task-id>`，而不是随机名 `wt/<task-id>`。这意味着你可以精确重现任何一个任务的代码状态。

### 3.4 你以为它在运行，其实它可能已经死了——Heartbeat 与故障恢复

多进程系统的噩梦是：一个 worker 静默挂了，没人知道。Kanban Swarm 用三重保护解决：

1. **显式心跳**：worker 可调用 `kanban_heartbeat(note=...)` 续命
2. **自动心跳桥**：Agent 每次 chunk 级活动自动续心跳（60 秒 dedup）——这意味着正常工作的 worker 不需要手动写 heartbeat
3. **跨平台 PID 存活检查**：Dispatcher 通过 `gateway.status._pid_exists` API 检测进程存活（跨平台实现，Windows 下通过系统 API，Linux 下额外读 `/proc/<pid>/status` 检测僵尸状态）——不依赖 TCP 长连接

**Reclaim 决策链**（每 60 秒执行一次，由 `_dispatch_once_locked` 驱动）：

| 步骤 | 阶段 | 检测条件 | 动作 |
|------|------|---------|------|
| 1 | `release_stale_claims` | TTL 过期 + PID **存活** | **延长** claim（worker 还在，只是心跳延迟） |
| 1 | `release_stale_claims` | TTL 过期 + PID **不存活** | **释放** claim，任务回 `ready` |
| 2 | `detect_stale_running` | PID 存活但 1h 无更新 | 视为卡死（wedged），回 `ready` |
| 3 | `detect_crashed_workers` | PID 不存活（刚崩溃） | 回 `ready` |
| 4 | `enforce_max_runtime` | 超 `max_runtime_seconds` | 标记 `timed_out`，允许重试 |
| 5 | `recompute_ready` | 父任务全部 done | 晋升子任务到 `ready` |
| 6 | `spawn` | 有空闲 slots | 派发 `ready` 任务给对应 profile |

连续失败 ≥ 2 次 → 自动 block，避免无限重试消耗资源。

### 3.5 你知道「完成」是什么意思吗？——Goal Mode

常规 Agent 模式是一个 turn 一个回复，但复杂任务往往需要多轮迭代。Goal Mode 引入了一个 **辅助 Judge** 来回答一个问题：「这件工作真的做完了吗？」

```
Worker 执行 1 个 turn
    ↓
辅助 Judge 检查回复 vs task title/body
    ↓
"done"? → kanban_complete
"continue"? → 继续下一 turn（默认最多 20 轮）
```

关键约束：goal_mode 下的 `kanban_complete` 受 judge 门控——你无法通过伪造完成来逃逸。同时 `kanban_block` 仅限 `dependency` 和 `needs_input` 两类，不能用「我搞不定」来绕过 judge。

**为什么重要？** 这实现了从「Agent 说自己做完了」到「有人（另一个 LLM）验证做完了」的质变。在内容生产、代码审查等质量敏感场景，这个差异是致命的。

---

## 四、实战：从一句话到一篇发布文章的全流程

以下是一个完整的博客写作流水线，展示 Kanban Swarm 如何把 5 个独立 Agent 串成一条生产线：

```
阶段一：Orchestrator 接收需求
  ├─ kanban_decompose → AI 辅助分解为子任务
  ├─ 创建 4 个 child task：
  │    ├─ [researcher-a] 调研 Kanban Swarm 技术架构    ← 无 parent，立即 ready
  │    ├─ [researcher-b] 调研竞品对比                   ← 无 parent，立即 ready
  │    ├─ [writer]      撰写博客初稿                    ← parent: researcher-a
  │    └─ [reviewer]    审查 + 发布                    ← parent: writer
  └─ Dispatcher 自动派发

阶段二：两个 Researcher 并行工作
  Worker A (researcher-a):
    kanban_show() → 读取任务详情 + 父任务 handoff
    web_search + read_file 进行源码级调研
    kanban_comment(key_finding="Kanban Swarm 仅 278 行")
    kanban_complete(summary="架构调研完成", metadata={...})

  Worker B (researcher-b):
    同步进行竞品分析...

阶段三：Writer 等待 researcher 完成
  自动晋升 ready → Dispatcher 派发
  kanban_show() → 读取两个 parent 的 handoff + metadata
  基于调研结果撰写博客
  kanban_complete(summary="初稿完成", artifacts=[...])

阶段四：Reviewer 审查
  读取初稿，审校内容
  kanban_block(kind="needs_input", reason="请确认发布日期")
  → 人工介入 → unblock → 发布
```

**全程无需人工干预**（除了最后的发布确认），5 个 Agent 各司其职，状态全持久化在 SQLite 中——即使进程崩溃也能自动恢复。

---

## 五、最佳实践与避坑指南

### 什么时候该用 Kanban Swarm？

| 场景 | 推荐方案 |
|------|---------|
| 5 分钟以内的并行推理 | `delegate_task` 更轻量 |
| 需要人工审批的流水线 | Kanban Swarm + `block(kind="needs_input")` |
| 多 profile 协作（调研→写→审→发） | Kanban Swarm 原生支持 |
| 需要跨会话持久化的长流程 | Kanban Swarm（数据落 SQLite） |
| 毫秒级实时交互 | 不适合（Dispatcher 60s tick） |

### 关键配置

```yaml
kanban:
  dispatch_in_gateway: true      # Gateway 内嵌 Dispatcher
  max_concurrent_tasks: null     # 全局并发上限
  failure_limit: 2               # 连续失败上限后 auto-block
  dispatch_stale_timeout_seconds: 14400  # 4h 无心跳即回收
```

### 常见坑

1. **新任务要等 60 秒**：Dispatcher 默认 60s tick 一次。急用可以用 `hermes kanban dispatch` 手动触发
2. **一个 worker 就是一个独立 hermes 进程**：启动开销 ~1s + ~2K tokens system prompt。不要为 30 秒就能完成的原子操作创建 Swarm
3. **worker 间通信不是实时的**：通过 `kanban_comment` + structured handoff 传递信息，不是消息队列。设计任务时确保下游 worker 能通过 `kanban_show()` 拿到所需全部上下文
4. **`needs_input` block 需要人主动 unblock**：养成定期 `hermes kanban ls --status blocked` 的习惯

---

## 六、总结：从「一个 AI」到「一支 AI 团队」

Kanban Swarm 的设计哲学是克制的——它没有重新发明调度器，没有引入新的通信协议，甚至没有新增一个后台服务。它只是把 Hermes 已有的 SQLite 板子、已有的 profile 机制、已有的子进程能力，用 278 行代码串联成一个**可审计、可恢复、可组合**的协作协议。

在 v0.18 中，Kanban Swarm 的意义不只是「多了一个功能」。它标志着 Hermes Agent 从一个**个人助理**进化为一个**团队基础设施**——你可以定义角色（researcher、writer、reviewer、publisher），分配任务依赖，让 Dispatcher 自动编排，然后在需要人类决策时优雅地暂停等待。

如果 delegate_task 是你手里的瑞士军刀，那么 Kanban Swarm 就是你的自动化车间。

下一步？也许是一个能够自我复制的 Swarm，Orchestrator 动态调整任务图，Goal Mode 的 Judge 学会了「不完美但足够好」。多 Agent 协作的故事，才刚开始。

---

*本文基于 Hermes Agent v0.18.2 实际源码分析撰写。调研过程中分析了 9 个核心源文件，涵盖 kanban_db.py（8981 行）、kanban_swarm.py（278 行）、kanban_decompose.py（477 行）等核心组件。*
