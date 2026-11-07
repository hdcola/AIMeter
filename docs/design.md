# AIMeter 设计文档：基于 LiteLLM 的计费与用户自服务

状态：草案 v0.1　日期：2026-11-07
依据：LiteLLM Proxy OpenAPI（https://docs.litellm.ai/openapi.json，版本 1.108.2）

## 1. 目标与非目标

### 目标
1. 用户可查看自己账户的**当前余额**。
2. 用户可查看**历史每月使用情况**（花费、tokens、请求数，按模型拆分）。
3. 账户余额耗尽时，**自动停止**对该用户的服务；充值后自动恢复。
4. **月度套餐（最低消费）**：用户按月预付套餐费，套餐费即当月最低消费，并折算为当月用量额度（月底未用完作废）；另可充值钱包余额，钱包余额不过期、滚存。套餐额度与钱包余额都耗尽才停服（见 4.8）。
5. 不修改 LiteLLM 源码，仅通过其管理 API 集成。

### 非目标（v1 不做）
- 支付网关对接细节（仅预留充值入口与回调接口，v1 提供管理员手动充值）。
- 正式发票开具、税务、多币种。
- 团队/组织级计费（v1 只做个人用户维度）。

## 2. 总体架构

```
用户 ──(自己的 virtual key)──────────────► LiteLLM Proxy ──► 上游模型
 │                                           ▲
 │ 浏览器                                    │ master key（仅后端持有）
 ▼                                           │
AIMeter 前端 ──► AIMeter 后端 ─────────────────┘
               │  ├─ 登录/鉴权
               │  ├─ 余额/用量查询（封装 LiteLLM API）
               │  ├─ 充值 / 账本
               │  └─ 对账与停服任务
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
| 用户登录凭据 | **不复用 LiteLLM**：邮箱密码由 AIMeter 自建（见 4.7），LiteLLM 侧不设 `password` |
| 用户 API key | Virtual Key（`/key/generate`，绑定 `user_id`） |
| 额度上限 | `max_budget`（用户级）。= 周期起始 `spend` + 套餐额度 + 钱包余额，由 AIMeter 每周期重算（见 4.8） |
| 累计消费 | `spend`（用户级，**不重置**，LiteLLM 自动累加，是只增的终身计数器） |
| **余额** | `max_budget − spend` = 套餐剩余额度 + 钱包余额 |
| 套餐额度 | 每月发放，月底作废；通过重算 `max_budget` 体现 |
| 钱包余额（充值） | 不过期、滚存；充值即钱包增加，同时 `max_budget += 充值额`（`/user/update`） |
| 欠费/手动停服 | `spend >= max_budget` 时 LiteLLM 自动拒绝；主动停服也通过预算实现（把 `max_budget` 设为当前 `spend`）。**不使用用户级 `blocked`**：规范中标注 `[Not Implemented Yet]`，且 `/v2/user/info` 不返回该字段 |

约束：**不使用 LiteLLM 的 `budget_duration` 自动重置，也不重置 `spend`。** 原因：`budget_duration` 到期会把 `spend` 清零、额度重新发放，无法表达"套餐额度作废、钱包余额滚存"这两种不同生命周期的额度。改为由 AIMeter 每月滚动重算 `max_budget`，LiteLLM 的 `spend` 保持终身累加，边界误差只会在相邻月份之间转移，不会丢失（详见 4.8）。

> 待验证：用户级 `max_budget` 在 key 属于该用户时是否对其所有 key 生效（LiteLLM 有 user / key / team 多层预算）。实现前需在测试环境验证，见第 9 节。

## 4. 功能设计

### 4.1 开户
1. AIMeter 创建用户记录（邮箱、角色、状态 `invited`），并发送设置密码邮件（见 4.7）。
2. 调 `POST /user/new`：`user_id`（取 AIMeter `users.id`）、`user_email`、`user_role=internal_user`、`max_budget=<初始额度，可为 0>`、`auto_create_key=false`；**不传 `password`**。
3. 调 `POST /key/generate`：绑定 `user_id`，**key 自身不设 `max_budget`**（避免与用户余额双重限制不一致），返回给用户一次（之后只显示后四位）。

### 4.2 余额查询
- 后端调 `GET /v2/user/info?user_id=...`，取 `spend`、`max_budget`。
- 返回：`balance = max(limit − spend, 0)`（`limit` 见 4.8），并给出服务状态 `in_service = (max_budget − spend > 0)`（不依赖 `blocked`）；同时按当期周期记录拆分为**套餐剩余额度** `max(0, plan_quota − (spend − spend_start))` 与**钱包余额** `balance − 套餐剩余额度`（见 4.8）。可用余额 = `max(balance − safety_reserve, 0)`（见 4.4）。
- 后端做 10–30 秒短缓存，防止前端刷新造成对 LiteLLM 的压力；**充值、标记已付、周期滚动、停服/恢复后立即失效缓存**。

### 4.3 月度用量
- 后端调 `GET /user/daily/activity?user_id=&start_date=&end_date=&timezone=`，返回按日的 `spend`、tokens、`api_requests`，并含 model/provider/api_key 拆分。
- **`timezone` 是整数分钟偏移，不是时区名**（规范原文：Timezone offset in minutes from UTC，沿用 JavaScript `Date.getTimezoneOffset()` 约定，符号相反：UTC+8 传 `-480`）；`start_date`/`end_date` 为 `YYYY-MM-DD` 字符串。
- 月度聚合统一使用**结算时区**（见 4.8，所有用户相同，不使用无夏令时的固定偏移以外的时区），由后端自行计算月边界后传 `start_date`/`end_date` 与对应偏移；不做按用户时区的聚合。
- **口径**：展示用 daily activity；计费结算用 `billing_cycles` 中记录的 `spend` 差值（4.8）。两者在边界上可能有微小差异（规范说明 daily 累计可能大于 `spend`），以 `billing_cycles` 为计费准，界面注明用量统计为参考口径。
- 重要：必须使用 daily activity，**不能用 `spend` 当月账单**。官方说明 daily activity 只增不减、不受预算重置影响。
- 已结束的月份固化到 AIMeter DB（`monthly_usage`），避免反复拉取，也防止上游数据保留策略变化。
- 分页：注意 `page`/`page_size` 参数，聚合时需循环取完。

### 4.4 余额耗尽停服（不允许透支）

**政策**：不允许用户余额为负，也不向用户追欠款。**停服统一通过预算实现，不使用用户级 `blocked`。**

**停服/恢复的定义**
- 停服态：把用户 `max_budget` 设为不高于当前 `spend`（取读取到的实时 `spend`），LiteLLM 因 `spend >= max_budget` 拒绝请求。边界判定（`>=` 还是 `>`）在 M0 验证。
- 恢复：按 4.8 公式重算 `max_budget` 并写入，无需"解封"。
- 手动停服（`/admin/users/{id}/hold`）：置 `users.manual_hold = true`，重算时始终写停服态；解除后恢复重算。

**三层保障**
1. **自动层（LiteLLM 原生）**：`spend >= max_budget` 即拒绝请求，主防线，无需 AIMeter 介入。
2. **主动层（AIMeter 对账任务）**：定时（如每分钟）分页扫描 `GET /user/list`（含 `spend`、`max_budget`），对每个用户按周期记录和实时 `spend` 计算剩余额度；剩余 ≤ 0、`manual_hold`、或 `spend < spend_start`（见 4.8 异常处理）时写停服态，并写审计、发通知；剩余 > 0 但处于停服态时（例如充值后）重算恢复。目的：处理预算字段被意外改动、并发超支后的收敛、欠费通知。
3. **防透支（尽量压低，非绝对）**：LiteLLM 在请求**完成后**才记账，并发请求可能小幅越过上限，仅靠预算无法做到绝对零透支。措施：
   - `safety_reserve`：下发给 LiteLLM 的 `max_budget = limit − safety_reserve`，把保留额留给在途请求，使并发超支落在真实余额之内；界面显示"可用余额 = 余额 − safety_reserve"。保留额大小见第 11 节。
   - 限制 `max_parallel_requests`、`rpm_limit`、`tpm_limit`，缩小并发窗口。
   - 若仍超出（`U` 大于 套餐额度 + 钱包 + 充值），**不向用户追缴、钱包不记负数**：超出部分计入 `billing_cycles.overdraft`，由平台承担，触发告警，作为调整 `safety_reserve`/限流的依据。
   - 更严格的实时校验（pre-call 钩子，在请求前按 AIMeter 余额拒绝）作为 v2 可选加固。

**预警**：可用余额低于阈值（如 20%）通过邮件通知（v1 可选）。

### 4.5 充值
- v1：管理员后台手动充值（`POST /admin/users/{id}/topup`）。
- 流程：写入充值账本（幂等，带 `order_id`）→ 在用户锁内（见 4.8）确定所属周期并重算 `max_budget` 写入 LiteLLM（同时即恢复服务）。
- **幂等与一致性**：先写本地账本（状态 pending）→ 调 LiteLLM → 标记 applied；失败可由重试任务补偿。禁止"读后写"式累加：`max_budget` 一律由当期周期记录重算（`spend_start + 套餐额度 + 钱包起始余额 + 当期充值合计`，见 4.8），保证重试安全。充值进入钱包，不过期。
- v2：接入 Stripe 等，回调验签后走同一流程。

### 4.6 自服务界面
- 概览：总余额，拆分显示套餐剩余额度（含到期日）与钱包余额；本月已用；套餐状态与下次扣费；服务状态（正常/已停服）。
- 用量：月度柱状图 + 按模型拆分表，可切换月份。
- Key 管理：查看 key 后四位、新建/轮换/删除 key（`/key/generate`、`/key/update`，只允许操作自己的）。
- 充值记录：账本列表。

### 4.7 用户认证（自建邮箱密码）

**决策**：AIMeter 自建邮箱 + 密码认证，不复用 LiteLLM 的登录体系。理由：LiteLLM 的账号密码与会话是为其 `/ui` 管理后台服务的（`/user/password/change` 仅限 UI 会话调用，`/session/logout` 仅接受 UI 会话 key），规范中没有面向第三方应用的"校验密码并签发令牌"接口；SSO 部分可能涉及企业版许可；且支付、通知、找回账号等需要自己掌控用户体系。

**与 LiteLLM 的唯一耦合点**：`users.id` 即 LiteLLM 的 `user_id`（4.1 开户时同时创建）。LiteLLM 侧不设密码，其 `/ui` 仅管理员使用，不对终端用户开放；终端用户模型调用只用 virtual key，不涉及登录。

**账户生命周期**
- v1 采用**管理员开户（邀请制）**：管理员创建用户 → 系统发送一次性"设置密码"链接 → 用户设置密码后状态变为 `active`。是否开放自助注册见第 11 节。
- 找回密码：邮箱发送一次性重置链接，令牌随机 32 字节、库中只存哈希、30 分钟过期、一次有效；重置成功后吊销该用户所有现有会话。
- 修改密码：需输入旧密码；成功后吊销其他会话。
- 邮箱变更：需新邮箱验证；同步调 `/user/update` 更新 LiteLLM 侧 `user_email`。

**密码**
- 使用 argon2id 哈希（带每用户随机盐），不可逆，库中不存明文。
- 最小长度 10，不强制复杂字符组合；拒绝常见弱密码列表。

**会话**
- 服务端会话：登录后生成随机会话 ID，存 `sessions` 表（仅存哈希），通过 Cookie 下发：`HttpOnly; Secure; SameSite=Lax`。
- 空闲 24 小时过期，绝对有效期 30 天；支持主动登出与"登出所有设备"（吊销即时生效，这是不用无状态 JWT 的原因）。
- 状态变更类请求做 CSRF 防护（SameSite + CSRF token）。

**防暴力破解**
- 登录接口按 IP + 邮箱双维度限流；连续失败 5 次后该账户临时锁定 15 分钟（指数退避）。
- 登录失败统一返回"邮箱或密码错误"，不暴露邮箱是否存在；找回密码接口同样不暴露。

**角色**
- `users.role`：`user` / `admin`。管理接口（`/admin/*`）要求 `admin`，与普通用户使用同一登录体系但独立授权。
- 被停服（余额耗尽或手动停服）或停用的用户：停服**只影响 LiteLLM 模型调用**，不影响登录，用户仍能登录查看账单并充值；仅账户 `disabled` 时禁止登录。

**接口**见第 5 节认证部分。

### 4.8 月度套餐与额度滚动

**业务规则**
- 套餐：每月固定费用 `fee`（即最低消费），对应每月用量额度 `quota`（v1 默认 `quota = fee`，可配置）。
- 月初预收：每个周期开始生成套餐费账单，**支付后**发放当月套餐额度。
- 两类额度：套餐额度（当月有效，月底作废）；钱包余额（用户充值，不过期，滚存）。
- 扣减顺序：**先扣套餐额度，再扣钱包余额**。
- 停服：套餐额度与钱包余额都为零时，由 LiteLLM 预算机制拒绝请求（见 4.4）；套餐额度用完但钱包有余额则继续服务；充值或下月发放新额度后恢复。
- 周期：按自然月，使用统一的结算时区（待定，见第 11 节），所有用户同一周期。

**与 LiteLLM 的映射：单一计数器 + 每月重算上限**

LiteLLM 每个用户只有一个 `max_budget`、一个 `spend`。我们让 `spend` 永远只增、不重置，把"两类额度"折叠进 `max_budget`：

```
周期开始：
  spend_start = 上一周期的 spend_end（首个周期：开户时读取的 LiteLLM spend）
  limit       = spend_start + plan_quota + wallet_start + topups_total   -- 套餐未付则 plan_quota=0
  max_budget  = limit − safety_reserve                                   -- 见 4.4；停服态则取实时 spend

周期内（每次变更都重算，不做增量累加）：
  充值 T：      topups_total += T，重算 limit / max_budget
  支付套餐费：  plan_quota 发放，重算（周期内晚付时）
  余额 = limit − spend = 套餐剩余 + 钱包余额

周期结束结算：
  spend_end   = 结算时读取的 LiteLLM spend（写入周期记录后不再改）
  U           = spend_end − spend_start                        -- 当月用量
  from_plan   = min(U, plan_quota)
  from_wallet = U − from_plan
  wallet_end  = max(0, wallet_start + topups_total − from_wallet)
  overdraft   = max(0, from_wallet − (wallet_start + topups_total))   -- 并发超支：平台承担，不追缴，记录并告警（见 4.4）
  下一周期：spend_start' = spend_end，wallet_start' = wallet_end
```

**基线说明**：`spend_end` 是唯一权威基线。写入新 `max_budget` 之前若有请求在途或任务失败重试，实时 `spend` 会略大于 `spend_end`，这部分差额因 `spend_start' = spend_end` 自然计入**新周期**消耗，不会丢失，也不需要额外补偿。

**异常处理（防 `spend` 被改动）**：每次重算 `max_budget` 前读取实时 `spend`，若 `spend < spend_start`（被重置或手动编辑），**停止重算、写停服态、告警**，由管理员处理后再恢复（宁可停服也不下发错误的额度）。开户时显式传 `budget_duration` 为空，且不设置会产生周期重置的默认用户参数。

**按用户串行化**：充值应用、标记已付、月度结算、对账任务对同一用户写 LiteLLM 前必须获取该用户的锁（数据库行锁或 advisory lock）。`topups.cycle_month` 在**应用时**（锁内）确定，取当时的开启周期；对已关闭周期的套餐费账单，付款后**不发放套餐额度**，金额按管理员确认转入钱包（默认策略，见第 11 节）。

套餐额度"作废"无需任何额外动作：下个周期重算 `max_budget` 时不再包含上月未用的部分。

**为什么不重置 `spend`**
- 避免 `/user/update` 清零与进行中请求之间的竞态：若结算时有请求在途，其费用会落入下一个周期，只造成月度归属的微小偏移，总额不丢失。
- `spend` 终身累加，可与 daily activity 互相校验。
- 月度用量展示仍使用 `/user/daily/activity`（见 4.3）。

**每月任务（幂等）**
1. 在周期边界执行；以 `billing_cycles (user_id, month)` 为幂等键，重复执行不重复结算。
2. 结算：读取 `spend_end` 并**一次性写入**周期记录，之后只用该记录计算。
3. 开启新周期：写入新记录，生成套餐费账单（`auto_renew` 的订阅），调 `/user/update` 设置新 `max_budget`（即恢复服务）；写入后回读 `/v2/user/info` 校验 `max_budget`，不一致则重试并告警。
4. **任务延迟的两面影响**：窗口内上月套餐额度已耗尽的用户继续被拒；反之，上月额度有剩余的用户会在窗口内继续用到"已过期"的额度（`max_budget` 仍是上月值，且这部分用量按旧周期结算，消耗的是旧额度）。该损失以窗口内的用量为上界，v1 接受。任务应在周期边界后立即运行，记录 `closed_at` 并监控延迟（超过阈值如 5 分钟告警）。

**套餐费未支付**
- 订阅状态 `past_due`：该周期 `plan_quota = 0`，用户仍可凭钱包余额使用；支付后立即发放当期额度。
- 周期内先用钱包余额、后支付套餐费：由于按周期总用量先扣套餐额度，这部分用量将被计入套餐额度，钱包余额相应在结算时补回。这对用户有利，v1 接受。

**v1 支付**：套餐费由管理员确认到账后标记已付（`/admin/invoices/{id}/mark-paid`）；接入 Stripe 后改为回调自动标记（M4）。

## 5. AIMeter 后端 API（对外）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/login` | 邮箱密码登录，下发会话 Cookie |
| POST | `/api/auth/logout` | 登出当前会话 |
| POST | `/api/auth/logout-all` | 登出所有设备 |
| POST | `/api/auth/password/set` | 凭邀请/重置令牌设置密码 |
| POST | `/api/auth/password/forgot` | 申请重置邮件（始终返回成功） |
| POST | `/api/me/password` | 修改密码（需旧密码） |
| GET | `/api/me` | 当前用户资料与角色 |
| GET | `/api/me/balance` | 余额、已用、状态 |
| GET | `/api/me/usage/monthly?from=&to=` | 月度汇总 |
| GET | `/api/me/usage/daily?month=YYYY-MM` | 某月按日明细（含模型拆分） |
| GET | `/api/me/keys` | 我的 key 列表（脱敏） |
| POST | `/api/me/keys` | 新建 key |
| DELETE | `/api/me/keys/{id}` | 删除 key |
| GET | `/api/me/topups` | 充值记录 |
| GET | `/api/me/plan` | 当前套餐、状态、套餐剩余额度与到期日 |
| GET | `/api/me/invoices` | 套餐费账单 |
| POST | `/admin/plans` | 创建/修改套餐 |
| PUT | `/admin/users/{id}/subscription` | 设置用户套餐、自动续费 |
| POST | `/admin/invoices/{id}/mark-paid` | 确认套餐费已付，发放当期额度 |
| POST | `/admin/users` | 开户 |
| POST | `/admin/users/{id}/topup` | 手动充值 |
| POST | `/admin/users/{id}/hold` / `unhold` | 手动停服/解除（`manual_hold`，通过预算实现，见 4.4） |

所有 `/api/me/*` 的 `user_id` 取自登录会话，**永不接受客户端传入**，防止越权。

## 6. LiteLLM 接口使用清单

| 用途 | 接口 |
|---|---|
| 开户 | `POST /user/new` |
| 修改额度/停服 | `POST /user/update`（`max_budget`；停服即把 `max_budget` 设为不高于当前 `spend`）。不使用 `blocked`（规范标注未实现） |
| 查余额 | `GET /v2/user/info` |
| 批量扫描 | `GET /user/list` |
| 月度/日度用量 | `GET /user/daily/activity` |
| 发/改/删 key | `POST /key/generate`、`/key/update`、`/key/delete` |
| 明细对账（可选） | `GET /spend/logs` |
| 终端客户模式（备选） | `/customer/new`、`/customer/block`、`/customer/daily/activity` |

备选：若用户并不登录 LiteLLM、只通过请求里的 `user` 字段标识，可使用 Customer（End User）模型。本设计选 Internal User 是因为需要为每个用户发独立 key。

## 7. 数据模型（AIMeter DB）

```
users(id  -- 同时作为 LiteLLM user_id,
      email UNIQUE, password_hash, role[user|admin],
      status[invited|active|disabled], manual_hold, failed_logins, locked_until, created_at)
sessions(id_hash PK, user_id, created_at, last_seen_at, expires_at, ip, user_agent, revoked_at)
auth_tokens(token_hash PK, user_id, purpose[invite|reset|email_change], expires_at, used_at)
topups(id, user_id, order_id UNIQUE, amount, currency, status[pending|applied|failed],
       source[manual|stripe], cycle_month  -- 应用时在用户锁内确定, created_at, applied_at)
monthly_usage(user_id, month, spend, prompt_tokens, completion_tokens,
              requests, breakdown_json, finalized_at,  PRIMARY KEY(user_id, month))
audit_log(id, actor, action, target_user, detail_json, created_at)
plans(id, name, monthly_fee, monthly_quota, currency, active)
subscriptions(user_id PK, plan_id, status[active|past_due|canceled], auto_renew, started_at)
invoices(id, user_id, cycle_month, kind[plan_fee], amount, status[pending|paid|void], created_at, paid_at)
billing_cycles(user_id, month, spend_start, spend_end, plan_quota, wallet_start, topups_total,
               wallet_end, overdraft, status[open|closed], closed_at, PRIMARY KEY(user_id, month))
```

（`topups` 即钱包充值；`applied` 的充值记入当期 `topups_total`。）

LiteLLM 的 `max_budget` 是账本的投影：始终由当期 `billing_cycles` 行重算（`spend_start + plan_quota + wallet_start + topups_total`），不做增量累加。

## 8. 安全

- master key 仅存于后端环境变量/密钥管理，不入库、不下发（详见 8.1）。
- 后端到 LiteLLM 走内网；LiteLLM 管理接口不对公网暴露，仅公开数据面（`/v1/*`）。
- 所有 `/api/me/*` 做会话鉴权 + 用户隔离；管理接口独立角色。认证细节见 4.7（argon2id、服务端会话、登录限流）。
- 新建 key 明文只返回一次。
- 审计所有额度变更与封禁动作。

### 8.1 Master key 说明与管理

**是什么**：LiteLLM Proxy 的超级管理员密钥，持有者等同 `proxy_admin`，可调用全部管理接口（创建/删除用户、发 key、改预算、封禁、查看全部花费）。AIMeter 后端调用 `/user/new`、`/user/update` 等接口时使用它。

**来源**：由部署者在部署 LiteLLM 时自行设定，LiteLLM 不会代为生成。二选一：

```yaml
# config.yaml
general_settings:
  master_key: sk-your-strong-secret
```
```bash
export LITELLM_MASTER_KEY="sk-your-strong-secret"
```

要求：
- 必须以 `sk-` 开头，用足够长的随机串，例如 `echo "sk-$(openssl rand -hex 32)"`。
- 必须同时配置数据库（`DATABASE_URL`），否则虚拟 key、用户、预算、花费记录等功能不可用。

**与 virtual key 的区别**：

| | Master key | Virtual key |
|---|---|---|
| 来源 | 部署时配置 | 通过 `/key/generate` 创建 |
| 权限 | 全部管理权限 | 只能调模型，受预算/模型/限流约束 |
| 数量 | 一个 | 多个，每用户一个或多个 |
| 持有者 | 仅 AIMeter 后端 | 终端用户 |

**使用规则**：
- 泄露等于整个网关失守：只放后端环境变量或密钥管理系统，不进代码仓库、不下发前端、不写入日志。
- 仅用于管理，不用它调用模型。
- 支持轮换：更换后同步更新 AIMeter 后端配置。
- 降权方案（待验证，纳入 M0）：用 `/key/generate` 创建仅允许特定管理路由的受限 key（`allowed_routes`）交给 AIMeter 后端使用，而不是直接使用 master key，以缩小泄露影响面。

## 9. 风险与待验证

| 风险 | 说明 | 应对 |
|---|---|---|
| 并发超支 | spend 在请求完成后记账，高并发会小幅超出 `max_budget`，无法做到绝对零透支 | `safety_reserve` + 并发/速率限制；对账任务收敛；残余超支由平台承担、记 `overdraft` 并告警（见 4.4）；v2 可加 pre-call 钩子 |
| 用户级 `blocked` 未实现 | 规范标注 `[Not Implemented Yet]`，且 `/v2/user/info` 不返回 | 不使用；停服统一用 `max_budget ≤ spend`；M0 验证边界判定 |
| BETA 接口变动 | `daily/activity` 标注 BETA | 锁定 LiteLLM 版本；封装适配层；升级前跑契约测试 |
| 企业版限制 | 部分报表/SSO 可能需 Enterprise 许可 | 实现前逐项核对许可证；核心路径只用开源接口 |
| 用户级预算对多 key 的行为 | 需确认用户 `max_budget` 是否约束其所有 key | **实现第一步在测试实例验证** |
| `/v2/user/info` 返回字段 | 本文未核对响应字段细节 | 对照实际响应落地 DTO |
| 时区与月边界 | `/user/daily/activity` 的 `timezone` 是整数分钟偏移（符号同 `getTimezoneOffset`），无法表达夏令时 | 统一结算时区（建议无夏令时的固定偏移）；自行计算月边界；固化已结束月份；套餐周期用同一时区 |
| 月度滚动任务失败/延迟 | 任务未执行时，新周期的 `max_budget` 未更新，套餐耗尽的用户会继续被拒 | 任务幂等、边界后立即运行、失败告警；可手动重跑 |
| 结算读取 `spend` 的边界误差 | 在途请求的费用落入下一周期 | `spend` 不重置，总额不丢，仅月度归属微小偏移；接受并在文档中说明 |
| 预算变更传播延迟 | 多实例且无 Redis 时，其他实例最多约 60 秒仍按旧预算拦截（来自 LiteLLM 自助文档，待验证） | 部署 Redis；充值/新周期后在界面提示可能有短暂延迟 |
| `spend` 是否被意外重置 | 方案依赖 `spend` 只增；若配置了 `budget_duration` 或默认用户参数会破坏 | 不设置 `budget_duration`、`max_internal_user_budget` 的周期；M0 验证；对账任务检测 `spend` 回退并告警 |

## 10. 里程碑

1. **M0 验证（1–2 天）**：起 LiteLLM 测试实例，验证预算超限拒绝、用户级预算对多 key、daily activity 返回结构；验证"`max_budget` 设为当前 `spend`"能否拒绝请求（`>=` 边界）、恢复后是否立即生效；验证受限管理 key（`allowed_routes`）能否替代 master key 完成后端所需的全部管理调用；验证不设 `budget_duration` 时 `spend` 只增不减、`/user/update` 调整 `max_budget` 不影响 `spend` 且立即生效；验证多实例下预算变更传播延迟。
2. **M1 后端核心**：自建认证（登录、会话、邀请/重置密码）、开户、余额、月度用量、手动充值（钱包）、账本、对账封禁任务。
3. **M1.5 月度套餐**：套餐与订阅、套餐费账单（手动标记已付）、周期记录与每月滚动重算任务、余额拆分（套餐/钱包）。
4. **M2 AIMeter 前端（用户自服务界面）**：概览（套餐/钱包拆分）、用量图、key 管理、充值记录、套餐与账单。
5. **M3 加固**：低余额通知、审计、监控告警、限流。
6. **M4 支付接入**：Stripe 等，回调幂等；套餐费自动扣款。

## 11. 待决策

- 技术栈（建议：后端 FastAPI + PostgreSQL，前端 React/Vue 任选；尚未确认）。
- ~~用户认证方式~~：已决定自建邮箱密码（见 4.7）。仍待定：是否开放自助注册（含邮箱验证）；SMTP/邮件发送服务选型；后续是否追加 OIDC 登录。
- 余额单位（美元计价 vs 自定义"点数"；LiteLLM 内部以 USD 记账，点数需在 AIMeter 层换算）。
- ~~是否允许小额透支~~：已决定**不允许**（钱包不记负数、不向用户追欠款）；因 LiteLLM 事后记账无法绝对杜绝，残余超支由平台承担（见 4.4）。仍待定：`safety_reserve` 的大小；`max_parallel_requests`/`rpm_limit` 默认值。
- 已关闭周期的套餐费账单被付款后的处理：默认转入钱包，不发放套餐额度，需确认。
- 套餐细节：套餐档位与价格；`quota` 是否等于 `fee`（是否赠送额外额度）；周期统一结算时区（UTC / Asia/Shanghai）；用户月中开通或升降级是否按比例折算；套餐费未付时是否设宽限期。

## 12. 评审待办（尚未处理）

来自一次独立评审，以下问题已确认但尚未写入正文设计：

- 开户（4.1）需同时创建当期 `billing_cycles` 记录，并处理月中开通的折算；否则"`max_budget` 始终由周期行推导"不成立。
- 取消订阅：缺取消接口，未定义当期已付额度与待付账单（应 `void`）的处理，以及是否退款。
- 金额类型：LiteLLM 预算为 USD 浮点，`plans.currency` 与之的换算未定义；数据库使用 Decimal，推送时统一舍入。
- 用户权限（M5）：`internal_user` 可通过数据面端口调用 `/key/*` 自建带预算的 key，需在网关限制路由或用 `allowed_routes`；`/key/regenerate` 规范中存在，应补入 key 轮换与 API 表。
- `billing_cycles` 增加同步状态字段（`synced_at`、最后下发的 `max_budget`、`plan_paid_at`、套餐费快照、任务 run id）；`/user/new` 与 `/key/generate` 中途失败的补偿。
- 钱包充值缺冲正/退款状态（Stripe 拒付）。
- API 表缺：取消订阅、查看订阅、邀请重发、改邮箱、邮箱验证、会话列表、管理员账单列表。
- 安全：`DELETE /api/me/keys/{id}` 的归属校验（IDOR）与 key 数量上限；管理员充值/标记已付需二次验证；登录锁定可被用于恶意锁号；`audit_log` 与 `topups`/`invoices`/`billing_cycles` 的关联。
- 里程碑：`billing_cycles` 应在 M1 建好（4.5 的充值依赖它）。
- 风险表中 `max_internal_user_budget` 周期的表述在规范中无法核实，需重新措辞。
