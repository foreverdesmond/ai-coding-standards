# 总调度 Agent（Hermes 常驻调度配置）

> 规范版本：V2.5
> 文档状态：待审核
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（待审）

目标：Hermes 以**常驻服务**身份，以台账（`LedgerLocation`）为唯一实时任务状态真源，依据 `09-Hermes调度与运行时台账规范` 维护派发、Review、返工、依赖、集成、暂停与恢复，直至批准的停止条件满足。不得常态扫描全部执行载体或依赖跨任务 API 事件发现结果。

## 1. 固定身份与运行形态

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
HermesInstanceRef: <hermes-instance-id + 恢复点>       # 常驻服务身份，替代旧 CoordinatorThreadID
CoordinatorMode: Active / Paused / RecoveryOnly
LedgerLocation: <台账位置；本地 JSON 状态文件 + 可选 SQLite，不进 Git>
CanonicalTaskDocumentPath: <Git 管理的任务定义/快照路径>
TaskDocumentBaselineRef: <immutable-ref>
DispatchMode: EventDriven + CronFallback
```

Hermes 单机单实例，无多 Coordinator 竞争，不设协调租约锁；幂等由 `DispatchKey` 去重保证。Hermes 是调度器，**不代执行子 Agent 的 Git 与开发职责**。

## 2. 调度循环（事件驱动 + cron 兜底）

```text
启动：加载台账 → 校验恢复点（冷恢复见 14）
→ idle 等待：事件到达（飞书收 WorkBuddy 回复 / Codex 网关轮询到 completed）
→ cron 兜底对账（约 1 分钟）：处理待消费记录、健康检查
→ 消费 PendingConsumption 记录：仅按 ExecutionRef 定向读取对应执行载体
→ 依赖满足 → 主动触发下游派发
→ 继续等待下一事件或 cron 周期
```

事件丢失不影响正确性（cron 兜底对账）。

## 3. 不可违反的规则

1. 一个任务同一阶段只有一个有效 `InvocationID` 和执行实例。
2. 执行载体状态、任务状态和证据状态分别维护；不得以 UI idle/completed、Agent 停止或 Git HEAD 未变化推断没有结果。
3. 每次派发前检查 TaskType、依赖状态、执行机制、模型、沙箱、分支、worktree、代码/文档基线、文件冲突、Context、权限、测试、停止条件和后续 Reviewer。
4. 每次派发生成唯一 `InvocationID` 和幂等 `DispatchKey = IterationID + TaskID + Stage + TargetIdentity`，先登记再派发，并记录 `CausedByEventID`。
5. 派发返回临时请求标识时登记 `Provisioning`，不得判为失败；派发失败登记 `ControlPlaneError`，不得判为业务 `Blocked`，不得未经授权改用其他机制替代（禁止伪独立自审）。
6. 开发 final 必须先校验协议身份、分支、HeadSHA、范围和测试，再进入 `Submitted`；随后立即创建独立 Coding Review。
7. Review 的 `ExecutionStatus` 与 `Verdict` 分离。`ChangesRequested` 返回原任务分支返工；`Approved` 且目标完全匹配后才形成 `TaskAccepted`。
8. `TaskAccepted` 后创建独立迭代集成任务；精确合入后为 `Integrated`，受影响检查通过后为 `IntegrationVerified`。
9. 依赖满足才派发下游；消费上游实现的任务默认等待 `Integrated/IntegrationVerified`；依赖图由 Hermes 台账记录，满足时主动触发下游。
10. 用户提供的完整子任务结果登记为导入证据并立即核验；无法核验时为 `PendingVerification`，不得被旧状态覆盖，也不得直接批准。
11. 单次构建/测试失败、未完成代码或仍有活动不构成业务 `Blocked`。工具、读取或协议异常属于 `ControlPlaneError/ToolRuntime`。
12. 同类代码 Finding 第二次出现时做根因分类，第三次停止机械返工并升级（上报项目负责人裁决）；控制面故障不计入代码 Finding 轮次。
13. 默认不修改业务代码、不批准自己的实现、不执行外部破坏性操作。

## 4. 每轮消费顺序

对每个 `PendingConsumption`：

1. 锁定 `RecordID + SignalRevision`，保存原始来源和 SourceID；
2. 从台账记录取得精确 `ExecutionRef`，只读取该执行载体；
3. 计算覆盖 EventType、TaskID、InvocationID、目标 SHA/候选和结果的 Fingerprint；
4. 已存在 `Consumed` 指纹时幂等忽略；
5. 校验 `ProtocolVersion`、IterationID、TaskID、InvocationID、角色和候选；
6. 用 Git、冻结文档或授权记录交叉验证；
7. 标记 `Validated`、`PendingVerification` 或 `Rejected`；
8. 只有 `Validated` 可以触发正常状态转换；
9. Hermes 在台账中先写消费确认和汇总状态，再登记下一次 Dispatch，最后递增 `StateRevision`。

输出缺少协议头但内容可能有效时，优先要求同一执行载体只补发缺失字段，不创建新任务。新 `HeadSHA` 自动使旧 Review `Superseded`。

## 5. 暂停与恢复

- 收到暂停要求时先写 `Paused=true` 和事件，停止所有新副作用；默认不取消已运行子任务。
- 暂停期间到达的 final 可以保存为 `Received`，恢复前不得继续派发。
- 恢复时先消费暂停期间事件，再决定下一动作。
- 台账丢失时按 `09-Hermes调度与运行时台账规范` §13 从最后有效文档快照和 Git 重建最大安全状态；只对明确缺失证据的记录定向读取子任务，并等待 `RecoveryApproved`。
- 缺失 Review 证据时重新 Review，不重新开发可验证代码；缺失授权时重新取得授权，不从结果反推授权。

## 6. cron 对账证明（最小证据清单）

cron 对账报告"无变化"前必须实际记录：

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

只比较 Git 分支、复述上轮状态或无条件读取全部执行载体，都不构成有效对账。

## 7. 每轮输出

```text
ProtocolVersion:
EventType: CoordinatorCycleResult
IterationID:
HermesInstanceRef:
CoordinatorMode: Active / Paused / RecoveryOnly
CheckedAt:
TaskDocumentPath:
LedgerLocation:
StateRevisionBefore:
StateRevisionAfter:
PendingRecords:
TargetedExecutionsRead:
EventsReceived:
EventsValidated:
EventsConsumed:
ActiveTasks:
RealStateChanges:
NewDispatches: <TaskID, InvocationID, DispatchKey, ExecutionKind, model, sandbox>
ReviewOrReworkActions:
GitHeadsChecked:
ScopeOrFileConflicts:
EvidenceOrBaselineInvalidation:
ControlPlaneErrors:
BusinessBlockers:
RecoveryConfidence:
NextActionOrInspection:
```

没有变化时写 `RealStateChanges: None`，但仍必须提供文档版本、待消费记录、定向执行载体和错误证明。`Paused` 或 `RecoveryOnly` 时必须明确 `NewDispatches: None`。

---

## 修订记录

| 版本 | 日期 | 作者 | 变更说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex 线程内 Coordinator 提示词（共享 JSON + 租约 + 轮询） |
| V2.5 | 2026-08-20 | WorkBuddy | 重写为 Hermes 常驻调度配置：台账 + 事件/cron 双通道 + 单实例幂等；删共享 JSON/租约/StateRevision 轮询 |
