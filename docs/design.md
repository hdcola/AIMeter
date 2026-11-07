# AIMeter 设计文档：基于 LiteLLM 的计费与用户自服务

状态：草案 v0.1　日期：2026-11-07
依据：LiteLLM Proxy OpenAPI（https://docs.litellm.ai/openapi.json，版本 1.108.2）

## 1. 目标与非目标

### 目标
1. 用户可查看自己账户的**当前余额**。
2. 用户可查看**历史每月使用情况**（花费、tokens、请求数，按模型拆分）。
3. 账户余额耗尽时，**自动停止**对该用户的服务；充值后自动恢复。
4. 不修改 LiteLLM 源码，仅通过其管理 API 集成。

### 非目标（v1 不做）
- 支付网关对接细节（仅预留充值入口与回调接口，v1 提供管理员手动充值）。
- 发票、税务、多币种。
- 团队/组织级计费（v1 只做个人用户维度）。

## 2. 总体架构

```
用户 ──(自己的 virtual key)──────────────► LiteLLM Proxy ──► 上游模型
 │                                           ▲
 │ 浏览器                                    │ master key（仅后端持有）
 ▼                                           │
自服务前端 ──► AIMeter 后端 ───────────────────┘
               │  ├─ 登录/鉴权
               │  ├─ 余额/用量查询（封装 LiteLLM API）
               │  ├─ 充值 / 账本
               │  └─ 对账与封禁任务
               ▼
            AIMeter DB（账本、充值订单、月度快照）
```

关键原则：
- **数据面与管理面分离**：用户调用模型直连 LiteLLM（用自己的 key）；查询余额/用量只走 AIMeter 后端。
- **master key 只存在于后端**，前端和用户永远拿不到。
- **LiteLLM 是消费数据的事实来源，AIMeter 账本是充值数据的事实来源。**

## 3. 核心概念映射

LiteLLM 没有"余额"字段，只有预算模型。映射如下：

| 业务概念 | LiteLLM 实现 |
|---|---|
| 用户 | Internal User（`/user/new`），`user_id` 与 AIMeter 用户一一对应 |
| 用户 API key | Virtual Key（`/key/generate`，绑定 `user_id`） |
| 累计充值总额 | `max_budget`（用户级） |
| 累计消费 | `spend`（用户级，LiteLLM 自动累加） |
| **余额** | `max_budget − spend` |
| 充值 | 增加 `max_budget`（`/user/update`） |
| 欠费停服 | `spend >= max_budget` 时 LiteLLM 自动拒绝；另有 `blocked` 主动封禁 |

约束：**v1 不使用 `budget_duration` 自动重置**（预付费模型下不应重置余额）。若未来做月度套餐，再引入。

> 待验证：用户级 `max_budget` 在 key 属于该用户时是否对其所有 key 生效（LiteLLM 有 user / key / team 多层预算）。实现前需在测试环境验证，见第 9 节。

## 4. 功能设计

### 4.1 开户
1. AIMeter 创建用户记录。
2. 调 `POST /user/new`：`user_id`、`user_email`、`user_role=internal_user`、`max_budget=<初始额度，可为 0>`、`auto_create_key=false`。
3. 调 `POST /key/generate`：绑定 `user_id`，**key 自身不设 `max_budget`**（避免与用户余额双重限制不一致），返回给用户一次（之后只显示后四位）。

### 4.2 余额查询
- 后端调 `GET /v2/user/info?user_id=...`，取 `spend`、`max_budget`。
- 返回：`balance = max(max_budget − spend, 0)`，并附 `blocked` 状态。
- 后端做 10–30 秒短缓存，防止前端刷新造成对 LiteLLM 的压力。

### 4.3 月度用量
- 后端调 `GET /user/daily/activity?user_id=&start_date=&end_date=&timezone=`，返回按日的 `spend`、tokens、`api_requests`，并含 model/provider/api_key 拆分。
- 后端按用户所属时区**按月聚合**。
- 重要：必须使用 daily activity，**不能用 `spend` 当月账单**。官方说明 daily activity 只增不减、不受预算重置影响。
- 已结束的月份固化到 AIMeter DB（`monthly_usage`），避免反复拉取，也防止上游数据保留策略变化。
- 分页：注意 `page`/`page_size` 参数，聚合时需循环取完。

### 4.4 余额耗尽停服（两层保障）
1. **自动层（LiteLLM 原生）**：`spend >= max_budget` 即拒绝请求。这是主防线，无需 AIMeter 介入。
2. **主动层（AIMeter 对账任务）**：
   - 定时（如每分钟）扫描 `GET /user/list` 或按活跃用户查询，余额 ≤ 0 的用户调 `POST /user/update {blocked:true}`，同时写入审计日志。
   - 充值成功后：增加 `max_budget`，若 `blocked=true` 则解封。
   - 目的：处理并发超支、预算字段被意外改动、以及需要给用户发"欠费通知"的场景。
3. **预警**：余额低于阈值（如 20%）通过邮件通知（v1 可选）。

### 4.5 充值
- v1：管理员后台手动充值（`POST /admin/users/{id}/topup`）。
- 流程：写入充值账本（幂等，带 `order_id`）→ 调 LiteLLM 增加 `max_budget` → 如被封禁则解封。
- **幂等与一致性**：先写本地账本（状态 pending）→ 调 LiteLLM → 标记 applied；失败可由重试任务补偿。禁止"读后写"式累加，改用账本总和重算 `max_budget = Σ充值`，保证重试安全。
- v2：接入 Stripe 等，回调验签后走同一流程。

### 4.6 自服务界面
- 概览：余额、本月已用、封禁状态。
- 用量：月度柱状图 + 按模型拆分表，可切换月份。
- Key 管理：查看 key 后四位、新建/轮换/删除 key（`/key/generate`、`/key/update`，只允许操作自己的）。
- 充值记录：账本列表。

## 5. AIMeter 后端 API（对外）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/me/balance` | 余额、已用、状态 |
| GET | `/api/me/usage/monthly?from=&to=` | 月度汇总 |
| GET | `/api/me/usage/daily?month=YYYY-MM` | 某月按日明细（含模型拆分） |
| GET | `/api/me/keys` | 我的 key 列表（脱敏） |
| POST | `/api/me/keys` | 新建 key |
| DELETE | `/api/me/keys/{id}` | 删除 key |
| GET | `/api/me/topups` | 充值记录 |
| POST | `/admin/users` | 开户 |
| POST | `/admin/users/{id}/topup` | 手动充值 |
| POST | `/admin/users/{id}/block` / `unblock` | 手动封禁/解封 |

所有 `/api/me/*` 的 `user_id` 取自登录会话，**永不接受客户端传入**，防止越权。

## 6. LiteLLM 接口使用清单

| 用途 | 接口 |
|---|---|
| 开户 | `POST /user/new` |
| 修改额度/封禁 | `POST /user/update`（`max_budget`、`blocked`） |
| 查余额 | `GET /v2/user/info` |
| 批量扫描 | `GET /user/list` |
| 月度/日度用量 | `GET /user/daily/activity` |
| 发/改/删 key | `POST /key/generate`、`/key/update`、`/key/delete` |
| 明细对账（可选） | `GET /spend/logs` |
| 终端客户模式（备选） | `/customer/new`、`/customer/block`、`/customer/daily/activity` |

备选：若用户并不登录 LiteLLM、只通过请求里的 `user` 字段标识，可使用 Customer（End User）模型。本设计选 Internal User 是因为需要为每个用户发独立 key。

## 7. 数据模型（AIMeter DB）

```
users(id, litellm_user_id, email, created_at, status)
topups(id, user_id, order_id UNIQUE, amount, currency, status[pending|applied|failed],
       source[manual|stripe], created_at, applied_at)
monthly_usage(user_id, month, spend, prompt_tokens, completion_tokens,
              requests, breakdown_json, finalized_at,  PRIMARY KEY(user_id, month))
audit_log(id, actor, action, target_user, detail_json, created_at)
```

`litellm max_budget` 始终由 `SUM(topups.amount WHERE applied)` 推导，是账本的投影。

## 8. 安全

- master key 仅存于后端环境变量/密钥管理，不入库、不下发。
- 后端到 LiteLLM 走内网；LiteLLM 管理接口不对公网暴露，仅公开数据面（`/v1/*`）。
- 所有 `/api/me/*` 做会话鉴权 + 用户隔离；管理接口独立角色。
- 新建 key 明文只返回一次。
- 审计所有额度变更与封禁动作。

## 9. 风险与待验证

| 风险 | 说明 | 应对 |
|---|---|---|
| 并发超支 | spend 在请求完成后记账，高并发会小幅超出 `max_budget` | 加 `rpm_limit`/`max_parallel_requests`；对账任务及时封禁；业务上接受小额透支 |
| BETA 接口变动 | `daily/activity` 标注 BETA | 锁定 LiteLLM 版本；封装适配层；升级前跑契约测试 |
| 企业版限制 | 部分报表/SSO 可能需 Enterprise 许可 | 实现前逐项核对许可证；核心路径只用开源接口 |
| 用户级预算对多 key 的行为 | 需确认用户 `max_budget` 是否约束其所有 key | **实现第一步在测试实例验证** |
| `/v2/user/info` 返回字段 | 本文未核对响应字段细节 | 对照实际响应落地 DTO |
| 时区与月边界 | 月度汇总依赖时区 | 统一使用用户时区，`timezone` 参数传入，并固化已结束月份 |

## 10. 里程碑

1. **M0 验证（1–2 天）**：起 LiteLLM 测试实例，验证预算超限拒绝、用户级预算对多 key、daily activity 返回结构、`blocked` 行为。
2. **M1 后端核心**：开户、余额、月度用量、手动充值、账本、对账封禁任务。
3. **M2 自服务前端**：概览、用量图、key 管理、充值记录。
4. **M3 加固**：低余额通知、审计、监控告警、限流。
5. **M4 支付接入**：Stripe 等，回调幂等。

## 11. 待决策

- 技术栈（建议：后端 FastAPI + PostgreSQL，前端 React/Vue 任选；尚未确认）。
- 用户认证方式（自建账号 / OIDC SSO）。
- 余额单位（美元计价 vs 自定义"点数"；LiteLLM 内部以 USD 记账，点数需在 AIMeter 层换算）。
- 是否允许小额透支。
