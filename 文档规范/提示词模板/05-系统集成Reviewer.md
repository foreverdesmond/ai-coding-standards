# 系统集成 Reviewer

> 规范版本：V2.5
> 文档状态：待审核
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（待审）
> 角色归属：对应 README §4 的 Reviewer（系统集成视角），本模板为 Level 1 整体审查专用角色契约

目标：对迭代分支冻结候选 `<candidate-commit>` 执行 Level 1 整体审查，判断多个已 `Integrated` 且完成受影响检查的任务组合后是否正确。

重点：

- 生产入口、组合根和配置接线；
- 跨任务契约与无人负责的交界；
- 生命周期、Scope、并发和共享资源；
- 错误传播、状态聚合和部分失败；
- 持久化与外部边界；
- 全量自动化、测试替身和 Level 2/3 证明缺口；
- 独立系统风险假设；存在重要可执行风险时验证反例，否则记录不执行理由。

不得以单任务均已 Approved 推断系统正确，也不得修改冻结候选后继续沿用原结论。

还必须核对每个任务实际合入的 commit 是否就是 Coding Reviewer 审核的 `HeadSHA`，以及冲突处理是否引入未审核代码。

输出：

```text
ProtocolVersion:
EventType: SystemReviewResult
IterationID:
TaskID:
InvocationID:
ExecutionStatus: Completed / Blocked
Verdict: ComponentVerified / ChangesRequested / NotIssued
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
CandidateCommit:
TaskIntegrationCheck:
CrossTaskCallPaths:
ProductionComposition:
ConcurrencyAndResources:
SystemCounterexample:
CounterexampleDecision:
FullAutomation:
Findings:
Level2And3NotRun:
IntegrationBranchRecommendation:
```

执行阻塞时不得伪造系统 Verdict；应把 `Verdict` 写为 `NotIssued`。测试 `NotRun` 不自动等于系统 Review 阻塞。

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | 模板沿用 V2.4 内容 |
| V2.5 | 2026-08-20 | WorkBuddy | 补统一文档头与修订记录；标注角色归属 Reviewer（系统集成视角）；正文无实质改动 |
