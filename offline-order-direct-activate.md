---
name: offline-order-direct-activate
overview: 线下订单创建后直接到账；审核拒绝时两步式追回（reject返回清单→前端删报名→confirm扣名额）。
todos:
  - id: refactor-applyOrderRecharge
    content: 将 handlePayOrderReview approve 路径的账户累加逻辑抽取为独立函数 applyOrderRecharge()，供线下订单创建和审核通过共用
    status: completed
  - id: modify-createOffline
    content: 修改 handlePayOrderCreateOffline：verification_status 保持 pending_review，事务内调用 applyOrderRecharge() 直接到账
    dependencies:
      - refactor-applyOrderRecharge
    status: completed
  - id: add-auto-enroll
    content: 在 handlePayOrderCreateOffline 事务内、账户累加完成后，若 session_id > 0 则内联报名消耗逻辑（扣 consumed_slots，写 enroll_consume 流水）→ ⚠️ 已改为前端协调模式：后端不再内联报名，session_id 仅记录在订单上供前端查询
    dependencies:
      - modify-createOffline
    status: cancelled
  - id: createOffline-return-fields
    content: handlePayOrderCreateOffline 返回增加 session_id/slot_quantity/course_id 字段，供前端判断是否需要同步报名 → 不需要，前端在提交时已知这些信息
    dependencies:
      - modify-createOffline
    status: cancelled
  - id: rewrite-reject
    content: 重写 handlePayOrderReview reject 路径：只改状态为rejected + 查询名额分布生成 withdraw_list（需退课的人）和 gift_revoke_list（转赠撤回通知），返回给前端
    status: completed
  - id: reclaim-confirm-api
    content: 新增 payOrderReclaimConfirm 接口：前端删完报名记录后调用，后端执行所有名额扣减（方案B直接扣减，不调withdraw），幂等设计（rejected→rejected_confirmed）
    dependencies:
      - rewrite-reject
    status: completed
  - id: fix-resubmit-and-universal
    content: 修改 handlePayOrderResubmit（rejected_confirmed 订单重新到账）和 handlePayUniversalRecharge（创建时立即到账）
    dependencies:
      - refactor-applyOrderRecharge
    status: completed
  - id: compile-and-test
    content: 编译验证（make），编写测试用例验证：线下订单直接到账、订单拒绝追回全链路
    dependencies:
      - modify-createOffline
      - rewrite-reject
      - reclaim-confirm-api
      - fix-resubmit-and-universal
    status: pending
  - id: run-existing-tests
    content: 运行现有测试脚本确认退课/转赠/代报名链路不受影响；检查并修复 test_pay_api.sh 中审核流程断言
    dependencies:
      - compile-and-test
    status: pending
  - id: write-new-tests
    content: 新增 test_offline_reject.sh（订单拒绝追回全链路）和 test_auto_enroll.sh（创建时自动报名），覆盖7组27个场景
    dependencies:
      - compile-and-test
    status: pending
---

## 产品概述

线下订单状态机保持不变（待审核→人工审核通过/拒绝），但名额到账时机提前：创建订单（待审核状态）时名额立即到账，用户可自由支配；审核通过时名额无变化（只计入业绩）；审核拒绝时两步式追回（reject返回清单→前端删报名→confirm扣名额）。

## 核心功能

1. **名额到账时机提前**：线下订单状态机不变（`pending_review` → `manual_verified` / `rejected` / `rejected_confirmed`），但 `handlePayOrderCreateOffline` 创建订单时（`pending_review` 状态）立即累加客户课程账户名额，用户可自由支配（报名/转赠）。审核通过（`manual_verified`）时名额无变化，只改状态。
2. **下单时传入 session_id**：报名由前端协调。后端在下单时仅记录 session_id 并返回给前端。前端收到成功返回后，调另一台服务器创建报名记录 + 调本服务器 `payCourseAccountDeduct` 扣名额。
3. **订单拒绝两步式追回**：reject 时只改状态+返回追回清单（退课清单+转赠撤回清单）；前端删完另一台服务器报名记录后调 `payOrderReclaimConfirm`，后端执行直接扣减（不调withdraw，沿gift单层链抹掉所有节点）。

## Tech Stack Selection

- 语言：C++20（现有项目，不引入新语言）
- 框架：Crow（现有，不变更）
- 数据库：MySQL 8.0，ORM：ORMPP（现有）
- 并发安全：乐观锁（`WHERE col >= N`），事务（`conn->begin/commit/rollback`）

---

## 🔵 前端协作接口说明（重要）

### 概述

本需求涉及 3 个需要前后端协作的关键场景。前端需要知道每个接口返回什么、何时调用下一接口、以及错误处理策略。

---

### 场景一：创建线下订单 + 同步报名

#### 后端行为

`POST /api/payOrderCreateOffline` 在事务内完成以下操作：
1. 插入订单记录（`status=paid`, `verification_status=pending_review`）
2. 调用 `applyOrderRecharge()` **立即给客户课程账户加名额**
3. commit 事务

**名额在 commit 后即已到账**，用户可立即支配（报名/转赠）。

#### 后端返回

```json
{
  "code": 200,
  "data": {
    "order_id": 123,
    "order_no": "OF20250101xxxx",
    "status": "paid",
    "belongs_to_staff_id": 5
  }
}
```

> 不需要返回 `session_id`/`slot_quantity`/`course_id`，前端提交时已知这些信息，收到成功返回后自行判断是否需要同步报名。

#### 前端联动逻辑

```
提交线下订单（传 session_id）
       │
       ▼
收到返回 code=200, session_id > 0
       │
       ▼
【同步报名】调另一台服务器创建报名记录（registrations）
       │       参数: customer_id, session_id, slot_quantity
       │
       ▼
调本服务器 POST /api/payCourseAccountDeduct
       参数: customer_id, course_id, slots
       消耗已到账的名额
       │
       ▼
报名完成
```

**关键确认**：前端收到 `code=200` 时，**名额已经到账了**（事务已 commit）。前端可以立即调 deduct 扣名额，不需要等待审核。

**失败处理**：
- 如果另一台服务器报名失败 → 名额已到账但未消耗，用户可手动报名或等待审核
- 如果 deduct 失败（名额被其他操作先消耗） → 提示用户名额不足，需重新充值

---

### 场景二：审核拒绝 + 追回确认

#### 步骤1：审核拒绝（payOrderReview reject）

财务审核拒绝时，后端返回两个清单。**此时仅改订单状态，不修改任何账户/转赠记录。**

**后端返回**：
```json
{
  "code": 200,
  "data": {
    "order_no": "OF20250101xxxx",
    "status": "paid",
    "verification_status": "rejected",
    "withdraw_list": [
      {"customer_id": 8201, "session_id": 2},
      {"customer_id": 8202, "session_id": 2}
    ],
    "gift_revoke_list": [
      {"gift_record_id": 101, "receiver_id": 8203, "slots": 1, "consumed": false},
      {"gift_record_id": 102, "receiver_id": 8204, "slots": 1, "consumed": true}
    ]
  }
}
```

**字段说明**：

| 字段 | 说明 |
|------|------|
| `withdraw_list` | 🔴 **必须操作**：需要从另一台服务器删除报名记录的人员清单 |
| `withdraw_list[].customer_id` | 需要删报名的用户ID |
| `withdraw_list[].session_id` | 对应的课期ID（去另一台服务器删 registrations 的 key） |
| `gift_revoke_list` | 🟢 **纯通知**：受影响的转赠清单，无需调用任何后端接口 |
| `gift_revoke_list[].gift_record_id` | 转赠记录ID |
| `gift_revoke_list[].receiver_id` | 被撤回转赠的用户ID（通知对象） |
| `gift_revoke_list[].slots` | 被撤回的名额数 |
| `gift_revoke_list[].consumed` | 是否已被消耗（true=receiver 已用这个名额报过名） |

**withdraw_list 包含哪些人**：
- 订单客户 A 通过 `payCourseAccountDeduct` 自报名的人
- 订单客户 A 通过 `payCourseAccountTransfer` 代报名的人（B 是被代报名人）
- 用转赠名额报名了的 receiver

**gift_revoke_list 包含哪些人**（status = pending 或 claimed 的转赠记录）：
- 转赠未领取（pending）：receiver 还没领，需通知 giver"转赠已失效"
- 转赠已领取未消耗（claimed, consumed=false）：receiver 已领取但未用，需通知 receiver"赠送被撤回"
- 转赠已领取已消耗（claimed, consumed=true）：receiver 已领取且已报名，需通知 receiver"名额被撤回，报名将失效"
- ❌ 已手动撤回（recalled）的不返回

#### 步骤2：前端操作

> ⚠️ **关键**：前端只需处理 `withdraw_list`（删报名记录）。`gift_revoke_list` 仅用于展示/通知，**不需要调任何后端接口**，后端在 `reclaimConfirm` 中自己完成转赠树的清理。

**操作顺序（严格按此执行）**：

```
1. 收到 reject 返回
       │
       ▼
2. 遍历 withdraw_list，调另一台服务器删除报名记录
   参数: customer_id + session_id
       │
       ├─ 任意一条删除失败 → 🛑 终止！不调 confirm，人工介入
       │
       ▼
3. 遍历 gift_revoke_list，通知 receiver 转赠被撤回
   （纯前端：弹窗/推送/列表标记，不需要后端参与）
       │
       ▼
4. 调本服务器 POST /api/payOrderReclaimConfirm
   参数: { "order_no": "xxx" }
       │
       ▼
5. 收到 code=200 → 追回完成，订单状态 rejected_confirmed
```

#### 步骤3：追回确认（payOrderReclaimConfirm）

后端收到 confirm 后自行完成所有名额扣减：
- 构建完整转赠树，按深度降序（叶子优先）逐层清理
- 扣减 A 的 total/consumed/frozen
- 清理下游 receiver 的 total/gifted/consumed/consumed_gifted
- 最终全链路归零，名额完全守恒

**请求**（只需传 order_no，后端自己查所有数据）：
```json
POST /api/payOrderReclaimConfirm
{
  "order_no": "OF20250101xxxx"
}
```

**成功返回**：
```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "order_no": "OF20250101xxxx",
    "verification_status": "rejected_confirmed"
  }
}
```

**幂等**（已 confirm 后再调，直接返回成功，不重复扣减）：
```json
{
  "code": 200,
  "msg": "Already confirmed",
  "data": {
    "order_no": "OF20250101xxxx",
    "verification_status": "rejected_confirmed"
  }
}
```

**失败返回示例**：
```json
// 订单不是 rejected 状态
{ "code": 400, "msg": "Order is not in rejected status (current: pending_review)" }

// 并发修改导致乐观锁失败
{ "code": 400, "msg": "Account update failed (concurrent modification or insufficient slots)" }
```

**前端重试策略**：
- confirm 幂等，可安全重试（如 3 次，间隔递增）
- 网络超时/5xx 同样可重试
- 400 错误不重试（先检查订单是否已是 rejected_confirmed）

---

### 场景三：重新提交（payOrderResubmit）

**前置条件**：订单必须是 `rejected_confirmed` 状态（`rejected` 未 confirm 不允许 resubmit）

**请求**：
```json
POST /api/payOrderResubmit
{
  "order_no": "OF20250101xxxx",
  "session_id": 99,
  "operator_id": 1
}
```

**成功时名额重新到账**（调 `applyOrderRecharge()`），订单回到 `pending_review` 状态，前端可重新编排报名。

---

## Implementation Approach

### 需求1：名额到账时机提前（状态机不变）✅ 已完成

**核心改动**：将 `handlePayOrderReview` approve 路径中的账户累加逻辑抽取为独立函数 `applyOrderRecharge()`，在 `handlePayOrderCreateOffline` 事务内直接调用。**状态机扩展**：创建时 `pending_review`，审核通过 `manual_verified`，审核拒绝 `rejected`，追回完成 `rejected_confirmed`（新增）。

**`applyOrderRecharge()` 函数逻辑**（行71-137，已实现）：

1. `INSERT INTO customer_course_accounts ... ON DUPLICATE KEY UPDATE total_slots += N, total_recharge_amount += N`
2. 查回 `acc_id`
3. 写 `course_account_transactions` 流水（`type="recharge"`）
4. 回写 `orders.course_account_id`

**修改点**：

- ✅ `handlePayOrderCreateOffline`（行1504）：`verification_status` 保持 `"pending_review"`，在 `conn->begin()` 后、`conn->commit()` 前，先 insert orders，再调用 `applyOrderRecharge()` 直接到账，最后 commit。
- ✅ `handlePayOrderReview` approve 路径（行1760）：仅旧数据（`course_account_id==0`）补到账，新订单创建时已到账不再重复。
- ✅ `handlePayOrderResubmit`（行3972）：`rejected_confirmed` 订单重新提交时调用 `applyOrderRecharge()` 重新到账。
- ✅ `handlePayUniversalRecharge`（行5030）：创建时立即到账（调用 `applyOrderRecharge()`）。

### 需求2：下单时 session_id 处理 ⚠️ 改为前端协调

**当前实现**：后端不在事务内自动报名。session_id 记录在 `orders.session_id` 字段上。

- 前端提交时已知 `session_id`，收到 `code=200` 后自行判断是否需要同步报名。
- 若 `session_id > 0`，前端调另一台服务器创建报名 + 调本服务器 `payCourseAccountDeduct` 扣名额。

### 需求3：订单拒绝追回（两步式）✅ 已完成

**核心思路**：reject 时本服务器只改状态 + 返回追回清单。前端删完另一台服务器的报名记录后，调 `payOrderReclaimConfirm`，后端再执行所有名额扣减。

**为什么不调 withdraw**：withdraw 是"归还到源头"逻辑，reject 追回需要"抹掉"所有节点，数学模型不同。直接扣减才能完全守恒。

**字段更新规则**（available 是计算值 `total-consumed-refunded-frozen`，自动跟随）：

| 名额状态 | A端（giver/订单客户） | B端（receiver） | available变化 |
|----------|----------------------|-----------------|--------------|
| available余额 | total-=1 | - | A:-1 |
| 自报名/代报名 consumed | total-=1, consumed-=1 | - | A:0 |
| 转赠未领取(pending) | total-=1, frozen-=1 | - | A:0 |
| 转赠已领取未消耗 | total-=1, frozen-=1 | total-=1, gifted-=1 | A:0, B:-1 |
| 转赠已领取已消耗 | total-=1, frozen-=1 | total-=1, gifted-=1, consumed-=1, consumed_gifted-=1 | A:0, B:0 |

**gift 链模型**：A→B→C 链式转赠时，`origin_giver_id` 始终为 A，追回时按深度降序（叶子优先）处理整棵树。

**举例验证**（A充5→A自报名1(session=2)→A代B报名1(session=2)→A转赠1给C(C未用)→A转赠2给D(D报名1(session=2),剩1未用)）：

- 初始：A(total=5,consumed=2,frozen=3,avail=0), C(total=1,gifted=1,avail=1), D(total=2,gifted=2,consumed=1,cg=1,avail=1)
- reject 返回：
  - withdraw_list: [{A,2}, {B,2}, {D,2}]
  - gift_revoke_list: [{C,101,1,false}, {D,102,1,true}, {D,103,1,false}]
- 前端删 A/B/D 的报名记录 → 调 confirm
- 后端扣减：
  - A: total-=5, consumed-=2, frozen-=3 → total=0, consumed=0, frozen=0
  - C: total-=1, gifted-=1 → total=0, gifted=0
  - D: total-=2, gifted-=2, consumed-=1, cg-=1 → total=0, gifted=0, consumed=0, cg=0
- **结果**：A/C/D 全归零✓，完全守恒

## Implementation Notes

1. **并发安全**：所有 `UPDATE customer_course_accounts` 必须带乐观锁条件，检查 `affected_rows == 1`
2. **事务边界**：`handlePayOrderCreateOffline` 已改为 `conn->begin/commit` 包裹 insert order + applyOrderRecharge ✅
3. **reject 追回不调 withdraw**：采用"直接扣减"模式，沿 gift 树叶子优先逐节点抹掉字段 ✅
4. **跨服务协调（两步式）**：reject 只改状态+返回清单；前端删报名后调 confirm ✅
5. **追回顺序**：available → 转赠已消耗 → 代报名 → 自报名 → 转赠冻结（LIFO）
6. **`payOrderReclaimConfirm` 新接口**：传 order_no，幂等设计（rejected→rejected_confirmed）✅
7. **新增状态 `rejected_confirmed`**：verification_status 新增值 ✅
8. **名额同质性**：名额不分订单来源，reject 追回 N=订单.slot_quantity ✅
9. **状态机**：`pending_review`（创建，名额已到账）→ `manual_verified`（审核通过）/ `rejected`（待追回）→ `rejected_confirmed`（追回完成）

## 影响点矩阵（开发前确认）

### 无需改动（基于账户余额，不依赖订单状态）

| 接口 | 行号 | 原因 |
| --- | --- | --- |
| `handlePayGiftCreate` | 6435 | 只检查 `transferable = total-consumed-refunded-frozen >= 1` |
| `handlePayGiftClaim` | 6749 | 基于 gift_records 领取 |
| `handlePayGiftRecall` | 7017 | 基于 gift_records 召回 |
| `handlePayCourseAccountDeduct` | 4031 | 自报名消耗，检查账户余额 |
| `handlePayCourseAccountTransfer` | 4527 | 代报名，检查账户余额 |
| `handlePayCourseAccountWithdraw` | 5182 | 退课，自己不涉及 reject 追回 |
| `handlePayOrderClose` | 1757 | 只处理 `pending_payment` |
| `handlePayRefund` | 2147 | pending_review 不可退款 |

### 需要改动

| 接口 | 行号 | 改动内容 | 影响级别 | 状态 |
| --- | --- | --- | --- | --- |
| `handlePayOrderCreateOffline` | 1504 | 事务包裹 + applyOrderRecharge() 直接到账 | 高（核心） | ✅ 已完成 |
| `handlePayOrderCreateOffline` 返回值 | 1622 | ~~补充字段~~ → 不需要，前端已知 | — | ✅ 已取消 |
| `handlePayOrderReview` approve | 1661 | 仅旧数据补到账，新订单只改状态 | 高（核心） | ✅ 已完成 |
| `handlePayOrderReview` reject | 1786 | 返回 withdraw_list + gift_revoke_list | 高（核心） | ✅ 已完成 |
| `handlePayOrderResubmit` | 3328 | rejected_confirmed → applyOrderRecharge() 重新到账 | 高 | ✅ 已完成 |
| `handlePayOrderModify` | 3615 | pending_review 锁定 slot_quantity/course_id | 中 | ✅ 已完成 |
| `handlePayUniversalRecharge` | 4393 | 创建时立即到账 | 中 | ✅ 已完成 |
| `payOrderReclaimConfirm` | 1861 | 新接口：两步式追回确认 | 高（核心） | ✅ 已完成 |

---

## 测试脚本影响评估

### 需要新增的测试场景

#### 场景组1：基础到账验证

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 1.1 纯充值 | A下单3名额，不传session_id | 创建后A.total=3,consumed=0,avail=3；审核通过后不变 |
| 1.2 充值+报名 | A下单3名额（返回session_id>0）→ 前端调deduct | 创建后A.total=3；deduct后consumed=3 |
| 1.3 万能充值 | payUniversalRecharge 3名额 | 创建后A(课程0).total=3,consumed=0,avail=3 |

#### 场景组2：reject 追回基础场景

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 2.1 全部available | A下单3名额，不花 → reject | reject返回withdraw_list空；confirm后A.total=0 |
| 2.2 自报名 | A下单3名额,session=2 → reject | withdraw_list=[{A,2}]；confirm后A.total=0,consumed=0 |
| 2.3 代报名 | A下单3名额,A代B报名1(session=2) → reject | withdraw_list=[{B,2}]；confirm后A.total=2,consumed=0 |
| 2.4 混合自报+代报 | A下单3名额,A自报1(session=2)+A代B报1(session=2) → reject | withdraw_list=[{A,2},{B,2}]；confirm后A.total=1 |

#### 场景组3：reject 追回含转赠

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 3.1 转赠未领取 | A下单3,A转赠1给B(pending) → reject | gift_revoke_list=[{B}]（pending也返回）；confirm后A.total=2,frozen=0 |
| 3.2 转赠已领取未消耗 | A下单3,A转赠1给B(B领取未用) → reject | gift_revoke_list=[{B}]；confirm后A.total=2,B.total=0 |
| 3.3 转赠已领取已消耗 | A下单3,A转赠1给B,B报名1(session=2) → reject | withdraw_list=[{B,2}],gift_revoke_list=[{B}]；confirm后A.total=2,B.total=0 |
| 3.4 转赠混合 | A下单5,A自报1+A代B报1+A转赠1给C(未用)+A转赠2给D(D报1,剩1) → reject | withdraw_list=[{A,2},{B,2},{D,2}]；confirm后A/C/D全归零 |
| 3.5 撤回后reject | A下单3→转赠B→转赠C→撤回B→reject | gift_revoke_list只有C(B已是recalled不返回)；C领取后 confirm 全归零 |

#### 场景组4：多订单叠加

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 4.1 余额充足 | A下单3(订单1)+A下单3(订单2),不花 → reject订单2 | confirm后A.total=3（订单1的还在） |
| 4.2 余额不足追回 | A下单3(订单1)+A下单3(订单2),A花掉4个 → reject订单2(N=3) | avail=2,需追回1；withdraw_list含1个session；confirm后A.total=3 |
| 4.3 跨订单转赠 | A下单3(订单1)+A下单3(订单2),A转赠2给B → reject订单2 | avail=4,扣3；confirm后A.total=3,B不变 |

#### 场景组5：边界与异常

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 5.1 reject未confirm | A下单3 → reject → 不调confirm | A名额不变，订单状态rejected |
| 5.2 confirm幂等 | A下单3 → reject → confirm → 再confirm | 第二次返回成功，名额不重复扣 |
| 5.3 rejected状态不可resubmit | A下单3 → reject(未confirm) → resubmit | 返回400"请先完成追回确认" |
| 5.4 rejected_confirmed可resubmit | A下单3 → reject → confirm → resubmit | 成功，名额重新到账 |
| 5.5 pending_review修改锁定 | A下单3 → Modify改slot_quantity | 返回400 |
| 5.6 pending_review修改备注 | A下单3 → Modify改remark | 成功 |

#### 场景组6：前端联动报名验证

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 6.1 session_id=0 | A下单3,不传session_id | 返回 session_id=0，前端不触发报名 |
| 6.2 session_id>0 | A下单3,session_id=2 | 返回 session_id=2，前端调另一台服务器报名 + deduct |
| 6.3 deduct失败 | A下单3,session=2 → 前端报名成功后deduct时名额被其他操作消耗 | deduct返回400，提示余额不足 |

## 已识别的问题与应对

### 问题1：创建订单返回是否需要补充字段（✅ 已决策：不需要）

**结论**：前端提交线下订单时已知 `session_id`、`slot_quantity`、`course_id`，收到 `code=200` 即名额已到账，自行判断是否需要调报名接口即可，不需要后端再返回这些字段。

### 问题2：前端删报名成功但 confirm 未到达后端（网络丢失）

**问题**：前端调另一台服务器删报名成功，但调 confirm 时网络丢失。
**应对**：
- 前端重试：confirm 幂等，重试 3 次，间隔递增
- 后端兜底：扫描 `verification_status='rejected'` 超过 1 小时的订单，WARNING 日志告警
- 超时标记：24 小时未 confirm 标记 `rejected_timeout`，需人工介入

### 问题3：reject 与 confirm 之间的状态变化

**问题**：reject 返回清单后、confirm 前，用户又操作了（转赠/报名）。
**应对**：confirm 重新查询当前状态执行扣减，不依赖 reject 时的清单。乐观锁检测并发变化，失败提示重试。

