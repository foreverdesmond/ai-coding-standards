# 调研分析：Hermes 常驻总调度 × Codex-Mimo 多 Agent 协作架构

> **任务**：Hermes 总调度指派 · 子任务 A（规范研究员）
> **基线**：ai-coding-standards V2.4（README.md / 07-子Agent任务委派与提示词规范 / 09-调度控制平面与运行时台账规范）
> **调研范围**：AGENTS.md · ACP · LangGraph Supervisor · Claude Code Subagents · OpenAI Agents SDK (Swarm/Mimo) · Anthropic orchestrator-worker
> **结论定位**：供 Hermes 迁移与规范重构（V3.x）使用的"调研 + 对比 + 建议"，不直接改规范正文
> **日期**：2026-08-20

---

## 0. 摘要（TL;DR）

| 维度 | 结论 |
|---|---|
| **Hermes 定位** | 常驻（resident）总调度 Agent = 长期运行的控制循环 + 持久状态层 + 执行机制适配层。V2.4 假设的"按需启动 Coordinator 任务"在 Hermes 下不再成立 |
| **五个调研对象** | AGENTS.md = 被动上下文层（Context L1 标准化载体）；ACP = 会话/传输协议层（执行机制适配）；LangGraph Supervisor = 显式图编排 + checkpoint 语义；Claude Code Subagents = 角色化 worker 落地形态；OpenAI Mimo/Swarm + Anthropic 工程经验 = handoff/guardrail/并行派发运行时模式 |
| **对 09 共享 JSON** | 保留"共享状态为唯一真源 + RecordID/SignalRevision 幂等消费 + 定向读取"；将"周期巡检任务"改写为"常驻调度循环 idle 轮询 + 事件加速通道" |
| **改动总量** | 裁剪 6 项 · 补漏 8 项 · 重写 3 项 · 修改 7 项（详见 §2.7 总表） |

---

## 1. 开源调研：五个多 Agent 协作规范/最佳实践

### 1.1 AGENTS.md —— 被动上下文层（Context L1 标准化载体）

| 字段 | 内容 |
|---|---|
| **核心机制** | 仓库根 `AGENTS.md`（Markdown 无强制 schema），子目录可嵌套，**closest-wins + 用户指令最高优先**；v2 增强 `@path` 导入与 glob 引用；60k+ 仓库、20+ 工具采用 |
| **一句话适配判断** | ✅ **强烈适配** —— 正是 Hermes Context L1 的标准化落点，所有 worker（WorkBuddy / Codex）共享可预测的操作说明；但它只解决上下文不解决调度，不能替代 09 |
| **适配要点** | V2.4 README §9 已提"Context L1 可复用 AGENTS"，但未定义文件位置、嵌套规则、冲突优先级——**必须补漏**（见 §2.4-1） |

### 1.2 ACP（Agent Client Protocol）—— 会话/传输协议层

| 字段 | 内容 |
|---|---|
| **核心机制** | JSON-RPC over stdio，Host（编排方）↔ Agent（执行方）；生命周期 `initialize → session/new → session/prompt → session/cancel`；两段式创建（clientThreadId → threadId）；`_meta` 扩展位 |
| **一句话适配判断** | ⚠️ **部分适配** —— 09 §3.1 线程生命周期（NotCreated/Provisioning/Running/Completed）与 ACP 会话生命周期一一对应，可形式化为执行机制适配契约；但 ACP 无调度、无持久台账、无多 Agent 依赖，不能当控制面 |
| **适配要点** | Hermes 派发任何执行载体（WorkBuddy / Codex / Human）时，可统一建模为 ACP 式会话——`ExecutionThreadID + session 方法`——不绑定具体机制，与 07 §2.1 ExpectedExecutionKind 理念一致 |

### 1.3 LangGraph Supervisor —— 显式图编排 + checkpoint 语义

| 字段 | 内容 |
|---|---|
| **核心机制** | StateGraph + supervisor 节点集中路由到专门 worker；显式状态 schema 作为单一状态面；checkpoint 持久化（thread_id）可中断/恢复/时间旅行重放；interrupt / human-in-the-loop |
| **一句话适配判断** | ⚠️ **语义借鉴，框架不迁移** —— supervisor 模式是"总调度"的教科书形态；但 LangGraph 进程内运行时 vs Hermes 跨会话/跨 worktree/跨执行载体的常驻调度，共享 JSON + 文档快照双层持久化反而更贴合 |
| **适配要点** | 借鉴三点语义：① 显式状态 schema → 09 §5.4 任务级字段；② checkpoint/重放 → 热/冷/灾难恢复；③ interrupt → 暂停/租约。**不要照搬其进程内运行时** |

### 1.4 Claude Code Subagents —— 角色化 worker 落地形态

| 字段 | 内容 |
|---|---|
| **核心机制** | 每个 subagent 独立上下文窗口 + 独立 worktree + 后台执行 + 工具权限边界（只读/读写/危险）；subagent 只看自己的指令和文件子集，不共享父 agent 全部历史 |
| **一句话适配判断** | ✅ **高度适配** —— 正是 V2.4 §5 "角色独立性"在运行时的落地方案。Hermes 派发 WorkBuddy 做开发、Codex 做 Review，各自独立上下文/工具权限 = 载体隔离 |
| **适配要点** | V2.4 已有 Git 权限矩阵（07 §5.1），但缺少"运行时工具权限边界"的形式化定义（只读 vs 读写 vs 危险操作）——**必须补漏**（见 §2.4-4） |

### 1.5 OpenAI Agents SDK (Swarm/Mimo) + Anthropic orchestrator-worker

| 字段 | 内容 |
|---|---|
| **核心机制** | Swarm：worker 间对等 handoff（`transfer_to_X`）；Mimo（Agents SDK）：structured output + tool guardrail + tracing；Anthropic orchestrator-worker：并行派发 + 子结果直写文件系统 + 外部记忆 + context window 管理 |
| **一句话适配判断** | ⚠️ **运行时经验借鉴** —— handoff 语义可映射为 V2.4 的"依赖满足才派发下游"；guardrail 映射为工具权限边界；但 Swarm 对等 handoff 与 Hermes 单实例总调度的层级拓扑冲突，不可照搬 |
| **适配要点** | ① 并行派发 + 结果汇总 → Hermes 可并行派发独立任务，汇总后由 Hermes 消费（符合 09 §8）；② 子结果直写文件系统 → 建议 Hermes worker 结果写共享状态而非依赖聊天上下文回传；③ guardrail → 工具权限分级（见 §2.4-4） |

---

## 2. 与 V2.4 基线对比分析

### 2.1 分析范围

| 文档 | 关键章节 | 本次关注 |
|---|---|---|
| README.md | §2 核心原则 · §4 角色 · §9-11 V2.4 决策 | 角色定义、状态机、风险路线 |
| 09-调度控制平面与运行时台账规范 | §3 三类状态 · §5 状态面 · §7-8 创建与消费协议 · §10 Review 分离 | 控制面核心：JSON 状态面、轮询、租约、幂等 |
| 07-子Agent任务委派与提示词规范 | §2.1 执行机制 · §3 提示词分层 · §5 角色独立 · §8 总调度规则 | 角色独立、提示词组装、派发前清单 |

### 2.2 裁剪项（应删除或弱化，共 6 项）

| # | 裁剪目标 | V2.4 原文 | 裁剪原因 |
|---|---|---|---|
| C-1 | 共享 JSON + 文件锁 + 原子脚本整套机制 | 09 §5.1 SharedRuntimeStatePath + TaskStateWriteLockPath + StateUpdateToolPath | Hermes 是常驻服务，自带持久化层（SQLite/文件）+ 单实例保证，不需要在 Codex 线程里用文件锁模拟调度器 |
| C-2 | 协调租约锁 | 09 §5.1 "取得短时文件锁" + §7 "获得未过期协调租约" | 单实例 Hermes 不存在多 Coordinator 竞争；幂等由 DispatchKey 去重保证，不需要租约 |
| C-3 | ExpectedExecutionKind: CodexThread / SubAgent | 07 §2.1 | 改为 WorkBuddy / Codex / Human / ApprovedEquivalent 四档（经 §8-B 纠错后） |
| C-4 | "创建 Codex 子任务失败不得用 spawn_agent 兜底" | 09 §7-10 · 07 §2.1 | 退回精神内核"禁止伪独立自审"即可；Hermes 下执行载体是 WorkBuddy/Codex/Human，不存在 Codex 创建失败的场景 |
| C-5 | Codex clientThreadId → threadId 映射逻辑 | 09 §3.1 | 统一为 ExecutionRef / SessionID，由 ACP 式适配层处理 |
| C-6 | "调度器不得只依赖聊天上下文"的措辞（保留精神） | 09 §2-3 | Hermes 是常驻服务，天然不依赖聊天上下文；改为"Hermes 状态以台账为真源，不以 API 事件或模型记忆为真源" |

### 2.3 补漏项（V2.4 缺失但 Hermes 迁移必须补充，共 8 项）

| # | 补漏目标 | V2.4 缺失处 | 补漏来源 | 补漏内容 |
|---|---|---|---|---|
| L-1 | AGENTS.md 位置/嵌套/优先级规则 | README §9 仅提"Context L1 可复用" | AGENTS.md 调研（§1.1） | 明确 Hermes 下 Context L1 = 仓库根 AGENTS.md；嵌套规则 closest-wins；用户指令最高优先级 |
| L-2 | Hermes 能力边界清单 | 全文无 | Hermes迁移大纲 §六-1 | cron 配置、事件来源（飞书/webhook）、台账存储、Agent 派发 API、工具权限 |
| L-3 | 执行机制适配层规范 | 07 §2.1 只列枚举 | ACP 调研（§1.2） | 定义 Hermes ↔ 执行载体的会话协议：session 创建/查询/取消；统一 ExecutionRef |
| L-4 | 运行时工具权限边界 | 07 §5.1 只有 Git 权限 | Claude Code Subagents（§1.4） | 定义只读 / 读写 / 危险三级；每级可执行的工具集；与 ExpectedExecutionKind 关联 |
| L-5 | 并行派发与结果汇总协议 | 09 §8 只描述串行消费循环 | Swarm/Anthropic（§1.5） | Hermes 可并行派发独立任务；汇总节点等待所有依赖 TaskAccepted 后统一消费 |
| L-6 | 事件加速通道 | 09 §8.2 只描述轮询 | LangGraph interrupt + Anthropic 工程经验 | Hermes 支持"事件到达立即处理"作为轮询补充；事件不得替代持久状态 |
| L-7 | Finding 三次升级根因分类保留 | 07 §8.4 第二轮分类根因 | Hermes迁移大纲 §八-E | 明确"第2轮分类根因 → 第3轮停机械返工并上报裁决"；Hermes 计数只是触发手段 |
| L-8 | 00-公共基座重写 | 提示词模板 00-公共基座 未在重写/保留清单中 | Hermes迁移大纲 §八-C | 00-公共基座耦合 SharedRuntimeStatePath 等 Codex 专属字段最重，必须改为 Hermes 注入协议 |

### 2.4 重写项（必须大幅改写，共 3 项）

| # | 重写目标 | V2.4 原文 | 重写方向 | 理由 |
|---|---|---|---|---|
| R-1 | 09-调度控制平面与运行时台账规范 | 整篇基于"Codex 线程模拟调度器" | 重建为"Hermes 常驻调度循环"：台账取代共享 JSON；cron + 事件驱动取代轮询巡检；单实例取代租约锁 | Hermes 本身就是调度器，09 的整套控制面机制是 Codex 专属适配，迁移后大部分不再需要 |
| R-2 | 提示词模板 07-总调度Agent | Codex 线程内 Coordinator 的提示词 | 重建为 Hermes 调度指令配置：cron 规则 + 事件订阅 + 派发策略 + 消费规则 | Hermes 是配置驱动的常驻服务，不是提示词驱动的临时线程 |
| R-3 | 提示词模板 00-公共基座 | 包含 SharedRuntimeStatePath / StateUpdateToolPath / PendingConsumption / ExecutionThreadID | 改为 Hermes 注入协议：台账路径 + ExecutionRef + 工具权限 + 基线引用 | 底座耦合最重，是所有角色提示词的公共依赖 |

### 2.5 修改项（保留但需适配，共 7 项）

| # | 修改目标 | V2.4 原文 | 修改内容 | 理由 |
|---|---|---|---|---|
| M-1 | 提示词组装（07 §4） | 公共基座 + 角色模板 + 基线 + 动态数据 | 动态数据改由 Hermes 注入（台账字段 + diff + test result） | Hermes 有持久状态，不需要子任务自己拼装基线 |
| M-2 | 依赖推进（07 §8） | Coordinator 轮询 StateRevision | Hermes 台账记录依赖图，依赖满足时由 Hermes 主动触发下游 | 常驻调度器可以主动推送，不需要依赖方轮询 |
| M-3 | 角色独立性（07 §5） | "Hermes 内置保证"（载体分离） | 改为"载体分离（WB 写 / Codex 审）＋ 保留独立风险模型与不可歧义审核目标" | 载体分离 ≠ 视角独立；不同载体仍可能共享上游上下文与判断 |
| M-4 | Finding 三次升级（07 §8.4） | "Hermes 计数判断停止返工升级" | 保留"第2轮分类根因 → 第3轮停机械返工并上报裁决"；Hermes 计数只是触发手段 | 只计数会丢失根因分类，退化为死循环计数器 |
| M-5 | 幂等键 | InvocationID 去重（Hermes迁移大纲原文） | DispatchKey = IterationID + TaskID + Stage + TargetIdentity | InvocationID 每次派发唯一，拿它去重等于没去重 |
| M-6 | 09 提示词模板 13-周期巡检 / 14-冷恢复 / 15-Canary | Codex 线程提示词 | 改为 Hermes cronjob 配置 + 事件订阅 + 恢复规则 | Hermes 常驻服务不需要提示词驱动巡检 |
| M-7 | ExpectedExecutionKind 枚举 | WorkBuddy / Codex / Hermes自执行 三档 | 改为 WorkBuddy / Codex / Human / ApprovedEquivalent 四档 | 漏掉 Human（Level 2/3）和 ApprovedEquivalent；"Hermes 自执行"归入 ApprovedEquivalent，不并列成常规档 |

### 2.6 保留不动项

| 保留目标 | 保留原因 |
|---|---|
| 角色矩阵（Implementer/Reviewer/System Reviewer/Merge Reviewer 等） | 角色能力定义与载体无关，Hermes 迁移后角色不变 |
| 风险路线（轻量/标准/高风险） | 风险分级逻辑与调度器无关 |
| Level 0–3 验证分层 | 验证层级是开发流程标准，不绑定调度机制 |
| Git 权限矩阵（07 §5.1） | Git 权限与调度器无关，保留 |
| 状态机 TaskState（Planned→...→Merged） | 状态机是开发流程标准，保留 |
| 核心原则（09 §2 / README §2） | "文档状态驱动""状态分离""持久可恢复"等原则在 Hermes 下同样成立，只是实现层变了 |
| 文档 01-06 / 08 | 任务背景/需求/设计/任务/变更控制/上下文包/验证门禁，与调度器无关 |
| 提示词模板 01-06 / 08-12 | 角色模板本身不绑定调度器；但 00-公共基座需要重写（见 R-3），会影响所有模板的注入方式 |

### 2.7 改动总表

| 改动类型 | 数量 | 涉及文档 |
|---|---|---|
| **裁剪** | 6 项 | 09（C-1/C-2/C-5/C-6）、07（C-3/C-4） |
| **补漏** | 8 项 | README（L-1）、新增能力边界清单（L-2/L-3/L-4/L-5/L-6）、07（L-7）、提示词（L-8） |
| **重写** | 3 项 | 09 整篇（R-1）、提示词 07（R-2）、提示词 00（R-3） |
| **修改** | 7 项 | 07（M-1/M-2/M-3/M-4/M-7）、提示词 13/14/15（M-6）、幂等键（M-5） |
| **保留** | 7 类 | 角色矩阵 / 风险路线 / Level 0-3 / Git 权限 / 状态机 / 核心原则 / 文档 01-06/08 |

---

## 3. 架构映射：V2.4 → Hermes 迁移

### 3.1 拓扑对比

```text
V2.4 拓扑（Codex 线程当调度器）:
  Coordinator 线程 → 共享 JSON → 轮询 → 子线程创建/消费
  └── 周期巡检任务（独立线程）
  └── 冷恢复任务（独立线程）

Hermes 迁移拓扑（常驻总调度）:
  Hermes 常驻服务（cron + 事件驱动）
    ├── 台账（SQLite/文件）= 唯一状态真源
    ├── 派发层（WorkBuddy 开发 / Codex Review / Human 验证）
    ├── 事件加速通道（飞书/webhook → 立即处理）
    └── 恢复机制（重启 → 读台账 + Git → 重建 DispatchKey 去重表）
```

### 3.2 状态映射

| V2.4 机制 | Hermes 替代 | 保留的核心语义 |
|---|---|---|
| SharedRuntimeStatePath（共享 JSON） | Hermes 台账（SQLite/文件） | 唯一状态真源 + RecordID/SignalRevision 幂等消费 |
| TaskStateWriteLockPath（文件锁） | Hermes 单实例保证 + DispatchKey 去重 | 无需锁，幂等键保证不重复派发 |
| 周期巡检任务（独立线程提示词） | Hermes cronjob + 事件驱动 | idle 轮询 + 可选事件加速（事件不替代持久状态） |
| 协调租约锁 | 无（单实例不需要） | — |
| Codex 冷恢复（独立线程提示词） | Hermes 重启恢复规则（配置） | 读台账 + Git → 最大安全状态重放 |
| 调度 Canary（独立线程提示词） | Hermes → WB → Codex → 验收 闭环 Canary | Canary 语义保留，实现改为 Hermes 配置 |

---

## 4. 优先级建议（Hermes 迁移落地顺序）

基于本次调研，建议按以下顺序推进 V3.x 规范重构：

| 优先级 | 行动 | 依赖 | 预计影响 |
|---|---|---|---|
| **P0** | 梳理 Hermes 能力边界清单（L-2） | 无 | 一切后续工作的前提 |
| **P0** | 重写 00-公共基座为 Hermes 注入协议（R-3/L-8） | L-2 | 所有角色提示词的公共底座 |
| **P1** | 重写 09 为 Hermes 常驻调度循环（R-1） | L-2 | 控制面核心重构 |
| **P1** | 定义执行机制适配层（L-3） | L-2 | Hermes ↔ 执行载体的统一接口 |
| **P1** | 定义工具权限边界（L-4） | L-3 | 运行时安全 |
| **P2** | 修改 07 提示词分层和总调度规则（M-1/M-2/M-3/M-4/M-7） | R-1/R-3 | 角色提示词适配 |
| **P2** | 重写提示词 07/13/14/15（R-2/M-6） | R-1 | 调度三件套 |
| **P2** | 补漏 AGENTS.md 规则（L-1） | 无 | Context L1 标准化 |
| **P3** | 定义并行派发与结果汇总（L-5） | R-1 | 并行优化 |
| **P3** | 定义事件加速通道（L-6） | R-1 | 响应性优化 |

---

## 5. 风险与待确认项

| # | 风险/待确认 | 说明 | 建议 |
|---|---|---|---|
| F-1 | Hermes 持久化能力未实测 | V2.4 假设 Hermes 自带持久化，但未经实际验证 | 并入能力边界清单，实测后确认 |
| F-2 | ACP 协议版本不稳定 | ACP v0.4.x 仍在演进，API 可能变化 | 使用抽象适配层，不直接依赖具体版本 |
| F-3 | 单实例 vs 分布式 | 当前假设 Hermes 单实例，未来可能需要多实例 | 单实例是 V3.0 基线；V3.x+ 再考虑分布式调度 |
| F-4 | "0磋"语义未知 | Hermes迁移大纲中标记为语义未知 | 需向原作者确认后决定保留或替换 |

---

## 6. 附录：术语映射

| V2.4 术语 | Hermes 迁移后术语 | 变化说明 |
|---|---|---|
| Coordinator | Hermes（常驻总调度） | 从"任务线程"变为"常驻服务" |
| SharedRuntimeStatePath | Hermes 台账路径 | 从共享 JSON 文件变为 Hermes 管理的持久存储 |
| StateUpdateToolPath | Hermes 台账 API | 从文件锁脚本变为 Hermes 内部 API |
| ExecutionThreadID | ExecutionRef / SessionID | 统一为执行载体引用，不绑定 Codex 线程 |
| ExpectedExecutionKind: CodexThread | ExpectedExecutionKind: WorkBuddy | 主力开发载体从 Codex 线程改为 WorkBuddy |
| 周期巡检提示词 | Hermes cronjob 配置 | 从提示词驱动变为配置驱动 |
| 冷恢复提示词 | Hermes 重启恢复规则 | 从提示词驱动变为配置驱动 |
| 协调租约锁 | DispatchKey 幂等去重 | 单实例不需要锁，幂等键保证唯一 |

---

*本文由 Hermes 子任务 A（规范研究员）产出，供总调度审阅后纳入 V3.x 规范重构计划。*
