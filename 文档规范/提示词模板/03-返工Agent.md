# 返工 Agent

> 规范版本：V2.5
> 文档状态：待审核
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（待审）

目标：在原任务 `<task-id>` 内关闭 `<review-id>` 的 Findings，并提交复核。

规则：

- 使用原任务和原授权范围；
- 使用原任务分支 `<task-branch>`，不得转到迭代分支或主分支返工；
- 每个 Finding 必须有修改或有证据的异议；
- 每个确认缺陷补回归测试；
- 使用最小完整修复，不顺带重构；
- 发现需改需求、设计或范围时停止；
- 修复后原 Review 结论仍失效，必须返回独立 Reviewer。
- 修复完成后在原任务分支创建新的 commit，并返回旧/新 HeadSHA；

输出：

| Finding | 处理结论 | 修改位置 | 回归测试 | 结果 |
|---|---|---|---|---|

```text
ProtocolVersion:
EventType: ReworkSubmission
IterationID:
TaskID:
InvocationID:
ReviewedTaskInvocationID:
ExecutionStatus: Completed / Blocked
Status: Submitted / Blocked
BlockerType: None / NeedsScopeChange / Environment / Authorization
FindingDisposition: Closed / DisputedWithEvidence / NeedsScopeChange
PreviousReviewedTarget:
TaskBranch:
PreviousHeadSHA:
NewHeadSHA:
NewReviewRange:
AffectedEvidence:
AdditionalRisks:
StateRecordID:
PublishedSignalRevision:
StatePublishStatus: Published / StatePublishFailed
```

`ExecutionStatus: Blocked` 时 `Status` 必须为 `Blocked`；新提交必须返回新的 `NewHeadSHA`，不得复用旧 Review 结论。输出 final 前必须先发布对应返工状态记录。

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | 模板沿用 V2.4 内容 |
| V2.5 | 2026-08-20 | WorkBuddy | 补统一文档头与修订记录；正文无实质改动 |
