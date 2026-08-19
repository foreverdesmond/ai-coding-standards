# 调研分析 · V2.4 → Hermes 调度模式重构大纲（WorkBuddy 稿）

> 状态：草稿（供 Hermes 总调度审阅）
> 作者：WorkBuddy（受 Hermes 委派）
> 上游基线：ai-coding-standards V2.4（待审核基线，V2.3 为上一已审核基线）
> 任务：子任务B — 开源规范调研 + V2.4 对比 + 重构大纲产出
> 日期：2026-08-19
> 关联：`Hermes迁移-规范重构大纲.md`（Richy 工作稿，本文与其互补，聚焦开源调研增量）

---

## 0. 核心结论（先给 5 条）

1. **V2.4 的 09 是在 Codex 线程里"硬造调度器"**，与开源共识相悖——业界主流做法是"原生 orchestrator/supervisor 进程 + 专业 worker"。Hermes 本身就是那个原生调度器，因此 09 的**机制层**（共享本地 JSON、短时锁、原子更新工具、轮询巡检、协调租约、从 JSON 冷恢复）应整体裁剪；但其**状态机思想**（三类状态分离、幂等派发、证据消费状态机、最大安全状态恢复）必须保留并映射到 Hermes 台账。
2. **开源主流拓扑高度收敛**：Anthropic orchestrator-worker、MCP composability、ACP Router Agent、LangGraph supervisor 四种来源指向同一个结构——"中心 supervisor + 专业 worker + 结构化 handoff"。这印证 Hermes 总调度方向正确，不需要再造一套通用控制面。
3. **07 的骨架可保留**：其"公共基座 + 角色增量 + 动态数据"的提示词分层、角色独立性、Git 权限矩阵，与 Claude Code subagent 实践（markdown 角色定义 + 权限 scope + 结构化失败信号）高度同构。仅需改 `ExpectedExecutionKind` 枚举与"Codex 专属兜底禁令"。
4. **README 是 V2.4 最成熟资产**：角色矩阵、三档风险路线、Level 0–3 验证、Git 权限矩阵、状态机语义、核心原则 2.1–2.9 全部保留。需重写的是第 10 节「Codex 与多 Agent 协作规则」整节，以及核心原则 2.10/2.11 中绑死 Codex 的表述。
5. **补漏方向（开源新增证据）**：AGENTS.md 式分层指令、ACP 式 agent 能力注册与幂等/可观测契约、MCP 企业试点的"持久化共享 scratchpad + 结构化 handoff"经验，应纳入 V2.5 的 Hermes 能力边界清单与台账规范。

---

## 1. 开源调研结果（5 个最值得参考的规范/实践）

> 调研方法：GitHub + 权威来源（OpenAI/Anthropic/Linux Foundation AAIF、agentcommunicationprotocol.dev、IBM developer、LangChain 官方等）。挑选标准：能被一个常驻总调度 Agent（Hermes）主导、且与 V2.4 的调度/委派痛点直接相关。

### 1.1 AGENTS.md（OpenAI → Linux Foundation AAIF）

- **是什么**：轻量 Markdown 标准，放在项目根目录，给 AI 编码 agent 提供项目特定指令（编码规范、构建步骤、测试要求）。2025-08 发布，2025-12-09 随 Agentic AI Foundation 捐赠给 Linux Foundation，已获 60,000+ 开源项目与 Amp/Codex/Cursor/Devin/Gemini CLI/Copilot/Jules/VS Code 等框架采用。
- **解决什么问题**：agent 进入仓库时"不知道这个项目的约定与边界"，需要一个可预测、机器可解析的指令入口。
- **为什么适合 Hermes 场景**：Hermes 作为总调度，给 WorkBuddy/Codex 派发任务前，需要一个**标准化的项目指令来源**作为"上下文 L1"的载体。AGENTS.md 的**分层约定**（global `~/.claude/AGENTS.md` → 项目 `./AGENTS.md` → 子包 `./packages/*/AGENTS.md`，就近覆盖）可直接借鉴到 V2.5 的上下文体系，替代 V2.4 里笼统的"Context L1 永久上下文"描述。
- **对 V2.4 的映射**：README 第 9 节"文件组织"中的 Context L1 定位与之同源，但 V2.4 未规定分层与就近覆盖规则，V2.5 可补。

### 1.2 ACP — Agent Communication Protocol（Linux Foundation，2025）

- **是什么**：REST-based 的 agent-to-agent 通信协议，标准化 agent 间消息格式、能力发现（agent registry / agent card）、幂等、可观测。定义了 Router Agent 模式：中心 agent 分解复杂请求 → 路由到专家 agent → 聚合响应。
- **解决什么问题**：多 agent 之间"怎么通信、怎么发现对方能力、怎么保证消息不丢不重"的互操作问题。
- **为什么适合 Hermes 场景**：V2.4 的 09 用"共享本地 JSON + 轮询"在 Codex 线程里**模拟** agent 间通信；ACP 给出的是**正式的 agent 通信层**。Hermes 是常驻服务，天然具备 REST/事件通道，可以直接采纳 ACP 的 Router 模式与幂等契约（message id + correlation id + 幂等键），替代共享 JSON 轮询。
- **对 V2.4 的映射**：09 的 `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`、`Fingerprint`、`RecordID + SignalRevision` 幂等消费，与 ACP 的幂等键/消息指纹思路一致——**这部分 V2.4 已经做对了，只需换载体**。

### 1.3 Anthropic《Building Effective Agents》+《Multi-Agent Research System》

- **是什么**：Anthropic 官方指南，提出 5 个可组合工作流模式（prompt chaining / routing / parallelization / orchestrator-workers / evaluator-optimizer），并公开其 Research 系统采用 orchestrator-worker 模式（lead agent 规划 + 并行 subagents 压缩返回）。8 条多 agent 设计原则：教会主控如何分工、按任务复杂度分配资源、工具设计即界面设计、让 agent 自我改进等。
- **解决什么问题**："多 agent 系统应该用哪种拓扑、复杂度该加到什么程度"的架构选型问题。核心立场：**最简单可行的模式优先**，复杂框架增加调试成本。
- **为什么适合 Hermes 场景**：Hermes 天然是 orchestrator-worker 的 lead agent。Anthropic 明确警告：**大多数 coding 任务并行度低、agent 实时协调还不成熟**——这直接支持 Richy 工作稿里"必须单实例、不做双轨"的决策。其"orchestrator 不干活、只做 handoff"的原则，对应 V2.4 的"总调度不自批"。
- **对 V2.4 的映射**：验证了 V2.4 角色分离（开发/Review/集成/合并）的正确性；同时提示 V2.4 的 09 控制面"重"了——对大多数任务，原生 orchestrator + 轻量 handoff 足够，不需要全量 JSON 控制面。

### 1.4 MCP Composability + 企业多 agent 试点经验

- **是什么**：MCP 的 composability（agent 既可作 client 也可作 server，形成层级编排）；IBM developer 架构文章与 silkate 三企业试点的生产经验总结。
- **解决什么问题**：多 agent 系统的三个真实痛点——工具名冲突、上下文窗口经济性、handoff 保真度。
- **关键结论（三条生产验证）**：
  1. **supervisor-worker，而非扁平 peer 网络**——每个 handoff 走一个 chokepoint，可观测性最强；
  2. **每个 bounded context 一个共享服务，而非每个 agent 一个**——避免数据访问逻辑重复；
  3. **持久化共享 scratchpad（按会话键控）**——handoff 传"你关心的 key"，不传全量对话。
- **为什么适合 Hermes 场景**：第 3 条直接印证 V2.4 的"开发任务文档状态交换区 + 共享状态"思路是对的（本质就是持久化 scratchpad），只是实现绑死在 Codex 线程。Hermes 的台账就是那个 scratchpad。
- **对 V2.4 的映射**：09 的"状态交换区只保存最近快照、共享状态是唯一真源"与"结构化 handoff"同构；但 09 把"唯一真源"实现为 Git 外共享 JSON 文件 + 锁，MCP 经验提示应改为 Hermes 原生持久化 store。

### 1.5 Claude Code Subagents / Agent Teams（2026 playbook）

- **是什么**：Anthropic Claude Code 的子 agent 机制与生产实践——subagent 用 markdown + frontmatter（name/description/tools/model）定义，lead 只看到 subagent 的 final summary；配套 Skill（教 how）、Hook（强制 rule）、Subagent（隔离 work）三件套。
- **解决什么问题**：如何把角色变成可配置、可权限隔离的执行单元；如何让失败可结构化、可重试。
- **关键结论**：
  - **权限 scope**：reviewer 只读（read/grep），writer 可写但禁生产 API——缩小爆炸半径；
  - **结构化失败信号**：`{status, reason, completed_portion, retry_possible}`，禁止无标记的部分输出；
  - **aggregator 模式**：orchestrator 派 N 个并行 subagent，再派 1 个聚合 agent 合成结果；
  - **3–5 个并发 subagent 是甜蜜点**，过多则合并成本反超收益。
- **为什么适合 Hermes 场景**：V2.4 的 07「角色提示词模板」与此同构——每个角色 = 一个 markdown 定义 + system prompt + 权限边界。V2.5 可直接采用"角色定义 markdown + 权限 scope"的格式，把 V2.4 的 13 个提示词模板重构为 Hermes 可派发的角色配置。
- **对 V2.4 的映射**：V2.4 的 `ExpectedExecutionKind`、`ExpectedModel` 正好对应 subagent 的 `tools/model` 配置；"载体分离 ≠ 视角独立"的提醒（Richy 工作稿 D 条）与此处"权限隔离"互补。

---

## 2. 对比分析：开源规范 vs V2.4

> 以下按"保留 / 裁剪 / 补漏 / 重写 / 修改"五类，逐项给结论与理由。重点评估 09、07、README。

### 2.1 保留不动（V2.4 成熟资产，与开源共识一致）

| 资产 | 理由 |
|---|---|
| 角色矩阵（项目负责人/总调度/开发者/Reviewer/返工/集成/合并…） | 与 Anthropic orchestrator-worker、Claude Code subagent 角色分离完全同构 |
| 三档风险路线（轻量/标准/高风险） | 与 Anthropic"最简单可行优先"一致，避免过度工程 |
| Level 0–3 验证分层 | 与 Anthropic evaluator-optimizer、generator-verifier 的验证思想一致 |
| Git 权限矩阵（§6.1） | 与 Claude Code 权限 scope 实践一致 |
| 状态机语义（TaskState / EvidenceState 全枚举） | 跨执行载体通用，是 09 里最有价值的部分 |
| 核心原则 2.1–2.9（事实优先/SOLID/职责分离/安全优先/上下文地图） | 与所有开源最佳实践一致，无争议 |
| 文档 01–06、08 的主体 | 不涉及 Codex 执行机制，通用 |

### 2.2 必须裁剪（Codex 专属 / 被 Hermes 原生能力取代）

| 原文 | 裁剪理由 | 替代 |
|---|---|---|
| `ExpectedExecutionKind: CodexThread/SubAgent` | Codex 专属枚举 | 改为 `WorkBuddy / Codex / Human / ApprovedEquivalent` |
| 「创建 Codex 子任务失败不得用 spawn_agent 兜底」 | 绑定 Codex 任务系统 | 退回「禁止伪独立自审」精神 |
| `clientThreadId → threadId` 映射 | Codex 线程概念 | 统一为 `ExecutionRef` |
| 共享本地 JSON + 短时锁 + 原子更新工具整套 | 在 Codex 线程里硬造调度器 | Hermes 台账 + 幂等键 |
| 周期轮询巡检（`StateRevision` 轮询） | Hermes 有 cron + 事件驱动 | cronjob + 飞书事件推送 |
| 协调租约锁（多 Coordinator 并发） | 单实例无需租约 | 单实例 + `DispatchKey` 幂等去重 |
| 从共享 JSON 冷恢复 | Hermes 重启读快照 + Git 天然恢复 | 显式幂等重建 |

### 2.3 必须补漏（开源有、V2.4 缺）

| 补漏项 | 来源 | 内容 |
|---|---|---|
| **Hermes 能力边界清单** | （工作稿已定） | cron 配置、事件来源、台账存储位置、Agent 派发方式，落地前必须先厘清 |
| **AGENTS.md 式分层指令** | 1.1 | global / 项目级 / 子包级，就近覆盖；规定"指令入口在哪、谁有权改" |
| **agent 能力注册与发现契约** | 1.2 ACP | 每个可派发角色/agent 的能力描述、输入输出 schema、幂等键，供 Hermes 路由 |
| **结构化 handoff 协议** | 1.4 MCP 试点 | handoff 传"关心的 key + 结构化状态"，不传全量对话；对应 V2.4 的"定向读取线程" |
| **权限 scope 显式化** | 1.5 | 每个角色的工具权限清单（read/write/bash 边界），缩小爆炸半径 |
| **结构化失败信号** | 1.5 | `{status, reason, completed_portion, retry_possible}`，禁止无标记部分输出 |

### 2.4 必须重写（整节替换）

| 对象 | 重写内容 |
|---|---|
| **09-调度控制平面与运行时台账规范** | 整份重构为《Hermes 调度与运行时台账规范》：删除共享 JSON/锁/轮询/租约/冷恢复，改为 Hermes 台账 + cron/事件 + 单实例幂等 + 重启恢复；**保留**三类状态分离、幂等 DispatchKey、证据消费状态机、最大安全状态恢复、Review 执行状态与 Verdict 分离、NotRun 语义 |
| **README 第 10 节「Codex 与多 Agent 协作规则」** | 整节重写为「Hermes 总调度与多 Agent 协作规则」，替换"Codex 线程""共享本地 JSON""周期巡检"等词 |
| **README 核心原则 2.10 / 2.11** | 改写表述：保留"文档状态驱动""状态分离""持久可恢复可审计"精神，删"共享 JSON""StateUpdateToolPath""轮询"实现细节 |
| **调度三件套提示词（07-总调度 / 13-周期巡检 / 14-冷恢复 / 15-Canary）** | 从"Codex 线程提示词"重建为"Hermes 配置 + 调度指令"；**并补 00-公共基座**（工作稿 C 条：它是耦合最重底座，`SharedRuntimeStatePath/StateUpdateToolPath/PendingConsumption` 全在其中） |

### 2.5 必须修改（保留骨架但适配）

| 对象 | 修改内容 |
|---|---|
| **提示词组装（§4）** | "公共基座 + 角色模板 + 基线 + Context L2 + 动态数据"保留；动态数据改由 Hermes 注入 |
| **依赖推进** | Hermes 台账记依赖图，由 Hermes 触发下游（替代"依赖 Integrated 后人工推进"） |
| **角色独立性（§5）** | 载体分离（WB 写 / Codex 审）+ 保留独立风险模型，不把"不同载体"等同于"独立视角"（工作稿 D 条） |
| **Finding 三次升级（§8.4）** | 保留"第 2 轮分类根因 → 第 3 轮停机械返工上报裁决"，Hermes 计数只是触发手段（工作稿 E 条） |
| **幂等键** | `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`；"天然幂等"改为"显式幂等"（工作稿 A 条） |

### 2.6 重点：09 / 07 / README 在 Hermes 模式下的适用性判定

**09（调度控制平面）**：**机制层不适用，状态机层适用。**
- 不适用：共享本地 JSON 唯一真源、短时锁、原子更新工具、轮询巡检、协调租约、RecoveryOnly 从 JSON 重建——这些是为"Codex 线程当总调度"设计的，Hermes 原生替代后全部多余。
- 适用（保留并映射）：ThreadStatus/TaskState/EvidenceState 三类状态分离、幂等 `DispatchKey`、`RecordID + SignalRevision` 幂等消费、证据状态机、最大安全状态恢复、Review `ExecutionStatus` 与 `Verdict` 分离、`NotRun` 语义、Canary 验收——这些是**跨执行载体通用的调度正确性内核**，是 V2.4 最值钱的部分。

**07（子 Agent 委派）**：**骨架适用，枚举与兜底禁令需改。**
- 适用：角色按需选择、提示词分层（公共基座+角色增量）、角色独立性、Git 权限、Reviewer 风险反证、SOLID 责任、总调度最低规则——与 Claude Code subagent 实践同构。
- 需改：`ExpectedExecutionKind` 枚举（去 CodexThread，加 WorkBuddy）；"Codex 子任务失败不得兜底"裁剪为"禁止伪独立自审"。

**README**：**主体适用，第 10 节与 2.10/2.11 需重写。**
- 适用：目的、核心原则 2.1–2.9、规范目录、角色、阶段地图、分支映射、Git 权限矩阵、验证等级、状态语义、文件组织、演进规则、V2.3 决定。
- 需重写：第 10 节整节；核心原则 2.10/2.11 的 Codex 实现细节表述。

---

## 3. V2.5 重构大纲（章节级）

> 目标：V2.4 → V2.5，迁移到"Hermes 服务器做总调度 + WorkBuddy 开发 + Codex Review"模式。

### 3.0 前置步骤（新增）
1. 梳理 **Hermes 能力边界清单**：cron 配置、事件来源（飞书推送等）、台账存储位置（SQLite/文件）、Agent 派发方式、重启恢复语义。
2. 确认台账载体归属（Hermes 自带 DB 还是项目本地文件）——工作稿 F 条存疑，先厘清。

### 3.1 文档层（09 重构）
- 新增《Hermes 调度与运行时台账规范》，替代旧 09：
  - 台账 schema（任务/阶段/状态/证据/依赖图）
  - Hermes cron + 事件驱动的调度循环（替代轮询）
  - 单实例 + `DispatchKey` 幂等派发（替代租约锁）
  - 重启恢复（读台账快照 + Git，替代冷恢复）
  - 保留三类状态分离 + 证据消费状态机 + 最大安全状态恢复
  - Canary 改为 Hermes→WB→Codex→验收闭环

### 3.2 委派层（07 重构）
- 角色定义格式改为 markdown + frontmatter（name/description/tools/model），对应 Claude Code subagent 格式
- `ExpectedExecutionKind` 枚举改 `WorkBuddy / Codex / Human / ApprovedEquivalent`
- 补结构化失败信号与权限 scope
- 重写调度三件套 + 00-公共基座

### 3.3 指令分层（AGENTS.md 式）
- 引入 global / 项目级 / 子包级 AGENTS.md 分层，就近覆盖
- 明确"上下文 L1"的物理载体与优先级

### 3.4 README 层
- 重写第 10 节为「Hermes 总调度与多 Agent 协作规则」
- 改写核心原则 2.10/2.11 表述
- 更新 V2.4 待审核决定 → V2.5 待审核决定（新增 DOC-DEC，标记 09/07 的迁移决议）

### 3.5 落地路径（呼应工作稿）
1. Hermes 能力边界清单 → 2. 建 Hermes 台账规范 → 3. 重写调度三件套 → 4. 写闭环 Canary → 5. 裁剪执行机制节 → 6. 开 V2.5 分支 → 7. V2.5 审核通过后更新飞书知识库。

---

## 4. 待澄清事项（交付给 Hermes）

1. **「0磋」语义**：原文出现于工作稿「五、保留不动」中，语义与出处未知，需向原文档作者确认后回填。
2. **台账载体归属**：Hermes 自带持久化 DB，还是项目本地文件？（决定 3.1 台账 schema 的落点）
3. **事件来源范围**：Hermes 的事件驱动目前覆盖哪些（飞书推送？git webhook？），是否足以替代全部周期巡检场景，还是仍需保留一个兜底 cron？
4. **Codex 角色定位**：迁移后 Codex 是否仅承担 Review（工作稿已暗示 WB 写 / Codex 审），还是保留部分开发能力？

---

## 附：调研来源清单

- OpenAI《Agentic AI Foundation》/ AGENTS.md：https://openai.com/blog/agentic-ai-foundation（2025-12）
- AGENTS.md Standard：https://agentic-ai.readthedocs.io/en/latest/Standards/agents-md/
- Agent Communication Protocol（ACP）：https://agentcommunicationprotocol.dev/core-concepts
- Anthropic《Building Effective Agents》：https://www.anthropic.com/engineering/building-effective-agents
- Anthropic《How we built our multi-agent research system》：https://www.anthropic.com/engineering/multi-agent-research-system
- IBM developer《MCP architecture patterns for multi-agent AI systems》：https://developer.ibm.com/articles/mcp-architecture-patterns-ai-systems/
- MCP Composability：https://mcpserverspot.com/learn/architecture/composability-in-mcp
- LangGraph Multi-Agent Supervisor：https://reference.langchain.com/python/langgraph-supervisor
- Claude Code Subagents Best Practices 2026：https://thepromptshelf.dev/blog/claude-code-subagents-best-practices-2026
