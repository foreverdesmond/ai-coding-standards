# 调度协议 Canary

目标：在不修改业务代码、不连接真实外部资源的条件下，验证 `<iteration-id>` 的任务创建、共享本地状态生产/消费、幂等、暂停和恢复协议。Canary 未通过时不得启动真实高风险任务；既有迭代仅可由项目负责人明确记录临时豁免。

输入：

```text
ProtocolVersion: <protocol-version>
IterationID: <iteration-id>
CanaryInvocationID: <invocation-id>
CoordinatorThreadID: <coordinator-thread-id>
RuntimeLedgerLocation: <isolated-canary-ledger>
CanonicalTaskDocumentPath: <isolated-shared-document-path>
SharedRuntimeStatePath: <isolated-shared-json-path>
TaskStateWriteLockPath: <isolated-lock-path>
StateUpdateToolPath: <atomic-update-tool-path>
ExpectedExecutionKind: <execution-kind>
ImplementerModel: <model>
ReviewerModel: <model>
SafeRepositoryTarget: <read-only or disposable target>
```

规则：

- 使用隔离 Canary 共享 JSON、任务文档快照、派生台账和无业务副作用目标；
- 不修改生产代码、业务分支、数据库、外部服务或真实配置；
- 项目要求 Codex 子任务时只使用 Codex 任务创建机制；
- 每项检查保留原始创建结果、任务 ID、StateRevision、状态记录、定向 final、Fingerprint 和状态转换证据；
- 失败时停止真实迭代启动，不通过扩大权限掩盖问题。

最低场景：

1. 创建只返回临时请求标识，进入 Provisioning 并最终绑定正式任务；
2. 子任务在巡检周期前完成，先写 `PendingConsumption`，Coordinator 按共享状态版本立即消费；
3. UI/跨任务事件不可用，但共享状态记录存在，仍能精确读取对应 final；
4. 缺协议字段时由同一任务补发，不重复创建；
5. 同一 `RecordID + SignalRevision` 重复读取只处理一次；
6. 导入一份用户提供的结构化结果，进入核验而非直接否定/批准；
7. 模拟 ToolRuntime 错误，不污染业务 Blocked；
8. 新 Head 使旧 Review Superseded；
9. Paused 后没有新派发或状态副作用；
10. 热恢复只依赖状态区和必要 Git 事实，保持状态和幂等键；
11. 删除派生 Canary 台账后，状态区仍可继续调度；
12. 创建失败不会触发其他执行机制兜底；
13. 不同 worktree 读取的不可变文档基线一致，但运行状态都通过统一工具写入同一个绝对 `SharedRuntimeStatePath`；
14. 并发触发时最多一个 Coordinator 获得写租约，两个生产者不会覆盖更高 `StateRevision`。

输出：

```text
ProtocolVersion:
EventType: CoordinatorCanaryResult
IterationID:
CanaryInvocationID:
ExecutionStatus: Completed / Blocked
ScenarioResults: <1-14 Pass/Fail + evidence>
DuplicateDispatchCount:
MissedEventCount:
IncorrectStateTransitionCount:
UnauthorizedFallbackCount:
RecoveryComparison:
ControlPlaneErrors:
Verdict: CanaryPassed / CanaryFailed / NotIssued
RequiredFixes:
```

只有 14 项全部通过、四个错误计数均为 0 且恢复比较一致时，才允许 `CanaryPassed`。
