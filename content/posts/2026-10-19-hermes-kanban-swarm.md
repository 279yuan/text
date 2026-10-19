---
title: "Hermes Agent v0.16 Kanban Swarm 功能深度解析"
date: 2026-10-19
draft: false
tags: ["hermes-agent", "multi-agent", "kanban", "ai-agents"]
---

## 开篇：一个让智能体"下班后继续干活"的问题

你有没有遇到过这样的场景：深夜你给一个 agent 派了个大活——"调研五个竞品、写三份报告、再让另一个 agent 审一遍"。结果第二天早上打开终端，发现因为你的电脑休眠、或者进程被 `/new` 清掉，整个任务树灰飞烟灭，连一句"我做到哪了"都问不出来。

这就是 Hermes Agent v0.16 引入的 **Kanban Swarm** 要解决的问题。它不是又一个"调用子智能体"的语法糖，而是一套把多智能体协作从"函数调用"升级为"持久化工作队列 + 状态机"的基础设施。这篇文章面向 AI 开发者，把它的原理、数据模型、状态机、编排模式和反模式一次讲透——所有默认值、字段名、CLI 动词都以官方文档为准。

## 一、定位：它和 delegate_task 到底差在哪

很多人第一次看到 Kanban 会问：我已经有 `delegate_task` 了，为什么还要一个看板？官方文档给了一句精准的区分：**`delegate_task` 是一次函数调用；Kanban 是一个工作队列，每一次交接都是一行任何 profile（或人）都能看到和编辑的记录。**

这张对照表是理解全篇的钥匙：

| 维度 | delegate_task | Kanban |
|---|---|---|
| 形态 | RPC 调用（fork → join） | 持久化消息队列 + 状态机 |
| 父任务 | 阻塞等待子任务返回 | `create` 之后 fire-and-forget |
| 子任务身份 | 匿名 subagent | 具名 profile，带持久记忆 |
| 可恢复性 | 无——失败即失败 | block → unblock → 重跑；崩溃 → reclaim |
| 人在回路 | 不支持 | 任意时刻 comment / unblock |
| 每任务智能体数 | 一次调用 = 一个 subagent | 任务生命周期内 N 个（重试/审查/跟进） |
| 审计 | 上下文压缩后丢失 | SQLite 永久落盘行 |
| 协调 | 层级（调用方 → 被调方） | 对等——任何 profile 可读写任何任务 |

**为什么重要**：`delegate_task` 的父进程一退出，子任务就丢了；它也不持久。而 Kanban 的每张卡是一行 SQLite 记录，崩溃了能被回收、卡住了能等人来 unblock、做错了能退回返工。选择标准很清晰——需要"短推理答案再继续、不涉及人类、结果回填父上下文"用 delegate_task；需要"跨智能体边界、跨重启存活、可能需要人类输入、事后可发现"用 Kanban。两者并非二选一：kanban worker 内部照样可以调用 `delegate_task`。至于 cronjob（持久化定时调度器，时间触发）和 tmux 多进程（无共享状态机的 ad-hoc 并行），它们各管一摊，不承担任务 DAG 编排。

## 二、数据模型：五个核心概念

Kanban 的世界由五个概念撑起，理解它们就理解了整套系统的骨架。

- **Board（看板）**：一个独立任务队列，自带 SQLite DB、workspaces 目录、dispatcher 循环。单安装可有多个 board（每项目/仓库/域一个）。新安装只有一个名为 `default` 的 board，DB 落在 `~/.hermes/kanban.db`（向后兼容）。
- **Card / Task（任务）**：一行记录，含 `title`、可选 `body`、一个 `assignee`（profile 名）、`status`、可选 `tenant`、可选 `idempotency_key`（重试自动化去重）。
- **Profile**：具名智能体身份，有独立配置（`~/.hermes/profiles/<name>/config.yaml`）与持久记忆。`assignee` 就是 profile 名；dispatcher 以 `hermes -p <assignee>` 拉起 worker，并注入 profile 作用域的 `HERMES_HOME`。
- **Dispatcher（调度器）**：长活循环，每 N 秒（默认 60）做四件事：回收过期 claim、回收崩溃 worker（PID 已消失但 TTL 未到）、把 ready 任务晋升、原子 claim 并 spawn 对应 profile。默认跑在 gateway 内（`kanban.dispatch_in_gateway: true`）。
- **Workspace（工作目录）**：worker 的操作目录，分三种形态（第四节细讲）。

**为什么重要**：Board 是硬隔离边界，Tenant 只是 board 内的软命名空间。Board 解析有固定优先级——CLI `--board <slug>` > `HERMES_KANBAN_BOARD` 环境变量 > `~/.hermes/kanban/current` 持久化 slug > `default`。slug 只允许小写字母数字 + 连字符 + 下划线、1–64 字符，斜杠/空格/点/`..` 在 CLI 层被拒绝，防路径穿越。跨 board 链接是不允许的——这是防呆设计，避免任务图悄悄跨越隔离域。

## 三、状态机：七个状态 + 一个终态

任务全状态集是 `triage | todo | ready | running | blocked | review | done | archived`，流转关系是理解整个系统行为的关键：

- **triage**：粗想法停放列。默认 `kanban.auto_decompose: true`，dispatcher 每 tick 对落入 triage 的任务跑 decomposer，产出子任务图并 fan-out；原任务成为所有子任务的 parent，`triage → todo`。
- **todo → ready**：依赖门开（所有 parent 为 done）后由 dispatcher 晋升。
- **ready → running**：dispatcher 原子 claim 并 spawn 对应 profile。
- **running → done / review / blocked**：分别对应 `kanban_complete`、`kanban_request_review`、`kanban_block`。
- **review → done / 回实现者 / blocked**：审查者 `kanban_complete` 批准；`kanban_request_changes` 退回返工；仅真·外部升级时 `kanban_block`。
- **blocked → 恢复安全源阶段**：审查来源且 parent 已完成 → `review`；实现工作且 parent 已完成 → `ready`；仍有 parent 未完成 → `todo`。unblock 绝不直接路由回 triage。

**为什么重要**：这里有一个特别精巧的熔断机制——**unblock-loop 熔断**。当 blocked → unblocked → 同因再次 block 达到 `BLOCK_RECURRENCE_LIMIT`（默认 2）次后，任务不再送回 blocked，而是路由到 triage 交人类裁决。这是确定性 DB 守卫，不是 LLM 判断；计数只在成功 complete 时重置。它防止了一个坏任务在"block → 人工解 → 又 block"之间无限空转。

## 四、workspace 三形态：scratch / dir / worktree

worker 在什么目录里干活，直接决定了文件的生命周期和协作方式：

- **scratch（默认）**：`~/.hermes/kanban/workspaces/<id>/` 下的全新临时目录，任务完成即删除。`kanban_complete(artifacts=[...])` 声明的文件会在清理前复制到持久 per-task 附件存储；缺失已声明的 scratch artifact 会让任务保持 in-flight，等你修正路径重试。
- **dir:\<path\>**：既有共享目录（Obsidian vault、邮件运维目录、per-account 目录）。必须是绝对路径，`dir:../tenants/foo/` 这类相对路径在派发时被拒绝。完成时保留，威胁模型是"受信本地用户、单主机"。
- **worktree**：`.worktrees/<id>/` 下的 git worktree，用于编码任务。`worktree:<path>` 钉死目标路径，worker 侧执行 `git worktree add`。完成时保留。

**为什么重要**：scratch 是"天生即弃"，适合一次性产出；dir 是"多人共享一块地"，适合要持续协作的场景；worktree 则是编码任务的标配，隔离分支又共享仓库。选错形态的代价是真实的——把需要人肉续改的文件丢进 scratch，任务一完成就没了。

## 五、依赖编排：fan-out / fan-in 的父子图

依赖由 `task_links` 表表达，`kanban_create(..., parents=[...])` 一次到位，或事后 `kanban_link(parent_id, child_id)` 补边。fan-out 是 orchestrator 用 `kanban_create` 产生多个并行子任务；fan-in 是子任务汇聚到一个合成任务。规则：cycle 与 self-link 被拒，跨 board 链接不允许。

**为什么重要**：官方有一条"编排铁律"——**在 fan-out 之前先做决定**。设计决策归 orchestrator，不归 worker。命名方案、schema、文件格式、API 形状这类"两张卡都要挑"的问题，必须由 orchestrator 拍板一次，写进**两张** card body。因为 worker 看不到兄弟卡的上下文，每个子卡 body 必须自带它所依赖的全部决策。另外还有一个新手必踩的坑：dispatcher 对未知 assignee 名是**静默丢弃**的——卡会永远停在 ready，没有任何报错。所以 orchestrator 落地每一张卡前，必须先用真实存在的 profile 名。

## 六、交接协议：summary / metadata / artifacts / created_cards

一次完整的交接长这样：

```python
kanban_complete(
    summary="1-3 句人类可读的收尾",
    metadata={"changed_files": ["a.py"], "tests_run": 12, "findings": [...]},
    created_cards=[...],  # 本 run 经 kanban_create 成功返回的子任务 id
    artifacts=["/abs/path/report.pdf"],
)
```

- `summary`：进 Run History，给人和下游读。
- `metadata`：机器可读的交接事实（`changed_files` / `tests_run` / `decisions` / `findings`），下游智能体、审查者、dashboard 可直接复用，不必刮散文。推荐的约定字段还有 `verification`、`dependencies`、`blocked_reason`、`retry_notes`、`residual_risk`。
- `created_cards`：内核校验每个 id 确实存在且由本 worker 创建，幽灵 id 会卡住 complete——这是防幻想的硬闸。
- `artifacts`：交付物绝对路径清单，gateway notifier 会把每个文件作为原生附件上传给订阅者（图片内联、其余作文件）。

**为什么重要**：metadata 是这条流水线的"机器接口"。没有它，下游智能体只能重读整段散文去猜你干了什么。红线也很明确——metadata 里不放 secrets、原始日志、token、OAuth 材料，只放指针和摘要。附件走 `kanban_attach`（base64 内联）或 `kanban_attach_url`（URL），25 MB 上限，存于 `<hermes-home>/kanban/attachments/<task_id>/`。

## 七、审查流水线：review 不是 block

`kanban_request_review(summary=..., metadata=..., reviewer=可选)` 把任务交审查，进 `review` 列并通知订阅者。审查者三种出口：批准 → `kanban_complete`；退回返工 → `kanban_request_changes(reason=...)`（关闭本轮审查、重施 parent 门控、路由回原实现者）；真正外部升级 → `kanban_block`。

**为什么重要**：审查**不是** block——它不计入 unblock-loop 检测。这意味着一个任务可以跨多次跟进循环反复审查，而永远不会被误升级到 triage。派发上，`review_dispatch: true`（默认）时 dispatcher 用内置 `sdlc-review` skill spawn 被分配 profile；`false` 则纯人工审查。

## 八、阻塞与恢复：block 的四种 kind

`kanban_block(reason=..., kind=...)` 按"为何卡住"路由：

- **dependency**：等另一个任务——进 todo，parent 完成后自动恢复，无需人类。
- **needs_input**：需要人类决策 / 回答。
- **capability**：硬墙——无访问权限、缺凭据、无智能体能做的动作。
- **transient**：可能自行消除的偶发故障。

`kanban_unblock(task_id)` 恢复到源阶段（review 或 ready），parent 仍开着则回 todo。

**为什么重要**：`kind` 是给 dispatcher 和人的信号，不是给自己看的备注。写错 kind 的后果很实际：一个该 dependency 的卡被你标成 needs_input，就会一直等人来点，而不是 parent 一完成就自动活过来。而同因反复 block 会触发第五节讲过的 unblock-loop 熔断，自动升级到 triage。

## 九、心跳与过期回收：为什么必须 heartbeat

`kanban_heartbeat(note=...)` 是纯副作用，报告存活。长操作每几分钟一次；若任务可能超过 1 小时，必须至少每小时一次。背后的回收逻辑是：dispatcher 回收"运行超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时 = 14400 秒）且最近 1 小时无心跳"的任务，判定为崩溃未清理。回收是良性的——任务回 ready 待重派，不计失败计数，但会丢失本次运行进度。

**为什么重要**：这是"崩溃可恢复"的代价与承诺。你只要按节奏 heartbeat，一个跑 3 小时的任务崩溃后能被干净地重派；你不 heartbeat，它就被当死进程回收，之前的进度白跑。另外两道防线：`failure_limit`（默认 2）——同一任务连续 spawn 失败达阈值自动 block；协议违规（protocol violation）——worker 退出码 0 但任务仍 running 时，先注入 2 次合成 nudge 提醒补 `kanban_complete`/`kanban_block`（`HERMES_KANBAN_STOP_NUDGE=0` 关闭），dispatcher 侧连续干净退出违规有界重试 `_PROTOCOL_VIOLATION_FAILURE_LIMIT`（默认 3）后自动 block，而非无限重 spawn。

## 十、Orchestrator 模式：正模式与反模式清单

一个合格的 orchestrator 不做实现工作——它把用户目标分解成任务、加链接、逐个 assign 给已设好的 profile，然后退后。内置自动分解器（triage auto-decomposer）用 `auxiliary.kanban_decomposer` 模型，读 profile 名册（含描述），产出 JSON 任务图；`kanban.auto_decompose_per_tick` 默认每 tick 3 个封顶防 burst。decomposer 绝不产出 `assignee=None`；LLM 选了未知 profile 时，子卡路由到 `kanban.default_assignee`（未设则活跃默认 profile）。`kanban.orchestrator_profile` 只决定 fan-out 后谁拥有根任务，**不**加载该 profile 的 prompt/skills。想跑 goal 循环用 `goal_mode`——每 turn 一个辅助 judge 对照卡 title+body 判定是否完成，未完成且预算未耗尽则同一 session 继续，预算耗尽则 block 该卡交人工审查（`goal_max_turns` 默认 20）。

**为什么重要**：这里是反模式的重灾区，官方列出的坑值得逐条背下来：

1. orchestrator 亲自执行实现工作——应把 toolsets 限到 board 操作（`kanban, gateway, memory`），从机制上断掉执行能力。
2. 未知 assignee——dispatcher 静默丢弃，卡永远 ready。
3. 两张并行卡各自独立决定同一问题——文件格式/命名/schema 两半永远对不上，orchestrator 必须提前拍板写进双方 body。
4. 无 gateway 运行时，ready 卡原地不动（`hermes kanban create` 会在创建时警告）。
5. 同时跑 gateway 内嵌 dispatcher 与独立 daemon 会 claim 竞争，不受支持（`hermes kanban daemon` 已弃用）。

## 十一、工具集分层：谁看得见什么

零 schema 足迹是刻意设计：普通 `hermes chat` 会话的 schema 里没有任何 `kanban_*` 工具，除非该 profile 显式启用 `kanban` toolset。dispatcher-spawned worker 因 `HERMES_KANBAN_TASK` 被设而获得 **task-scoped** 的 kanban 工具，不能改无关任务；orchestrator 通过配置获得更宽路由面（`kanban_list`、`kanban_create`、`kanban_link`、`kanban_unblock`、对外卡 `kanban_comment`）。完整工具集共 15 个：`kanban_show`、`kanban_list`、`kanban_complete`、`kanban_request_review`、`kanban_request_changes`、`kanban_block`、`kanban_heartbeat`、`kanban_comment`、`kanban_attach`、`kanban_attach_url`、`kanban_attachments`、`kanban_create`、`kanban_link`、`kanban_unblock`、`kanban_schedule`。

**为什么重要**：worker 不 shell 出 `hermes kanban`，而是模型直接调工具、工具经 Python `kanban_db` 层读写 board——三个原因：远程终端后端可移植、无 shlex/argparse 引号脆弱性、结构化错误更好。worker 生命周期也被自动注入（无需安装/配置）：spawn 时 `kanban_show()` 读上下文 → `cd $HERMES_KANBAN_WORKSPACE` 工作 → 长操作间 `kanban_heartbeat` → 收尾 `kanban_complete` 或卡住 `kanban_block`。

## 十二、可操作配置与反模式速查

`config.yaml` 的 `kanban` 段关键默认值（这是你调参时的地图）：

| Key | 默认 | 用途 |
|---|---|---|
| `kanban.dispatch_in_gateway` | `true` | 调度器内嵌 gateway |
| `kanban.dispatch_interval_seconds` | `60` | 调度 tick 间隔 |
| `kanban.review_dispatch` | `true` | 审查任务用内置 sdlc-review skill spawn |
| `kanban.failure_limit` | `2` | 连续 spawn 失败自动 block 阈值 |
| `kanban.dispatch_stale_timeout_seconds` | `14400`（4h） | 无心跳回收阈值 |
| `kanban.auto_decompose` | `true` | 对 triage 自动跑 decomposer |
| `kanban.auto_decompose_per_tick` | `3` | 每 tick 分解封顶 |
| `kanban.orchestrator_profile` | `""` | 分解后根任务归属 profile（空=活跃默认） |
| `kanban.default_assignee` | `""` | 未知 profile 兜底 assignee |
| `kanban.auto_subscribe_on_create` | `true` | create 后终态事件唤醒发起智能体 |
| `kanban.done_sub_retention_days` | `30` | done 订阅保留天数，0 关闭清理 |

配套环境变量：`HERMES_KANBAN_TASK`、`HERMES_KANBAN_BOARD`、`HERMES_KANBAN_WORKSPACE`、`HERMES_KANBAN_ATTACHMENTS_ROOT`、`HERMES_KANBAN_DISPATCH_IN_GATEWAY`、`HERMES_KANBAN_STOP_NUDGE`；`BLOCK_RECURRENCE_LIMIT` 默认 2。

面向人类的 CLI 动词（模型不用，脚本/cron/人用）：`init` `create` `list`（alias `ls`）`show` `assign` `link` `unlink` `comment` `complete` `block` `unblock` `archive` `tail`；次常用 `watch` `stats` `runs` `log` `dispatch` `daemon` `gc`；boards 相关 `boards list / create / switch / show / rename / rm`。

## 结尾：把它当成"会落盘的团队"

回到开篇那个问题。Kanban Swarm 的本质，是把"给 agent 派活"这件事从一场脆弱的函数调用，变成了一本会落盘、可恢复、可审计、可人工介入的团队台账。你不再需要担心进程退出丢任务，不再需要靠上下文压缩去回忆"做到哪了"，也不再需要人肉盯着每一个 worker——状态机、依赖门、回收器、审查流水线把这些都接管了。

对 AI 开发者来说，真正要内化的不是那 15 个工具名，而是三件事：**board 是硬隔离、决策在 fan-out 之前拍板、review 不是 block**。把这三条吃透，你就能用它搭出跨重启存活、跨智能体协作、人可在任意时刻插手的真正自治流水线。
