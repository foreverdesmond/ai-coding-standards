# 调度协议 Canary（Hermes 闭环 Canary 配置）

> 规范版本：V2.5
> 文档状态：已审核通过（V2.5 定稿基线）
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（已审核）

目标：在不修改业务代码、不连接真实外部资源的条件下，验证 `<iteration-id>` 的任务派发、台账状态生产/消费、幂等、暂停和恢复协议。Canary 未通过时不得启动真实高风险任务；既有迭代仅可由项目负责人明确记录临时豁免。

## 1. 输入

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
CanaryInvocationID: <invocation-id>
HermesInstanceRef: <hermes-instance-id>
LedgerLocation: <isolated-canary-ledger>
CanonicalTaskDocumentPath: <isolated-shared-document-path>
DispatchMode: EventDriven + CronFallback
ExpectedExecutionKind: <execution-kind>
ImplementerModel: <model>
ReviewerModel: <model>
SafeRepositoryTarget: <read-only or disposable target>
```

## 2. 规则

- 使用隔离 Canary 台账、任务文档快照、派生证据目录和无业务副作用目标；
- 不修改生产代码、业务分支、数据库、外部服务或真实配置；
- 每项检查保留原始派发结果、ExecutionRef、StateRevision、状态记录、定向 final、Fingerprint 和状态转换证据；
- 失败时停止真实迭代启动，不通过扩大权限掩盖问题。

## 3. 最低场景（保留原 14 场景精神 + 新增常驻场景）

1. 派发只返回临时请求标识，进入 Provisioning 并最终绑定正式 `ExecutionRef`；
2. 执行载体在 cron 对账周期前完成，先回报 `PendingConsumption`，Hermes 按台账版本立即消费；
3. 事件（飞书 / Codex 网关）不可用，但台账状态记录存在，仍能精确读取对应 final；
4. 缺协议字段时由同一执行载体补发，不重复创建；
5. 同一 `RecordID + SignalRevision` 重复读取只处理一次；
6. 导入一份用户提供的结构化结果，进入核验而非直接否定/批准；
7. 模拟 ToolRuntime 错误，不污染业务 Blocked；
8. 新 Head 使旧 Review Superseded；
9. Paused 后没有新派发或状态副作用；
10. 热恢复只依赖台账和必要 Git 事实，保持状态和幂等键；
11. 删除派生 Canary 台账后，状态真源仍可继续调度；
12. 指定执行载体失败时不会触发其他执行机制兜底（除非 `ApprovedEquivalent` 授权）；
13. 不同 worktree 读取的不可变文档基线一致，运行状态都通过 Hermes 写入同一个台账；
14. 沙箱分级正确：workspace-write 拦 git、danger-full-access 放行 git；
15. **【新增】多实例防重**：模拟两个触发源，最多一个产生副作用，DispatchKey 幂等去重生效；
16. **【新增】事件丢失后 cron 兜底**：模拟事件丢失，cron 对账仍能消费且不重复派发；
17. **【新增】Hermes 重启热恢复**：模拟进程重启，读台账 + Git 重建后状态一致；
18. **【新增】Canary 失败处置闭环**：模拟某场景失败进入 `CanaryFailed`，记录失败场景/根因、责任人修复、重跑、连续 3 次失败告警 Richy、通过后才解除真实任务 `Planned/Ready` 冻结（与 `09` §14.1 一致）。

## 4. 输出

```text
ProtocolVersion:
EventType: CoordinatorCanaryResult
IterationID:
CanaryInvocationID:
ExecutionStatus: Completed / Blocked
ScenarioResults: <1-18 Pass/Fail + evidence>
DuplicateDispatchCount:
MissedEventCount:
IncorrectStateTransitionCount:
UnauthorizedFallbackCount:
RecoveryComparison:
ControlPlaneErrors:
CanaryFailedScenarios: <失败场景编号 + 根因 + 处置进度 + 是否告警 Richy>
Verdict: CanaryPassed / CanaryFailed / NotIssued
RequiredFixes:
```

只有 18 项全部通过、四个错误计数均为 0 且恢复比较一致时，才允许 `CanaryPassed`。任一场景进入 `CanaryFailed` 时按 `09` §14.1 处置：记录根因 → 修复 → 重跑 → 连续 3 次失败告警 Richy，真实高风险迭代保持冻结直至 Canary 通过或 Richy 明确记录临时豁免。

---

## 修订记录

| 版本 | 日期 | 作者 | 变更说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex 调度协议 Canary 提示词（隔离共享 JSON + 锁路径 + Codex 创建兜底禁令） |
| V2.5 | 2026-08-20 | WorkBuddy | 重写为 Hermes 闭环 Canary：删隔离共享 JSON/锁/Codex 兜底禁令；新增多实例防重、事件丢失 cron 兜底、Hermes 重启热恢复 |
| V2.5（待审核） | 2026-08-20 | Hermes | 增场景 18 Canary 失败处置闭环（CanaryFailed→修复→重跑→连续3次告警 Richy），输出增 CanaryFailedScenarios 字段，与 09 §14.1 一致 |
| V2.5 定稿 | 2026-08-20 | WorkBuddy | 评审通过，标记为 V2.5 正式基线 |
