# 项目功能开发与多 Agent 协作规范

<div align="center">

  **Multi-Agent Development & Collaboration Standards (V2.5) | 多 Agent 开发与协作规范 (V2.5)**

  A collection of guidelines, standards, and best practices for AI-assisted (vibe-coded) software development and multi-agent collaboration

  面向 AI 辅助（vibe coding）开发与多 Agent 协作的规范、标准与最佳实践集合

  [![X (Twitter)](https://img.shields.io/badge/X-@Richyisaflower-black?logo=x)](https://x.com/Richyisaflower)
  [![MIT License](https://img.shields.io/badge/License-MIT-green.svg)](https://choosealicense.com/licenses/mit/)
</div>

<p align="center">
  <a href="#简体中文">简体中文</a> · <a href="#english">English</a>
</p>

---

## English

### Overview

This repository defines a development and multi-agent collaboration standard for AI-assisted software projects. It turns loosely defined goals into auditable, implementable, verifiable, mergeable, and traceable software changes. The standard covers requirements, design, development tasks, independent review, layered verification (Level 0–3), and merge gates, and is not tied to any specific business, language, framework, or model.

### Contents

- Purpose & risk routes (lightweight / standard / high-risk)
- Core principles (facts-first, boundary-setting, independence of review, layered verification, safety-first)
- Standards directory & single responsibility (docs 01–08)
- Common roles, stage map, default branch role mapping
- Verification levels & status semantics
- Hermes-orchestrated multi-agent collaboration rules
- Evolutionary notes & V2.3 approved decisions, V2.5 pending decisions

Full Chinese specification: continue reading below (简体中文).

---

## 简体中文

# 项目功能开发与多 Agent 协作规范

> 规范版本：V2.5
> 规范状态：待审核（V2.3 仍为上一已审核基线）
> 基线说明：Hermes 总调度基线（总调度由 Hermes 常驻服务承担，WorkBuddy/Codex 平行协作执行）
> 适用范围：新功能、旧功能升级、缺陷修复、技术迁移、数据迁移、架构重构及其他软件开发任务
> 使用对象：项目负责人、分析与设计人员、总调度（Hermes）、开发者、子 Agent、代码审查者、测试与运维人员
> 作者：WorkBuddy（受 Hermes 总调度委派）
> 原定稿日期：2026-08-11
> 修订日期：2026-08-20
> 审核人：Richy（待审）

## 1. 目的

本规范用于把尚未完全明确的开发目标，逐步转化为可审核、可实现、可验证、可合并和可追溯的软件变更。

它不绑定特定业务、语言、框架、数据库、分支名称或模型。每个项目应在开发任务文档中填写本项目的实际技术栈、分支映射、环境能力、模型选择和人工授权边界。

高风险任务的完整链路为：

```text
任务背景与现状分析
  → 需求规格
  → 详细设计
  → 详细开发任务
  → 开发上下文包生成与校验
  → 调度控制平面初始化与 Canary
  → 独立任务分支开发、提交与独立 Review（Level 0）
  → 独立迭代集成任务合并已审核提交
  → 迭代整体集成验证与系统 Review（Level 1）
  → 集成分支人工/半自动系统验证（Level 2）
  → 真实外部环境版本验收（Level 3）
  → 最终合并审核
  → 独立主分支合并任务与后续发布
```

部署、上线和生产变更可以另行编写执行文档。本目录规定它们与开发完成、合并批准之间的状态边界，但不默认授权任何外部操作。

### 1.1 风险路线

本规范不要求所有任务走同样重的流程。项目应预先配置默认路线：

| 路线 | 适用场景 | 最低流程 |
|---|---|---|
| 轻量 | 局部、可逆、影响小、自动化充分 | 任务简报、定向验证、独立第二视角 Review |
| 标准 | 少量模块、风险可控、需要设计和集成 | 可合并的开发说明、Level 0、持续集成和适用 Level 1–3 |
| 高风险 | 跨模块、多 Agent、并发、关键数据、安全、迁移或外部不确定性 | 完整文档链、风险触发 Context L2、系统集成责任和 Level 0–3 |

具体选择和升级规则以 `04` 为准。路线简化不能豁免 SOLID、证据真实性、独立 Review 和外部操作安全。

## 2. 核心原则

### 2.1 先查事实，再形成方案

- 现状必须以当前代码、配置、数据、日志、运行入口和可复核证据为准。
- 历史文档、上下文包和 Agent 总结都是导航材料，不自动等同于当前事实。
- 事实、测量、推断、建议、已批准决定和未知项必须分开记录。
- 未经验证的假设不得被下游静默继承为事实。

### 2.2 上游定边界，下游做展开

- 背景文档回答“为什么做、当前实际如何工作、风险在哪里”。
- 需求文档回答“必须实现什么、什么不做、如何分级验收”。
- 详细设计回答“如何实现、运行时如何协作、失败时怎样处理”。
- 开发任务回答“如何拆分、谁负责交界处、怎样证明完成”。
- 上下文包是高风险或复杂委派的受控导航材料，不能创造新需求或替代代码探索；轻量任务不强制生成。
- 下游不得静默改变上游已经批准的行为、风险接受或范围。

### 2.3 局部正确不等于系统正确

- 单元测试通过不证明组件接线正确。
- Mock/Fake 集成测试通过不证明生产对象图、真实 Scope、并发资源和外部系统正确。
- 全部开发任务 `TaskAccepted` 不代表迭代已经完成系统集成验证。
- 自动化测试通过不代表真实环境已经验证。
- 真实环境验证完成不代表已经获得合并、部署或发布授权。

### 2.4 上下文是地图，不是领土

- Context L1 永久上下文和按风险触发的 Context L2 单任务上下文包用于降低大型任务的重复理解成本。
- 开发者仍必须读取 Context L3 当前代码，独立检查调用者、生产入口、间接依赖和测试替身边界。
- “允许修改范围”不等于“只允许阅读范围”；合理的只读调查范围应覆盖受影响的完整运行路径。
- 上下文与代码冲突时必须报告，不得挑选更方便的一方继续实现。

### 2.5 独立审查必须具有不同视角

- 开发者负责实现和自证，不能批准自己的任务。
- Coding Reviewer 必须以实际 diff 和精确提交为准，不以开发者总结为准。
- Reviewer 不应只机械重跑开发者测试，必须主动寻找设计遗漏、反例和测试证明缺口。
- 系统集成 Review 与单任务 Coding Review 是不同职责，前者重点检查任务交界、生产入口、并发和资源所有权。

### 2.6 验证分层且风险驱动

- Level 0 验证单任务局部正确性。
- Level 1 验证迭代内多个任务组合后的自动化行为。
- Level 2 在集成分支验证生产形态的系统运行路径，允许人工执行并发现问题后返工。
- Level 3 验证无法由本地自动化充分替代的真实外部环境和版本级业务结果。
- 不适用的测试层级必须写明理由和批准人，不得直接省略。

### 2.7 安全优先于测试便利

- 自动化测试不得默认获得创建、删除、清空或重建数据库、Schema、存储卷、外部资源和真实数据的权限。
- 破坏性测试操作必须针对精确目标获得项目负责人单次明确授权。
- 环境能力不足时应记录 `NotRun`、风险和后续责任人，不能通过扩大权限冒险补齐证据。
- 项目可根据数据可恢复性和风险偏好，批准在现有集成环境上人工验证，但必须记录范围和保护措施。
- 任何可能修改数据库或其他持久资源的验证，不得由开发、返工、Coding Review 或合并执行任务完成；必须转为独立验证任务，默认由项目负责人授权并人工执行。

### 2.8 设计一致性与 SOLID 是硬门禁

- 构建成功、测试通过和代码风格良好不能抵消需求或设计偏离。
- 新增或变化的设计必须接受职责、依赖方向、替换性、接口粒度和依赖倒置审查。
- 旧代码可以登记为既有债务，但不能自动豁免本次新增风险。
- 需要改变设计时必须先走变更流程。

### 2.9 开发、审核与合并职责分离

- 真实编码任务必须在当前迭代所属的独立任务分支中开发并提交；worktree 只是工作目录，不是审核身份。
- 普通开发者和返工执行者只能向自己的任务分支提交，不得直接提交或合并到迭代开发分支、长期集成分支、稳定分支或项目默认主分支。
- `TaskAccepted` 只表示指定任务提交通过 Level 0，不表示已经进入迭代候选。
- 已审核提交只能由独立迭代集成任务合并到迭代开发分支；合并后还必须形成 `Integrated` 和 `IntegrationVerified` 证据。
- 最终合并审核者只判断资格；只有独立主分支合并执行任务可以在候选、审核结论和项目负责人授权精确匹配时执行主分支合并。

### 2.10 文档状态驱动调度，事件驱动 + cron 定期对账消费

- Hermes 台账记录任务状态，是唯一任务状态发现入口；子任务通过 Hermes 台账接口幂等写入待消费信号，再输出结构化协议头。详细开发任务文档只保存任务定义和最近一次 Git 持久快照（`TASK-STATE-EXCHANGE` 块）。
- 总调度以事件驱动为主、cron 定期对账兜底：发现待消费记录后，才定向读取该记录绑定的执行载体内容，不得常态遍历全部任务。
- 周期对账负责发现和消费文档状态变化，同时检查停滞、冲突、上下文失效和真实阻塞。
- 单次编译失败、测试失败或未完成代码属于正常中间状态，不得在仍有活动时推断为 `Blocked` 或中断任务。

### 2.11 调度状态必须持久、可恢复并可审计

- 执行载体状态、开发任务状态和证据消费状态必须分别维护，不得用 UI 的 idle/completed 推断 Review 或任务结论。
- 调度状态持久于 Hermes 台账，可恢复、可审计；不再有共享 JSON 文件锁。任务状态、待消费信号和消费确认统一通过 Hermes 台账接口幂等写入。
- 开发任务文档中的状态块是 Git 管理的持久快照，不是活动状态真源；台账缺失、损坏或版本回退时按 `09-Hermes调度与运行时台账规范` 的恢复协议处理（热自动续跑 / 冷读台账 + Git 重建 / 灾难级需 Richy 介入批准）。
- 指定执行机制（WorkBuddy/Codex/Human）不得被未经授权替代，禁止伪独立自审。
- 高风险迭代启动前必须通过调度协议 Canary，真实开发任务不得兼作控制平面测试。

## 3. 规范目录与单一职责

| 文件 | 核心职责 |
|---|---|
| [01-任务背景与现状分析文档规范](./文档规范/01-任务背景与现状分析文档规范.md) | 建立当前事实、运行拓扑、证据和风险基线 |
| [02-需求规格文档规范](./文档规范/02-需求规格文档规范.md) | 定义范围、行为、验收层级和成功标准 |
| [03-详细设计文档规范](./文档规范/03-详细设计文档规范.md) | 设计组件、运行时所有权、失败语义和测试策略 |
| [04-详细开发任务文档规范](./文档规范/04-详细开发任务文档规范.md) | 拆分任务、依赖、文件所有权、交界责任和状态 |
| [05-文档审核与变更控制规范](./文档规范/05-文档审核与变更控制规范.md) | 管理批准、版本、变更传播、候选冻结和证据有效性 |
| [06-开发上下文包生成与使用规范](./文档规范/06-开发上下文包生成与使用规范.md) | 规定上下文包的生成、校验、探索、失效和对账 |
| [07-子Agent任务委派与提示词规范](./文档规范/07-子Agent任务委派与提示词规范.md) | 规定角色契约、提示词结构、权限和输出状态 |
| [08-开发验证、代码审查与合并门禁规范](./文档规范/08-开发验证、代码审查与合并门禁规范.md) | 规定 Level 0–3、Review 分工、分支门禁和合并资格 |
| [09-Hermes调度与运行时台账规范](./文档规范/09-Hermes调度与运行时台账规范.md) | 规定 Hermes 台账状态面、定向消费、事件/cron 对账、暂停、恢复和 Canary |

角色提示词模板位于 [`提示词模板`](./文档规范/提示词模板/README.md)。模板是委派起点，不得替代任务实际基线和自主探索。

## 4. 通用角色

规范定义角色能力，不绑定具体模型或供应商。V2.5 起收敛为 6 种核心 Agent 角色（Hermes 总调度 + 5 种执行/审核角色）：

| 角色 | 核心职责 | 关键边界 |
|---|---|---|
| Coordinator（总调度） | 派发、巡检、判 gate | 由 Hermes 常驻服务承担；不做子 Agent 实操，不自行批准代码 |
| Implementer（开发） | 首次开发 + 返工 + Level 0 单元测试 | 只在自己 feature 分支；跑 L0 并落证据；不批准自己 |
| Reviewer（审核） | 审核开发成果（diff/commit） | 基于 diff + 开发者 L0 证据审，只抽查关键路径，不重跑全套单测；只读，不 merge |
| Integrator（集成） | 合并代码：feature → iteration | 只合并已 Review 通过的精确 commit；解决冲突；不自己审自己的 merge |
| Validator（测试验证） | 合并后的全量测试与验证（集成 L1 + 回归） | 在集成分支跑全量；只读/测试环境；不 merge |
| Doc/Design Reviewer（文档/设计审核） | 一切文档审核 + 设计 + 工作包/上下文设计 | 上游文档、需求、设计、任务包、Context 的设计与审核 |

**合并来源**（V2.4 → V2.5 角色收敛）：Reworker → Implementer（返工是开发的续集）；System Reviewer → Reviewer（按 Level 区分范围）；Main Merge → Integrator（同为合并执行，仅目标分支不同）；Requirements / Design / Task 三个审核 → 合一 Doc/Design Reviewer。

**项目负责人（人类）**：不参与上述 Agent 角色，是最终业务决策与授权者——批准业务决定、风险例外、外部操作、合并与发布，并在真实环境门禁亲自验证。任何 Agent 不得自证批准。

**保留底线（不可妥协）**：开发与 Review 分离；合并执行与审查分离；最终批准不问责开发者自己。每个任务按需启用 2~N 个角色（轻量任务只需 Implementer + Reviewer）；低风险任务可兼任部分角色，但开发与最终批准不得由同一执行主体自证完成。

## 5. 阶段地图

| 阶段 | 进入下一阶段的核心结论 | 权威规范 |
|---|---|---|
| 背景 | 当前事实、运行拓扑、风险和未知项足以支持需求 | `01` |
| 需求 | 范围、口径、Level 0–3 和 NotRun 可验收 | `02` |
| 设计 | 需求落点、运行时所有权、失败语义、SOLID 和测试策略明确 | `03` |
| 任务 | 路线、任务边界、系统集成责任和 Context 策略可执行 | `04` |
| Context/委派 | 适用的 Context L2、角色提示词和不可变基线有效 | `06`、`07` |
| 调度控制面 | 文档状态生产/消费、锁、租约、恢复和 Canary 可执行 | `09` |
| Level 0–3 | 对应验证状态和证据满足项目门禁 | `08` |
| 变更/合并 | 冻结候选、批准和证据仍然有效 | `05`、`08` |

各阶段只以权威规范中的本地审核清单为准，本 README 不复制完整门禁。

## 6. 默认分支角色映射

下表是推荐默认值，不是唯一分支模型。项目可以使用 trunk-based、发布分支、功能开关或其他策略，但必须保留等价的状态、证据和授权门禁。

| 逻辑角色 | 项目实际分支 | 默认进入条件 |
|---|---|---|
| 任务分支 | `<task-branch>` | 单任务开发、返工和本地提交 |
| 当前迭代开发分支 | `<iteration-branch>` | 独立集成任务合并 `TaskAccepted` 的精确提交 |
| 长期集成分支 | `<integration-branch>` | Level 1 通过 |
| 稳定/主分支 | `<stable-or-main-branch>` | Level 3、最终合并审核和负责人授权通过后，由独立合并任务执行 |

项目可以调整或合并分支角色，但真实编码任务默认不得直接在共享迭代分支或主分支开发。确需采用 trunk-based 等策略时，必须由项目负责人事先批准，并记录等价的候选隔离、独立 Review、回退和冲突控制方式。

### 6.1 Git 权限矩阵

| 角色 | 任务分支 commit | 迭代分支 commit/merge | 主分支 merge |
|---|---:|---:|---:|
| Implementer（开发） | 仅自己的 feature 分支 | 禁止 | 禁止 |
| Reviewer（审核） | 禁止 | 禁止 | 禁止 |
| Integrator（集成） | 禁止改写任务实现 | 允许合并精确已审核提交 | 仅在最终审核 + 负责人授权后允许 |
| Validator（测试验证） | 禁止 | 禁止 | 禁止 |
| Doc/Design Reviewer | 禁止 | 禁止 | 禁止 |
| Coordinator（Hermes） | 禁止 | 禁止 | 禁止 |

说明：Coordinator（Hermes）不代执行任何 git 操作，只派发与判 gate。需要提交权限的角色（Implementer / Integrator）使用 `danger-full-access` 沙箱；Reviewer / Validator / Doc-Design Reviewer 为只读沙箱。worktree 由 Implementer 自建、Integrator 统一清理。

## 7. 验证等级概览

| 等级 | 目标 | 典型执行者 | 默认分支动作 |
|---|---|---|---|
| Level 0 | 单任务编译、单元测试、定向组件测试和 Coding Review | 开发者 + 独立 Reviewer | 形成 `TaskAccepted` 的精确任务提交 |
| Level 1 | 全任务组合、组件/Fake 集成、全量自动化和系统 Review | 系统集成 Reviewer | 合入长期集成分支 |
| Level 2 | 生产形态入口、小范围系统冒烟、现有集成环境真实路径 | 项目负责人运行 + Agent 辅助 | 保持在长期集成分支并返工闭环 |
| Level 3 | 真实外部环境和版本级验收 | 项目负责人授权，Agent 辅助 | 申请合入稳定分支 |

各等级的详细定义、`NotRun`、测试重复控制和合并规则见 `08`。

## 8. 文档与代码状态

### 8.1 文档状态

统一使用：`草稿`、`待审核`、`修改中`、`已审核通过`、`待同步`、`已取代`、`已归档`。

### 8.2 任务与版本状态

统一使用：

- `Pending`、`Ready`、`InProgress`、`Submitted`、`InReview`、`ChangesRequested`、`TaskAccepted`、`Blocked`、`Cancelled`；
- `MergePending`、`Integrated`、`IntegrationVerified`；
- `ComponentVerified`、`SystemVerified`、`ExternalVerified`；
- `MergeBlocked`、`MergeApproved`、`Merged`；
- `ReleaseReady`。

状态必须绑定证据，并写入开发任务文档的状态交换区；不得只根据聊天消息、UI 标签或 Agent 是否停止来推断。

### 8.3 线程与证据状态

- 执行载体状态使用：`NotCreated`、`Provisioning`、`Running`、`Idle`、`NeedsAttention`、`Completed`、`Unavailable`、`Cancelled`；
- 证据状态使用：`Unread`、`Received`、`PendingVerification`、`Validated`、`Rejected`、`Superseded`、`Consumed`；
- `Idle`/`Completed` 不等于没有 final，`Received` 不等于已经验证，代码存在不等于流程门禁通过；
- 详细语义和转换规则以 `09` 为准。

## 9. 文件组织

```text
<项目配置的架构资料位置>/          # Context L1，可复用现有 AGENTS、ADR 或 docs
设计文档/<功能名称>/
├── 01-任务背景与现状分析.md
├── 02-需求规格.md
├── 03-详细设计.md
├── 04-详细开发任务.md
├── 任务上下文包/                  # 按风险需要时使用 Context L2
├── 验证记录/                      # Level 0–3 证据
├── 05-上线执行清单.md             # 按需
└── README.md
```

小型、低风险任务可以经项目负责人批准合并部分文档，但不得省略事实调查、验收层级、设计风险、独立 Review 和合并资格判断。

## 10. Hermes 总调度与多 Agent 协作规则

1. 先完整读取适用规范和已审核上游文档。
2. 任务委派使用 `07` 的角色契约和对应模板；动态数据由 Hermes 注入。
3. 上下文包必须按 `06` 生成，由 Doc/Design Reviewer 在依赖满足后生成，并显式暴露未知项和自主探索要求。
4. 一个任务同一阶段只能有一个有效开发实例，除非明确拆分为互不冲突的子任务。
5. 真实编码任务必须在独立任务分支提交；Reviewer 必须审查准确的 `CodeBaseSHA..HeadSHA`，不能只审查开发者报告、worktree 或当前主工作区。
6. Review 失败后返回原任务返工，再由原 Reviewer 或批准的替代 Reviewer 复核。
7. `TaskAccepted` 后由 Integrator 合并准确的已审核提交；需要使用上游实现的下游任务默认依赖 `Integrated`。
8. 派发前由 Hermes 初始化台账记录、Invocation、幂等 DispatchKey、不可变文档基线；高风险迭代先通过 Canary，除非项目负责人对既有迭代明确记录临时豁免。
9. 派发后子任务通过 Hermes 台账接口幂等写入完成/求助状态；Hermes 以事件驱动 + cron 兜底消费待处理记录，只对命中的记录定向读取，不重复派发。
10. 任务创建返回临时请求标识时登记 `Provisioning` 并继续绑定正式任务，不得判定为失败；指定执行机制（WorkBuddy/Codex/Human）不得被未经授权替代。
11. 台账缺失、损坏或版本回退时按 `09-Hermes调度与运行时台账规范` 的恢复协议处理（热自动续跑 / 冷读台账 + Git 重建 / 灾难级需 Richy 介入），仅对明确缺失证据的记录定向读取任务历史。
12. 任务仍有活动时，不得因中间编译或测试失败推断阻塞并中断。
13. Review 必须使用独立执行实例，不得发回开发任务要求其自审；禁止伪独立自审。
14. 未运行测试必须如实记录，不能以推断或其他测试替代。
15. 未经授权不得连接、修改或删除真实外部资源。
16. 文档、执行载体、任务、证据、代码、验证、合并和发布状态分别维护。
17. 需要提交权限的角色（Implementer / Integrator）使用 `danger-full-access` 沙箱；Hermes 单实例运行，以 DispatchKey 幂等去重替代协调租约锁。

## 11. 规范自身的演进

- 现实运行中发现的稳定流程缺陷，应形成复盘并反向检查本目录。
- 新规则必须说明解决的通用风险，不能只针对单次事故写死实现细节。
- 规范变更同样需要审核；本目录状态为“待审核”时，不得宣称新版本已经成为正式基线。
- 提示词模板经过多次项目验证后，可以进一步封装为 Skill，但 Skill 不能取代仓库内已审核规范。

## 12. V2.3 已审核决定

- [x] DOC-DEC-001：采用 Level 0–3 验证等级及对应状态语义。
- [x] DOC-DEC-002：Level 0–3 是通用状态门禁；迭代、集成和稳定分支映射作为推荐默认策略，由项目配置覆盖。
- [x] DOC-DEC-003：Level 2 默认允许项目负责人在项目指定的集成候选上人工运行，发现问题后按返工闭环处理。
- [x] DOC-DEC-004：Level 3 和最终合并审核通过后，按项目策略进入稳定候选；默认申请稳定分支合并。
- [x] DOC-DEC-005：默认禁止 Agent 自动创建、删除、清空或重建数据库及其他持久资源；例外需精确目标的单次授权。
- [x] DOC-DEC-006：上下文包和子 Agent 角色提示词分别形成独立规范。
- [x] DOC-DEC-007：规范只定义角色能力和分支逻辑角色，不绑定具体模型、供应商或项目分支名称。
- [x] DOC-DEC-008：采用轻量、标准、高风险三条路线；Context L2 仅按风险触发。
- [x] DOC-DEC-009：Context L1 只在稳定事实实质变化时更新；`NoChange` 不建立独立任务或 Review。
- [x] DOC-DEC-010：SOLID 保持所有路线的硬门禁，证据展开程度可以随风险调整。
- [x] DOC-DEC-011：真实编码任务默认使用独立任务分支和不可变提交作为 Review 候选；普通开发者只可向自己的任务分支提交。
- [x] DOC-DEC-012：迭代集成与主分支合并分别由独立执行任务完成，Reviewer 只审核资格，不执行合并。
- [x] DOC-DEC-013：任务完成采用开发任务文档状态驱动；周期巡检消费文档信号并按需定向取证。

## 13. V2.5 待审核决定

- [ ] DOC-DEC-021：以 Hermes 台账替代共享本地 JSON 作为任务运行状态唯一真源；开发任务文档 `TASK-STATE-EXCHANGE` 块仍为 Git 持久快照。（取代 V2.4 DOC-DEC-014 / 019）
- [ ] DOC-DEC-022：Hermes 单实例运行，以 `DispatchKey = IterationID+TaskID+Stage+TargetIdentity` 幂等去重替代协调租约锁。（取代 V2.4 DOC-DEC-016）
- [ ] DOC-DEC-023：调度采用事件驱动 + cron 定期对账兜底，替代共享 JSON 的 `StateRevision` 周期巡检。（取代 V2.4 DOC-DEC-016 / 019）
- [ ] DOC-DEC-024：`ExpectedExecutionKind` 统一为四档枚举 `WorkBuddy / Codex / Human / ApprovedEquivalent`；指定执行机制不得被未经授权替代，禁止伪独立自审。（取代 V2.4 DOC-DEC-017）
- [ ] DOC-DEC-025：通用角色收敛为 6 种（Coordinator / Implementer / Reviewer / Integrator / Validator / Doc-Design Reviewer），保留开发与 Review 分离、合并执行与审查分离底线。

---

## 修订记录

| 版本 | 日期 | 作者 | 变更说明 |
|---|---|---|---|
| V2.3 | 2026-08-11 | — | V2.3 已审核通过基线 |
| V2.4 | 2026-08-15 | — | 引入共享 JSON 状态交换区（Codex 线程调度）、角色 10 种、Codex 协作规则 |
| V2.5 | 2026-08-20 | WorkBuddy | 标题/版本升 V2.5（Hermes 总调度基线）；§2.10/§2.11 去共享 JSON；§3 目录表 09 改名；§4 角色收敛 6 种；§6.1 Git 权限矩阵按 6 角色更新；§10 重写为 Hermes 协作规则；§13 改写为 V2.5 待审核决定（DOC-DEC-021~025） |
