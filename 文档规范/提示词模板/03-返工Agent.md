# 返工 Agent

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
Status: Submitted / Blocked
FindingDisposition: Closed / DisputedWithEvidence / NeedsScopeChange
PreviousReviewedTarget:
TaskBranch:
PreviousHeadSHA:
NewHeadSHA:
NewReviewRange:
AffectedEvidence:
AdditionalRisks:
```
