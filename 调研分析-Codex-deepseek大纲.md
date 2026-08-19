# 调研分析：常驻总调度 Agent（Hermes）主导的多 Agent 软件协作规范

> 任务：Hermes 总调度指定的规范研究员（子任务 A）产出
> 基线：本仓库 V2.4（`README.md`、`文档规范/07-子Agent任务委派与提示词规范.md`、`文档规范/09-调度控制平面与运行时台账规范.md`）
> 调研对象：AGENTS.md、ACP、LangGraph Supervisor、Claude Code Subagents、OpenAI Agents SDK/Swarm 与 Anthropic orchestrator-worker 经验
> 结论定位：供 Hermes 迁移与规范重构（V3.x）使用，属于"调研 + 对比 + 建议"，不直接改规范正文
> 日期：2026-08-19

---

## 0. 摘要（TL;DR）

1. **Hermes 的定位**：常驻（resident）总调度 Agent = 一个长期运行的控制循环 + 持久状态层 + 执行机制适配层，而不是 V2.4 所假设的"按需启动的 Coordinator 任务"。这是本次对比的总纲。
2. **五个调研对象各有可取层**：
   - AGENTS.md = 被动上下文层（Context L1 的标准化载体，不能当控制面）；
   - ACP = 主机↔Agent 会话/传输协议层（适配 `CreationRequestId/Provisioning/ThreadBound` 生命周期，不能解决调度）；
   - LangGraph Supervisor = 显式图编排与持久化语义（checkpoint/重放/中断，可借鉴状态机与幂等重放设计，不迁移其进程内运行时）；
   - Claude Code Subagents = 角色化 worker 的落地形态（上下文隔离、worktree 隔离、后台执行、工具权限边界）；
   - OpenAI Agents SDK/Swarm + Anthropic 工程经验 = handoff/guardrail/tracing 与"子结果直写文件系统、外部记忆、并行派发"等运行时经验。
3. **对 09 共享 JSON + 轮询的结论**：**在 Hermes 常驻调度下适用，但需改造**——保留"共享 JSON 为唯一实时真源 + `RecordID/SignalRevision` 幂等消费 + 定向读取"；把"周期巡检任务"改写为"常驻调度循环的 idle 轮询 + 可选事件加速通道"；明确事件加速不得替代持久状态（与 09 §3.3 禁止推断一致）。
4. **对比输出**：裁剪 4 项、补漏 6 项、重写 2 项、修改 5 项（详见 §2.3–§2.6 与 §2.7 总表）。

---

## 1. 开源调研

### 1.1 调研范围与方法

| # | 对象 | 类型 | 出处 |
|---|---|---|---|
| 1 | AGENTS.md | 开放文件规范（Linux Foundation / Agentic AI Foundation 托管） | agents.md；openai/agents.md；60k+ 仓库、20+ 工具采用 |
| 2 | ACP（Agent Client Protocol） | 主机↔Agent 的 JSON-RPC/stdio 会话协议 | agentclientprotocol.github.io；agentclientprotocol/agent-client-protocol |
| 3 | LangGraph Supervisor | 多 Agent 编排框架/模式（含 langgraph-supervisor、swarm） | langchain-ai.github.io/langgraph；langgraph-supervisor 文档 |
| 4 | Claude Code Subagents | 编码 Agent 的子代理机制 | code.claude.com/docs；anthropics/claude-code 仓库与社区实践 |
| 5 | OpenAI Agents SDK / Swarm + Anthropic orchestrator-worker | 编排 SDK 与生产级多 Agent 工程经验 | openai.com（Agents SDK/Swarm）；anthropic.com/engineering/multi-agent-research-system |

筛选标准：与"常驻总调度 Agent 主导多 Agent 软件开发"直接相关；覆盖"文件/协议/框架/运行时/经验"五个层面；均有公开可查的规范文本或生产案例。

### 1.2 AGENTS.md——被动上下文层（"给 Agent 的 README"）

**核心机制**
- 仓库根 `AGENTS.md`，可用 Markdown 任意分节（setup/build/test/风格/PR 指引），**无强制字段、无 schema**；被视为"活的文档"。
- **嵌套与优先级**：子目录可放自己的 `AGENTS.md`，Agent 自动读取"最近的"一份；**离被编辑文件最近者优先；用户当前显式指令优先级最高**。大型 monorepo 可放数十份（OpenAI 主仓库 88 份）。
- **v2 增强**：`@path` 导入/引用、glob 包含/排除（`@docs/*.md`）、Markdown 链接引用，支持按文件粒度组装上下文，而非整目录灌入。
- 采用面广（Codex、Copilot、Cursor、Claude Code、Gemini CLI、Amp、Factory、Aider 等），格式稳定。

**为何适合 Hermes**
- Hermes 需要一份"所有 worker 共享、位置可预测"的仓库级操作说明。AGENTS.md 正是 Context L1 的**标准化落点**，且"closest-wins + 用户指令优先"给了明确的冲突消解规则。
- 它**只解决上下文，不解决调度**：无状态、无事件、无门禁——不能替代 09，但可以成为 09 之上被所有角色引用的地图层。

**对 V2.4 的提示**：README §9 已写"Context L1 可复用现有 AGENTS"，但未定义文件位置、嵌套规则、冲突优先级——是明确的补漏点（§2.4-1）。

### 1.3 ACP（Agent Client Protocol）——主机↔Agent 会话协议层

**核心机制**
- JSON-RPC over stdio，双方角色：**Host（IDE/编排方）与 Agent（执行方）**；协议 v0.4.x，官方 TypeScript SDK，Gemini CLI 为生产实现。
- 生命周期方法：`initialize`（版本协商 + 能力通告）→ `session/new`（创建会话）→ `session/prompt`（下发任务）→ `session/cancel`（取消）；通知：`session/update`、`session/agent_message`、`session/agent_thought`（流式思考）、`session/config_change`（权限/配置变更）、`session/status_update`。
- `_meta` 保留扩展位：不假设键值语义，允许实现自定义元数据。
- 提供的是**会话/传输契约**，不含任务状态机、依赖、验证等级或合并门禁。

**为何适合 Hermes**
- 09 §3.1 的线程生命周期（`NotCreated/Provisioning/Running/Idle/NeedsAttention/Completed`）与 ACP 会话生命周期（create→provision→prompt→update/cancel）**一一对应**；09 中"只返回 clientThreadId 视为 Provisioning 成功"正是 ACP 式两段式创建的语义。
- 常驻 Hermes 需要一个"与任何执行载体解耦"的适配层：把 Codex 子线程、subagent、人工执行统一建模成 ACP 式会话，Hermes 只面向 `ExecutionThreadID + 会话方法`，不绑定具体机制——这与 07 §2.1 `ExpectedExecutionKind` 的理念一致，可形式化为协议适配层。
- 局限：ACP 无调度、无持久台账、无多 Agent 依赖；只能当"执行机制适配契约"，不能当控制面。

### 1.4 LangGraph Supervisor——显式图编排与持久化语义

**核心机制**
- `StateGraph`：节点 = Agent/工具，边 = 控制流；**supervisor 节点**集中路由到专门 worker（`create_supervisor`、`create_handoff_tool`）；另有 swarm（worker 间对等 handoff）、hierarchical teams（多级 supervisor）等模式。
- **显式状态 schema**：图的共享状态有类型定义，是"可审计的单一状态面"。
- **checkpoint 持久化**：带 `thread_id` 的 checkpointer，可中断、恢复、时间旅行重放。
- **interrupt / human-in-the-loop**：暂停点显式化，等待人工输入后继续。
- 进程内库：编排逻辑与运行时同进程，跨异构执行载体（外部 Agent）能力弱。

**为何适合 Hermes**
- supervisor 模式 = "总调度"的教科书形态：中央节点决定派发，worker 只做专门工作；层级化 supervisor 对应 Hermes + 迭代级协调者。
- **可借鉴的不是框架而是语义**：① 显式状态 schema（09 §5.4 任务级字段可视为其文件化版本）；② checkpoint/重放（映射 09 §13 热/冷/灾难恢复的"检查点 + 最大安全状态重放"）；③ interrupt（映射 09 §12 暂停/租约）。
- 局限：LangGraph 的持久化在进程内运行时，Hermes 面对的是"跨会话、跨 worktree、跨执行载体"的常驻调度，共享 JSON + 文档快照的双层持久化反而更贴合；不要照搬其运行时。

### 1.5 Claude Code Subagents——角色化 worker 的落地形态

**核心机制**
- 自定义子代理 = `.claude/agents/*.md`，frontmatter 声明：`name/description/tools/disallowedTools/model/permissionMode/maxTurns/skills/mcpServers/hooks/memory/background/isolation`。
- **description 驱动路由**：父代理按任务与子代理描述匹配自动委派；子代理拥有**独立上下文窗口 + 独立系统提示 + 独立工具/权限约束**；可嵌套、可并行。
- **后台执行**（v2.1.198+ 默认）：父会话保持响应，完成后回收结果；**worktree 隔离**（`isolation: worktree`）：在专用 git worktree 运行，无改动自动清理，有改动保留分支。
- 局限性：委派是模型启发式（描述匹配），无正式状态机、无依赖图、无验证门禁；父代理作为唯一汇聚点。

**为何适合 Hermes**
- 它是 07 "角色契约"的工程化样本：把"角色 = 系统提示 + 工具白名单 + 权限模式 + 停止条件"落到可声明文件——Hermes 的提示词模板目录可对齐此结构。
- worktree 隔离 = 07 §5.1 Git 权限（任务分支/独立 worktree）的机制实现；背景执行 = 常驻调度的"派发后不阻塞"。
- 局限：Hermes 不是"在一个父会话里描述匹配"，而是显式派发 + 状态跟踪，所以只借鉴 worker 形态，不借鉴自动路由。

### 1.6 OpenAI Agents SDK / Swarm 与 Anthropic orchestrator-worker 经验

**核心机制**
- **OpenAI Agents SDK**：Agent（指令+工具）+ **Handoff**（显式控制转移）+ **Guardrails**（输入/输出双侧校验）+ **Tracing/Observability**（执行轨迹可视化）；Swarm 为其轻量原型（`transfer_to_XXX` 函数、context variables）。
- **Anthropic 多 Agent 研究系统经验**：orchestrator-worker 架构（计划→并行派发→汇总）；**子代理结果直写文件系统**以减少"传话游戏（game of telephone）"；**阶段完成摘要写入外部记忆**以应对上下文上限；**从错误点恢复**；对并行 worker 的 token 成本与失败重试做工程控制。

**为何适合 Hermes**
- Handoff = 07/09 中"派发后由子任务写状态、巡检消费"的控制转移语义；Guardrail = 09 §8 交付校验（协议头/Invocation/HeadSHA）的输入输出双侧化建议。
- Tracing = 09 §6 派生事件日志（events.jsonl）的可观测目标；"结果直写文件系统"是 09 目前**缺失**的一环（状态 JSON 只应存指针，长证据内容应落盘——见 §2.4-3）。
- 局限：Swarm 无状态、无持久化，SDK 面向 API 编排，均不面向"常驻调度 + 人为参与的门禁流程"，只取其概念。

### 1.7 横向对比表

| 维度 | AGENTS.md | ACP | LangGraph Supervisor | Claude Code Subagents | OpenAI SDK/Swarm + Anthropic 经验 |
|---|---|---|---|---|---|
| 定位 | 被动上下文文件 | 会话/传输协议 | 图编排框架（进程内） | 角色化 worker 机制 | 编排 SDK + 生产经验 |
| 是否有持久状态 | 无 | 会话级 | 有（checkpoint/thread_id） | 无（会话级） | 无/平台 tracing |
| 是否有调度/依赖/门禁 | 无 | 无 | 有（图 + supervisor 路由，无业务门禁） | 无（描述匹配路由） | handoff 无门禁 |
| 幂等/恢复语义 | — | 无 | 重放/中断/时间旅行 | 无 | 无 |
| 对 Hermes 的最大价值 | Context L1 标准载体 + 冲突消解规则 | 执行机制适配契约（会话生命周期） | 显式状态 schema + checkpoint 语义 | worker 角色化 + worktree/上下文隔离 | handoff/guardrail/tracing + 结果落盘经验 |
| 不可照搬处 | 不能当控制面 | 不含调度与台账 | 进程内运行时 | 启发式路由、无状态机 | 无持久化、无人工门禁 |

### 1.8 对 Hermes 的启示汇总（可直接借用清单）

1. **地图层**：以 AGENTS.md 作为 Context L1 标准入口，明确 closest-wins 与"用户指令 > 就近 AGENTS.md > 远程文档"的冲突优先级。
2. **协议层**：把执行载体统一建模为 ACP 式会话（new→prompt→update→cancel），`CreationRequestId`→`ExecutionThreadID` 两段式绑定；`config_change` 对应权限/模型变化通知。
3. **状态层**：保留 LangGraph 式"显式 schema + 检查点重放"语义，落地为 09 的共享 JSON + 文档快照 + 最大安全状态恢复。
4. **worker 层**：角色模板对齐 subagent frontmatter（tools 白名单、permissionMode、worktree isolation、background）；派发后不阻塞。
5. **经验层**：子结果直写证据文件（反传话游戏）、阶段摘要外部记忆（对应 06 对账）、guardrail 双侧校验、tracing 审计、并行成本与失败重试控制。

---

## 2. 与 V2.4 对比

### 2.1 基线要点回顾（对比输入）

- **README（V2.4 待审核）**：§2.10 文档状态驱动调度、§2.11 持久可恢复；§10 Codex 协作规则 8–11 要求共享本地 JSON、统一原子工具、Invocation、幂等键、不可变基线、协调租约、RecoveryOnly；§13 DOC-DEC-014~020 为 09 相关待审核决定。
- **07**：角色按需选择、`ExpectedExecutionKind` 显式配置、提示词分层（公共基座 + 角色增量）、角色独立性、总调度最低规则（§8）、派发前清单（§8.1）、状态生产/消费与周期巡检（§8.2）、Review 返工循环升级（§8.4）。
- **09**：三类状态分离（ThreadStatus/TaskState/EvidenceState）、不可变执行基线、`SharedRuntimeStatePath` 唯一真源 + `CanonicalTaskDocumentPath` 快照、派生审计缓存、事件与幂等（`DispatchKey`、`RecordID+SignalRevision`）、创建与消费协议、周期巡检（§11）、暂停/租约（§12）、三级恢复（§13）、Canary（§14）。

### 2.2 重点评估：09 共享 JSON + 轮询在 Hermes 常驻调度下是否适用

**结论：适用，但必须从"任务式周期巡检"改造为"常驻调度循环的持久状态层"。**

| 09 现有设计 | 常驻 Hermes 下的判断 | 建议 |
|---|---|---|
| 共享本地 JSON 为唯一实时真源 | **保持**：常驻下更不能依赖聊天上下文/UI 事件/模型记忆；跨会话、跨 worktree、多执行载体仍需要一个所有角色可达的绝对路径真源 | 不变，补充"常驻进程持有/读写同一路径"的说明 |
| 轮询 `StateRevision` + 定向读取 `PendingConsumption` | **保持为兜底通道**：子任务完成信号以持久状态到达，不依赖 Hermes 恰好"在线看到事件" | 不变 |
| 周期巡检作为独立可重复任务（§11） | **重写**：常驻 Hermes 的 idle 循环即巡检；有活动时不空转轮询；巡检提示词变为"常驻调度循环的协议定义" | 见 §2.5-1 |
| 轮询间隔/心跳 | 常驻下可加**事件加速通道**（线程完成通知、文件变更监听）减少延迟 | 事件只作加速，事件丢失不得影响正确性（仍以轮询兜底），与 09 §3.3 禁止推断一致 |
| 短时锁 + 协调租约 | **保留**：常驻下触发源更多（用户消息、完成信号、手动 continue、潜在多实例），租约仍是防重复副作用的关键 | 不变 |
| `CoordinatorThreadID`/Coordinator 作为任务 | **语义调整**：改为 Hermes 的持久身份（可含实例 ID + 恢复点），而非"某次启动的 Coordinator 任务" | 见 §2.5-2 |
| 派生审计缓存（events.jsonl） | **保留**：可对齐 SDK tracing 语义；同时补充"结果内容落盘"（状态 JSON 只存指针） | 见 §2.4-3 |

**为何仍是最优解**：LangGraph 的 checkpoint 在进程内，ACP/Swarm 无持久化，UI 事件不可靠；共享 JSON + 文档快照是唯一满足"持久、跨执行载体、可审计、可冷/热恢复"且与人为门禁流程兼容的方案。轮询在常驻下代价更低（idle 循环天然空闲），且天然免疫"事件丢失/顺序错乱"。

### 2.3 裁剪点（Trim）

| # | 对象 | 裁剪内容 | 原因 |
|---|---|---|---|
| T-1 | 09 §11 | 周期巡检的"独立任务化"表述（巡检提示词、巡检作为可重复协议） | Hermes 常驻后巡检不是独立任务而是调度循环内置步骤；保留其"无 PendingConsumption 不遍历线程"的核心 |
| T-2 | 09/07 | 与具体执行载体强耦合的细节（Codex 子任务创建、clientThreadId、跨 worktree 写入） | 应由 ACP 式会话适配层统一抽象，规范层只保留"执行机制不可替代"原则（07 §2.1 已含） |
| T-3 | 07 §3 | 各角色"共享基座 + 增量"中过度细化的字段罗列 | 对齐 subagent frontmatter 的最小必要字段，减少提示词组装篇幅 |
| T-4 | README §2.10 | "调度者和周期巡检只轮询"的并列表述 | 常驻下"调度者 = 巡检循环"，避免读者误以为是两个实体 |

### 2.4 补漏点（Gap）

| # | 缺失 | 补齐内容 | 借鉴来源 | 原因 |
|---|---|---|---|---|
| G-1 | Context L1 无标准入口与冲突规则 | 定义 AGENTS.md（或等价文件）为 Context L1 载体：位置、嵌套 closest-wins、用户指令 > 就近 AGENTS.md > 远程文档；与 05 变更控制衔接 | AGENTS.md | README §9 只提"可复用现有 AGENTS"，无细则；AGENTS.md 已是 20+ 工具的事实标准 |
| G-2 | 常驻总调度自身的生命周期未定义 | 新增"常驻调度运行模式"章节：启动/热恢复、idle 循环、多实例防重、版本升级时状态兼容 | LangGraph checkpoint/thread_id；ACP session | 09 假设 Coordinator 是任务；Hermes 是长期进程，需要自身恢复点与身份 |
| G-3 | 结果/证据内容无落盘规范 | 规定"状态 JSON 只存指针，长结果/证据写入派生证据目录"，子任务直接写文件减少传话 | Anthropic 多 Agent 研究系统 | 09 只有派生审计缓存，未规定结果内容落盘；状态 JSON 塞长文会膨胀且难审计 |
| G-4 | 输入/输出校验不对称 | 交付校验扩展为 guardrail 双侧：派发前校验输入基线，消费前校验输出协议头/Invocation/HeadSHA | OpenAI Agents SDK guardrails | 09 §8 已有消费侧校验，缺派发侧输入校验 |
| G-5 | 阶段摘要/外部记忆无规范 | 子任务完成阶段时写结构化摘要到证据目录，供恢复与下游上下文对账 | Anthropic 外部记忆 | 06 上下文包对账缺少"子任务阶段摘要"这一输入源 |
| G-6 | 权限/模型变更无通知语义 | 状态记录补充 `config_change` 类事件（权限/模型/工具白名单变化） | ACP session/config_change | 07/09 有权限字段但无"运行中变更需记录并可能使旧派发失效"的语义 |

### 2.5 重写点（Rewrite）

| # | 对象 | 现状 | 重写方向 | 原因 |
|---|---|---|---|---|
| R-1 | 09 §11 周期巡检 → **常驻调度循环** | 巡检是"可重复执行的持久协议"，轮询 StateRevision | 定义为 Hermes 常驻 idle 循环：启动即加载状态、进入空闲轮询、事件加速可选、暂停即静默、恢复先对账；保留"无待消费不遍历线程" | 常驻后巡检与调度是同一循环，独立巡检任务语义过时 |
| R-2 | 07 §8 总调度最低规则 → **常驻总调度角色契约** | "使用 Coordinator 时的最低规则"（面向按需任务） | 升级为 Hermes 常驻运行契约：持久身份/恢复点、会话适配层（ACP 式）、派发/消费/恢复循环、多触发源并发协调、Guardrail 双侧校验 | Hermes 是常驻 Agent，其运行契约应从"规则"提升为"协议章节" |

### 2.6 修改点（Modify）

| # | 对象 | 修改内容 | 原因 |
|---|---|---|---|
| M-1 | 09 §5.1 字段 | `ExpectedExecutionKind` 增加 ACP 式会话适配层字段（SessionEndpoint/ProtocolAdapter/Version），`CoordinatorThreadID` 语义改为"常驻调度身份 + 实例/恢复点" | 执行载体统一抽象，常驻身份明确 |
| M-2 | 09 §6.1 事件表 | 新增 `SessionConfigChanged`、`EventAccelerated`（事件加速通道命中）等事件；明确事件加速不得替代状态写入 | 对齐 ACP 通知与常驻事件驱动 |
| M-3 | 09 §8/07 §8.2 | 消费循环增加"事件加速可选 + 轮询兜底"双通道描述，并保持幂等键不变 | 常驻下低延迟与可靠性兼得 |
| M-4 | 09 §14 Canary | 增加常驻场景用例：多实例防重、事件加速丢失后轮询兜底、Hermes 热恢复 | 常驻是新的主运行形态，Canary 需覆盖 |
| M-5 | README §2.10/§2.11、§13 决定 | 措辞从"调度者和周期巡检"改为"常驻调度循环/巡检"，DOC-DEC-014/016 附注"共享 JSON 适用于常驻调度，轮询为兜底、事件为加速" | 与 R-1/R-2 保持一致 |

### 2.7 对比总表

| 维度 | V2.4 现状（07/09） | 调研最佳实践 | 建议 | 类型 |
|---|---|---|---|---|
| 上下文入口 | Context L1 概念存在，无标准文件 | AGENTS.md closest-wins | 定义 AGENTS.md 载体与冲突优先级 | 补漏 G-1 |
| 执行载体抽象 | ExpectedExecutionKind + 两段式创建 | ACP 会话生命周期 | 统一为会话适配层 | 裁剪 T-2 / 修改 M-1 |
| 状态真源 | 共享 JSON + 文档快照 | LangGraph checkpoint | 保持，补充常驻语义 | 保留 + 修改 M-1 |
| 巡检 | 独立周期巡检任务 | 常驻 idle 循环 | 重写为调度循环 | 重写 R-1 |
| 结果传递 | 状态 JSON + 定向读取线程 | 结果直写文件系统 | 状态只存指针，证据落盘 | 补漏 G-3 |
| 校验 | 消费侧交付校验 | Guardrail 双侧 | 增加派发侧输入校验 | 补漏 G-4 |
| 可观测 | 派生审计缓存 events.jsonl | SDK tracing | 保持并语义对齐 | 保留 |
| 暂停/恢复 | 租约 + 三级恢复 | interrupt/checkpoint 重放 | 保持，补充常驻身份 | 修改 M-1/M-4 |
| worker 形态 | 提示词模板（07） | subagent frontmatter | 模板对齐 frontmatter 最小字段 | 裁剪 T-3 |
| 权限通知 | 无运行中变更语义 | ACP config_change | 增加 config 变更事件 | 补漏 G-6 |

### 2.8 影响范围与优先级建议

- **高优先（V3.0 主干）**：R-1（常驻调度循环）、R-2（常驻总调度契约）、G-2（常驻生命周期）——三者是 Hermes 迁移的骨架。
- **中优先（随主干落地）**：G-1（AGENTS.md 入口）、G-3（证据落盘）、M-1/M-2/M-3（字段与事件）。
- **低优先（可与运行复盘合并）**：T-1~T-4 措辞裁剪、G-4/G-5/G-6、M-4/M-5 增量。
- **不建议**：用 LangGraph 替换控制面、用 ACP 替换台账、用 subagent 自动路由替换显式派发、用事件通道替代共享状态轮询。

---

## 3. 结论与下一步

1. 调研确认：没有任何单一开源标准覆盖"常驻总调度 + 人为门禁 + 多执行载体"全场景；V2.4 的"共享 JSON 真源 + 幂等消费 + 定向读取 + 三级恢复"方向正确，是分层借鉴后的合理基座。
2. 建议 V3.0 以"常驻调度循环（R-1/R-2）"为骨架，先落地 AGENTS.md 上下文入口（G-1）与证据落盘（G-3），再用 Canary 扩展覆盖常驻场景（M-4）。
3. 后续子任务建议：B（协议/状态 schema 细化）、C（提示词模板与角色契约对齐）、D（Canary 用例与恢复演练）。

## 4. 参考资料

- AGENTS.md：https://agents.md/ （Agentic AI Foundation / Linux Foundation 托管；60k+ 仓库、20+ 工具）
- ACP：https://agentclientprotocol.github.io/agent-client-protocol/ （v0.4.x；TypeScript SDK；Gemini CLI 生产实现）
- LangGraph：https://langchain-ai.github.io/langgraph/ （StateGraph、checkpoint、interrupt；langgraph-supervisor/swarm 模式）
- Claude Code Subagents：https://code.claude.com/docs/ （.claude/agents frontmatter；background/isolation/worktree）
- OpenAI Agents SDK / Swarm：https://openai.com/ （Handoffs、Guardrails、Tracing；Swarm transfer_to_XXX）
- Anthropic：How we built our multi-agent research system（2025-06，orchestrator-worker、结果直写文件系统、外部记忆）
- 本仓库基线：README.md（V2.4）、文档规范/07、文档规范/09（V2.4 待审核）

---
*本文件由子任务 A 产出，只作为调研与对比依据，不修改任何规范正文。*
