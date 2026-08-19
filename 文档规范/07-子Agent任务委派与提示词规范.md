# 子任务与 Agent 委派及提示词规范

> 规范版本：V2.5
> 规范状态：待审核（V2.3 仍为上一已审核基线）
> 适用范围：使用 Agent 执行开发、Review、测试、调度或合并审核
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 修订日期：2026-08-20
> 审核人：Richy（待审）

## 1. 单一职责

本文只规定如何选择执行机制和角色、组装提示词以及保持角色独立。各角色的详细执行契约以 [`提示词模板`](./提示词模板/README.md) 为唯一真源，不在正文重复；任务身份、事件、台账、巡检和恢复以 [`09-Hermes调度与运行时台账规范`](./09-Hermes调度与运行时台账规范.md) 为唯一真源。

规范定义能力，不绑定模型、供应商或版本。具体执行实例由项目配置。

## 2. 角色按需选择

V2.5 收敛为 6 种核心角色（详见 README §4），每个任务按需启用 2~N 个：

| 角色 | 核心职责 | 关键边界 |
|---|---|---|
| Coordinator（总调度） | 派发、巡检、判 gate | Hermes 常驻服务承担，不做子 Agent 实操 |
| Implementer（开发） | 首次开发 + 返工 + L0 单测 | 只在自己 feature 分支；不批准自己 |
| Reviewer（审核） | 审核开发成果（diff/commit） | 基于 diff + L0 证据审；只读，不 merge |
| Integrator（集成） | 合并 feature → iteration | 只合并已 approved 的精确 commit；不自己审自己的 merge |
| Validator（测试验证） | 合并后全量测试（L1+回归） | 在集成分支跑全量；只读/测试环境；不 merge |
| Doc/Design Reviewer（文档/设计审核） | 文档审核 + 设计 + 工作包/上下文 | 纯文档工作归其；低风险设计不审、高风险同角色他审 |

**合并来源（V2.4 → V2.5）**：Rework Implementer → Implementer（返工是开发的续集）；System Reviewer → Reviewer（按 Level 区分）；Merge Reviewer → Reviewer（合并资格审核视角）；Iteration Integrator / Main Merge Executor → Integrator（同为合并执行，仅目标分支不同）；Requirements/Design/Task Reviewer → Doc/Design Reviewer（合一）。

**保留底线**：开发与 Review 分离；合并执行与审查分离；最终批准不问责开发者自己。轻量任务通常只需 Implementer + 独立 Reviewer。

### 2.1 执行机制必须显式配置

角色和执行机制是两个不同概念。项目必须为每次派发记录：

```text
ExpectedExecutionKind: WorkBuddy / Codex / Human / ApprovedEquivalent
ExpectedModel
Role
InvocationID
```

- 执行载体（WorkBuddy / Codex / Human）在派发时显式指定；Hermes 按 `09-Hermes调度与运行时台账规范` 派发并绑定 `ExecutionRef`；
- 派发返回临时请求标识表示 `Provisioning`，不是失败；
- 派发失败或载体不可用时登记 `ControlPlaneError` 并停止，不得改用其他机制未经授权替代（禁止伪独立自审）；
- 只有项目负责人事先批准 `ApprovedEquivalent` 时才允许等价执行机制；
- 角色独立性不能仅靠不同名称证明，必须有不同 `InvocationID` 和执行实例。

## 3. 提示词分层

不再要求所有角色共享一份开发任务基座。按角色使用：

### 3.1 真正公共基座

只包含：

- 协议版本、IterationID、TaskID、InvocationID、父调度身份和执行机制；
- 仓库、工作区和审核目标；
- `CodeBaseSHA` 与适用的需求、设计、任务、Context 不可变基线；
- 用户已有改动保护；
- 外部与破坏性操作授权；
- 证据真实性和 `NotRun`；
- 角色允许状态和停止条件。

### 3.2 开发角色增量

- 已审核需求、设计和任务简报；
- Context L2（仅适用时）；
- 修改职责、必须调查范围和完成证据；
- 自主探索和 SOLID 硬门禁。
- 唯一任务分支、worktree、`CodeBaseSHA`、允许 commit 的范围和禁止合并的目标分支。
- 结构化 `DevelopmentSubmission` 事件头和必须返回的 Invocation 身份。

### 3.3 Review 角色增量

- 需求和设计基线；
- 不可歧义审核目标与实际 diff；
- Reviewer 先独立形成风险模型，再读取开发上下文包；
- SOLID、测试证明边界和独立风险假设；
- 不允许直接修改代码或更新最终任务状态，除非明确授权角色变化。
- `TaskBranch`、`CodeBaseSHA`、`HeadSHA` 与准确 `CodeBaseSHA..HeadSHA`；Review 不得改审当前主工作区或其他分支。
- `ExecutionStatus` 与 `Verdict` 分离的结构化 `CodingReviewResult` 事件头。

### 3.4 候选版本角色增量

- 候选提交/PR revision/稳定快照；
- Level 证据、NotRun、分支策略和目标门禁；
- 不要求单任务 ID 或 Context L2。

### 3.5 合并执行角色增量

- 来源分支、目标分支、准确候选 SHA、有效 Review 结论和授权记录；
- 允许的 Git 动作、冲突处理边界、合并后证据和禁止夹带开发修改；
- `Iteration Integrator` 与 `Main Merge Executor` 必须使用不同于普通开发任务的权限契约。

## 4. 提示词组装

```text
公共基座（Hermes 注入协议）
  + 一个角色模板
  + 该角色需要的正式基线
  + 可选 Context L2
  + 本轮动态数据（Invocation、diff、Findings、测试或候选证据）
```

动态数据（Invocation、台账字段、diff、Findings、测试或候选证据）由 Hermes 在派发时注入，执行 Agent 不自行拼装基线。

不得只发送“按文档完成任务”，也不得无差别灌入整个文档目录。

## 5. 角色独立性

- 开发与 Review 使用不同执行实例或明确的独立复核人；
- Reviewer 不以开发者总结为事实，也不先被开发者推荐的风险范围限制；
- 同一模型可以承担不同任务，但不能在同一任务中把自己的实现直接批准；
- 高风险最终 Review 应使用项目当前可用的强独立推理能力；
- 执行实例替换时记录恢复点和需要重新读取的材料。
- 执行实例替换必须生成新 `InvocationID`，旧实例标记失效或取消；不得让两个实例同时成为同一阶段的有效真源。
- Coding Review 必须创建独立任务或独立执行实例，不得把“请审核”发回原开发任务；
- Merge Reviewer 只审核合并资格，Merge Executor 只执行已经批准的精确合并；审核和执行默认分离。
- 载体分离（WorkBuddy 写 / Codex 审）不等于视角独立；不同载体仍可能共享上游上下文与判断，必须保留独立的审核目标与风险模型，不得仅凭载体不同即视为独立审核。

## 5.1 Git 权限

- Implementer 与 Rework Implementer 只能在自己的任务分支创建 commit；
- 普通开发角色不得直接 commit 或 merge 到迭代开发分支、长期集成分支、稳定分支或主分支；
- Reviewer、Validator 和 Coordinator 默认只读，不得通过修改代码形成通过结论；
- Iteration Integrator 只能合并指定且已 `TaskAccepted` 的提交到迭代开发分支；
- Main Merge Executor 只能在 `MergeApproved` 和项目负责人明确授权后合并准确候选到主分支；
- 发生需要修改实现的冲突时，合并执行者停止并返回开发/返工闭环。

## 6. Reviewer 的风险反证

Reviewer 必须形成独立风险假设。

- 存在可执行且重要的风险时，至少验证一个开发者未覆盖的反例；
- 没有值得执行的额外反例时，说明判断依据；
- 不得为了满足数量配额制造无意义测试；
- 反例不能代替需求、设计和 SOLID 全面审查。

## 7. SOLID 角色责任

- Implementer 在提交时说明本次改动的 SOLID 影响；
- Coding Reviewer 独立完成 S/O/L/I/D 判断；
- System Reviewer 检查任务组合后是否出现职责泄漏、错误抽象或依赖倒置破坏；
- 低风险任务可以简洁记录，高风险或存在取舍时必须展开；
- SOLID 始终是硬门禁，不因 Context L2 省略或开发路线简化而取消。

## 8. 总调度最低规则

使用 Coordinator 时：

- 一个任务同一阶段只保留一个有效开发实例；
- 状态以实际结果、审核目标和 diff 为准；
- 开发完成进入独立 Review，失败回原任务返工；
- Review 通过先形成 `TaskAccepted`；之后派发独立迭代集成任务，形成 `Integrated`，再执行受影响集成验证形成 `IntegrationVerified`；
- 依赖满足才派发下游；需要上游实现的依赖默认等待 `Integrated`；依赖图由 Hermes 台账记录，依赖满足时由 Hermes 主动触发下游；
- 派发后由 Hermes 通过事件（飞书 / Codex 网关轮询）+ cron 兜底消费台账；执行 Agent 通过载体通道回报，Hermes 写入 `PendingConsumption` 后立即处理；
- 对账只处理状态变化，不重复发送同一指令；
- 活动心跳和长日志可以保存在派生证据/审计缓存中；任务状态、Invocation、执行引用、待消费信号和消费确认必须写入 Hermes 台账；
- 运行时状态由 Hermes 按 `09-Hermes调度与运行时台账规范` 写入台账；台账不可用时按恢复协议处理；
- 每次派发具有 `InvocationID`、`DispatchKey`、不可变代码/文档基线和因果事件；
- 创建请求、正式任务、线程状态、任务状态和证据状态分别登记；
- 所有状态信号按 `RecordID + SignalRevision` 至少一次读取并幂等消费；
- 项目暂停时停止新副作用，恢复时先处理未消费事件；
- 总调度不批准自己的实现。

总调度至少维护以下运行时映射，不得仅凭 UI 标签推断：

```text
RecordID, TaskID, TaskType, InvocationID, ExpectedExecutionKind, ExpectedModel,
ExecutionRef, TaskBranch, WorktreePath,
CodeBaseSHA, RequirementsBaselineRef, DesignBaselineRef,
TaskDocumentBaselineRef, HeadSHA, ImplementerTask, ReviewerTask,
CarrierStatus, TaskState, EvidenceState, SignalRevision, SignalState,
ProducedAt, ConsumedAt, LastEventFingerprint
```

### 8.1 派发前强制清单

任何任务派发前必须检查：

- TaskType、角色模板和允许输出状态；
- 依赖类型及所需状态；
- 任务分支、worktree、CodeBaseSHA、MergeTarget 和候选形式；
- InvocationID、执行机制、预期模型、父调度身份和幂等 DispatchKey；
- 需求、设计、任务和 Context 的不可变基线；
- 文件范围与其他活跃任务是否冲突；
- Context L2 是否适用、存在且仍有效；
- Git 权限、外部副作用权限和停止条件；
- 预期测试范围、交付证据和后续 Reviewer；
- 同一阶段是否已有有效实例，防止重复派发。
- Hermes 台账（LedgerLocation）可写、单实例幂等（DispatchKey 去重）、当前不处于 Paused/恢复中；

任何必填项缺失或冲突未解决时不得派发。

### 8.2 状态生产、消费与周期巡检

标准调度循环：

```text
Hermes 在台账登记 DispatchKey/Invocation/状态记录并派发
→ 绑定 ExecutionRef（统一执行引用）
→ 事件（飞书/Codex 网关轮询）到达或 cron 兜底对账触发
  → PendingConsumption：只读取记录绑定的执行载体，校验交付，再幂等派发下一动作
  → 无新状态：不遍历载体，只做台账/文档/Git 健康对账
```

交付校验至少包括：协议版本、Invocation、最终状态、`HeadSHA`、预期文件、测试证据、范围外改动和后续 Review/集成动作。任务在第一分钟完成时必须立即处理，不等待第七分钟或其他配置周期。

UI 显示 idle/completed、Git HEAD 未变化或读取接口暂时无结果，都不能单独证明“没有 Review/完成状态”。任务状态发现只以开发任务文档为准；用户提供的完整子任务结果应写入对应记录的 `PendingVerification` 并立即核验，不得被旧状态覆盖。

### 8.3 中断与阻塞判断

以下中间现象不能单独作为中断理由：未完成代码、单次编译失败、单次测试失败、长时间测试仍有输出或 worktree 持续变化。

只有执行者明确报告 `Blocked`/`NeedsScopeChange`、出现授权或文件范围冲突、项目负责人要求停止，或连续检查确认无活动且无恢复路径时，才允许中断。中断前必须读取最新消息、任务状态和可用活动证据。

### 8.4 Review 返工循环升级

- 第一轮 `ChangesRequested` 正常返工；
- 第二轮出现同类 Finding 时，Hermes 必须比较两轮结论并分类根因；
- 第三次仍出现同类问题时停止机械派发，判断实现能力不匹配、需求/设计歧义、候选绑定错误或证据规则错误，并向项目负责人报告需要的裁决；
- 代码 `HeadSHA` 未变化且仅更新状态区/证据时，不重新执行完整 Coding Review 或全量测试。

## 9. 提示词质量门禁

- [ ] 角色和允许输出状态明确；
- [ ] 协议版本、IterationID、TaskID、InvocationID、父调度身份和执行机制明确；
- [ ] 审核目标不可歧义；
- [ ] 只提供该角色真正需要的基线；
- [ ] Context L2 只在触发时提供；
- [ ] 修改权限和只读调查分开；
- [ ] 任务分支、CodeBaseSHA、HeadSHA、ReviewedCommitRange/ReviewedCommitSet 和合并目标明确；
- [ ] 代码、需求、设计、任务和 Context 基线不可歧义；
- [ ] SOLID、测试、证据和 NotRun 责任明确；
- [ ] 外部操作边界明确；
- [ ] Review 视角未被开发上下文提前限制；
- [ ] 停止条件明确；
- [ ] 已通过派发前清单，并配置文档状态生产/消费与周期巡检；
- [ ] 已配置唯一台账（`LedgerLocation`）、幂等 `DispatchKey`、单实例幂等去重（无协调租约锁）、暂停、三级恢复和适用 Canary；
- [ ] 指定执行机制（WorkBuddy/Codex/Human）不被未经授权替代（禁止伪独立自审）；
- [ ] 未绑定不必要的具体模型、业务或工具。

## 10. 模板维护

模板可以按项目扩展，但角色目标、权限、输出状态和禁止行为不能互相混用。若提示词变化影响正在执行的任务，应发送完整更新后的角色契约，而不是零散补充造成冲突。

## 11. 完成定义

只有当项目选择了最小必要角色、为每个角色填写正确基线与动态数据、保持开发和 Review 独立，并通过提示词质量门禁后，Agent 委派才可启动。

---

## 修订记录

| 版本 | 日期 | 作者 | 变更说明 |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 已审核通过基线 |
| V2.4 | 2026-08-15 | — | 引入 CodexThread/SubAgent 执行机制与共享 JSON 轮询 |
| V2.5 | 2026-08-20 | WorkBuddy | §2.1 枚举四档（WorkBuddy/Codex/Human/ApprovedEquivalent）；§4 动态数据 Hermes 注入；§5 载体分离≠视角独立；§8 共享 JSON→Hermes 台账 + 事件/cron；ExecutionThreadID→ExecutionRef |
| V2.5 | 2026-08-20 | Hermes | 审阅修订：§2 角色列表收敛为 6 种核心角色，扩展角色并入映射说明 |
