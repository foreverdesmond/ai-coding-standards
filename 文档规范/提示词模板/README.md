# 子 Agent 提示词模板索引

> 模板版本：V2.5
> 模板状态：已审核通过（V2.5 定稿基线）
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-21
> 审核人：Richy（已审核）
> 上游规范：[07-子Agent任务委派与提示词规范](../07-子Agent任务委派与提示词规范.md)
> 调度协议：[09-Hermes调度与运行时台账规范](../09-Hermes调度与运行时台账规范.md)
> 修订记录：见文末

## 使用方法

每次派发使用：

```text
00-公共基座
  + 一个按需角色模板
  + 该角色所需基线
  + Context L2（仅触发时）
  + 当前 Invocation 与动态数据
```

尖括号字段必须替换。模板不能直接空白派发，也不能替代执行者自主探索。

所有异步任务必须先填充 `ProtocolVersion`、`IterationID`、`TaskID`、`InvocationID`、`ParentCoordinatorRef`（Hermes 身份）和 `ExpectedExecutionKind`。指定执行机制（WorkBuddy / Codex / Human）不得被未经授权替代，禁止伪独立自审；只有项目负责人事先批准 `ApprovedEquivalent` 时才允许等价执行机制。

## 模板

| 文件 | 角色 |
|---|---|
| [00-公共基座](./00-公共基座.md) | 所有角色共享约束（Hermes 注入协议） |
| [01-开发Agent](./01-开发Agent.md) | 首次实现 |
| [02-Coding-Reviewer](./02-Coding-Reviewer.md) | Level 0 独立代码审查 |
| [03-返工Agent](./03-返工Agent.md) | Review 问题关闭 |
| [04-测试验证Agent](./04-测试验证Agent.md) | 测试执行与证据分类 |
| [05-系统集成Reviewer](./05-系统集成Reviewer.md) | Level 1 整体审查 |
| [06-最终合并审核Agent](./06-最终合并审核Agent.md) | 冻结候选合并资格 |
| [07-总调度Agent](./07-总调度Agent.md) | Hermes 常驻派发、台账与事件/cron 对账 |
| [08-需求审核Agent](./08-需求审核Agent.md) | 需求范围、口径和分级验收审查 |
| [09-设计审核Agent](./09-设计审核Agent.md) | 设计覆盖、运行时风险和测试策略审查 |
| [10-开发任务审核Agent](./10-开发任务审核Agent.md) | 任务拆分、上下文、调度和门禁审查 |
| [11-迭代集成执行Agent](./11-迭代集成执行Agent.md) | 合并精确已审核任务提交到迭代开发分支 |
| [12-主分支合并执行Agent](./12-主分支合并执行Agent.md) | 在最终门禁和授权后执行主分支合并 |
| [13-总调度周期对账](./13-总调度周期对账.md) | Hermes cron 定时对账：台账消费与幂等健康检查 |
| [14-总调度冷恢复](./14-总调度冷恢复.md) | Hermes 重启恢复：从台账 + Git 重建最大安全状态 |
| [15-调度协议Canary](./15-调度协议Canary.md) | 在无业务副作用场景验收调度控制平面 |

角色模板数量不代表每个迭代必须启动同样数量的 Agent。角色已按 README §4 收敛为 6 种核心角色；下表模板为按需角色契约，具体角色归属见 `07` §2 的映射说明。

## 动态数据最低要求

- 仓库和工作区；
- ProtocolVersion、IterationID、TaskID、InvocationID、ParentCoordinatorRef、ExpectedExecutionKind 和模型；
- 来源与目标分支；
- TaskType、任务分支、worktree、CodeBaseSHA、HeadSHA、ReviewedCommitRange、ReviewedCommitSet 和 MergeTarget；
- RequirementsBaselineRef、DesignBaselineRef、TaskDocumentBaselineRef 和适用 ContextBaselineRef；
- 不可歧义基线或候选目标；
- 上游文档及版本；
- 上下文包路径与状态（适用时）；
- 当前任务、风险和权限；
- 测试、Findings 或 NotRun；
- 期望的结构化输出。
- 所有异步角色由 Hermes 注入 `LedgerLocation`、`CanonicalTaskDocumentPath`、`StateRecordID` 与派发时台账版本；执行 Agent 不直接写台账，通过其载体通道输出结构化协议头，由 Hermes 以事件驱动 + cron 兜底消费后幂等写入台账（无文件锁）。Coordinator/巡检还必须提供暂停状态、恢复策略和停止/通知条件。

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | Codex 线程调度基座（SharedRuntimeStatePath/StateUpdateToolPath/短时锁/Codex 子任务兜底禁令） |
| V2.5 | 2026-08-20 | WorkBuddy | 整体升 V2.5：文档头、调度协议引用改 09-Hermes、删 Codex 兜底禁令改"指定执行机制不可被未经授权替代"、共享 JSON 三件套改 Hermes 台账注入描述、模板表 07/13/14 描述同步 |
| V2.5 定稿 | 2026-08-20 | WorkBuddy | 评审通过，标记为 V2.5 正式基线 |
| V2.5 勘误 | 2026-08-21 | WorkBuddy | 模板表 13 行同步：文件名/链接改为 13-总调度周期对账，描述术语统一「对账」 |
