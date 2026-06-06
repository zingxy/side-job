# 支付与订单系统 HTTP 接口说明

## 架构说明

本系统由两层服务组成：

- **C++ 后端**（端口 8080）：业务逻辑层，所有前端请求都调到这里（路径 `/api/{action}`）
- **Go 支付服务**（端口 9090）：微信 API 对接层，负责签名/验签/AES 解密，由 C++ 内部调用

**前端开发者只需要知道**：所有接口都是 `POST /api/{action}`，调 C++ 的 8080 端口。Go 服务不对外暴露接口。

## HTTP 调用协议

- **协议**：HTTP（开发环境）/ HTTPS（生产环境）
- **域名与端口**：`http://{server}:8080`（C++ 服务端口，生产环境由 Nginx 反代到 443）
- **请求方法**：全部 `POST`
- **请求路径**：`/api/{action}`，如 `/api/payCreate`、`/api/payOrderQuery`
- **Content-Type**：`application/json`
- **请求体**：JSON 对象，字段见各接口说明
- **响应体**：统一 JSON 格式 `{ "code": 200, "msg": "success", "data": {...} }`
- **认证**：当前接口无鉴权，前端通过传入 `customer_id`/`operator_id` 等标识区分操作人（生产环境应加 JWT 或 Session 鉴权）

**调用示例（curl）：**

```bash
curl -X POST http://localhost:8080/api/payCreate \
  -H "Content-Type: application/json" \
  -d '{"customer_id":1,"course_id":10,"slot_quantity":1,"amount":9900}'
```

**调用示例（JavaScript fetch）：**

```js
const res = await fetch('http://localhost:8080/api/payCreate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ customer_id: 1, course_id: 10, slot_quantity: 1, amount: 9900 }),
});
const data = await res.json();
```

## 通用约定

- **请求方式**：所有接口均为 `POST`
- **请求路径**：`/api/{action}`，如 `/api/payCreate`
- **请求体**：`application/json`
- **响应格式**：统一 JSON 结构

```json
{
  "code": 200,
  "msg": "success",
  "data": { ... }
}
```

- **code**：`200` 成功，`400` 参数错误，`404` 未找到，`500` 服务器错误，`502` 下游服务不可用
- **data**：业务数据，失败时可能为空或包含部分错误上下文

---

## 分页与筛选（所有列表接口通用）

所有列表类接口（`*List`、`*AllList`）均支持以下公共参数：

### 分页参数

| 字段 | 类型 | 默认值 | 范围 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| page | int | 1 | ≥1 | 第几页 |
| page_size | int | 20 | 1~200 | 每页条数，最大 200 |

### 分页响应

列表接口返回的 `data` 中统一包含分页元数据：

```json
{
  "code": 200,
  "data": {
    "list": [...],
    "total": 156,
    "page": 2,
    "page_size": 20,
    "total_pages": 8
  }
}
```

不传 `page`/`page_size` 时使用默认值，向后兼容。

### 各接口筛选参数一览

| 接口 | 筛选参数 | 新增（★） |
| :--- | :--- | :--- |
| **payOrderAllList** | `start_time`, `end_time`, `status`★, `source`★, `customer_id`★, `course_id`★, `created_by_staff_id`★ | |
| **payStaffPerformance** | `created_by_staff_id`★, `start_time`★, `end_time`★ | 纯统计，0=查全部业务员，按人分组返回 |
| **payOrderList** | `customer_id`(必填), `status`★, `start_time`★, `end_time`★, `created_by_staff_id`★ | |
| **payRefundAllList** | `start_time`, `end_time`, `status`★, `source`★, `applicant_customer_id`★ | |
| **payRefundPendingList** | `start_time`★, `end_time`★ | 之前全表扫描无筛选 |
| **payRefundList** | `order_no`(必填), `status`★, `start_time`★, `end_time`★ | |
| **payOperationLogAll** | `start_time`, `end_time`, `action_type`★, `operator_id`★, `order_no`★ | |
| **payOperationLogList** | `order_no`(必填), `action_type`★ | |
| **payCourseAccountList** | `customer_id`(必填), `course_id` | 已有筛选 |
| **payCourseAccountTransactions** | `customer_id`, `course_id`, `start_time`, `end_time` | 已有筛选 |
| **payMySlots** | `customer_id`(必填), `course_id` | 新增分页 |
| **payGiftList** | `giver_id`(必填), `course_id`, `status`★, `start_time`★, `end_time`★ | |

> ★ 表示本次新增的参数。所有筛选参数均为**选填**，不传 = 查全部，向后兼容。

---

## 一、支付流程

### 1.1 payCreate — 小程序创建订单（统一下单）

**使用场景：** 小程序端客户选择课程后，点击"立即支付"触发此接口。支付层只关注课程充值，`course_id` 必填。财务核销/支付成功后自动累加课程账户名额。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 下单客户 ID |
| amount | int | 是 | 总金额（分） |
| course_id | int | 是 | 课程 ID |
| body | string | 否 | 商品描述，默认"课程订单" |
| open_id | string | 否 | 用户微信 OpenID |
| source | string | 否 | 订单来源，默认 `mini_program` |
| slot_quantity | int | 否 | 名额数量，默认 1 |
| created_by_staff_id | int | 否 | 绑定的业务员 ID，用于业绩归属。不传默认 0（未绑定） |
| recharge_type | string | 否 | 充值类型，后端自动判断：有历史充值记录 → `additional_recharge`，否则 → `recharge` |
| payment_method | string | 否 | 充值方式，后端自动设为 `wechat` |

**注意：** `recharge_type` 和 `payment_method` 由后端自动填充，前端无需传入。小程序端仅区分 `recharge`（充值）和 `additional_recharge`（追加充值）。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_no | string | 订单号（如 `ORD20260519123456123456`） |
| order_id | int | 订单自增 ID |
| payment_params | object | 微信支付调起参数（见下表） |

**payment_params 字段：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| appId | string | 小程序 AppID |
| timeStamp | string | 时间戳（秒） |
| nonceStr | string | 随机字符串 |
| package | string | 预支付交易会话标识，格式 `prepay_id=...` |
| signType | string | 签名算法，固定 `RSA` |
| paySign | string | 对以上参数的签名 |

**示例：**

```json
// 请求 POST /api/payCreate
{
  "customer_id": 1,
  "course_id": 10,
  "slot_quantity": 5,
  "amount": 50000,
  "body": "课程充值"
}

// 请求 POST /api/payCreate（单名额）
{
  "customer_id": 1,
  "course_id": 10,
  "slot_quantity": 1,
  "amount": 9900,
  "body": "测试课程"
}

// 响应
{
  "code": 200,
  "msg": "success",
  "data": {
    "order_no": "ORD20260519123456123456",
    "order_id": 1,
    "payment_params": {
      "appId": "wx1234567890abcdef",
      "timeStamp": "1684923456",
      "nonceStr": "a1b2c3d4e5f6",
      "package": "prepay_id=wx20260519123456abcdef",
      "signType": "RSA",
      "paySign": "MIGfMA0GCSqGSIb3DQEBAQUAA4..."
    }
  }
}
```

---

### 1.2 payNotify — 微信支付回调（内部接口）

**使用场景：** 用户完成支付后，微信服务器异步通知支付结果。此接口由 **Nginx 反向代理直接路由到 Go 支付服务**（不经过 C++），Go 负责验签和解密后再通知 C++ 更新订单状态。**前端不需要调用此接口。**

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| out_trade_no | string | 是 | 订单号 |
| trade_state | string | 是 | 支付状态：`SUCCESS` / `FAIL` |
| paid_at | string | 否 | 到账时间，不传则用当前时间 |

**响应：**

- 成功返回 `{ "code": 200, "msg": "success" }`
- 已处理的订单幂等直接返回成功（同一订单重复回调不会重复更新）
- `trade_state=SUCCESS` 时：订单 `status → paid`，小程序订单 `verification_status → auto_verified`，支付流水 `status → success`
- `trade_state` 非成功时：订单 `status → closed`，支付流水 `status → failed`，写入操作日志 `pay_failed`

**补充说明：**
- 如果回调丢失（网络异常等），定时查单任务（`startOrderCheckTimer`）会每隔 5 分钟兜底扫描超 30 分钟仍为 `pending_payment` 的订单，主动调微信查单 API 补偿。
- 并发回调保护：数据库层使用 `WHERE status='pending_payment'` 原子更新，防止同一订单被并发回调重复处理。

---

## 二、订单查询

### 2.1 payOrderQuery — 订单详情查询

**使用场景：**
- 前端支付完成后跳转到订单详情页查看支付结果
- 业务员后台查询某个订单的详细信息
- 退款申请前先查订单确认状态和金额

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 订单自增 ID |
| order_no | string | 订单号 |
| customer_id | int | 客户 ID |
| course_id | int | 课程 ID |
| course_account_id | int | 关联课程账户 ID（充值订单核销后写入） |
| type | string | 订单类型：`purchase`（购买）/ `recharge`（充值） |
| source | string | 订单来源：`mini_program`（小程序）/ `offline`（线下代录） |
| slot_quantity | int | 名额数量 |
| unit_price_amount | int | 单价（分） |
| total_amount | int | 总金额（分） |
| status | string | 支付状态（见下方枚举） |
| verification_status | string | 核销状态（见下方枚举） |
| payment_no | string | 支付流水号（小程序订单查 payment_records，线下订单取 external_payment_no） |
| external_payment_no | string | 线下支付单号（线下订单才有） |
| payment_proof_url | string | 付款凭证图片 URL |
| refund_amount | int | 已退金额（分），状态为 completed 的退款累加 |
| refundable_amount | int | 可退余额（分），= total_amount - refund_amount |
| has_pending_refund | bool | 是否有进行中（pending 状态）的退款 |
| payment_deadline_at | string | 支付截止时间 |
| paid_at | string | 支付时间 |
| verified_at | string | 核销时间 |
| verified_by | int | 核销人 ID（0 表示未指定） |
| reviewed_at | string | 财务审核时间（通过/拒绝都记录） |
| rejected_reason | string | 拒绝原因 |
| remark | string | 业务员备注（供财务审核参考） |
| recharge_type | string | 充值类型（见下方枚举） |
| payment_method | string | 充值方式：`wechat` / `alipay` / `bank_card` |
| created_by_staff_id | int | 业务员 ID（线下订单创建人） |
| deleted_at | string | 软删除时间（不为空表示已删除） |
| created_at | string | 创建时间 |
| updated_at | string | 更新时间 |

**示例：**

```json
// 请求 POST /api/payOrderQuery
{ "order_no": "ORD20260519123456123456" }

// 响应
{
  "code": 200,
  "msg": "success",
  "data": {
    "id": 1,
    "order_no": "ORD20260519123456123456",
    "customer_id": 1,
    "type": "purchase",
    "source": "mini_program",
    "total_amount": 9900,
    "status": "paid",
    "verification_status": "auto_verified",
    "payment_no": "PAY20260519123456654321",
    "refund_amount": 0,
    "refundable_amount": 9900,
    "has_pending_refund": false,
    "created_at": "2026-05-19 12:34:56"
  }
}
```

---

### 2.2 payOrderList — 按客户查询订单列表

**使用场景：**
- 小程序"我的订单"页面，展示客户所有订单
- 业务员后台查看某个客户的历史订单

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 客户 ID |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| status | string | 否 | 按订单状态筛选（见状态枚举） |
| start_time | string | 否 | 起始时间 |
| end_time | string | 否 | 结束时间 |
| recharge_type | string | 否 | 按充值类型筛选：`recharge` / `additional_recharge` / `retrain_recharge` / `qixin_recharge` |
| payment_method | string | 否 | 按充值方式筛选：`wechat` / `alipay` / `bank_card` |
| created_by_staff_id | int | 否 | 业务员 ID（筛选该业务员经办的订单） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 订单数组，每项字段同 `payOrderQuery` 的 data（含 `course_id`、`course_account_id`），额外包含： |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |
| refunds | array | 该订单的退款明细数组（仅当有退款记录时返回，见下方退款明细字段） |
| refund_amount | int | 已退总金额（分） |
| refundable_amount | int | 可退余额（分） |
| has_pending_refund | bool | 是否有进行中的退款 |

**退款明细字段（list[].refunds[]）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| refund_no | string | 退款单号 |
| status | string | 退款状态 |
| amount | int | 退款金额（分） |
| type | string | 退款类型：`full` / `partial` |
| channel | string | 退款渠道 |
| source | string | 退款来源：`customer` / `staff` |
| reason | string | 退款原因 |
| created_at | string | 申请时间 |
| handled_at | string | 审核时间 |
| rejected_reason | string | 拒绝原因 |

---

### 2.3 payOrderAllList — 全部订单查询

**使用场景：** 财务/管理员查看系统全部订单，支持时间范围筛选和分页。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| start_time | string | 否 | 起始时间，格式 `YYYY-MM-DD HH:MM:SS` |
| end_time | string | 否 | 结束时间，格式 `YYYY-MM-DD HH:MM:SS` |
| status | string | 否 | 按订单状态筛选（见状态枚举） |
| source | string | 否 | 按订单来源筛选（`mini_program` / `offline`） |
| recharge_type | string | 否 | 按充值类型筛选：`recharge` / `additional_recharge` / `retrain_recharge` / `qixin_recharge` |
| payment_method | string | 否 | 按充值方式筛选：`wechat` / `alipay` / `bank_card` |
| customer_id | int | 否 | 按客户 ID 筛选 |
| course_id | int | 否 | 按课程 ID 筛选 |
| created_by_staff_id | int | 否 | 业务员 ID（选填，不传=查全部业务员） |

- 同时传 `start_time` + `end_time`：查询该时间范围内的订单
- 只传 `start_time`：查询该时间之后的订单
- 只传 `end_time`：查询该时间之前的订单
- 都不传：返回最新订单（按 created_at 倒序），分页默认 20 条/页
- 传 `created_by_staff_id`：仅查该业务员绑定的订单，用于业绩统计

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 订单数组，字段同 `payOrderQuery` 的 data（含 `course_id`、`course_account_id`），额外包含 `payment_no`、`refund_amount`、`refundable_amount`、`has_pending_refund`，以及 `refunds` 退款明细数组（仅当有退款记录时返回，字段同 `payOrderList` 的 `refunds`） |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

---

### 2.4 payStaffPerformance — 业务员业绩查询

**使用场景：** 纯统计接口，查业务员业绩（总业绩、净业绩、退款额）。`created_by_staff_id=0` 时返回所有业务员的业绩（按人分组），`>0` 时返回指定业务员。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| created_by_staff_id | int | 否 | 业务员 ID，默认 0。0 = 查全部业务员，按人分组返回 |
| start_time | string | 否 | 起始时间，格式 `YYYY-MM-DD HH:MM:SS` |
| end_time | string | 否 | 结束时间，格式 `YYYY-MM-DD HH:MM:SS` |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list[].staff_id | int | 业务员 ID |
| list[].total_orders | int | 订单数 |
| list[].total_amount | int | 总业绩（分） |
| list[].total_refund_amount | int | 退款总额（分） |
| list[].net_amount | int | 净业绩 = total_amount - total_refund_amount（分） |
| summary | object | 所有业务员合计（字段同上） |

**示例请求：**

```json
// 查全部业务员业绩
POST /api/payStaffPerformance
{ "created_by_staff_id": 0 }

// 查业务员 100 在 2026 年 6 月的业绩
POST /api/payStaffPerformance
{
    "created_by_staff_id": 100,
    "start_time": "2026-06-01 00:00:00",
    "end_time": "2026-06-30 23:59:59"
}
```

**示例响应：**

```json
{
    "code": 200,
    "msg": "success",
    "data": {
        "list": [
            {
                "staff_id": 100,
                "total_orders": 15,
                "total_amount": 198000,
                "total_refund_amount": 19800,
                "net_amount": 178200
            },
            {
                "staff_id": 200,
                "total_orders": 3,
                "total_amount": 30000,
                "total_refund_amount": 5000,
                "net_amount": 25000
            }
        ],
        "summary": {
            "total_orders": 18,
            "total_amount": 228000,
            "total_refund_amount": 24800,
            "net_amount": 203200
        }
    }
}
```

---

## 三、线下订单

### 3.1 payOrderCreateOffline — 创建线下订单

**使用场景：** 业务员在线下收到客户转账/现金后，在后台代录充值订单。客户对指定课程预存名额，订单进入待审核，财务核销通过后自动累加 `customer_course_accounts` 课程账户名额。`course_id` 必填。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 客户 ID |
| course_id | int | 是 | 课程 ID |
| amount | int | 是 | 总金额（分） |
| slot_quantity | int | 否 | 名额数量，默认 1 |
| external_payment_no | string | 否 | 线下支付单号（转账流水号等） |
| payment_proof_url | string | 否 | 付款凭证图片 URL |
| created_by_staff_id | int | 否 | 业务员 ID |
| remark | string | 否 | 备注（供财务审核参考） |
| recharge_type | string | 否 | 充值类型：`recharge`（充值）/ `retrain_recharge`（复训充值）/ `additional_recharge`（追加充值）/ `qixin_recharge`（齐心会充值），业务员自由填写 |
| payment_method | string | 否 | 充值方式：`wechat`（微信）/ `alipay`（支付宝）/ `bank_card`（银行卡），业务员自由填写 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | int | 订单自增 ID |
| order_no | string | 订单号 |
| status | string | 固定 `paid` |

---

### 3.2 payOrderReview — 财务审核线下订单

**使用场景：** 财务人员登录后台，查看业务员提交的线下订单，核对转账凭证后决定通过或拒绝。

**校验规则：**
1. 订单已软删除（`deleted_at` 不为空）→ 拒绝
2. 仅线下订单（`source=offline`）且已支付（`status=paid`）可审核
3. 仅 `verification_status=pending_review` 的订单可审核（已审核/已拒绝的不可重复审核）

**审核通过时：**
- 所有订单：`verification_status → manual_verified`
- **有 course_id 的订单**：自动累加 `customer_course_accounts` 课程账户（ON DUPLICATE KEY UPDATE），写入 `course_account_transactions` 流水（type=recharge），回写 `course_account_id` 到订单

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| action | string | 是 | `approve`（通过）/ `reject`（拒绝） |
| verified_by | int | 是 | 审核人 ID |
| rejected_reason | string | 否 | 拒绝原因（action=reject 时建议填写） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_no | string | 订单号 |
| status | string | 订单支付状态 |
| verification_status | string | 核销状态：`manual_verified`（通过）/ `rejected`（拒绝） |

---

### 3.3 payOrderResubmit — 业务员重新提交被拒订单

**使用场景：** 财务拒绝某笔线下订单后（如凭证不清晰），业务员补充材料后重新提交审核。只有 `verification_status=rejected` 的订单可重新提交。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| operator_id | int | 是 | 操作人 ID（当前登录业务员 ID） |
| external_payment_no | string | 否 | 修改后的线下支付单号 |
| payment_proof_url | string | 否 | 修改后的付款凭证 URL |
| total_amount | int | 否 | 修改后的金额（仅被拒绝订单可修改） |
| slot_quantity | int | 否 | 修改后的名额数量（仅被拒绝订单可修改） |
| customer_id | int | 否 | 修改后的客户 ID（仅被拒绝订单可修改） |
| course_id | int | 否 | 修改后的课程 ID（仅被拒绝订单可修改） |
| remark | string | 否 | 修改后的备注 |

**注意：** 待审核订单（`pending_review`）不可修改金额/客户等核心字段，只允许修改备注、凭证。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_no | string | 订单号 |
| verification_status | string | 固定 `pending_review` |

---

### 3.4 payOrderClose — 关闭订单

**使用场景：** 手动关闭超期未支付的订单。

**校验规则：**
1. 订单已软删除（`deleted_at` 不为空）→ 拒绝
2. 仅 `pending_payment` 状态的订单可关闭

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| operator_id | int | 是 | 操作人 ID（当前登录用户 ID） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_no | string | 订单号 |
| status | string | 固定 `closed` |

---

### 3.5 payOrderDelete — 业务员软删除订单

**使用场景：** 业务员录入的线下订单有误，在审核通过前可以删除。仅线下订单可删除，且仅 `verification_status=pending_review`（待审核）或 `rejected`（审核拒绝）的状态允许删除。已核销/已退款的订单不可删除。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| operator_id | int | 是 | 操作人 ID（当前登录用户 ID） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_no | string | 订单号 |
| deleted_at | string | 软删除时间 |

---

## 四、退款流程

### 4.1 payRefund — 退款申请

**使用场景：**
- 小程序端：客户在订单详情页点击"申请退款"，前端判断名额可退性后提交
- 业务员后台：选中订单后点击"退款"，填写退款金额和原因提交

前端负责判断名额是否可退（签到状态、座位分配、转赠状态、开课时间等），后端只校验订单本身的状态和金额。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| applicant_customer_id | int | 是 | 申请人客户 ID（退款归属） |
| amount | int | 是 | 退款金额（分） |
| refund_slot_quantity | int | 是 | 退款名额数量 |
| reason | string | 否 | 退款原因 |
| source | string | 否 | 退款来源：`customer`（客户申请）/ `staff`（工作人员），默认 `customer` |
| operator_id | int | staff 时必填 | 操作人 ID，`source=staff` 时必填，`source=customer` 时可省略 |

**后端校验规则：**

1. 订单必须存在
2. 订单未被软删除（`deleted_at IS NULL`）
3. 订单状态为 `paid` / `verified` / `refunding` / `refunded`
4. 退款金额 ≤ 订单总金额
5. 退款金额 + 已完成退款总额 ≤ 订单总金额
6. 同一订单有 `pending` 状态的退款时不可重复申请（并发保护）
7. 同一订单有进行中的退款时，不允许重复申请（并发保护 + 幂等控制）
8. 线下订单（`source=offline`）只能由工作人员发起退款（`source=staff`）
9. **退款名额自动截断**：退款名额超过可退名额时自动截断为可退名额数，最低为 0，不返回错误。仅校验退款金额是否超过可退款金额

**退款类型自动判断：** 后端根据退款金额自动判定 `type`：
- 等于订单总金额 → `full`（全额退款）
- 小于订单总金额 → `partial`（部分退款）

审核通过时，`full` 退款将订单状态改为 `refunded`，`partial` 退款改为 `refunding`（可继续申请后续部分退款）。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| refund_no | string | 退款单号（如 `RF20260519123456123456`） |
| refund_id | int | 退款自增 ID |
| status | string | 固定 `pending` |
| type | string | 退款类型：`full`（全额）/ `partial`（部分），自动判断 |
| refund_slot_quantity | int | 实际退款名额数（自动截断后） |

**示例：**

```json
// 请求 POST /api/payRefund（客户申请，退 1 个名额）
{
  "order_no": "ORD20260519123456123456",
  "applicant_customer_id": 1,
  "amount": 9900,
  "refund_slot_quantity": 1,
  "reason": "个人原因申请退款",
  "source": "customer"
}

// 请求 POST /api/payRefund（工作人员代申请，退多个名额）
{
  "order_no": "ORD20260519123456123456",
  "applicant_customer_id": 1,
  "amount": 29700,
  "refund_slot_quantity": 3,
  "reason": "客户要求退款",
  "source": "staff",
  "operator_id": 1
}

// 响应
{
  "code": 200,
  "msg": "success",
  "data": {
    "refund_no": "RF20260519123456123456",
    "refund_id": 1,
    "status": "pending",
    "type": "full"
  }
}
```

---

### 4.2 payRefundReview — 退款审核

**使用场景：** 财务人员登录后台，查看待审核的退款申请，逐一审核。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| refund_no | string | 是 | 退款单号 |
| action | string | 是 | `approve`（通过）/ `reject`（拒绝） |
| handled_by | int | 否 | 审核人 ID |
| rejected_reason | string | 否 | 拒绝原因（action=reject 时建议填写） |

**审核通过时：**
- 小程序订单（`channel=wechat_original_route`）：先调微信退款 API，成功后更新退款单状态为 `completed`
- 线下订单（`channel=offline_manual`）：直接更新退款单状态为 `completed`
- 同时根据退款类型更新订单状态：`full` → `refunded`，`partial` → `refunding`

**审核拒绝时：** 更新退款单状态为 `rejected`，订单状态不变，前端可重新申请退款。

只有 `pending` 状态的退款可审核（幂等保护，已审核的不允许再次审核）。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| refund_no | string | 退款单号 |
| status | string | 更新后的退款状态：`completed` / `rejected` |
| order_status | string | 更新后的订单状态（仅 action=approve 时返回）：`refunded` / `refunding` |

---

## 五、退款查询

### 5.1 payRefundQuery — 退款详情查询

**使用场景：** 财务审核前查看退款申请的完整信息（金额、原因、申请人等），辅助审核决策。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| refund_no | string | 是 | 退款单号 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 退款自增 ID |
| refund_no | string | 退款单号 |
| order_id | int | 关联订单 ID |
| applicant_customer_id | int | 申请人客户 ID |
| source | string | 退款来源：`customer` / `staff` |
| type | string | 退款类型：`full` / `partial` |
| channel | string | 退款渠道：`wechat_original_route`（微信原路退回）/ `offline_manual`（线下人工退款） |
| status | string | 退款状态（见下方枚举） |
| amount | int | 退款金额（分） |
| refund_slot_quantity | int | 退款名额数量 |
| reason | string | 退款原因 |
| rejected_reason | string | 拒绝原因 |
| handled_by | int | 审核人 ID |
| handled_at | string | 审核时间 |
| created_at | string | 申请时间 |

---

### 5.2 payRefundList — 按订单查询退款记录

**使用场景：** 在订单详情页展示该订单的所有退款历史。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| status | string | 否 | 按退款状态筛选（pending / completed / rejected 等） |
| start_time | string | 否 | 起始时间 |
| end_time | string | 否 | 结束时间 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 退款记录数组，每条记录包含： |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**退款记录字段（list[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 退款自增 ID |
| refund_no | string | 退款单号 |
| order_id | int | 关联订单 ID |
| order_no | string | 关联订单号 |
| order_total_amount | int | 关联订单总金额（分） |
| order_status | string | 关联订单支付状态 |
| applicant_customer_id | int | 申请人客户 ID |
| source | string | 退款来源：`customer` / `staff` |
| type | string | 退款类型：`full` / `partial` |
| channel | string | 退款渠道 |
| status | string | 退款状态 |
| amount | int | 退款金额（分） |
| refund_slot_quantity | int | 退款名额数量 |
| reason | string | 退款原因 |
| rejected_reason | string | 拒绝原因 |
| handled_by | int | 审核人 ID |
| handled_at | string | 审核时间 |
| created_at | string | 申请时间 |
| updated_at | string | 更新时间 |

---

### 5.3 payRefundPendingList — 待审核退款列表（财务专用）

**使用场景：** 财务后台查看待处理退款申请。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| start_time | string | 否 | 起始时间 |
| end_time | string | 否 | 结束时间 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 待审核（status=`pending`）退款记录，字段： |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**退款记录字段（list[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 退款自增 ID |
| refund_no | string | 退款单号 |
| order_id | int | 关联订单 ID |
| applicant_customer_id | int | 申请人客户 ID |
| source | string | 退款来源：`customer` / `staff` |
| type | string | 退款类型：`full` / `partial` |
| channel | string | 退款渠道 |
| amount | int | 退款金额（分） |
| refund_slot_quantity | int | 退款名额数量 |
| reason | string | 退款原因 |
| created_at | string | 申请时间 |

---

### 5.4 payRefundAllList — 全部退款记录

**使用场景：** 财务或运营查看所有退款记录，支持时间范围、状态、来源、申请人筛选。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| start_time | string | 否 | 起始时间，格式 `YYYY-MM-DD HH:MM:SS` |
| end_time | string | 否 | 结束时间，格式 `YYYY-MM-DD HH:MM:SS` |
| status | string | 否 | 按退款状态筛选（pending / approved / completed / rejected） |
| source | string | 否 | 按退款来源筛选（customer / staff） |
| applicant_customer_id | int | 否 | 按申请人客户 ID 筛选 |

- 时间筛选规则同 `payOrderAllList`，都不传时返回最新退款记录，分页默认 20 条/页。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 退款记录数组，每条记录包含退款基础字段（同 `payRefundList`）以及关联订单信息： |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |
| order_no | string | 关联订单号 |
| order_total_amount | int | 关联订单总金额（分） |
| order_status | string | 关联订单支付状态 |

---

## 六、操作流水

### 6.1 payOperationLogList — 订单操作流水

**使用场景：** 订单详情页底部展示该订单的完整操作历史，用于追溯。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| order_no | string | 是 | 订单号 |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| action_type | string | 否 | 按操作类型筛选（approve / reject / refund_apply 等） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 操作记录数组 |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**操作记录字段：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 流水自增 ID |
| order_no | string | 订单号 |
| operator_id | int | 操作人 ID |
| operator_role | string | 操作人角色：`sales`（业务员）/ `finance`（财务）/ `customer`（客户） |
| action_type | string | 操作类型（见下方枚举） |
| from_status | string | 变更前状态 |
| to_status | string | 变更后状态 |
| reason | string | 操作原因/拒绝原因 |
| proof_snapshot | string | 凭证快照（JSON） |
| created_at | string | 操作时间 |

---

### 6.2 payOperationLogAll — 全部操作流水

**使用场景：** 管理员/运营查看所有订单的操作流水，全局审计追踪。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| start_time | string | 否 | 起始时间，格式 `YYYY-MM-DD HH:MM:SS` |
| end_time | string | 否 | 结束时间，格式 `YYYY-MM-DD HH:MM:SS` |
| action_type | string | 否 | 按操作类型筛选（approve / reject / refund_apply 等） |
| operator_id | int | 否 | 按操作人 ID 筛选 |
| order_no | string | 否 | 按订单号筛选 |

- 时间筛选规则同 `payOrderAllList`，都不传时返回最新操作记录，分页默认 20 条/页。

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 全部操作记录数组，字段同 `payOperationLogList` |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

---

## 七、其他

### 7.1 payHealth — Go 支付服务健康检查

**使用场景：** 运维或启动时检查 Go 支付服务（负责微信 API 调用）是否正常。

**请求参数：** 无

**响应：**

```json
{
  "code": 200,
  "msg": "Go pay service is alive",
  "data": { ... }
}
```

Go 服务不可达时返回 `502`。

---

## 八、课程账户管理

充值订单财务核销通过后，自动累加学员的课程账户名额。以下为课程账户相关接口。

### 8.1 payCourseAccountList — 查询学员课程账户列表

**使用场景：** 前端报名页展示学员在某课程下的可用账户，确认有多少名额可用于报名。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 学员 ID |
| course_id | int | 否 | 课程 ID（传则只返回该课程账户，不传返回全部） |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 课程账户数组，每项包含： |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**账户记录字段（list[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 课程账户 ID |
| customer_id | int | 学员 ID |
| course_id | int | 课程 ID |
| total_recharge_amount | decimal | 累计充值金额 |
| consumed_amount | decimal | 已消耗金额 |
| refunded_amount | decimal | 已退款金额 |
| total_slots | int | 累计充值获得名额数 |
| consumed_slots | int | 已消耗名额数 |
| refunded_slots | int | 已退款名额数 |
| total_gifted_slots | int | 累计被转赠名额数（别人转给自己的） |
| consumed_gifted_slots | int | 已消耗的转赠名额数 |
| available_slots | int | 可用名额（计算值 = total_slots - consumed_slots - refunded_slots） |
| created_at | string | 创建时间 |
| updated_at | string | 更新时间 |

---

### 8.2 payCourseAccountTransactions — 查询课程账户流水

**使用场景：**
1. 传 `customer_id`（+ 可选 `course_id`）：查某学员的账户变动明细
2. 仅传 `course_id`：查该课程下所有学员的流水（**哪些学员报过名**）

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 否 | 学员 ID（与 course_id 至少填一个） |
| course_id | int | 否 | 课程 ID（单独传 = 查该课程全部学员流水） |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| start_time | string | 否 | 起始时间 |
| end_time | string | 否 | 结束时间 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 流水记录数组（按 created_at 倒序） |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**流水记录字段（list[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| id | int | 流水 ID |
| customer_id | int | 学员 ID（按课程查询时可用于区分不同学员） |
| course_id | int | 课程 ID |
| type | string | 变动类型（见下表） |
| type_label | string | 变动类型中文：充值 / 报名消耗 / 退款扣除 / 帮朋友报名 / 转赠转出 / 转赠收入 |
| to_customer_id | int | 关联学员ID（不同流水类型含义不同：`transfer_out`=被报名人ID，`gift_out`=接收方ID，`gift_in`=赠与人ID） |
| slots_change | int | 名额变动（正数=增加，负数=减少） |
| amount_change | decimal | 金额变动（正数=增加，负数=减少） |
| ref_order_id | int | 关联订单 ID（充值/报名时） |
| ref_refund_id | int | 关联退款单 ID（退款扣除时） |
| remark | string | 备注 |
| created_at | string | 变动时间 |

**变动类型：**

| type 值 | type_label | 说明 |
| :--- | :--- | :--- |
| recharge | 充值 | 充值订单核销后累加 |
| enroll_consume | 报名消耗 | 学员报名时消耗名额 |
| refund_deduct | 退款扣除 | 退款审核通过后扣除 |
| transfer_out | 帮朋友报名 | A用自有名额帮B报名，A扣名额，备注记录B的ID |
| gift_out | 转赠转出 | 分享链接模式：领取成功后发起方记录 |
| gift_in | 转赠收入 | 分享链接模式：领取成功后接收方记录 |

---

### 8.3 payCourseAccountDeduct — 扣减课程账户名额

**使用场景：** 学员自己报名时调用，扣减对应课程账户的可用名额。

**扣减策略：** 消耗 `consumed_slots`（名额不可退款）。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 扣减方学员 ID |
| course_id | int | 是 | 课程 ID |
| deduct_slots | int | 是 | 扣减名额数量 |
| deduct_amount | decimal | 是 | 扣减金额（分，由调用方传入，后端不做单价校验） |
| ref_order_id | int | 否 | 关联报名订单 ID |
| remark | string | 否 | 备注 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否扣减成功 |
| remaining_slots | int | 扣减后可用名额 |

**错误码：** `400` — 账户不存在 / 可用名额不足 / 参数校验失败

**并发安全**：使用 `WHERE` 条件做乐观锁，`affected_rows = 0` 表示名额不足。

---

### 8.4 payCourseAccountAdd — 增加课程账户名额

**使用场景：** 单独为某学员增加课程账户名额（不增加金额）。充值场景不使用此接口，充值由核销后处理自动累加。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 增加方学员 ID |
| course_id | int | 是 | 课程 ID |
| add_slots | int | 是 | 增加名额数量 |
| remark | string | 否 | 备注 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否增加成功 |
| course_account_id | int | 账户 ID |

**错误码：** `400` — 参数校验失败；`500` — 数据库写入失败

---

### 8.5 payCourseAccountTransfer — 帮朋友报名（A给B/C报名）

**使用场景：** A用自有名额帮朋友B报名。A消耗自有名额（`consumed_slots+N`），B的报名信息记录在其他系统（不在支付系统课程账户中）。一次调用扣减A名额并写A的流水，备注中记录帮谁报名。给多人报名时循环调用，每次处理一个。

默认每次帮报名1个名额。具体报哪个活动/场次由前端决定，后端只负责扣减名额并记录帮谁报名。

**示例**：A用课程X的自有名额给B报名。

```json
{
  "from_customer_id": 1,
  "to_customer_id": 2,
  "course_id": 10,
  "remark": "A给B报名第4期"
}
```

给C报名再调一次，`to_customer_id` 改为 C 的 ID。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| from_customer_id | int | 是 | 扣减方学员 ID（A） |
| to_customer_id | int | 是 | 被报名学员 ID（B） |
| course_id | int | 是 | 课程 ID |
| transfer_slots | int | 否 | 报名名额数量，默认 1 |
| remark | string | 否 | 备注 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否报名成功 |
| remaining_slots | int | 扣减方剩余可用名额 |

**错误码：** `400` — 账户不存在 / 可用名额不足 / 帮报名数量为 0

**并发安全**：使用 `WHERE` 条件乐观锁，`affected_rows = 0` 回滚事务。

**操作内容**：
1. A的 `consumed_slots + N`（消耗自有名额）
2. 写A的流水：`type=transfer_out`，`type_label=帮朋友报名`，`slots_change=-N`，备注记录B的ID

---

### 8.6 payCourseAccountGift — 名额转赠（A转给B持有）

**使用场景：** A将自己拥有的课程名额转赠给B。转赠后 A 的可用名额减少，B 的可用名额增加（名额所有权转移）。B 可以用转赠来的名额报名课期。不支持链式转赠（B 收到后不能再转给别人），由表单服务器控制。

一次调用完成 A 扣减 + B 增加，数据库事务保证原子性。A/B 必须不同人，转赠数量必须大于 0。

**示例**：A 转赠 2 个课程 X 的名额给 B。

```json
{
  "from_customer_id": 1,
  "to_customer_id": 2,
  "course_id": 10,
  "gift_slots": 2,
  "remark": "转赠给朋友"
}
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| from_customer_id | int | 是 | 转出方学员 ID（A） |
| to_customer_id | int | 是 | 接收方学员 ID（B） |
| course_id | int | 是 | 课程 ID |
| gift_slots | int | 是 | 转赠名额数量 |
| remark | string | 否 | 备注 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否转赠成功 |
| from_remaining_slots | int | 转出方剩余可用名额 |

**错误码：** `400` — 账户不存在 / 可用名额不足 / 转赠给自己 / 转赠数量为 0

**并发安全**：事务包裹扣减 A + 增加 B + 写两条流水，A 扣减使用乐观锁 `WHERE (total_slots - consumed_slots - refunded_slots) >= gift_slots`。

**操作内容**：
1. A 的 `consumed_slots + N`（扣减转出方名额）
2. B 的 `total_slots + N`（增加接收方名额，首次转赠自动创建账户）
3. 写 A 的流水：`type=gift_out`，`slots_change=-N`，`to_customer_id=B`
4. 写 B 的流水：`type=gift_in`，`slots_change=+N`，`to_customer_id=A`

---

### 8.7 payMySlots — 我的名额（聚合查询）

**使用场景：** 前端展示"我的名额"页面，一次性返回学员在所有课程中的名额总数和来源明细，区分购买和转赠来源。

**与 `payCourseAccountList` 的区别：** 本接口额外返回 `items` 数组，包含每笔名额的来源明细（购买订单号/金额、转赠来源人），前端只需调一次即可完整展示。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 学员 ID |
| course_id | int | 否 | 课程 ID（不传返回所有课程） |
| page | int | 否 | 页码（控制 items 明细分页，默认 1） |
| page_size | int | 否 | 每页条数（控制 items 明细数量，默认 20，最大 200） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| accounts | array | 课程账户数组，每项包含： |
| total | int | 符合条件的总记录数（items 明细总数） |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**账户记录字段（accounts[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| course_id | int | 课程 ID |
| total_slots | int | 累计名额 |
| consumed_slots | int | 已消耗名额 |
| refunded_slots | int | 已退款名额 |
| available_slots | int | 可用名额 |
| items | array | 名额来源明细数组（见下表） |

**名额来源明细字段（accounts[].items[]）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| source | string | 来源类型：`purchase`（购买）/ `gift`（转赠） |
| slots | int | 该笔来源的名额数量 |
| order_no | string | 订单号（仅 `purchase` 时有值） |
| total_amount | int | 订单总金额（分）（仅 `purchase` 时有值） |
| ref_order_id | int | 订单自增 ID（仅 `purchase` 时有值） |
| from_customer_id | int | 赠与人 ID（仅 `gift` 时有值） |
| created_at | string | 获取时间 |

**示例：**

```json
// 请求 POST /api/payMySlots
{ "customer_id": 1 }

// 响应
{
  "code": 200,
  "msg": "success",
  "data": {
    "accounts": [
      {
        "course_id": 1,
        "total_slots": 5,
        "consumed_slots": 1,
        "refunded_slots": 0,
        "available_slots": 4,
        "items": [
          {
            "source": "purchase",
            "slots": 3,
            "order_no": "ORD20260531195458455736",
            "total_amount": 29700,
            "ref_order_id": 1,
            "created_at": "2026-05-31 19:54:58"
          },
          {
            "source": "gift",
            "slots": 2,
            "from_customer_id": 4002,
            "created_at": "2026-05-31 20:00:00"
          }
        ]
      }
    ]
  }
}
```

---

### 8.8 payCourseAccountRefundDeduct — 退款扣除课程账户名额（内部调用）

**使用场景：** 退款审核通过后，退款模块内部调用此接口扣除学员的课程账户名额和金额。前端不直接调用。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| customer_id | int | 是 | 学员 ID |
| course_id | int | 是 | 课程 ID |
| deduct_slots | int | 是 | 扣除名额数量 |
| deduct_amount | decimal | 是 | 扣除金额（分） |
| ref_refund_id | int | 是 | 关联退款单 ID |
| remark | string | 否 | 备注 |

**注意**：退款扣除的名额**不恢复** available_slots。钱已退，名额彻底扣除。

---

### 8.9 payGiftCreate — 创建转赠（分享链接模式）

**使用场景：** 小程序端学员想把自己的课程名额分享给其他人。发起转赠后冻结对应名额，返回 `claim_code`，前端将其拼接为分享链接。每个名额对应一个独立链接。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| giver_id | int | 是 | 发起方客户 ID |
| course_id | int | 是 | 课程 ID |
| gift_slots | int | 是 | 转赠名额数（固定为 1） |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| claim_code | string | 12 位随机码，用于拼接分享链接 |
| expires_at | string | 过期时间（YYYY-MM-DD HH:MM:SS），创建时间 +24 小时 |

**示例：**

```json
// 请求 POST /api/payGiftCreate
{
  "giver_id": 4001,
  "course_id": 1,
  "gift_slots": 1
}

// 响应
{
  "code": 200,
  "msg": "success",
  "data": {
    "claim_code": "Ab3x9KmNpQr2",
    "expires_at": "2026-06-03 14:30:00"
  }
}
```

**后端校验规则：**
1. `gift_slots` 固定为 1，传其他值返回 400
2. 发起方在该课程的可用名额 ≥ 1
3. 冻结名额：`consumed_slots + 1`（available_slots 减少）
4. 生成 12 位随机 `claim_code`（大写字母+数字），写入转赠记录表（status=pending）

**错误码：** `400` — 账户不存在 / 可用名额不足 / gift_slots 不为 1

---

### 8.10 payGiftQuery — 查询转赠详情

**使用场景：** 接收方点击分享链接后，调用此接口查看转赠信息（发起人、课程、名额、状态等）。**无需鉴权**，未登录用户也能查。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| claim_code | string | 是 | 转赠编码 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| claim_code | string | 转赠编码 |
| giver_id | int | 发起方客户 ID |
| giver_name | string | 发起方昵称 |
| course_id | int | 课程 ID |
| course_name | string | 课程名称（预留，当前为空） |
| gift_slots | int | 转赠名额数 |
| status | string | 转赠状态：`pending` / `claimed` / `expired` / `recalled` |
| receiver_id | int | 接收方 ID（已领取时有值） |
| receiver_name | string | 接收方昵称（已领取时有值） |
| expires_at | string | 过期时间 |
| created_at | string | 创建时间 |

**过期检测：** 查询时若 status=pending 且当前时间超过 expires_at，自动更新为 expired。

---

### 8.11 payGiftClaim — 领取转赠

**使用场景：** 接收方点击分享链接后，登录后点击"领取名额"调用此接口。领取成功后名额从发起方转移到接收方。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| claim_code | string | 是 | 转赠编码 |
| receiver_id | int | 是 | 接收方客户 ID |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否领取成功 |
| gift_slots | int | 领取到的名额数 |
| giver_id | int | 发起方 ID |

**后端校验规则：**
1. `claim_code` 存在、status=pending、未过期
2. `receiver_id ≠ giver_id`（不能领取自己的转赠）
3. 接收方尚未领取过该 code（幂等）
4. 事务内完成：发起方名额正式消耗 + 接收方名额增加 + 写流水 + 标记 claimed

**流水记录：** 发起方 type=`gift_out`（转赠转出），接收方 type=`gift_in`（转赠收入），同时记录对方用户 ID。

**错误码：** `400` — 转赠已过期 / 已领取 / 非本人 / 领取失败；`404` — claim_code 不存在

**并发安全：** `UPDATE ... WHERE status='pending'` 原子更新，affected_rows=0 表示已被其他人领取。

---

### 8.12 payGiftRecall — 召回转赠

**使用场景：** 发起方发起转赠后，对方尚未领取前，发起方可以主动召回，恢复被冻结的名额。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| claim_code | string | 是 | 转赠编码 |
| giver_id | int | 是 | 发起方客户 ID |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| success | bool | 是否召回成功 |
| remaining_slots | int | 召回后发起方可用名额 |

**后端校验规则：**
1. `claim_code` 存在、发起人是 `giver_id`、status=pending、未过期
2. 解冻名额：`consumed_slots - 1`（available_slots 恢复）
3. 标记转赠记录 status=recalled

**错误码：** `400` — 转赠已过期 / 已被领取 / 已召回；`403` — 非发起人无权召回

---

### 8.13 payGiftList — 查询转赠记录列表

**使用场景：** 查询某用户的转赠历史记录，支持按课程、状态、时间范围筛选。

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
| :--- | :--- | :--- | :--- |
| giver_id | int | 是 | 发起方客户 ID |
| page | int | 否 | 页码，默认 1 |
| page_size | int | 否 | 每页条数，默认 20，最大 200 |
| course_id | int | 否 | 课程 ID（不传返回该用户所有课程的转赠记录） |
| status | string | 否 | 按转赠状态筛选（pending / claimed / expired / recalled） |
| start_time | string | 否 | 起始时间 |
| end_time | string | 否 | 结束时间 |

**响应 data：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| list | array | 转赠记录数组（按 created_at 倒序） |
| total | int | 符合条件的总记录数 |
| page | int | 当前页码 |
| page_size | int | 每页条数 |
| total_pages | int | 总页数 |

**转赠记录字段（list[].）：**

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| claim_code | string | 12 位转赠编码 |
| giver_id | int | 发起方客户 ID |
| course_id | int | 课程 ID |
| gift_slots | int | 转赠名额数 |
| status | string | 转赠状态（pending / claimed / expired / recalled，过期状态实时判定不写库） |
| receiver_id | int | 接收方 ID（claimed 时有值） |
| expires_at | string | 过期时间 |
| created_at | string | 创建时间 |
| claimed_at | string | 领取时间（claimed 时有值） |

---

## 状态枚举值

### 订单支付状态（orders.status）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `pending_payment` | 待支付 | 小程序订单创建后、未支付前 |
| `paid` | 已支付 | 支付成功，但尚未核销或财务审核 |
| `verified` | 已核销 | 订单已生效（系统自动核销后更新为此状态） |
| `rejected` | 已拒绝 | 财务审核拒绝（仅限线下订单） |
| `refunding` | 部分退款 | 已有一笔或多笔部分退款完成，可继续退款 |
| `refunded` | 已退款 | 全额退款完成，不可再退 |
| `closed` | 已关闭 | 超时未支付或支付失败被关闭 |

### 订单核销状态（orders.verification_status）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `none` | 未核销 | 初始状态（小程序订单创建后或线下订单被拒后） |
| `auto_verified` | 自动核销 | 小程序支付成功时系统自动核销（verified_by=-1） |
| `manual_verified` | 人工核销 | 财务审核通过后标记 |
| `pending_review` | 待审核 | 线下订单创建后等待财务审核 |
| `rejected` | 审核拒绝 | 财务拒绝线下订单 |

### 退款状态（refunds.status）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `pending` | 待处理 | 退款申请已提交，等待财务审核 |
| `rejected` | 已拒绝 | 财务审核拒绝退款 |
| `completed` | 已完成 | 退款已到账（审核通过后直接设为 completed） |
| `cancelled` | 已取消 | 退款申请被取消 |

### 转赠状态（course_gift_records.status）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `pending` | 等待领取 | 发起方已冻结名额，等待接收方领取 |
| `claimed` | 已领取 | 接收方已成功领取 |
| `expired` | 已过期 | 超过 24 小时未领取，链接失效 |
| `recalled` | 已召回 | 发起方主动召回，名额恢复 |

### 充值类型（orders.recharge_type）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `recharge` | 充值 | 首次充值（小程序）/ 业务员填写 |
| `additional_recharge` | 追加充值 | 小程序端有历史充值记录时自动判断，或业务员线下填写 |
| `retrain_recharge` | 复训充值 | 仅业务员线下填写 |
| `qixin_recharge` | 齐心会充值 | 仅业务员线下填写 |

---

### 支付流水状态（payment_records.status）

| 值 | 含义 | 说明 |
| :--- | :--- | :--- |
| `pending` | 待支付 | 已创建支付流水，等待用户支付 |
| `success` | 支付成功 | 用户已完成支付 |
| `failed` | 支付失败 | 支付失败或用户取消 |
| `closed` | 已关闭 | 支付超时关闭 |

### 操作类型（action_type）

| 值 | 含义 |
| :--- | :--- |
| `create_order` | 业务员/系统提交订单 |
| `approve` | 财务审核通过 |
| `reject` | 财务审核拒绝 |
| `resubmit` | 业务员修改后重新提交 |
| `close` | 关闭订单 |
| `delete` | 删除订单（软删除） |
| `pay_failed` | 支付失败 |
| `refund_apply` | 申请退款 |
| `refund_approve` | 退款审核通过 |
| `refund_reject` | 退款审核拒绝 |
| `refund_fail` | 微信退款 API 调用失败 |
