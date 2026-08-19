# Hermes 能力边界清单

> 规范版本：**V2.5（草案）**
> 文档状态：**已审核通过（V2.5 定稿基线）**
> 作者：Hermes（Tiffany）
> 创建日期：2026-08-20
> 最后更新：2026-08-20
> 审核人：Richy（已审核）
> 修订记录：见文末
> 定位：V2.5 升级计划中的 **P0 前提**。三份大纲、审阅报告、上线计划都假设 Hermes 具备某些能力，本清单把 **Hermes 实际能力** 实测确认并写死，作为 00-公共基座重写、09-台账规范重写、派发机制实现的依赖底座。

---

## 1. Hermes 运行形态

| 项 | 确认结果 |
|---|---|
| 部署形态 | **单机单实例**（一台服务器跑一套 Hermes，仅产生一个调度器实例） |
| 推论 | 无多 Coordinator 竞争 → **无需协调租约锁**；幂等由 `DispatchKey` 去重保证 |
| 常驻性 | **常驻服务**（非临时线程），持续运行，天然是"总调度" |

## 2. 状态持久化 / 台账载体

| 项 | 确认结果 |
|---|---|
| 自身存储 | Hermes 框架用 **SQLite 会话库** 保存会话/记忆（框架级，非调度台账） |
| 项目台账载体 | **本地 JSON 状态文件**（项目目录内、**不进 Git**）+ 可选 SQLite；由 Hermes 维护 |
| 台账内容 | 任务/阶段/状态/证据指针/依赖图/调度消费水位(ConsumedRevision) |
| 不可变快照 | 开发任务文档的 `TASK-STATE-EXCHANGE` 块仍作为 **Git 持久快照**（跨重启/灾难恢复用） |
| 持久可恢复性 | 台账持久化，Hermes 重启可读台账 + Git 重建 |

> **重要区分**：**调度台账 ≠ Hermes 会话库**。Hermes 会话库是框架用来保存聊天/记忆的 SQLite；调度台账是 Hermes 为开发迭代专门维护的本地 JSON 状态文件（调度专用，含任务/阶段/证据/依赖），两者是不同存储，不可混用。

## 3. 事件源与巡检能力

| 事件/通道 | 覆盖场景 | 说明 |
|---|---|---|
| ① 飞书长连接 (lark-ws) | 收 **WorkBuddy** 回复消息 | 被动，可能丢消息（已实证：WB 回复但 Hermes 未收到）→ 需兜底 |
| ② Codex 网关轮询 `GET /v1/threads/:id` | 查 **Codex** 线程进度 | 主动轮询 status：inProgress→completed/failed |
| ③ Hermes cron 定时轮询 | **兜底对账** | 频率 ~1 分钟；补事件丢漏 |
| ④ TG 推送 | 通知 Richy 关键节点 | **0 token**（读台账结构化字段+模板拼装，不走 LLM） |

**巡检结论**：采用 **事件(①②) + cron 定时轮询(③) 双通道**；事件丢失不影响正确性（cron 兜底对账）。

## 4. 定时调度（cron）能力

| 项 | 确认结果 |
|---|---|
| 支持语法 | `30m` / `every 2h` / 标准 5 段 cron `0 9 * * *` / ISO 一次性时间戳 |
| 最小精度 | **分钟级**（5 段 cron，无秒级） |
| 能否承载调度对账 | **能**。cron 每 1 分钟对账一次，处理待消费记录、健康检查，符合"事件+cron 兜底" |

## 5. Agent 派发能力（Hermes → 各载体）

| 载体 | 派发方式 | 是否同步 | 巡检方式 | Git 权限 |
|---|---|---|---|---|
| **WorkBuddy** | 飞书 post 消息 @WB | 异步（单向投递） | 被动等 WB 回 interactive 消息 + cron 兜底 | ✅ 默认可提交 |
| **Codex** | `POST /v1/threads`（网关） | 可同步(wait) / 异步 | 主动轮询 `/threads/:id` | ⚠️ 需 `sandbox:danger-full-access` |

**派发参数**（Codex 网关）：
```
POST http://10.192.241.5:4501/v1/threads
body: { prompt, cwd, model, modelProvider, sandbox, approval }
- modelProvider: openai(原生模型) / opencodex(CodexSplit第三方)
- sandbox: read-only / workspace-write(写文件) / danger-full-access(需要git时)
```

## 6. 工具权限分级（指派给 Agent 的沙箱策略）

| 任务类型 | sandbox | 说明 |
|---|---|---|
| 只读调研/Review | read-only | Reviewer、审阅 |
| 普通写文件 | workspace-write | 写文档/证据 |
| **需要 git 提交**（开发/集成/设计） | **danger-full-access** | `.git` 在 workspace-write 下是 protected path，会拦 git |

**实测依据**（2026-08-19 已验）：
- workspace-write 下 `git add` 报 `index.lock: Operation not permitted`（.git 被 sandbox 拦）
- danger-full-access 下 `git status/worktree add/add/commit/push/reset` **全部成功**
- 自定义 Permission Profile 在 managed 环境被宿主 requirements 钳制，不可靠

## 7. 重启恢复语义（A4 三级）

| 级别 | 触发 | Hermes 动作 | 失败处理 |
|---|---|---|---|
| 热恢复 | 单次工具/调用偶发异常 | 记日志，从最近台账快照自动续跑；重试 | 仍不行，等 cron 对账兜底 |
| 冷恢复 | 进程崩溃/重启 | **全自动**读台账+Git 重建`ConsumedRevision`→RecoveryOnly 校验→**自动校验通过即完成（无需 Richy 介入）** | ❌ **失败自动升级灾难级**（灾难级才需 Richy） |
| 灾难恢复 | 台账丢失/损坏（或冷恢复失败） | 从 Git 任务文档快照重建，确定可证明状态，其余降级待核实 | **需 Richy 介入批准** |

## 8. 通知与呈现

| 项 | 确认结果 |
|---|---|
| 台账看板 | **不新增开发看板**；Hermes 维护台账真源 |
| 推送 | TG 推送 **关键节点 + 0 token + 台账链接**：Submitted/Approved/ChangesRequested/Integrated/需授权/异常失败 |
| 链接 | 台账条目路径，Richy 按需点开查看，不主动灌全量 |

## 9. 已知限制（诚实披露）

1. **飞书推送可能丢**（已实证）：必须靠 cron 轮询兜底，不能单依赖事件。
2. **Codex 部分模型输出偶发异常**：deepseek-flash 短任务偶尔提前结束/流中断返回空；pro 更稳；网关已按最后一条 agentMessage 提取最终答案。
3. **GPT 原生模型走 Codex 会话流量**，额度用尽报 usageLimitExceeded（约每月 20 日重置）。
4. **Hermes 不代执行 git**：worktree 创建/commit/merge 由相应角色 Agent 用 danger-full-access 自办，Hermes 只派发参数、巡检、判 gate。
5. **多任务并行需约束**：存在冲突的任务不设并行；真冲突需 Richy 协调（见《Hermes流程与边界决议》C2）。

---

*本清单为 Hermes 能力实测确认，是 00-公共基座、09-台账规范重写及派发实现的依赖底座。未尽之处在实施过程中补充。*

---

## 修订记录

| 版本 | 日期 | 修订人 | 说明 |
|---|---|---|---|
| V2.5（草稿） | 2026-08-20 | Hermes | 首次发布：确认 Hermes 运行形态/台账/事件/cron/派发/权限/恢复能力；补齐文档版本号与审核状态元信息（Richy 反馈） |
| V2.5（草稿） | 2026-08-20 | Hermes | 审阅修订：修笔误'原生存章'→'原生模型'；§7冷恢复同步09(自动完成无需Richy)；§2明确区分调度台账与Hermes会话库 |
| V2.5 定稿 | 2026-08-20 | WorkBuddy | 评审通过，标记为 V2.5 正式基线 |
