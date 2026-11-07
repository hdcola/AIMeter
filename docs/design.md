# AIMeter 设计文档：基于 LiteLLM 的计费与用户自服务

状态：草案 v0.1　日期：2026-11-07
依据：LiteLLM Proxy OpenAPI（https://docs.litellm.ai/openapi.json，版本 1.108.2）

## 1. 目标与非目标

### 目标
1. 用户可查看自己账户的**当前余额**。
2. 用户可查看**历史每月使用情况**（花费、tokens、请求数，按模型拆分）。
3. 账户余额耗尽时，**自动停止**对该用户的服务；充值后自动恢复。
4. **预付钱包 + 月底结算 + 最低月费**：全部费用预付，用户充值进入钱包（不过期、滚存）；每月底按 `max(当月实际用量, 最低月费)` 从钱包扣费；钱包余额不足最低月费时，下个月不开通服务，充值达标后恢复（见 4.8）。
5. 不修改 LiteLLM 源码，仅通过其管理 API 集成。

### 非目标（v1 不做）
- 支付网关对接细节（仅预留充值入口与回调接口，v1 提供管理员手动充值）。
- 正式发票开具、税务、多币种（v1 仅美元 USD，界面以 `$` 显示）。
- 邮件发送（v1 不做）：提醒走站内通知；邀请开户与重置密码由管理员转交一次性链接；不开放自助注册。
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

### 2.1 技术栈

**后端**
- 语言与框架：Python + **FastAPI**（异步），Pydantic v2 做请求/响应校验，自动生成 OpenAPI 文档。
- 数据库：**仅支持 PostgreSQL**，不支持 SQLite。访问层用 SQLAlchemy 2.0（异步，asyncpg 驱动），迁移用 Alembic。
- 调用 LiteLLM：用 httpx（异步）封装成独立的适配层模块，集中处理超时、重试、鉴权头、版本差异；锁定 LiteLLM 版本，并保留针对测试实例的契约测试（见第 9 节"BETA 接口变动"）。
- 密码哈希：argon2-cffi（argon2id）。会话存 PostgreSQL（见 4.7），不使用无状态 JWT。
- 配置：部署相关配置（数据库连接串、LiteLLM 地址、master key）用环境变量（pydantic-settings）；业务参数（4.9）存数据库 `system_config` 表，管理员可通过 API 修改。
- 后台任务：月底结算、对账停服、月底前提醒、`safety_reserve` 刷新（见 4.4、4.8、4.9）。同一份代码以独立 worker 进程运行，也可通过开关在 API 进程内运行（单实例时）。任务用 PostgreSQL advisory lock 做互斥，保证多实例下同一任务不会重复执行。

**PostgreSQL 使用约定（因只支持 PG，可直接使用其特性）**
- 按用户串行化（4.8）：使用 `pg_advisory_xact_lock`（以用户 id 的哈希为键）或对 `users` 行 `SELECT ... FOR UPDATE`，在事务内完成"读周期记录 → 写 LiteLLM → 更新周期记录"。
- 金额：钱包、充值、扣费等用 `NUMERIC(12,2)`（精确到分）；`spend_start`、`spend_end` 这类来自 LiteLLM 的浮点基线用 `NUMERIC(18,8)` 保存，避免浮点误差累积。
- 时间：一律 `timestamptz`，以 UTC 存储，展示和周期计算时按 `billing_timezone` 转换。
- 幂等：充值 `order_id`、`billing_cycles (user_id, month)`、`notifications` 都用唯一约束 + `INSERT ... ON CONFLICT` 实现。
- 部署：AIMeter 使用独立的数据库和独立的数据库账号；若 LiteLLM 已有 PostgreSQL 服务器，可复用同一服务器但**不共用数据库**，也不直接读写 LiteLLM 的表，一律通过其 API。

**测试**
- pytest + pytest-asyncio；数据库测试使用真实的 PostgreSQL（容器），不用 SQLite 替代。
- 对 LiteLLM 的调用在单元测试中用 mock（如 respx），另有针对 LiteLLM 测试实例的契约测试，覆盖 M0 的验证项。

**前端**：框架尚未确定（React / Vue），见第 11 节。

## 3. 核心概念映射

LiteLLM 没有"余额"字段，只有预算模型。映射如下：

| 业务概念 | LiteLLM 实现 |
|---|---|
| 用户 | Internal User（`/user/new`），`user_id` 与 AIMeter 用户一一对应 |
| 用户登录凭据 | **不复用 LiteLLM**：邮箱密码由 AIMeter 自建（见 4.7），LiteLLM 侧不设 `password` |
| 用户 API key | Virtual Key（`/key/generate`，绑定 `user_id`） |
| 额度上限 | `max_budget`（用户级）。= 周期起始 `spend` + 钱包可用额度 − `safety_reserve`，由 AIMeter 重算并下发（见 4.8） |
| 累计消费 | `spend`（用户级，**不重置**，LiteLLM 自动累加，是只增的终身计数器） |
| 币种 | 全部为**美元（USD）**，界面显示 `$`；AIMeter 的钱包、账单与 LiteLLM 的 `spend`/`max_budget` 使用同一币种，无汇率换算（LiteLLM 的 `spend` 币种由模型单价决定，预期为 USD，M0 核实） |
| **余额（钱包）** | `limit − spend`，其中 `limit = 周期起始 spend + 周期起始钱包 + 当期充值`；钱包只有一类，不过期、滚存 |
| 充值 | 钱包增加，重算并下发 `max_budget`（`/user/update`） |
| 月底结算 | 在 AIMeter 内完成：`扣费 = max(当月用量, 最低月费)`，从钱包扣除；LiteLLM 只负责实时限额 |
| 欠费/手动停服 | `spend >= max_budget` 时 LiteLLM 自动拒绝；主动停服也通过预算实现（把 `max_budget` 设为当前 `spend`）。**不使用用户级 `blocked`**：规范中标注 `[Not Implemented Yet]`，且 `/v2/user/info` 不返回该字段 |

约束：**不使用 LiteLLM 的 `budget_duration` 自动重置，也不重置 `spend`。** 原因：月底扣费（含最低月费保底）与钱包滚存都由 AIMeter 的账本完成，LiteLLM 只需要"钱包还剩多少"这一个上限；`budget_duration` 到期会把 `spend` 清零、重新发放额度，与"余额滚存、按月结算扣费"冲突。因此由 AIMeter 每月滚动重算 `max_budget`，LiteLLM 的 `spend` 保持终身累加，边界误差只会在相邻月份之间转移，不会丢失（详见 4.8）。

> 待验证：用户级 `max_budget` 在 key 属于该用户时是否对其所有 key 生效（LiteLLM 有 user / key / team 多层预算）。实现前需在测试环境验证，见第 9 节。

## 4. 功能设计

### 4.1 开户
1. AIMeter 创建用户记录（邮箱、角色、状态 `invited`），并生成一次性"设置密码"链接，由管理员转交用户（v1 不发邮件，见 4.7）。
2. 调 `POST /user/new`：`user_id`（取 AIMeter `users.id`）、`user_email`、`user_role=internal_user`、`max_budget=0`（钱包为 0，尚未开通，处于停服态）、`budget_duration` 留空、`max_parallel_requests=2`（见 4.4）、`auto_create_key=false`；**不传 `password`**。
3. 调 `POST /key/generate`：绑定 `user_id`，**key 自身不设 `max_budget`**（避免与用户余额双重限制不一致），返回给用户一次（之后只显示后四位）。
4. 创建首个 `billing_cycles` 记录（`spend_start` = 开户时读取的 LiteLLM `spend`，`wallet_start = 0`，未开通）。首次充值达到最低月费后开通服务（见 4.8）。

### 4.2 余额查询
- 后端调 `GET /v2/user/info?user_id=...`，取 `spend`、`max_budget`。
- 返回：钱包余额 `balance = max(limit − spend, 0)`（`limit` 见 4.8），服务状态 `in_service = (max_budget − spend > 0)`（不依赖 `blocked`），本月已用 `U = spend − spend_start`，**预计月底扣费** `max(U, min_fee_cycle)`（已开通周期；未开通周期为 0）。可用余额 = `max(balance − safety_reserve, 0)`（见 4.4）。
- 后端做 10–30 秒短缓存，防止前端刷新造成对 LiteLLM 的压力；**充值、周期滚动、停服/恢复后立即失效缓存**。

### 4.3 月度用量
- 后端调 `GET /user/daily/activity?user_id=&start_date=&end_date=&timezone=`，返回按日的 `spend`、tokens、`api_requests`，并含 model/provider/api_key 拆分。
- **`timezone` 是整数分钟偏移，不是时区名**（规范原文：Timezone offset in minutes from UTC，沿用 JavaScript `Date.getTimezoneOffset()` 约定，符号相反：UTC+8 传 `-480`）；`start_date`/`end_date` 为 `YYYY-MM-DD` 字符串。
- 月度聚合统一使用**结算时区**（系统配置 `billing_timezone`，见 4.9，所有用户相同），由后端自行计算月边界后传 `start_date`/`end_date` 与对应的分钟偏移；不做按用户时区的聚合。
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
2. **主动层（AIMeter 对账任务）**：定时（如每分钟）分页扫描 `GET /user/list`（含 `spend`、`max_budget`），对每个用户按周期记录和实时 `spend` 计算剩余额度；未开通、剩余 ≤ `safety_reserve`、`manual_hold`、或 `spend < spend_start`（见 4.8 异常处理）时写停服态，并写审计、发通知；已开通且剩余 > `safety_reserve` 但处于停服态时（例如充值后）重算恢复。目的：处理预算字段被意外改动、并发超支后的收敛、欠费通知。
3. **防透支（尽量压低，非绝对）**：LiteLLM 在请求**完成后**才记账，并发请求可能小幅越过上限，仅靠预算无法做到绝对零透支。措施：
   - **并发限制：每用户 `max_parallel_requests = 2`**。不单独设置 `max_tokens` 上限：单个请求的输出以各模型自身的最大输出 token（LiteLLM 模型元数据 `max_output_tokens`）为上界。
   - `safety_reserve = 并发上限 × 单个请求最坏成本`（并发 2 即 2 倍）。开放哪些模型以 LiteLLM 为准，最坏成本由 LiteLLM 的模型元数据自动计算（`GET /model_group/info`，见 4.9）：单个请求最坏成本 = `max_input_tokens × input_cost_per_token + max_output_tokens × output_cost_per_token`，取所有开放模型的最大值。下发给 LiteLLM 的 `max_budget = limit − safety_reserve`，把保留额留给在途请求，使并发超支落在真实余额之内。保留额仍在钱包里、月底只按实际用量结算，**不会损失**；界面显示"可用余额 = 余额 − safety_reserve"。
   - v1 暂不设置 `rpm_limit`、`tpm_limit`（见 4.9）；若 M0 测得 `spend` 记账延迟较大，再启用以缩小并发窗口。
   - 若仍超出（`charge` 大于 钱包 + 当期充值），**不向用户追缴、钱包不记负数**：超出部分计入 `billing_cycles.overdraft`，由平台承担，触发告警，作为调整 `safety_reserve`/限流的依据。
   - 更严格的实时校验（pre-call 钩子，在请求前按 AIMeter 余额拒绝）作为 v2 可选加固。

**预警**：月底前提醒见 4.8「月底前提醒」；另可在可用余额低于阈值（如 20%）时通知（可选）。

### 4.5 充值
- v1：管理员后台手动充值（`POST /admin/users/{id}/topup`）。
- 流程：写入充值账本（幂等，带 `order_id`）→ 在用户锁内（见 4.8）确定所属周期并重算 `max_budget` 写入 LiteLLM（同时即恢复服务）。
- **幂等与一致性**：先写本地账本（状态 pending）→ 调 LiteLLM → 标记 applied；失败可由重试任务补偿。禁止"读后写"式累加：`max_budget` 一律由当期周期记录重算（`spend_start + wallet_start + topups_total − safety_reserve`，见 4.8），保证重试安全。充值进入钱包，不过期；钱包未达最低月费时，充值后重新判定开通条件。
- v2：接入 Stripe 等，回调验签后走同一流程。

### 4.6 自服务界面
- 概览：钱包余额与可用余额；本月已用；本月最低月费与**预计月底扣费**（`max(已用, 最低月费)`）；服务状态（正常/已停服，停服时给出原因：余额不足最低月费 / 余额耗尽 / 手动停服）。
- 月度账单：每月用量、最低月费、实际扣费、期末余额。
- 用量：月度柱状图 + 按模型拆分表，可切换月份。
- Key 管理：查看 key 后四位、新建/轮换/删除 key（`/key/generate`、`/key/update`，只允许操作自己的）。
- 充值记录：账本列表。

### 4.7 用户认证（自建邮箱密码）

**决策**：AIMeter 自建邮箱 + 密码认证，不复用 LiteLLM 的登录体系。理由：LiteLLM 的账号密码与会话是为其 `/ui` 管理后台服务的（`/user/password/change` 仅限 UI 会话调用，`/session/logout` 仅接受 UI 会话 key），规范中没有面向第三方应用的"校验密码并签发令牌"接口；SSO 部分可能涉及企业版许可；且支付、通知、找回账号等需要自己掌控用户体系。

**与 LiteLLM 的唯一耦合点**：`users.id` 即 LiteLLM 的 `user_id`（4.1 开户时同时创建）。LiteLLM 侧不设密码，其 `/ui` 仅管理员使用，不对终端用户开放；终端用户模型调用只用 virtual key，不涉及登录。

**账户生命周期**
- v1 采用**管理员开户（邀请制）**，**不发邮件、不开放自助注册**：管理员创建用户 → 系统生成一次性"设置密码"链接，管理员复制后通过其他渠道转交用户 → 用户设置密码后状态变为 `active`。
- 找回密码：v1 无自助找回。用户联系管理员，由管理员为其生成一次性重置链接（`/admin/users/{id}/reset-link`）转交。令牌随机 32 字节、库中只存哈希、邀请/重置链接有效期 24 小时、一次有效；重置成功后吊销该用户所有现有会话。
- 修改密码：需输入旧密码；成功后吊销其他会话。
- 邮箱变更：v1 由管理员修改（无邮件验证），同步调 `/user/update` 更新 LiteLLM 侧 `user_email`，并写审计。

**密码**
- 使用 argon2id 哈希（带每用户随机盐），不可逆，库中不存明文。
- 最小长度 10，不强制复杂字符组合；拒绝常见弱密码列表。

**会话**
- 服务端会话：登录后生成随机会话 ID，存 `sessions` 表（仅存哈希），通过 Cookie 下发：`HttpOnly; Secure; SameSite=Lax`。
- 空闲 24 小时过期，绝对有效期 30 天；支持主动登出与"登出所有设备"（吊销即时生效，这是不用无状态 JWT 的原因）。
- 状态变更类请求做 CSRF 防护（SameSite + CSRF token）。

**防暴力破解**
- 登录接口按 IP + 邮箱双维度限流；连续失败 5 次后该账户临时锁定 15 分钟（指数退避）。
- 登录失败统一返回"邮箱或密码错误"，不暴露邮箱是否存在。

**角色**
- `users.role`：`user` / `admin`。管理接口（`/admin/*`）要求 `admin`，与普通用户使用同一登录体系但独立授权。
- 被停服（余额耗尽或手动停服）或停用的用户：停服**只影响 LiteLLM 模型调用**，不影响登录，用户仍能登录查看账单并充值；仅账户 `disabled` 时禁止登录。

**接口**见第 5 节认证部分。

### 4.8 预付钱包与月底结算（最低月费）

**业务规则**
- 全部预付：用户充值进入**钱包**，余额不过期、滚存；只有这一类额度。
- 月内用量通过 LiteLLM 预算实时限额：用户可以用到钱包余额（扣除 `safety_reserve`，见 4.4）。
- **月底结算**：`本月扣费 = max(当月实际用量, 最低月费)`，从钱包扣除。用量超过最低月费按实际用量扣；不足则按最低月费扣（差额即保底）。
- 最低月费 `min_fee`：默认取全局配置，可对单个用户覆盖（`users.min_fee`）。结算时快照到 `billing_cycles.min_fee`。全局默认值 `default_min_fee` 是系统配置项（见 4.9），由管理员设置。
- **开通门槛**：钱包余额 ≥ 本周期最低月费 `min_fee_cycle`（且 > `safety_reserve`）才开通该周期的服务（周期开始时 `min_fee_cycle = min_fee`，月中开通按天折算，见下）。每月结算后若余额不足下个周期的 `min_fee`，**下个月不开通服务**（停服态）；充值达标后立即开通。
- 保底只对**开通过**的周期收取：周期内一直未达门槛的月份不收最低月费。月中开通的当月**按天折算**：`min_fee_cycle = min_fee × 剩余天数 / 当月总天数`，剩余天数从开通当日起算（含当日），按结算时区计算，金额按分四舍五入。开通只在**充值时**判定，不会因折算额随日期下降而自动开通。注意：月中开通者按折算额达标即可开通，但下个周期仍需余额 ≥ 完整 `min_fee`，月末余额可能不足而下月停服，界面和提醒应明确说明。
- 已开通周期内因用量耗尽而停服的用户，充值后立即恢复，无需再过门槛（保底已在本月承诺）。
- 周期：自然月，使用统一的结算时区（系统配置 `billing_timezone`，见 4.9），所有用户同一周期。

**为什么余额一定够扣**：开通要求钱包 ≥ `min_fee_cycle`；月内用量被预算限制在钱包之内。因此 `max(用量, min_fee_cycle) ≤ 钱包`，月底扣费不会出现余额不足的情况；唯一例外是并发超支，按 4.4 由平台承担并记录 `overdraft`。

**与 LiteLLM 的映射：单一计数器 + 每月重算上限**

LiteLLM 每个用户只有一个 `max_budget`、一个 `spend`。我们让 `spend` 永远只增、不重置，用 `max_budget` 表达"钱包还能花多少"：

```
周期开始：
  spend_start  = 上一周期的 spend_end（首个周期：开户时读取的 LiteLLM spend）
  wallet_start = 上一周期的 wallet_end（首个周期：0）
  开通判定：wallet_start ≥ min_fee 且 wallet_start > safety_reserve → activated_at = 周期开始，min_fee_cycle = min_fee；否则未开通
  limit        = spend_start + wallet_start + topups_total
  max_budget   = limit − safety_reserve          -- 已开通且未手动停服
               = 实时 spend                      -- 未开通 / manual_hold / 剩余 ≤ safety_reserve（停服态）

周期内（每次变更都重算，不做增量累加）：
  充值 T：   topups_total += T；若本周期未开通，重新判定开通条件：
               min_fee_cycle = min_fee × 剩余天数 / 当月总天数（剩余天数含充值当日）
               wallet_start + topups_total ≥ min_fee_cycle 且 > safety_reserve → 置 activated_at；重算 limit / max_budget
  余额       = limit − spend
  预计月底扣费 = max(spend − spend_start, min_fee_cycle)     -- 已开通周期

周期结束结算：
  spend_end    = 结算时读取的 LiteLLM spend（写入周期记录后不再改）
  usage        = spend_end − spend_start
  charge       = max(usage, min_fee_cycle)  若本周期已开通，否则 usage
  wallet_end   = max(0, wallet_start + topups_total − charge)
  overdraft    = max(0, charge − (wallet_start + topups_total))   -- 并发超支：平台承担，不追缴，记录并告警（见 4.4）
  下一周期：spend_start' = spend_end，wallet_start' = wallet_end
```

**数值示例**（`min_fee = $50`，`safety_reserve` 忽略不计）：
| 情形 | 月初钱包 | 当月用量 | 扣费 | 月末钱包 | 下月 |
|---|---|---|---|---|---|
| 用得少 | 120 | 20 | 50（保底） | 70 | 开通（70 ≥ 50） |
| 用得多 | 120 | 80 | 80 | 40 | **不开通**（40 < 50），充值到 ≥ 50 后开通 |
| 用光 | 120 | 120 | 120 | 0 | 不开通 |
| 月中开通（30 天的月，16 日充值，剩 15 天，`min_fee_cycle = 25`） | 30（未开通）→ 充 40 = 70 | 10 | 25（按天折算的保底） | 45 | 不开通（45 < 50） |

**基线说明**：`spend_end` 是唯一权威基线。写入新 `max_budget` 之前若有请求在途或任务失败重试，实时 `spend` 会略大于 `spend_end`，这部分差额因 `spend_start' = spend_end` 自然计入**新周期**用量，不会丢失，也不需要额外补偿。

**异常处理（防 `spend` 被改动）**：每次重算 `max_budget` 前读取实时 `spend`，若 `spend < spend_start`（被重置或手动编辑），**停止重算、写停服态、告警**，由管理员处理后再恢复（宁可停服也不下发错误的额度）。开户时显式传 `budget_duration` 为空，且不设置会产生周期重置的默认用户参数。

**按用户串行化**：充值应用、月度结算、对账任务对同一用户写 LiteLLM 前必须获取该用户的锁（数据库行锁或 advisory lock）。`topups.cycle_month` 在**应用时**（锁内）确定，取当时的开启周期。

**为什么不重置 `spend`**
- 避免 `/user/update` 清零与进行中请求之间的竞态：若结算时有请求在途，其费用会落入下一个周期，只造成月度归属的微小偏移，总额不丢失。
- `spend` 终身累加，可与 daily activity 互相校验。
- 月度用量展示仍使用 `/user/daily/activity`（见 4.3）。

**每月任务（幂等）**
1. 在周期边界执行；以 `billing_cycles (user_id, month)` 为幂等键，重复执行不重复结算。
2. 结算：读取 `spend_end` 并**一次性写入**周期记录，之后只用该记录计算 `usage`、`charge`、`wallet_end`。
3. 开启新周期：写入新记录，按开通判定设置 `activated_at`，调 `/user/update` 设置新 `max_budget`（未达门槛则写停服态）；写入后回读 `/v2/user/info` 校验 `max_budget`，不一致则重试并告警。对未达门槛的用户写入站内通知"余额不足最低月费，请充值"。
4. **任务延迟的影响**：窗口内 `max_budget` 仍是上月值——余额已耗尽的用户继续被拒；未达新门槛的用户仍可在旧上限内使用，这部分用量计入旧周期并在结算时扣费，以其钱包余额为上界，v1 接受。任务应在周期边界后立即运行，记录 `closed_at` 并监控延迟（超过阈值如 5 分钟告警）。

**月底前提醒（若不充值，下月将停服）**
- 目的：在周期结束前告知用户，如果不充值，下个月会因钱包余额不足最低月费而不开通服务。
- 提醒任务每日运行，按结算时区计算"距周期结束还有几天"。对**已开通**的用户计算（假设此后不再产生用量）：
  `projected_wallet_end = wallet_start + topups_total − max(usage_so_far, min_fee_cycle)`
- 若 `projected_wallet_end < min_fee`（下个周期的完整最低月费），则生成**站内提醒**（v1 不发邮件，见 4.7），内容包括：当前余额、本月已用、预计月底扣费、预计月末余额、**至少需充值** `min_fee − projected_wallet_end`（此后用量越多，所需越多），以及"不充值则下月停服"的明确说明。
- 提醒时间点由配置 `reminder_days_before_end` 决定，默认 `[7, 1]`（距月底 7 天、1 天各一次）。以 `notifications (user_id, type, cycle_month, variant)` 唯一键保证同一周期同一时间点只发一次；用户充值后条件不再满足则不再发送。
- 展示：用户登录后在概览页显示醒目横幅与所需充值额，并写入站内通知（`GET /api/me/notifications`）。**局限**：v1 没有邮件，用户若不登录就看不到提醒；后续可增加邮件或其他渠道（`notifications.channel` 已预留）。
- 兜底：月底结算后若余额仍不足，下个月按"开通门槛"不开通服务（停服态），并写入"下月已停服，充值达标后立即恢复"的站内通知（每月任务第 3 步）。

**v1 支付**：充值由管理员确认到账后手动录入（`/admin/users/{id}/topup`）；接入 Stripe 后改为回调自动入账（M4）。

### 4.9 系统配置项

以下参数放在系统配置里（存于 `system_config` 表，管理员可通过 `/admin/config` 修改，变更写审计），不写死在代码中：

| 配置项 | 说明 | 取值 |
|---|---|---|
| `billing_timezone` | 结算时区（IANA 名称，如 `Asia/Shanghai`、`UTC`）。决定周期边界、月度聚合和提醒的"距月底天数"。**修改后从下一个周期起生效**，避免打乱进行中的周期 | 待配置 |
| `default_min_fee` | 全局默认最低月费，可被 `users.min_fee` 覆盖 | 由管理员配置 |
| `default_rpm_limit` / `default_tpm_limit` | 每用户每分钟请求数 / token 数上限（可选的额外防护） | v1 不设置（留空） |
| `max_parallel_requests` | 每用户并发上限 | 2 |
| `safety_reserve_mode` | `auto`：按 LiteLLM 模型元数据自动计算；`fixed`：管理员指定固定金额 | 默认 `auto` |
| `reminder_days_before_end` | 月底前提醒的时间点（距月底天数） | `[7, 1]` |

**开放模型以 LiteLLM 为准**：AIMeter 不单独维护模型列表，用户可用的模型就是 LiteLLM 中配置并开放给该用户的模型。

**`safety_reserve` 自动计算（`auto`）**：后端定时（如每天，以及启动时）调用 `GET /model_group/info`，读取各模型组的 `max_input_tokens`、`max_output_tokens`、`input_cost_per_token`、`output_cost_per_token`，按 4.4 公式取最大值并乘以并发上限，得到全局 `safety_reserve`。注意：
- 不设 `max_tokens_cap`，输出按各模型的 `max_output_tokens` 计，最大输出很长的模型同样会抬高最坏成本。超长上下文模型也会使最坏成本很大（整个上下文窗口按输入计价），可能让保留额远高于实际常见请求。此时可改用 `fixed` 模式，或缩小开放模型范围；因残余超支会记入 `overdraft`，可按实际观测调整。
- 推理 token、缓存读写、分级定价（`tiered_pricing`）可能使实际成本高于线性估算，上线后以 `overdraft` 为依据校准。
- 模型列表或价格变化时 `safety_reserve` 随之变化，下发的 `max_budget` 在下次重算时生效。

**时区与 LiteLLM 的差异**：`/user/daily/activity` 的 `timezone` 参数是整数分钟偏移，不是时区名。若配置了有夏令时的时区，同一个月内偏移可能变化，需分段查询；建议配置无夏令时的时区。`timezone` 参数的确切语义纳入 M0 验证。

**币种（美元）**
- 全部金额为**美元（USD）**，界面以 `$` 显示，v1 仅此一种币种。钱包、充值、最低月费、月度账单与 LiteLLM 的 `spend`、`max_budget` 同币种，不做汇率换算。
- LiteLLM 的 `spend` 以其模型单价计，预期为 USD（M0 核实）。
- 精度：数据库金额使用 `NUMERIC`，统一四舍五入到分（见 2.1）。

## 5. AIMeter 后端 API（对外）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/login` | 邮箱密码登录，下发会话 Cookie |
| POST | `/api/auth/logout` | 登出当前会话 |
| POST | `/api/auth/logout-all` | 登出所有设备 |
| POST | `/api/auth/password/set` | 凭邀请/重置令牌设置密码 |
| POST | `/api/me/password` | 修改密码（需旧密码） |
| GET | `/api/me` | 当前用户资料与角色 |
| GET | `/api/me/balance` | 余额、已用、状态 |
| GET | `/api/me/usage/monthly?from=&to=` | 月度汇总 |
| GET | `/api/me/usage/daily?month=YYYY-MM` | 某月按日明细（含模型拆分） |
| GET | `/api/me/keys` | 我的 key 列表（脱敏） |
| POST | `/api/me/keys` | 新建 key |
| DELETE | `/api/me/keys/{id}` | 删除 key |
| GET | `/api/me/topups` | 充值记录 |
| GET | `/api/me/statements` | 月度账单：用量、最低月费、实际扣费、期末余额 |
| GET | `/api/me/notifications` | 站内提醒（月底前提醒、下月停服通知） |
| PUT | `/admin/users/{id}/min-fee` | 设置用户最低月费（空则用全局默认） |
| POST | `/admin/users/{id}/reset-link` | 生成一次性重置密码链接，由管理员转交用户 |
| GET | `/admin/config` | 读取系统配置（4.9） |
| PUT | `/admin/config/{key}` | 修改系统配置项（写审计） |
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
| 模型列表与单价（计算 `safety_reserve`） | `GET /model_group/info` |
| 终端客户模式（备选） | `/customer/new`、`/customer/block`、`/customer/daily/activity` |

备选：若用户并不登录 LiteLLM、只通过请求里的 `user` 字段标识，可使用 Customer（End User）模型。本设计选 Internal User 是因为需要为每个用户发独立 key。

## 7. 数据模型（AIMeter DB）

```
users(id  -- 同时作为 LiteLLM user_id,
      email UNIQUE, password_hash, role[user|admin],
      status[invited|active|disabled], manual_hold, min_fee  -- 空=全局默认,
      failed_logins, locked_until, created_at)
sessions(id_hash PK, user_id, created_at, last_seen_at, expires_at, ip, user_agent, revoked_at)
auth_tokens(token_hash PK, user_id, purpose[invite|reset], expires_at, used_at)
topups(id, user_id, order_id UNIQUE, amount, currency, status[pending|applied|failed],
       source[manual|stripe], cycle_month  -- 应用时在用户锁内确定, created_at, applied_at)
monthly_usage(user_id, month, spend, prompt_tokens, completion_tokens,
              requests, breakdown_json, finalized_at,  PRIMARY KEY(user_id, month))
audit_log(id, actor, action, target_user, detail_json, created_at)
billing_cycles(user_id, month, spend_start, spend_end, min_fee  -- 本周期快照,
               min_fee_cycle  -- 按天折算后的保底额,
               activated_at  -- 空=本周期未开通, wallet_start, topups_total,
               usage, charge, wallet_end, overdraft, status[open|closed], closed_at,
               PRIMARY KEY(user_id, month))
notifications(id, user_id, type[month_end_reminder|next_month_stopped|...],
              channel[in_app|email]  -- v1 仅 in_app, cycle_month,
              variant  -- 如提醒时间点, created_at, read_at,
              UNIQUE(user_id, type, cycle_month, variant, channel))
system_config(key PK, value_json, updated_by, updated_at)
```

（`topups` 即钱包充值；`applied` 的充值记入当期 `topups_total`。月度扣费记录在 `billing_cycles.charge`。）

LiteLLM 的 `max_budget` 是账本的投影：始终由当期 `billing_cycles` 行重算（`spend_start + wallet_start + topups_total − safety_reserve`，停服态则取实时 `spend`），不做增量累加。

## 8. 安全

- master key 仅存于后端环境变量/密钥管理，不入库、不下发（详见 8.1）。
- 后端到 LiteLLM 走内网；LiteLLM 管理接口不对公网暴露，仅公开数据面（`/v1/*`）。
- 所有 `/api/me/*` 做会话鉴权 + 用户隔离；管理接口独立角色。认证细节见 4.7（argon2id、服务端会话、登录限流）。
- 新建 key 明文只返回一次。
- 审计所有额度变更与停服动作。

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
| 并发超支 | spend 在请求完成后记账，高并发会小幅超出 `max_budget`，无法做到绝对零透支 | 每用户并发 2 + `safety_reserve`（按模型最大输入/输出计算）；对账任务收敛；残余超支由平台承担、记 `overdraft` 并告警（见 4.4）；v2 可加 pre-call 钩子 |
| 用户级并发限制是否覆盖所有 key | `max_parallel_requests` 若只在 key 级生效，用户多发 key 即可绕过"并发 2"，保留额随之失效 | M0 验证用户级限制；若不生效，限制每用户 key 数量或改在网关层做限制 |
| 用户级 `blocked` 未实现 | 规范标注 `[Not Implemented Yet]`，且 `/v2/user/info` 不返回 | 不使用；停服统一用 `max_budget ≤ spend`；M0 验证边界判定 |
| BETA 接口变动 | `daily/activity` 标注 BETA | 锁定 LiteLLM 版本；封装适配层；升级前跑契约测试 |
| 企业版限制 | 部分报表/SSO 可能需 Enterprise 许可 | 实现前逐项核对许可证；核心路径只用开源接口 |
| 用户级预算对多 key 的行为 | 需确认用户 `max_budget` 是否约束其所有 key | **实现第一步在测试实例验证** |
| `/v2/user/info` 返回字段 | 本文未核对响应字段细节 | 对照实际响应落地 DTO |
| 时区与月边界 | `/user/daily/activity` 的 `timezone` 是整数分钟偏移（符号同 `getTimezoneOffset`），无法表达夏令时 | 统一结算时区（建议无夏令时的固定偏移）；自行计算月边界；固化已结束月份；结算周期用同一时区 |
| 月度结算任务失败/延迟 | 任务未执行时，新周期的 `max_budget` 未更新：余额已耗尽的用户继续被拒；余额不足下月最低月费的用户仍可在窗口内用完旧上限内的额度（以其钱包余额为界） | 任务幂等、边界后立即运行、失败告警；可手动重跑 |
| 结算读取 `spend` 的边界误差 | 在途请求的费用落入下一周期 | `spend` 不重置，总额不丢，仅月度归属微小偏移；接受并在文档中说明 |
| 预算变更传播延迟 | 多实例且无 Redis 时，其他实例最多约 60 秒仍按旧预算拦截（来自 LiteLLM 自助文档，待验证） | 部署 Redis；充值/新周期后在界面提示可能有短暂延迟 |
| `spend` 是否被意外重置 | 方案依赖 `spend` 只增；若配置了 `budget_duration` 或默认用户参数会破坏 | 不设置 `budget_duration`、`max_internal_user_budget` 的周期；M0 验证；对账任务检测 `spend` 回退并告警 |

## 10. 里程碑

1. **M0 验证（1–2 天）**：起 LiteLLM 测试实例，验证预算超限拒绝、用户级预算对多 key、daily activity 返回结构；验证"`max_budget` 设为当前 `spend`"能否拒绝请求（`>=` 边界）、恢复后是否立即生效；验证受限管理 key（`allowed_routes`）能否替代 master key 完成后端所需的全部管理调用；验证不设 `budget_duration` 时 `spend` 只增不减、`/user/update` 调整 `max_budget` 不影响 `spend` 且立即生效；验证多实例下预算变更传播延迟；验证用户级 `max_parallel_requests=2` 是否对该用户所有 key 生效；测量请求完成后 `spend` 的记账延迟（决定是否需要启用 `rpm_limit`/`tpm_limit`）；核对 `/model_group/info` 返回的 `max_input_tokens`/`max_output_tokens`/单价与实际开放模型一致，并估算 `safety_reserve` 的实际金额是否可接受；核实 LiteLLM `spend`/模型单价的币种（预期为 USD）。
2. **M1 后端核心**：自建认证（登录、会话、邀请/重置密码）、开户（含首个周期记录）、余额、月度用量、手动充值（钱包）、周期记录与月底结算任务（含最低月费与开通门槛）、月底前站内提醒任务、系统配置、对账停服任务。
3. **M2 AIMeter 前端（用户自服务界面）**：概览（余额、预计月底扣费）、用量图、key 管理、充值记录、月度账单。
4. **M3 加固**：低余额阈值通知、邮件等其他通知渠道（含自助找回密码）、审计、监控告警、限流。
5. **M4 支付接入**：Stripe 等，回调幂等。

## 11. 待决策

- ~~技术栈~~：后端已决定 FastAPI + PostgreSQL（仅支持 PostgreSQL，不支持 SQLite，见 2.1）。仍待定：前端框架（React / Vue）。
- ~~用户认证方式~~：已决定自建邮箱密码（见 4.7）；**v1 不做邮件发送**：账号由管理员开户，邀请/重置通过管理员转交一次性链接，不开放自助注册。仍待定：后续是否增加邮件渠道与自助注册；是否追加 OIDC 登录。
- ~~余额单位~~：已决定为美元（USD），界面以 `$` 显示，与 LiteLLM 同币种，不做汇率换算。
- ~~是否允许小额透支~~：已决定**不允许**（钱包不记负数、不向用户追欠款）；因 LiteLLM 事后记账无法绝对杜绝，残余超支由平台承担（见 4.4）。已决定每用户并发限制为 2；开放模型以 LiteLLM 为准，`safety_reserve` 由 LiteLLM 模型元数据自动计算（见 4.4、4.9）；**不设 `max_tokens_cap`**（输出按各模型 `max_output_tokens` 计）；**v1 不设置 `rpm_limit`/`tpm_limit`**。
- ~~结算时区~~：已决定放在系统配置 `billing_timezone`（见 4.9），具体取值待配置，建议无夏令时的时区。
- ~~月底前提醒~~：已决定要提醒（v1 仅站内提醒，不发邮件），余额不足最低月费的用户下个月停服（见 4.8）。提醒时间点已确定为 `[7, 1]`；仍待定：是否增加其他提醒渠道。
- ~~最低月费默认金额~~：已决定为系统配置项 `default_min_fee`（见 4.9），金额由管理员设置。
- ~~月中开通~~：已决定按天折算（见 4.8）。

## 12. 评审待办（尚未处理）

来自一次独立评审，以下问题已确认但尚未写入正文设计：

- 金额精度：LiteLLM 的 `max_budget`/`spend` 是浮点，下发时需统一舍入规则，并避免浮点误差影响 `spend >= max_budget` 的判定。
- 用户权限：`internal_user` 可通过数据面端口调用 `/key/*` 自建带预算的 key，需在网关限制路由或用 `allowed_routes`；`/key/regenerate` 规范中存在，应补入 key 轮换与 API 表。
- `billing_cycles` 增加同步状态字段（`synced_at`、最后下发的 `max_budget`、任务 run id）；`/user/new` 与 `/key/generate` 中途失败的补偿。
- 钱包充值缺冲正/退款状态（Stripe 拒付）。
- API 表缺：会话列表、管理员查看月度账单；邮件上线后补：邀请重发、自助找回密码、改邮箱验证。
- 安全：`DELETE /api/me/keys/{id}` 的归属校验（IDOR）与 key 数量上限；管理员充值/调整需二次验证；登录锁定可被用于恶意锁号；`audit_log` 与 `topups`/`billing_cycles` 的关联。
- 风险表中 `max_internal_user_budget` 周期的表述在规范中无法核实，需重新措辞。
