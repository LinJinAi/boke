---
title: Hermes Agent 多智能体 Kanban 编排机制：卡片、Profile、Dispatcher 与串行 parents 链
date: 2026-09-13
tags: ["Hermes Agent", "AI Agent", "多智能体编排"]
author: LinJinAi
---

你有没有过这种体验：一个想法在周五晚上诞生，周一早上你希望它已经被拆成卡片、查完资料、写完初稿、审完校、发上线？Agent 编排的常见回答是「加一层多智能体协作」。但多数实现里，协作只停留在内存——进程一挂，中间交接就丢了。Hermes Agent 的 Kanban 把这件事反过来做：协作的每一次交接都是数据库里的一行，任何 profile、任何时刻都能看见，甚至直接改。

本文只讲已经核实到的机制细节。选题原始的「v0.16」措辞在公开 release notes 里查不到对应内容，因此标题不含版本号；正文提到 Swarm 首发时，写的是 v0.15.0（2026-05-28），不写 v0.16。

## 1. 一个 Kanban 卡是什么

Kanban 不是一块可视化看板，它是一个持久队列，恰好也长着一块看板的样子。任务板落在一个 SQLite 文件里，两个入口共享同一个 DB 层：

> The board has two front doors, both backed by the same ~/.hermes/kanban.db

人类和脚本走 `hermes kanban …` CLI 或 `/kanban` 斜杠命令；Agent 走专用的 `kanban_*` 工具集——模型直接读表和路由任务，不需要 shell 出去调 CLI。

一张卡就是一行记录：

> **Task** — a row with title, optional body, one assignee (a profile name), status (triage | todo | ready | running | blocked | review | done | archived), optional tenant namespace, optional idempotency key

值得注意的设计约束有两个：一是「one assignee」——一张卡只能派给一个 profile，多角色协作必须拆成多张卡；二是状态机有 8 个状态，triage、todo、ready、running、blocked、review、done、archived，其中 review 和 blocked 的语义差异后文要专门展开。

除了卡本身，还有两个原语：

> **Link** — task_links row recording a parent → child dependency. The dispatcher promotes todo → ready when all parents are done.
>
> **Comment** — the inter-agent protocol. Agents and humans append comments; when a worker is (re-)spawned it reads the full comment thread as part of its context.

Link 是依赖关系，Comment 是协作协议。Comment 的关键在于「worker 被（重新）拉起时读完整评论线程作为上下文的一部分」——这让它成为跨会话的隐性记忆，而不只是留言。

**为什么重要**：把「谁做了什么、下一棒是什么」固化成数据库行，而不是模型上下文里的字符串，你就获得了断点续跑能力。这是 Kanban 区别于所有「进程内多智能体」框架的根本点。

## 2. Dispatcher：唯一负责「派活」的循环

> **Dispatcher** — a long-lived loop that, every N seconds (default 60): reclaims stale claims, reclaims crashed workers (PID gone but TTL not yet expired), promotes ready tasks, atomically claims, spawns assigned profiles. Runs inside the gateway by default (kanban.dispatch_in_gateway: true). One dispatcher sweeps all boards per tick; workers are spawned with HERMES_KANBAN_BOARD pinned so they can't see other boards. After kanban.failure_limit consecutive spawn failures on the same task (default: 2) the dispatcher auto-blocks it with the last error as the reason

一段话里五个动作：回收过期认领、回收崩溃 worker（PID 没了但 TTL 未过期）、提升 ready、原子认领、拉起目标 profile。默认跑在 gateway 进程内，一个 dispatcher 扫所有 board；worker 通过固定的 `HERMES_KANBAN_BOARD` 环境变量被限制只能看到自己的 board；同一任务连续 2 次 spawn 失败会自动 block，理由就是最后那次错误。

**为什么重要**：「原子认领」和「连续失败自动 block」是 Kanban 能无人值守的前提。前者避免两个 worker 抢同一张卡，后者避免同一张坏卡把 dispatcher 打成死循环。

## 3. Worker 的生命周期：四步协议

每个跑 Kanban 任务的 profile 自动获得 worker 生命周期——它被注入到 worker 系统提示的 `KANBAN_GUIDANCE` 块里，没有任何技能需要安装：

> Every profile that works kanban tasks automatically gets the worker lifecycle — it's injected into the worker's system prompt at spawn (the KANBAN_GUIDANCE block), so there is nothing to install or configure.

协议本体就四步：`kanban_show()` → `cd $HERMES_KANBAN_WORKSPACE` → 长任务期间 `kanban_heartbeat(note=...)` → `kanban_complete(...)` 或 `kanban_block(...)`。heartbeat 频率有明确规则：

> Call kanban_heartbeat(note="...") every few minutes during long operations. If your work may run longer than 1 hour, call kanban_heartbeat at least once an hour — the dispatcher reclaims tasks that have been running past kanban.dispatch_stale_timeout_seconds (default 4 h) with no heartbeat in the last hour

超过 1 小时的任务每小时至少一次 heartbeat；4 小时无心跳会被判定为崩溃并 reclaim。

**为什么重要**：heartbeat 是 worker 与 dispatcher 之间的存活信号，也是任务在长时训练、批量编码这类场景下不被误杀的唯一手段。

## 4. 三种合法的终结方式：review 不是 block

每次 claim 必须以三种之一收场，这是 worker lanes 文档里写死的约束：

> kanban_request_review(summary=..., metadata=..., reviewer=...) — same-card implementation is complete and enters first-class review; status flips to review.
>
> kanban_block(reason=...) — task waits for human input, status flips to blocked. The dispatcher respawns when kanban_unblock runs.

关键区别：review 是**非阻塞**的独立状态，不影响 block-loop 计数；block 才是真正等人工输入。文档在描述 request_review 时点明了这一点：「it is not a block and does not affect block-loop accounting」。

选型还有一个容易踩的坑——任务图里如果已经预建了下游 review/QA/release 子卡：

> Pre-created downstream review/QA/release card: kanban_show lists child IDs; inspect those cards with kanban_show(task_id=...) before choosing the terminal action. When a child is the downstream review/QA/release phase, call kanban_complete on the implementation phase. It cannot promote until this parent is done/archived. Do not additionally request same-card review and never sticky-block the parent with review-required: — either choice strands or duplicates the downstream lane.

父卡 done 之后子卡才会自动 promote，此时再对父卡调 `kanban_request_review` 就会 strand 或重复下游 lane。

**为什么重要**：把 review 当 block 用，会触发 unblock-loop 的误判升级；有下游子卡时又用 same-card review，下游 lane 直接卡住。这是两个最容易在真实流水线里翻车的分支。

## 5. 工具集与 delegate_task 的分工

> kanban | kanban_attach, kanban_attach_url, kanban_attachments, kanban_block, kanban_comment, kanban_complete, kanban_create, kanban_heartbeat, kanban_link, kanban_list, kanban_request_changes, kanban_request_review, kanban_show, kanban_unblock | Multi-agent coordination tools. Registered for dispatcher-spawned task workers (HERMES_KANBAN_TASK) ... delegate_task children are not Kanban run owners: their schema strips/disables this toolset and runtime guards reject direct board mutations, even if parent HERMES_KANBAN_* env vars are present.

数一数，一共 14 个 `kanban_*` 工具。编排者（orchestrator）额外拿到 `kanban_list` 和 `kanban_unblock`；而 `delegate_task` 的子 agent **拿不到**这个工具集——它们的 schema 直接剥离，运行时守卫会拒绝直接改动看板，即使父进程里 `HERMES_KANBAN_*` 环境变量都存在。

Kanban 与 `delegate_task` 的关系是分工，不是替代：

> **One-sentence distinction:** delegate_task is a function call; Kanban is a work queue where every handoff is a row any profile (or human) can see and edit.

表格里的两行最能说明问题：

> | Resumability | None — failed = failed | Block → unblock → re-run; crash → reclaim |
>
> | Human in the loop | Not supported | Comment / unblock at any point |

`delegate_task` 是一次同步 RPC（fork→join）：子 agent 匿名、失败不可恢复、上下文随压缩消失；Kanban 是持久队列加状态机：profile 有名有姓、可 block/unblock、失败可 reclaim、有完整审计行。两者可以并存——一个 Kanban worker 的任务里完全可以调 `delegate_task`，反之不行。`delegate_task` 默认最多 10 个并发子 agent，可配 JSON Schema 做结构化输出校验，失败时父进程校验一次，只给一次有边界的纠正回合。

Profile 与 dispatcher 的关系值得单独点一句：assignee 就是 profile 名，每个 profile 有自己的 `~/.hermes/profiles/<name>/config.yaml`，dispatcher 拉起 worker 时注入 profile 作用域的 `HERMES_HOME`——这就是「orchestrator 跑前沿模型、worker 跑廉价模型」这条成本策略的技术基础：

> Run your orchestrator/dispatcher profile on a frontier model and point worker profiles at inexpensive models.

按任务挂技能也很直接——`kanban_create(..., skills=["translation"])`，dispatcher 会为每个技能发一个 `--skills <name>` 参数，worker 启动时全部加载；但技能名必须已经装在 assignee 的 profile 上，没有运行时安装。

## 6. 并行与串行：parents 链不只是调度门

文档给了几种协作模式，最典型的两种：

> **P1 Fan-out** | N siblings, same role | "research 5 angles in parallel"
>
> **P2 Pipeline** | role chain: scout → editor → writer | daily brief assembly

Fan-out 是 N 个同角色 sibling 并行；Pipeline 是角色链串行。真正的重头戏在 `--parent` 的语义——它同时是调度门和上下文交接通道：

> A parent link is not just a scheduling gate — it is the context handoff channel from a completed card to a new one.
>
> The parent's handoff rides along. The worker context assembled for the child (build_worker_context, what kanban_show() returns) contains a ## Parent task results section with each parent's completion summary and metadata, verbatim
>
> This is why the pattern for follow-up work on a finished card is a new child card, not reopening the done card. Completed cards are immutable history — their context flows forward through the parent link.

父卡全 done 时，子卡创建即 ready；子卡 worker 的上下文里会带一段 `## Parent task results`，逐字包含每个父卡的 `summary` 和 `metadata`。所以「在已完成卡上追加后续工作」的正确姿势是新建子卡，done 卡是不可变历史。

```python
# 串行 parents 链：researcher → writer → reviewer
writer = kanban_create(
    title="写作：多智能体 Kanban 编排",
    assignee="writer",
    body="写约 2000 字技术文章，遵循上游调研报告的核实结论",
    parents=[researcher.id],
    skills=["blog-writing"],   # 必须已装在 writer profile 上
)
# writer 完成（kanban_complete）后，下游 reviewer 卡才会从 todo 自动 promote 到 ready
# reviewer 的 kanban_show() 里会带 ## Parent task results 段，
# 逐字包含 writer 的 summary 和 metadata
```

**为什么重要**：这条链解决了多智能体协作里最难的两个问题——中间产物不丢、上下文不压缩。它替代的是「在内存里传 message」的做法，代价是每次交接都落库一次。

## 7. Kanban Swarm v1 与「无第二调度器」

Swarm v1 首发于 v0.15.0（2026-05-28，The Velocity Release），官方定位是「Kanban grew into a real multi-agent platform across 104 PRs」。一条命令生成一张完整图：

> hermes kanban swarm creates a durable Kanban Swarm v1 graph in one shot: a completed root/blackboard card, N parallel worker cards, a verifier card gated on all workers, and a synthesizer card gated on the verifier. Shared swarm context (the "blackboard") is stored as structured JSON comments on the root card so any worker can read it.

拓扑是 root（立即完成、承载 blackboard）→ N 个并行 worker → verifier（等所有 worker）→ synthesizer（等 verifier）。blackboard 是 root 卡上结构化 JSON 形式的 comments，所以任何 worker 都能读。

源码文件头注释点出了整个 Swarm v1 的设计哲学——「刻意不加第二个调度器」：

> Kanban Swarm v1: thin swarm topology helpers on top of Kanban. Deliberately no second scheduler — a small task graph written into the existing Kanban kernel:
>
> The shared blackboard is structured JSON comments on the root task, so all state lives in existing task_comments/task_events rows and the dashboard, notifier, slash command and dispatcher keep working without a new service.

blackboard 就是 root 任务上的结构化 JSON comments，所有状态落在既有的 `task_comments` / `task_events` 行里，看板、通知器、斜杠命令、dispatcher 都不用改，直接能读，不需要新服务。另外这张图是原子性提交的——「dispatchers and dashboard readers see either no new swarm or the complete topology, never a partially linked root/worker/verifier graph」。

一个交叉印证：Swarm v1 相关的 open issue #35600 里写着 `The kanban swarm system (v1, shipped in v0.15.0) is Hermes's most powerful multi-agent orchestration primitive`。该 issue 仍未合并，仅作交叉印证，不是官方引用。

**为什么重要**：「无第二调度器」意味着 Swarm 只是一层薄薄的拓扑 helper，而不是又一套编排引擎。这是它能在不破坏 Kanban 内核的前提下引入并行 + gate + 综合的经典模式的原因。

## 8. 工作区与 goal-mode：两个容易忽略的旋钮

工作区有三种类型，保留策略差异很大：

> scratch (default) — fresh tmp dir under ~/.hermes/kanban/workspaces/<id>/ ... Deleted when the task completes — scratch is ephemeral by design. Files explicitly declared through kanban_complete(artifacts=[...]) are copied into durable per-task attachment storage before cleanup
>
> dir:<path> — an existing shared directory ... Must be an absolute path. ... Preserved on completion.
>
> worktree — a git worktree under .worktrees/<id>/ for coding tasks. ... Preserved on completion.

scratch 默认完成即删，除非用 `kanban_complete(artifacts=[...])` 显式声明的产物会被复制到持久附件存储；dir 和 worktree 都保留。

goal-mode 是另一个旋钮：

> By default each worker gets one shot at its card ... Pass --goal (CLI) or goal_mode=True (the kanban_create tool / dashboard) to instead run that worker in a goal loop ... after every turn an auxiliary judge checks the worker's output against the card's title + body (treated as the acceptance criteria)
>
> or the budget runs out (which blocks the card for human review rather than exiting silently)

默认每个 worker 只有一次机会；加 `--goal` 就走 goal loop，每轮由一个辅助 judge 拿卡片 title + body 当验收标准来评。目标达成就继续；预算耗尽会 **block** 卡等人看，而不是静默退出。

## 9. 选型判断

| 场景 | 方案 |
| --- | --- |
| 一次性探索，结果可丢弃 | `delegate_task` |
| 需要持久、可恢复的跨日工作流 | Kanban 串行 parents 链 |
| 需要结构化 fan-out + gate + 综合 | `hermes kanban swarm` |
| 需要验收标准循环直到达成 | `kanban_create(goal_mode=True)` |

三个最常见的错误用法：把 review 当 block 用；有预建下游 review 卡时在父卡上再调 `kanban_request_review`；在已完成卡上追加工作而不是新建子卡。

## 参考来源

1. https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
2. https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes
3. https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference
4. https://hermes-agent.nousresearch.com/docs/reference/tools-reference
5. https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation
6. https://github.com/NousResearch/hermes-agent/releases/tag/v2026.5.28
7. https://github.com/NousResearch/hermes-agent/blob/main/hermes_cli/kanban_swarm.py
8. https://github.com/NousResearch/hermes-agent/issues/35600 （open，未合并；仅用于交叉印证 Swarm v1 首发版本）
