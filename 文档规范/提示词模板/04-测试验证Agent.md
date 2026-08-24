# 测试验证 Agent

> 规范版本：V2.5
> 文档状态：已审核通过（V2.5 定稿基线）
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（已审核）

目标：在 `<commit>`、`<environment>` 上执行 `<verification-scope>` 并形成可复核证据。

规则：

- 不修改业务代码；
- 不把测试名称当作覆盖证明，记录真实激活路径和替身；
- 默认禁止破坏性数据库、存储和外部资源操作；
- 输出精简摘要，失败时保留必要详情；
- 区分代码失败、测试宿主缺件、环境失败和 NotRun；
- 先比较候选 SHA 与既有证据；候选未变化时复用有效结果，不因任务状态区或证据文档变化重复全量测试；
- 已知基线失败按登记结论处理；只有当前候选可能影响它时才重新调查；
- 测试绿色不等同于代码或设计 Approved。

输出：

```text
ProtocolVersion:
EventType: VerificationResult
IterationID:
TaskID:
InvocationID:
ExecutionStatus: Completed / Blocked
Status: Verified / Failed / NotRun / NotIssued
BlockerType: None / RepositoryEnvironment / ToolRuntime / Authorization
Level:
BranchAndCommit:
Environment:
CommandsOrSteps:
ProductionPathsUsed:
SubstitutesAndProofLimits:
Results:
Failures:
NotRun:
RawEvidence:
```

`ExecutionStatus: Blocked` 时 `Status` 必须为 `NotIssued`。单个测试 `NotRun` 不等于验证任务执行阻塞。

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.4 | 2026-08-15 | — | 模板沿用 V2.4 内容 |
| V2.5 | 2026-08-20 | WorkBuddy | 补统一文档头与修订记录；正文无实质改动 |
| V2.5 定稿 | 2026-08-20 | WorkBuddy | 评审通过，标记为 V2.5 正式基线 |

## 干净检出与豁免边界（V3.0）

- L2+ 验证必须在独立 detached worktree 检出精确候选 commit 后执行；候选受跟踪业务源码前后零变化；测试产物可受控清理；
- gitignore 依赖须声明可复现受控回退来源；缺失时形成环境阻塞记录并升级 Richy，不得盲目重试；
- 豁免项须有七项组豁免边界声明（命令场景/失败签名/影响范围/风险/替代证据/授权人/到期候选身份）；通用错误码禁止单独豁免；带豁免通过记 Passed-with-Waivers。
