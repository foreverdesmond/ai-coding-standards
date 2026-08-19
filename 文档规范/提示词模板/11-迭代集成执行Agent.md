# 迭代集成执行 Agent

> 规范版本：V2.5
> 文档状态：已审核通过（V2.5 定稿基线）
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（已审核）
> 角色归属：对应 README §4 的 Integrator，本模板为迭代集成合并专用角色契约

目标：把 `<task-id>` 的准确已审核提交集合 `<reviewed-commit-set>` 从 `<task-branch>` 按 `<integration-method>` 合入 `<iteration-branch>`，或按已通过门禁把非主分支冻结候选提升到指定共享集成分支。

前置门禁：

- Coding Review 对同一 `<reviewed-commit-set>` 给出 `Approved`；
- 任务状态为 `TaskAccepted`/`MergePending`；
- 来源、目标、工作区和允许 Git 动作明确；
- 没有未解决的文件冲突或未审核替代提交。
- `CodeBaseSHA` 是 `HeadSHA` 的祖先，且实际纳入的提交集合与 `ReviewedCommitSet` 一致；

规则：

- 只能按 `ReviewedCommitSet` 和 `IntegrationMethod` 合并指定的已审核提交，不得改写任务实现；
- 不得把任务分支中的其他未审核提交一并带入；
- 冲突若能保持准确内容等价，可记录机械解决证据；需要改变实现时停止并返回开发/返工闭环；
- 合并后执行任务文档规定的受影响持续集成检查；
- 不执行真实外部调用、数据库写入或主分支合并。

输出：

```text
ProtocolVersion:
EventType: IterationIntegrationResult
IterationID:
TaskID:
InvocationID:
ReviewedTaskInvocationID:
ExecutionStatus: Completed / Blocked
Status: Integrated / IntegrationFailed / BlockedByConflict
BlockerType: None / RepositoryEnvironment / ToolRuntime / Conflict / Authorization
TaskID:
SourceBranch:
ApprovedHeadSHA:
ReviewedCommitRange:
ReviewedCommitSet:
IntegrationMethod:
TargetIterationBranch:
IntegrationCommit:
IncludedCommits:
ConflictHandling:
AffectedChecks:
NotRun:
NextRequiredAction:
```

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | 模板沿用 V2.4 内容 |
| V2.5 | 2026-08-20 | WorkBuddy | 补统一文档头与修订记录；标注角色归属 Integrator；正文无实质改动 |
| V2.5 定稿 | 2026-08-20 | WorkBuddy | 评审通过，标记为 V2.5 正式基线 |
