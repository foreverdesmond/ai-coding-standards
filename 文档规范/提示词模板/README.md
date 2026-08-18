# 子 Agent 提示词模板索引

> 模板版本：V2.4
> 模板状态：待审核（V2.3 仍为上一已审核基线）
> 上游规范：[07-子Agent任务委派与提示词规范](../07-子Agent任务委派与提示词规范.md)
> 调度协议：[09-调度控制平面与运行时台账规范](../09-调度控制平面与运行时台账规范.md)
> 更新日期：2026-08-15

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

所有异步任务必须先填充 `ProtocolVersion`、`IterationID`、`TaskID`、`InvocationID`、父调度身份和执行机制。明确要求 Codex 子任务时，不得使用子 Agent 或当前任务自行执行作为创建失败的兜底。

## 模板

| 文件 | 角色 |
|---|---|
| [00-公共基座](./00-公共基座.md) | 所有角色共享约束 |
| [01-开发Agent](./01-开发Agent.md) | 首次实现 |
| [02-Coding-Reviewer](./02-Coding-Reviewer.md) | Level 0 独立代码审查 |
| [03-返工Agent](./03-返工Agent.md) | Review 问题关闭 |
| [04-测试验证Agent](./04-测试验证Agent.md) | 测试执行与证据分类 |
| [05-系统集成Reviewer](./05-系统集成Reviewer.md) | Level 1 整体审查 |
| [06-最终合并审核Agent](./06-最终合并审核Agent.md) | 冻结候选合并资格 |
| [07-总调度Agent](./07-总调度Agent.md) | 派发、巡检和状态真源 |
| [08-需求审核Agent](./08-需求审核Agent.md) | 需求范围、口径和分级验收审查 |
| [09-设计审核Agent](./09-设计审核Agent.md) | 设计覆盖、运行时风险和测试策略审查 |
| [10-开发任务审核Agent](./10-开发任务审核Agent.md) | 任务拆分、上下文、调度和门禁审查 |
| [11-迭代集成执行Agent](./11-迭代集成执行Agent.md) | 合并精确已审核任务提交到迭代开发分支 |
| [12-主分支合并执行Agent](./12-主分支合并执行Agent.md) | 在最终门禁和授权后执行主分支合并 |
| [13-总调度周期巡检](./13-总调度周期巡检.md) | 按开发任务文档版本消费状态并执行幂等健康巡检 |
| [14-总调度冷恢复](./14-总调度冷恢复.md) | 状态区损坏时从最后有效文档状态和 Git 重建最大安全状态 |
| [15-调度协议Canary](./15-调度协议Canary.md) | 在无业务副作用场景验收调度控制平面 |

## 动态数据最低要求

- 仓库和工作区；
- ProtocolVersion、IterationID、TaskID、InvocationID、父调度身份、预期执行机制和模型；
- 来源与目标分支；
- TaskType、任务分支、worktree、CodeBaseSHA、HeadSHA、ReviewedCommitRange、ReviewedCommitSet 和 MergeTarget；
- RequirementsBaselineRef、DesignBaselineRef、TaskDocumentBaselineRef 和适用 ContextBaselineRef；
- 不可歧义基线或候选目标；
- 上游文档及版本；
- 上下文包路径与状态（适用时）；
- 当前任务、风险和权限；
- 测试、Findings 或 NotRun；
- 期望的结构化输出。
- 所有异步角色都必须提供 `CanonicalTaskDocumentPath`、`SharedRuntimeStatePath`、`StateUpdateToolPath`、`StateRecordID`、派发时状态区版本和短时锁；Coordinator/巡检还必须提供暂停状态、恢复策略和停止/通知条件。
