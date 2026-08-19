# V2.5 Hermes 总调度模式 · 流程与边界决议

> 状态：草稿（讨论留痕，供后续实施）
> 日期：2026-08-20
> 参与：Richy + Hermes（Tiffany），延续《大纲审阅报告》与《规范2.5升级计划》的评审讨论
> 说明：本文记录 Hermes 迁移 V2.5 过程中，Richy 与 Hermes 逐项讨论确认的**完整流程**与**全部边界决议**。它是评审报告 / 升级计划的落地细化，是后续《Hermes能力边界清单》与实施排程的事实基线。

---

## 一、总体流程（最终版）

```text
[Richy + GPT-5.6-Sol]：需求分析 → 边界划分 → 详细设计 → 拆分开发任务
        │              （任务清单中注明每个任务的 载体Agent 与 模型）
        ▼
[Hermes 审核开发任务：合理 / 可执行]──反馈──► [Richy + GPT-5.6-Sol 修改]
        ▼
Hermes：建台账(SQLite) → 按任务清单派发
        ▼
Implementer（载体按清单）：建 worktree → 开发 → L0 单测(落证据) → commit
        ▼
Reviewer（一对一，直接进该 worktree 只读审 commit）
        ▼  Approved / ChangesRequested(返工 → 只审修改部分)
Integrator：合并到 iteration → 统一清理 worktree
        ▼
Validator：L1 + 回归（自动化）
        ▼  [通过]
════════════ 真实环境门禁：Richy 授权 + Richy 亲自在真实测试环境验证 ════════════
        ▼  批准
Release → production/main
```

---

## 二、完整边界决议明细（讨论确认留痕）

### A. 能力载体（Hermes 能力边界）

| # | 决议项 | 结论 |
|---|---|---|
| A1 | Hermes 台账载体 | 由 Hermes 定（倾向 SQLite），后续《Hermes能力边界清单》由 Hermes 独立完成 |
| A2 | 事件源与巡检 | **收消息 + 定时轮询（约 1 分钟）双通道**；飞书推送可能丢(已实证 WB 回复但 Hermes 未收到)，需轮询兜底 |
| A3 | 部署形态 | **单机单实例**（需 Hermes 确认：一台服务器一套 Hermes 只产生单实例） |
| A4 | 恢复语义 | **热**：单次异常自动从最近台账快照续跑；**冷**：重启后全自动读台账+Git 重建，失败自动升级灾难级；**灾难**：台账丢失/损坏（或冷恢复失败），从 Git 任务文档快照重建，需 Richy 介入批准 |
| A5 | cron 精度 | 由 Hermes 确认最小精度与能否承载调度对账 |

### B. 角色与边界

| # | 决议项 | 结论 |
|---|---|---|
| B1 | Doc/Design Reviewer 边界 | 凡纯文档工作归其完成；(设计+工作包+文档审核)；**低风险设计不需审，高风险设计由同角色其他 agent 审** |
| B2 | L0 信任 | **信任开发者 L0 证据**（有后续集成/全量测试兜底），Reviewer 不重跑全套单测 |
| B3 | 触发集成 | **沿用 V2.4**：TaskAccepted → 创建独立集成任务 |
| B4 | 载体-角色映射 | **不建立固定映射**；载体 Agent 与模型在**设计阶段**由详细任务清单指定（价格多变，临时指定） |

### B5. 角色精简（本评审新增决议，落地于 README §4 / §6.1 / 提示词模板）

| 精简后角色 | 核心职责 | 关键边界 |
|---|---|---|
| Implementer（开发） | 首次开发 + 返工 + L0 单测 | 只在自己 feature 分支；跑 L0 并落证据；不批准自己 |
| Reviewer（审核） | 审核开发成果（diff/commit） | 基于 diff+开发者 L0 证据，只抽查关键路径不重跑全套；只读，不 merge |
| Integrator（集成） | 合并 feature→iteration | 只合并已 approved 的精确 commit；解决冲突；不自己审自己的 merge |
| Validator（测试验证） | 合并后全量测试（L1+回归） | 在集成分支跑全量；只读/测试环境；不 merge |
| Doc/Design Reviewer（文档/设计审核） | 一切文档审核+设计+工作包/上下文 | 纯文档工作归其；低风险设计不审、高风险同角色他审 |
| Coordinator（总调度） | 派发/巡检/判 gate | Hermes 承担，不做子 Agent 实操 |

合并来源：Reworker→Implementer；System Reviewer→Reviewer；Main Merge→Integrator；Requirements/Design/Task 三审核→合一 Doc/Design Reviewer。
保留底线：开发与 Review 分离；合并执行与审查分离；最终批准不问责开发者自己。

### C. 流程编排

| # | 决议项 | 结论 |
|---|---|---|
| C1 | 派发协议头设计 | Hermes 设计（原则：省 token、不引风险、语义明确） |
| C2 | 并行 | **允许多任务并行**；存在冲突的任务不设为并行；实施中真产生冲突 → 找 Richy 协调 |
| C3 | 证据落盘格式 | Hermes 设计（同 C1 原则） |
| C4 | 返工 Review | **沿用：只对修改部分 Review**（省 token） |
| C5 | worktree 创建 | **方案 Y + Hermes 统一编排**：Implementer 自建（danger-full-access 下实测可行），Hermes 派发时给出 TaskID/base分支/worktree目录；**不亲自执行 git worktree add** |
| C6 | worktree 清理 | **由最终合并代码的 Integrator 统一清理**（D2） |

### D. Git / 沙箱 / 安全

| # | 决议项 | 结论 |
|---|---|---|
| D1 | danger-full-access 使用 | 需要提交权限的角色（开发/集成/设计）均可使用 |
| D2 | worktree 生命周期 | 创建=Implementer（方案Y）；使用=Implementer；审核=Reviewer只读；清理=Integrator统一 |
| D3 | Reviewer 拿 snapshot 方式 | **D3b：Reviewer 直接进开发者 worktree 只读审核**（严格串行不重叠；开发读写/审核只读；一个开发 agent 只负责一个功能；避免跨 worktree 丢失不能进 git 的本地内容） |

### E. 需求/设计/任务拆分的责任

| # | 决议项 | 结论 |
|---|---|---|
| E1 | 需求分析/边界划分/详细设计/拆分任务 | **均由 Richy + GPT-5.6-Sol 完成**，Hermes 不主导 |
| E2 | Hermes 对开发任务的角色 | **审核**是否合理/可执行，反馈给 Richy，Richy+GPT 修改。Hermes 不拍板核心设计决策 |

### F. 台账与通知

| # | 决议项 | 结论 |
|---|---|---|
| F1 | 台账呈现 | **不新增开发看板**；Hermes 维护台账真源（SQLite/文件） |
| F2 | 台账通知 | **台账更新时推送到 TG，关键节点 + 0 token + 台账链接**：结构化字段读台账+模板拼装（不走 LLM），关键状态(Submitted/Approved/ChangesRequested/Integrated/需授权/异常失败)才推；细节凭链接自行查看 |
| F3 | 迭代推进/Release | **需 Richy 授权**；Level 1+回归通过不代表可上线；真实环境版本验收由 Richy 亲自在真实测试环境执行（符合 V2.4 Level2/3 由项目负责人运行+授权） |

---

## 三、上下文包（Context L2）责任与分阶段生成（本评审新增决议）

- **责任人**：Doc/Design Reviewer。
- **生成时机**：开发前（派发时刻），且**依赖已满足**。
- **规则**：
  1. 上游基线就绪后、开发派发前，生成 Context L2 骨架（目标/风险/已确凿事实/探索范围/修改边界/禁止项）。
  2. 依赖任务已完成（此刻依赖已满足）才把被依赖任务的已验证产出/证据纳入包内；**绝不预写依赖尚未产出的内容**，避免与实际代码矛盾。
  3. 依赖完成后增量补充仍由 Doc/Design Reviewer 负责（复用证据落盘）。
- **核心原则**：Context L2 在开发前生成（依赖已满足）；宁可留"待下游产出后补充"，也不写可能冲突的占位。

> 注意：该决议同时记录了 V2.4 06 规范未明确"工作包由谁生成"（模板仅 `GeneratedBy:` 字段名）这个缺口，此处补全。

---

## 四、测试职责分层（防 Token 浪费，本评审新增决议）

| 层级 | 责任角色 | 动作 |
|---|---|---|
| Level 0（单测/组件） | Implementer | 开发后跑一次，结果落证据目录 |
| Review | Reviewer | 不重跑全套单测；基于 diff + L0 证据审查，只对关键路径冒烟 |
| Level 1 + 回归 | Validator | 仅 merge 前跑全量一次 |

Hermes 派发注入测试边界（prompt 显式写）：
- 给 Implementer："实现后运行 Level 0 单测，测试结果作为证据提交，供 Reviewer 审阅。"
- 给 Reviewer："开发者已跑 Level 0 单测（见证据）。你只需审查 diff 逻辑 + 对关键路径冒烟。不要重跑全部单元测试。"

核心原则：测试一次由责任方跑，其他方基于证据审；集成/回归只在 merge 前跑（采纳 deepseek 大纲 G-3 证据落盘作为复用载体）。

---

*本文为 Hermes 迁移讨论留痕，具体执行由 Hermes 按后续《Hermes能力边界清单》与派发机制落地。*
