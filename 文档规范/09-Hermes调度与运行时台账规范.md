# Hermes 调度与运行时台账规范

> 规范版本：V2.5
> 规范状态：待审核
> 适用范围：由 Hermes 常驻总调度主导、WorkBuddy / Codex / Human 平行协作执行的开发迭代
> 前身：`09-调度控制平面与运行时台账规范`（V2.4，Codex 线程当总调度）
> 修订日期：2026-08-20
> 依赖底座：《Hermes能力边界清单》（实测能力）、《Hermes流程与边界决议》（边界决策）
> 作者：WorkBuddy（重写）
> 审核人：Richy（待审）
> 修订记录：见文末

## 1. 目的与单一职责

本文规定 Hermes 常驻总调度模式下的控制平面，包括：

- Hermes 台账（本地 JSON 状态文件 + 可选 SQLite）作为任务运行状态唯一真源；
- 派发、执行载体身份绑定、状态生产/消费与幂等；
- 事件驱动 + cron 兜底对账的双通道调度循环；
- 三类状态（执行载体状态、任务状态、证据状态）的分离；
- 单实例下的幂等去重（无协调租约锁）；
- 暂停、三级恢复（热/冷/灾难）与最大安全状态；
- 派发沙箱分级与 `danger-full-access` 授权边界；
- 调度协议的 Canary 验收。

角色选择与提示词组装以 `07-子Agent任务委派与提示词规范` 为准；开发、Review、验证与合并门禁以 `08` 为准。本文定义控制协议；Hermes 台账是该迭代任务运行状态的唯一真源，`CanonicalTaskDocumentPath` 中的 `TASK-STATE-EXCHANGE` 块保存任务定义与最近一次持久快照。

本规范不授权业务代码修改、外部操作、Review 通过或分支合并。Hermes 是调度器，**不代执行子 Agent 的 Git 与开发职责**。

## 2. 核心原则

1. **文档状态驱动**：没有开发任务文档中更高版本的待消费状态和不可歧义目标，不启动跨任务读取或推进任务状态。
2. **状态分离**：执行载体是否运行、代码任务达到什么阶段、证据是否已消费必须分别维护。
3. **持久可恢复**：调度器不依赖聊天上下文、UI 标签或模型记忆；Hermes 重启可读台账 + Git 重建。
4. **文档先发布、至少一次消费**：子任务先回报结果；允许重复读取同一记录，不允许重复产生副作用。
5. **最大安全状态**：恢复时只恢复到证据支持的最高状态，无法证明的内容保持待核实。
6. **控制面故障不污染业务状态**：派发、事件读取或台账写入故障不得伪装成代码 `Blocked` 或 Review Finding。
7. **单实例幂等**：Hermes 单机单实例，无多 Coordinator 竞争；幂等由 `DispatchKey` 去重保证，不设协调租约锁。
8. **事件驱动 + cron 兜底**：事件（飞书长连接 / Codex 网关轮询）优先，cron 定时对账兜底；事件丢失不影响正确性。
9. **执行机制显式配置**：每项派发记录 `ExpectedExecutionKind`、模型、沙箱；执行载体不可被未经授权替代。

## 3. 三类状态

### 3.1 执行载体状态（原 ThreadStatus）

描述执行载体的运行状态，不代表开发任务结论：

| 状态 | 含义 |
|---|---|
| NotCreated | 尚未派发创建 |
| Provisioning | 派发已发出，正式执行标识尚未绑定 |
| Running | 执行中或仍可继续 |
| Idle | 当前无活动；可能已存在未消费 final |
| NeedsAttention | 请求输入、授权或发生可恢复异常 |
| Completed | 执行已结束；出现待消费信号后必须读取 final 才能判断任务结论 |
| Unavailable | 当前无法读取或执行载体丢失 |
| Cancelled | 已明确取消执行载体 |

Codex 网关返回正式 `thread id` 时直接登记执行标识；只返回请求标识时表示 `Provisioning`，不得判定为创建失败。统一用 `ExecutionRef` 引用执行载体，不区分 `clientThreadId/threadId`。

### 3.2 任务状态（TaskState）

描述开发流程状态：

```text
Planned → ContextGenerationPending → Ready → InProgress → Submitted → InReview
  → ChangesRequested → ReworkInProgress → Submitted
  → TaskAccepted → MergePending → Integrated → IntegrationVerified
```

`ContextGenerationPending` 仅在 `ContextL2Policy=Required` 时出现：Hermes 先派发 Doc/Design Reviewer 生成 Context L2（须在其依赖满足后），其完成信号写回台账后才将任务推进到 `Ready`，再派发 Implementer。

`Blocked`、`Cancelled` 是有证据的旁路状态。候选版本继续使用 `ComponentVerified`、`SystemVerified`、`ExternalVerified`、`MergeApproved` 和 `Merged`。

### 3.3 证据状态（EvidenceState）

描述一个事件或结论是否已经可靠消费：

| 状态 | 含义 |
|---|---|
| Unread | 已知可能存在但尚未读取 |
| Received | 已取得内容，尚未完成校验 |
| PendingVerification | 来源存在但暂时不能交叉验证 |
| Validated | 身份、目标和内容已经校验 |
| Rejected | 证据不属于目标任务、格式不可恢复或内容无效 |
| Superseded | 被新 `HeadSHA`、新候选或批准的替代证据取代 |
| Consumed | 已完成幂等状态转换或后续动作 |

以下推断一律禁止：

- `Idle`、`Completed` 或 UI 停止 ⇒ 没有 Review 结论；
- Git HEAD 未变化 ⇒ 没有 Review、求助或环境事件；
- 派发请求标识尚未绑定正式执行标识 ⇒ 派发失败；
- 测试 `NotRun` ⇒ Review 必然阻塞；
- 代码已在迭代分支 ⇒ 合规达到 `Integrated`。

## 4. 不可变执行基线

每次派发必须绑定以下基线；不适用时明确 `N/A` 及理由：

```text
CodeBaseSHA
RequirementsBaselineRef
DesignBaselineRef
TaskDocumentBaselineRef
ContextBaselineRef
```

真实编码任务默认使用 Git 身份：

```text
TaskBranch
WorktreePath
CodeBaseSHA
HeadSHA
ReviewedCommitRange
ReviewedCommitSet
MergeTarget
```

文档基线必须使用不可变 Git commit/tree、受控文档 revision、diff hash 或项目负责人批准的稳定快照。不得把另一个持续变化工作区的绝对路径当作唯一设计基线。

## 5. 台账状态面

### 5.1 唯一状态真源

- **Hermes 台账**（`LedgerLocation`，本地 JSON 状态文件 + 可选 SQLite，**项目目录内、不进 Git**）是活动迭代中任务运行状态的**唯一实时真源**，由 Hermes 单实例维护。**台账绝不提交至 Git——即便执行载体有 ```danger-full-access``` 提交权限，任何 add/commit 也不得包含台账文件**（台账随开发变动会产生与提交的循环引用/自包含哈希，必须与版本库隔离）。
- **开发任务文档**（`CanonicalTaskDocumentPath`）中的 `TASK-STATE-EXCHANGE` 块是 **Git 持久快照**，用于跨重启/灾难恢复。
- 执行 Agent **不直接写台账**；通过载体通道回报结构化结果，由 Hermes 消费后写入台账（见 §8）。
- 台账只存定位与验证所需的摘要、SHA、Verdict、`ExecutionRef` 与下一动作；长日志、完整 diff、完整 final 不写入台账，落入派生证据目录。

### 5.2 迭代级最小字段

```text
SchemaVersion, ProtocolVersion, IterationID, HermesInstanceRef,
CoordinatorMode, Paused, CodeBaseSHA, RequirementsBaselineRef,
DesignBaselineRef, TaskDocumentBaselineRef, CanonicalTaskDocumentPath,
LedgerLocation, StateRevision, ConsumedRevision, UpdatedAt
```

### 5.3 任务级最小字段

```text
RecordID, TaskID, TaskType, Stage, InvocationID, ParentIterationID, Role,
ExpectedExecutionKind, ExpectedModel, ActualModel, ModelProvider, Sandbox,
TaskBranch, WorktreePath, CodeBaseSHA, HeadSHA,
ExecutionRef, CarrierStatus, TaskState, EvidenceState,
SignalRevision, SignalState, ProducedAt, ConsumedAt, ConsumedBy,
LastEventFingerprint, BlockerType, RecoveryConfidence, NextAction
```

`ExpectedModel` 是项目要求；无法验证 `ActualModel` 时记录 `Unknown`，不得自行宣称模型匹配。

## 6. 事件、幂等与派发去重

### 6.1 事件源与双通道调度

| 通道 | 覆盖 | 性质 |
|---|---|---|
| 飞书长连接（lark-ws） | 收 WorkBuddy 回复消息 | 事件，可能丢（已实证） |
| Codex 网关轮询 `GET /v1/threads/:id` | 查 Codex 线程进度 | 事件，主动轮询 |
| CI / 集成验证回写 | Integrator 合并精确 commit 后，运行/触发受影响集成检查，通过其载体通道（飞书/Codex）回报结构化协议头（含 `IntegrationCommit` + `IntegrationStatus`），由 Hermes 消费写入台账 | 事件，由执行载体回报，不依赖外部 webhook |
| 候选冻结（human-gated） | 项目负责人冻结迭代候选后告知 Hermes（或 Hermes 轮询到冻结标记），触发派发 Level 1（Validator / System Reviewer） | 事件，human-gated |
| Hermes cron 定时轮询 | **兜底对账**（约 1 分钟） | 兜底，补事件丢漏 |
| TG 推送 | 通知关键节点 | 输出，0 token |

**结论**：事件（飞书 + Codex 轮询 + CI 回写 + 候选冻结）优先消费，cron 定时对账兜底；事件丢失不影响正确性。

> **IntegrationVerified 回写说明**：`Integrated` 之后的 `IntegrationVerified` 不再依赖外部 CI webhook。执行主体为 **Integrator**（或独立 `IntegrationValidationTask`）：其合并精确 commit 后，运行/触发受影响集成检查，并通过自身载体通道（飞书/Codex）回报结构化协议头（含 `IntegrationCommit` + `IntegrationStatus=Passed/Failed`）；Hermes 消费该信号后写入台账，触发下游 Level 1。回报缺失视为待消费信号，由 cron 停滞检测升级（见 §11）。

### 6.2 幂等身份

- 派发幂等键：`DispatchKey = IterationID + TaskID + Stage + TargetIdentity`。同一 `DispatchKey` 已存在有效实例时禁止再次派发。
- 消费幂等键：`RecordID + SignalRevision`。重复读取同一记录只更新读取元数据，不重复创建 Review、返工或集成任务。
- 事件指纹 `Fingerprint` 至少覆盖事件类型、任务、Invocation、目标 SHA/候选和规范化结果。

### 6.3 因果关系

任何产生副作用的新派发必须记录：

```text
CausedByEventID
DispatchKey = IterationID + TaskID + Stage + TargetIdentity
```

## 7. 派发与身份绑定

### 7.1 派发 API

Hermes → 执行载体：

| 载体 | 派发方式 | 同步性 |
|---|---|---|
| WorkBuddy | 飞书 post 消息 @WB | 异步（单向投递） |
| Codex | `POST /v1/threads`，body `{prompt, cwd, model, modelProvider, sandbox, approval}` | 可同步 / 异步 |
| Human | 由 Hermes 通过 TG/飞书通知，人工作业 | 异步 |

`modelProvider`：`openai`（原生）/ `opencodex`（CodexSplit 第三方）。

### 7.2 ExecutionRef 统一身份

- 所有执行载体统一建模为 `ExecutionRef`，替代旧 `clientThreadId → threadId` 两段式映射。
- Codex 返回正式 thread id 时登记 `ExecutionRef = thread id`；只返回请求标识时登记 `Provisioning`，随后解析正式标识。
- 派发创建调用失败时登记 `ControlPlaneError`，不得登记业务 `Blocked`，不得改用其他执行机制兜底（除非项目负责人批准 `ApprovedEquivalent`）。
- **载体不可用升级阈值**：同一执行载体（按 `ExecutionRef` 识别）连续派发/读取失败达到 **3 次**仍不可用，Hermes 不得继续空转重试，须升级并告警项目负责人（Richy）；仅当 Richy 批准 `ApprovedEquivalent` 时才可换等效载体，否则保持 `Provisioning`/`NeedsAttention`/`PendingVerification` 并等待 Richy 介入。

### 7.3 沙箱分级与 git 授权

| 任务类型 | sandbox | 说明 |
|---|---|---|
| 只读调研 / 代码审核 | read-only | Reviewer、只读调查 |
| 写文档 / 写证据 | workspace-write | 不触碰 `.git` |
| **需要 git 提交**（开发 / 集成 / 合并） | danger-full-access | `.git` 在 workspace-write 下为 protected path |

需要 `git add/commit/merge/push` 的角色（Implementer、Integrator）使用 `danger-full-access`。Hermes 不代执行 git；worktree 创建由 Implementer 自办（Hermes 只派发 TaskID/base 分支/worktree 目录），审核由 Reviewer 只读进入，清理由 Integrator 统一执行（详见《Hermes流程与边界决议》C5/D2/D3b）。

## 8. 状态生产与消费协议

标准循环：

```text
Hermes 读台账
→ 事件到达（飞书收 WB 回复 / Codex 网关轮询到 completed）或 cron 对账触发
→ 收集 SignalState = PendingConsumption 的记录
→ 仅按该记录的 ExecutionRef 定向读取详细 final/求助
→ 校验后更新汇总状态、标记 Consumed 并派发后续
→ 无新信号：只做台账完整性与必要 Git 健康检查
```

事件消费顺序：

1. 收集待消费状态记录；没有 `PendingConsumption` 时禁止遍历任务系统寻找结果；
2. 以 `RecordID + SignalRevision` 计算幂等身份；
3. 仅对该记录绑定的 `ExecutionRef` 读取原始 final/求助内容；
4. 已消费则幂等忽略；
5. 记录 `Received`，不得先推进业务状态；
6. 校验 `ProtocolVersion`、`InvocationID`、`TaskID`、角色、分支和候选身份；
7. 用 Git、不可变文档或对应权威来源交叉验证事实；
8. 标记 `Validated`、`PendingVerification` 或 `Rejected`；
9. 仅 `Validated` 证据可以触发正常状态转换；
10. Hermes 在台账中写入汇总状态、消费确认和后续阶段记录，最后递增 `StateRevision`。

跨任务读取失败时保留 `PendingConsumption`，记录控制面错误并通知；下一轮仍只处理该待消费记录，不得退化为扫描所有执行载体，也不得把读取失败改写成业务 `Blocked`。同一待消费记录连续读取失败达到 **3 次**仍无法处理时，按 §7.2 载体不可用升级阈值告警 Richy。

若 final 缺少结构化协议头但内容可能有效，应先要求同一执行载体补发"仅协议头/缺失字段"，不得直接否定结果或机械创建新任务。执行载体不可用时保留原始内容并进入 `PendingVerification`。

## 9. 证据归属与冲突处理

不同来源只证明其职责范围：

| 事实 | 权威来源 |
|---|---|
| 任务范围、依赖和授权边界 | 已审核任务文档 |
| 提交、祖先和分支包含关系 | Git |
| 开发是否正式提交 | 台账记录 + Implementer final + Git 校验 |
| Review 结论 | 台账记录 + 独立 Reviewer final |
| Review 审核的是哪个代码 | Reviewer final + Git SHA |
| 执行载体是否运行或结束 | 飞书消息 / Codex 网关线程历史 |
| 项目负责人授权 | 明确用户消息或正式授权记录 |

来源冲突时必须并列记录。例如代码已经出现在迭代分支，但缺少 Review/集成证据时：

```text
CodePresence: PresentInIteration
ProcessState: IntegrationUnverified
RequiredAction: 审计来源、候选等价性和门禁，不自动回滚或宣布 Integrated
```

## 10. Review 执行状态与结论分离

Reviewer 输出必须分别包含：

```text
ExecutionStatus: Completed / Blocked
Verdict: Approved / ChangesRequested / NotIssued
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
```

规则：

- 测试 `NotRun` 不自动要求 `ExecutionStatus: Blocked`；Reviewer 仍可根据可验证反例形成 `ChangesRequested`；
- 工具调用格式、任务读取或模型执行协议异常属于 `ToolRuntime`，不是代码 Finding；
- 审核目标无法解析、仓库对象缺失等才属于 `RepositoryEnvironment`；
- 未经授权的真实外部操作属于 `Authorization`；
- `ExecutionStatus: Blocked` 时 `Verdict` 必须为 `NotIssued`；
- `Approved`/`ChangesRequested` 必须绑定精确审核目标；新 `HeadSHA` 自动使旧结论 `Superseded`。

测试职责分层（防 token 浪费，依据《Hermes流程与边界决议》§四）：

- Level 0 由 Implementer 跑一次并落证据目录；
- Reviewer 基于 diff + L0 证据审查，只对关键路径冒烟，**不重跑全套单测**；
- Level 1 + 回归由 Validator 仅在 merge 前跑全量一次。

## 11. cron 对账（周期巡检）

Hermes cron（约 1 分钟）对账每轮：

1. 读台账与 `Paused`；
2. `Paused=true` 时只报告暂停，不读取或派发业务后续；
3. 收集全部 `PendingConsumption` 记录；
4. 只对待消费记录定向读取其 `ExecutionRef`，校验并消费；
5. 无待消费记录时不调用任务列表或线程读取 API，只检查台账完整性、文档快照、必要 Git 依赖和健康；
6. 记录实际读取的文档版本、状态记录、定向执行载体和错误；
7. 继续等待下一轮或事件触发；
8. **停滞检测与升级**（见下）。

### 11.1 停滞检测

cron 对账在每轮额外检查长期无进展的任务，避免静默卡死（注意：仍在活动、有最近心跳或正在等待人类授权的任务不误报）：

- 任务持续处于 `ContextGenerationPending` / `Ready` / `InProgress` / `PendingConsumption` 超过 **2 个 cron 周期**（约 2 分钟）无任何状态推进或读取活动 → 记录停滞并告警；仍无进展超过 **4 个 cron 周期** → 升级项目负责人（Richy）。
- 任务处于 `Integrated` 但缺少 `IntegrationVerified` 回写（即无对应 CI/集成验证回写信号）超过 **2 个 cron 周期** → 记录停滞并告警；超过 **4 个 cron 周期** → 升级 Richy，由 Hermes 重派 `IntegrationValidationTask` 或人工介入。
- 依赖环检测：任务因上游长期 `Ready`/`InProgress` 未满足而无法推进时，cron 在停滞升级时一并报告依赖阻塞图，便于定位环依赖（环依赖的拒登记规则见 `04` §5.2）。

没有以下最小证据时不得报告"无变化"：

```text
TaskDocumentPath
LedgerLocation
StateRevisionBefore
StateRevisionAfter
PendingRecords
TargetedExecutionsRead
GitHeadsChecked
ControlPlaneErrors
```

对账不得只比较 Git HEAD，也不得重复派发已存在 `DispatchKey` 的指令；没有 `PendingConsumption` 时不得为了"确认无变化"逐载体读取。

## 12. 暂停与并发

### 12.1 暂停

项目负责人要求暂停时：

- 先设置 `Paused=true` 并记录事件；
- 停止新派发、返工、Review、集成和业务状态推进；
- 默认不取消已运行的子任务，除非项目负责人明确要求；
- 后续到达的事件可以原样保存为 `Received`，但恢复前不执行副作用；
- cron 对账仅报告暂停状态。

恢复必须记录 `Resumed`，并先执行未消费事件对账。

### 12.2 并发与并行

- Hermes 单机单实例，无多 Coordinator 竞争，**不设协调租约锁**；重复副作用由 `DispatchKey` 幂等去重阻断。
- 允许多任务并行；存在冲突（共享文件/同一 worktree）的任务不设为并行；实施中真产生冲突，找项目负责人协调（《Hermes流程与边界决议》C2）。
- 一个任务同一阶段只保留一个有效执行实例，除非已拆成互不冲突的子任务。

## 13. 恢复协议（三级）

| 级别 | 触发 | Hermes 动作 | 失败处理 |
|---|---|---|---|
| 热恢复 | 单次工具/调用偶发异常 | 记日志，从最近台账快照自动续跑；重试 | 连续重试 **3 次**仍失败 → 自动升级冷恢复 |
| 冷恢复 | 进程崩溃/重启 | 全自动读台账 + Git 重建 `ConsumedRevision` → `RecoveryOnly` 校验 → 恢复 | 失败自动升级灾难级 |
| 灾难恢复 | 台账丢失/损坏（或冷恢复失败） | 从 Git 任务文档快照重建，确定可证明状态，其余降级待核实 | **需项目负责人（Richy）介入批准** |

### 13.1 恢复升级阈值（显式）

- **热 → 冷**：同一热恢复动作连续失败达到 **3 次**仍无法续跑，Hermes 自动升级为冷恢复，不再无限重试；
- **冷 → 灾难**：冷恢复（读台账 + Git 重建 + `RecoveryOnly` 校验）失败，自动升级灾难级；
- **灾难**：必须等待项目负责人（Richy）介入批准 `RecoveryApproved` 后才退出 `RecoveryOnly`；
- 每次升级均记入恢复报告与台账，避免静默无限循环。

### 13.1 冷恢复步骤

1. 读台账；其缺失或损坏时，从 `CanonicalTaskDocumentPath` 最后一个结构有效的 `TASK-STATE-EXCHANGE` 快照恢复计划图、角色、依赖、基线、稳定状态和未消费记录；
2. 扫描 Git 分支、worktree、提交、祖先关系和迭代分支包含关系；
3. 仅对记录为 `PendingConsumption`、`PendingVerification` 或证据缺失的阶段，按其 `ExecutionRef` 定向发现执行载体；
4. 只读取上述目标任务历史；禁止把冷恢复重新实现为全量跨载体扫描；
5. 为 final、求助、提交、Review、返工和集成计算事件指纹；
6. 按来源职责交叉验证，并生成 `RecoveredEvidence`；
7. 对每项任务计算证据支持的最大安全状态；
8. 对冲突、缺失、错误执行机制和未知合入形成清单；
9. 写回台账并保存旧状态备份；在稳定门禁点把最新状态复制为开发任务文档持久快照；
10. 输出恢复报告；冷恢复**自动校验通过即完成（无需项目负责人介入）**，校验通过即退出 `RecoveryOnly` 恢复调度。

### 13.2 最大安全状态

- 仅发现任务分支 commit：`CodePresence=PresentInTaskBranch`、`TaskState=InProgress`（已有状态）；
- Implementer final 与 Git 均有效：最多恢复为 `Submitted`；
- 有效 `ChangesRequested`：恢复为 `ChangesRequested`，并检查是否已有新 Head；
- 有效 `Approved` 且目标等于当前 Head：可以恢复 `TaskState=TaskAccepted`，但 `RecoveryOnly` 仍禁止据此派发，必须先完成恢复审计；
- 当前 Head 已变化：旧 Review 标记 `Superseded`；
- 代码已在迭代分支但缺少合规集成证据：`IntegrationUnverified`，不自动回滚，也不宣称 `Integrated`；
- Review 历史丢失：重新 Review，不重新开发可验证提交；
- 授权记录丢失：重新取得授权，不从代码结果反推授权存在。

恢复事件必须带 `Recovered=true`、来源、原始时间、恢复时间和置信度：`Verified / Strong / Partial / Unknown`。只有 `Verified` 或项目规范明确允许的 `Strong` 可以支持正常状态转换。

## 14. 协议 Canary 与验收

真实高风险迭代启动前必须在无业务副作用场景测试控制平面，至少覆盖：

1. 派发只返回临时请求标识，能正确进入 `Provisioning` 并绑定正式 `ExecutionRef`；
2. 子任务早于 cron 对账周期完成，事件或下一次对账时能消费；
3. 飞书事件不可用，但台账存在 `PendingConsumption` 时仍能精确定位结果；
4. final 缺字段时要求原任务补发，不重复创建执行实例；
5. 重复读取同一 `RecordID + SignalRevision` 不重复派发；
6. 用户粘贴完整结果时进入核验，而不是直接否定或直接批准；
7. 工具运行错误进入 `ControlPlaneError`，不污染业务 `Blocked`；
8. 新 `HeadSHA` 使旧 Review 失效；
9. `Paused=true` 后不产生任何新副作用；
10. Hermes 可仅从台账和必要 Git 事实热恢复；台账丢失时可从开发任务文档快照冷恢复；
11. 删除派生证据目录后，台账不丢失且调度可以继续；
12. 指定执行载体失败时不会触发其他执行机制兜底（除非 `ApprovedEquivalent` 授权）；
13. 不同 worktree 解析出的需求、设计和任务基线内容一致；
14. 沙箱分级正确：workspace-write 拦 git、danger-full-access 放行 git；
15. cron 兜底在事件丢失后仍能对账消费，不产生重复副作用。

Canary 未通过时，真实业务任务保持 `Planned/Ready`，不得用真实开发验证调度协议。

### 14.1 Canary 失败处置（CanaryFailed）

Canary 任一场景失败时进入 `CanaryFailed` 状态，并按下述路径闭环，避免任务静默卡在 `Planned/Ready` 无人知：

1. 记录失败场景编号、现象与根因（控制面缺陷 / 载体异常 / 协议不匹配等）；
2. 责任人（Hermes 工程侧）修复控制平面或载体配置；
3. 重跑 Canary → 通过后解除真实业务任务的 `Planned/Ready` 派发冻结；
4. 连续 **3 次**重跑仍未通过 → 告警项目负责人（Richy）人工介入，不得继续自动重试掩盖问题；
5. 全程通过 TG / 飞书通知 Richy 当前 `CanaryFailed` 状态与处置进度。

`CanaryFailed` 不是业务 `Blocked`，不污染代码任务状态；它只冻结真实高风险迭代的启动，直至 Canary 通过或 Richy 明确记录临时豁免。

## 15. 审核清单

- [ ] 执行载体状态、任务状态和证据状态已经分离；
- [ ] 唯一 `LedgerLocation`、`CanonicalTaskDocumentPath`、Schema 明确；台账不进 Git，`TASK-STATE-EXCHANGE` 为持久快照；
- [ ] 任务具有 Invocation、`ExecutionRef` 和执行载体状态；
- [ ] 所有派发具有幂等 `DispatchKey` 和因果事件；
- [ ] 代码与文档基线均不可歧义；
- [ ] 状态消费使用 `RecordID + SignalRevision`，没有待消费记录时不扫描执行载体；
- [ ] 派发请求标识等 Provisioning 成功未被当作失败；
- [ ] 项目要求的执行机制不可被未经授权替代；
- [ ] Review 执行状态和 Verdict 已分离；
- [ ] 沙箱分级正确，需要 git 的角色使用 danger-full-access；
- [ ] cron 对账能证明实际读取台账，并仅对 `PendingConsumption` 记录定向读取执行载体；
- [ ] 暂停和 `DispatchKey` 幂等去重能阻止重复副作用；
- [ ] 热、冷、灾难恢复均有明确门禁；
- [ ] Canary 覆盖重复状态、早完成、事件丢失、跨 worktree 统一、沙箱分级和并发对账；
- [ ] 恢复结果必须审核后才能重新派发。

## 16. 完成定义

只有当项目已配置 Hermes 台账、不可变执行基线、合法状态转换、幂等派发、暂停、三级恢复和 Canary，并通过项目负责人审核后，Hermes 才可调度真实高风险迭代。

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.5（待审核） | 2026-08-20 | WorkBuddy | 由 V2.4「09-调度控制平面与运行时台账规范」重命名+重写为 Hermes 常驻总调度模式（台账schema/三类状态/DispatchKey幂等/事件+cron双通道/单实例无租约/ExecutionRef/三级恢复/沙箱分级） |
| V2.5（待审核） | 2026-08-20 | Hermes | 补齐作者/审核人/修订记录元信息 |
| V2.5（待审核） | 2026-08-20 | Hermes | 审阅修订：台账绝不可提交至Git(防循环引用/自包含哈希)；冷恢复自动校验通过即完成无需Richy；EvidenceIncomplete改为已有状态InProgress |
| V2.5（待审核） | 2026-08-20 | Hermes | 补流程缺口7条：§3.2增 ContextGenerationPending；§6.1增 CI/集成验证回写 + 候选冻结 human-gated 事件；§7.2/§8载体不可用连续3次升级 Richy；§11.1停滞检测（含 IntegrationVerified 超时升级）；§13.1恢复升级阈值（热→冷→灾难，热3次）；§14.1 CanaryFailed 状态+处置路径+连续3次告警 Richy |
