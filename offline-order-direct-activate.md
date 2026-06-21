---
name: offline-order-direct-activate
overview: 线下订单创建后直接到账；审核拒绝时两步式追回（reject返回清单→前端删报名→confirm扣名额）。
todos:
  - id: refactor-applyOrderRecharge
    content: 将 handlePayOrderReview approve 路径的账户累加逻辑抽取为独立函数 applyOrderRecharge()，供线下订单创建和审核通过共用
    status: pending
  - id: modify-createOffline
    content: 修改 handlePayOrderCreateOffline：verification_status 保持 pending_review，事务内调用 applyOrderRecharge() 直接到账
    dependencies:
      - refactor-applyOrderRecharge
    status: pending
  - id: add-auto-enroll
    content: 在 handlePayOrderCreateOffline 事务内、账户累加完成后，若 session_id > 0 则内联报名消耗逻辑（扣 consumed_slots，写 enroll_consume 流水）
    dependencies:
      - modify-createOffline
    status: pending
  - id: rewrite-reject
    content: 重写 handlePayOrderReview reject 路径：只改状态为rejected + 查询名额分布生成 withdraw_list（需退课的人）和 gift_revoke_list（转赠撤回通知），返回给前端
    status: pending
  - id: reclaim-confirm-api
    content: 新增 payOrderReclaimConfirm 接口：前端删完报名记录后调用，后端执行所有名额扣减（方案B直接扣减，不调withdraw），幂等设计（rejected→rejected_confirmed）
    dependencies:
      - rewrite-reject
    status: pending
  - id: fix-resubmit-and-universal
    content: 修改 handlePayOrderResubmit（rejected_confirmed 订单重新到账）和 handlePayUniversalRecharge（创建时立即到账）
    dependencies:
      - refactor-applyOrderRecharge
    status: pending
  - id: compile-and-test
    content: 编译验证（make），编写测试用例验证：线下订单直接到账、自动报名、订单拒绝追回全链路
    dependencies:
      - modify-createOffline
      - add-auto-enroll
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
2. **下单时传入 session_id 自动报名**：账户累加完成后，若 `session_id > 0`，自动消耗对应名额（扣 `consumed_slots`，写 `enroll_consume` 流水，标记 `consumed_session_id`）。
3. **订单拒绝两步式追回**：reject 时只改状态+返回追回清单（退课清单+转赠撤回清单）；前端删完另一台服务器报名记录后调 `payOrderReclaimConfirm`，后端执行直接扣减（不调withdraw，沿gift单层链抹掉所有节点）。

## Tech Stack Selection

- 语言：C++20（现有项目，不引入新语言）
- 框架：Crow（现有，不变更）
- 数据库：MySQL 8.0，ORM：ORMPP（现有）
- 并发安全：乐观锁（`WHERE col >= N`），事务（`conn->begin/commit/rollback`）

## Implementation Approach

### 需求1：名额到账时机提前（状态机不变）

**核心改动**：将 `handlePayOrderReview` approve 路径中的账户累加逻辑（行 1661-1733）抽取为独立函数 `applyOrderRecharge()`，在 `handlePayOrderCreateOffline` 事务内直接调用。**状态机扩展**：创建时 `pending_review`，审核通过 `manual_verified`，审核拒绝 `rejected`，追回完成 `rejected_confirmed`（新增）。

**`applyOrderRecharge()` 函数逻辑**（与现有 approve 路径一致）：

1. `INSERT INTO customer_course_accounts ... ON DUPLICATE KEY UPDATE total_slots += N, total_recharge_amount += N`
2. 查回 `acc_id`
3. 写 `course_account_transactions` 流水（`type="recharge"`）
4. 回写 `orders.course_account_id`

**修改点**：

- `handlePayOrderCreateOffline`（约行 1428）：`verification_status` 保持 `"pending_review"` 不变，但在 `conn->begin()` 后、`conn->commit()` 前，先 insert orders，再调用 `applyOrderRecharge()` 直接到账，若 `session_id > 0` 再执行报名消耗（需求2），最后 commit。
- `handlePayOrderReview` approve 路径：**移除**账户累加逻辑（因为创建时已经到账），只改 `verification_status` 为 `manual_verified` + 写操作日志。
- `handlePayOrderResubmit`（约行 3463）：`rejected_confirmed` 订单重新提交时，调用 `applyOrderRecharge()` 重新到账（名额在 confirm 时已被清零），状态回到 `pending_review`。注意：`rejected`（未 confirm）不允许 resubmit。
- `handlePayUniversalRecharge`（约行 4444）：同理，`verification_status` 保持 `pending_review`，创建时立即到账（调用 `applyOrderRecharge()`），审核通过时只改状态。

### 需求2：下单时自动报名

**触发条件**：`session_id > 0`（在 `applyOrderRecharge()` 成功之后执行）

**逻辑**（内联 `handlePayCourseAccountDeduct` 核心逻辑，约行 4056-4300）：

1. 计算 `from_course = min(slot_quantity, avail_course)`，`from_universal = slot_quantity - from_course`
2. 扣减 `customer_course_accounts.consumed_slots += from_course`（带乐观锁）
3. 写 `course_account_transactions` 流水：`type="enroll_consume"`，`slots_change=-slot_quantity`，`session_id=session_id`
4. 标记转赠消耗：对 `from_course` 每个名额，找 `course_gift_records` 中 `receiver_id=customer_id AND course_id=course_id AND status='claimed' AND consumed_session_id IS NULL` 的记录，设 `consumed_session_id=session_id`，同时 `consumed_gifted_slots++`

**注意**：报名消耗在事务内完成，与账户累加原子性保证。

### 需求3：订单拒绝追回（两步式：reject返回清单 + confirm执行扣减）

**核心思路**：reject 时本服务器只改状态 + 返回追回清单（退课清单+转赠撤回清单）。前端删完另一台服务器的报名记录后，调 `payOrderReclaimConfirm` 接口，后端再执行所有名额扣减。

**为什么不调 withdraw**：withdraw 是"归还到源头"逻辑，reject 追回需要"抹掉"所有节点（A和receiver一起清零），数学模型不同。直接扣减才能完全守恒。

**gift 链模型**：A→B→C 链式转赠时，中间人 B 转出后相关名额信息直接清除，**退化为 A→C 单层**。追回时无需遍历中间人，只处理 giver 和最终 receiver 两个节点。

**字段更新规则**（available 是计算值 `total-consumed-refunded-frozen`，自动跟随，无需单独操作）：

| 名额状态 | A端（giver/订单客户） | B端（receiver） | available变化 |
|----------|----------------------|-----------------|--------------|
| available余额 | total-=1 | - | A:-1 |
| 自报名/代报名 consumed | total-=1, consumed-=1 | - | A:0 |
| 转赠未领取(pending) | total-=1, frozen-=1 | - | A:0 |
| 转赠已领取未消耗 | total-=1, frozen-=1 | total-=1, gifted-=1 | A:0, B:-1 |
| 转赠已领取已消耗 | total-=1, frozen-=1 | total-=1, gifted-=1, consumed-=1, consumed_gifted-=1 | A:0, B:0 |

- `refunded_slots`：**不变**（拒绝≠退款）
- `consumed_gifted_slots`：追回转赠已消耗时递减（仅记账）

**两步式流程**：

#### 步骤1：reject 接口（改状态 + 返回清单）

本服务器查询订单客户的名额分布，生成两个清单：

**withdraw_list**（需从另一台服务器删报名记录的人）：
- A 的自报名：`{customer_id: A, session_id: X}`
- A 的代报名：`{customer_id: B, session_id: X}`（B 是被代报名人）
- 转赠已消耗的 receiver：`{customer_id: D, session_id: X}`（D 用转赠名额报了名）

**gift_revoke_list**（转赠被撤回的人，前端通知展示用）：
- 转赠已领取未消耗：`{customer_id: C, gift_record_id: 101, slots: 1}`
- 转赠已领取已消耗：`{customer_id: D, gift_record_id: 102, slots: 1}`（D 也需要通知）
- 转赠未领取（pending）：不需要通知（receiver 还不知道有转赠）

reject 响应：
```json
{
  "code": 200,
  "data": {
    "order_no": "xxx",
    "verification_status": "rejected",
    "withdraw_list": [
      {"customer_id": 8201, "session_id": 2},
      {"customer_id": 8202, "session_id": 2},
      {"customer_id": 8204, "session_id": 2}
    ],
    "gift_revoke_list": [
      {"customer_id": 8203, "gift_record_id": 101, "slots": 1},
      {"customer_id": 8204, "gift_record_id": 102, "slots": 1}
    ]
  }
}
```

#### 步骤2：前端操作

1. 遍历 `withdraw_list`，调另一台服务器删除对应的报名记录（registrations）
   - 失败 → 终止，人工介入
2. 遍历 `gift_revoke_list`，通知用户转赠被撤回（纯展示，无接口调用）
3. 全部成功后，调 `POST /api/payOrderReclaimConfirm`

#### 步骤3：payOrderReclaimConfirm 接口（后端执行名额扣减）

```json
POST /api/payOrderReclaimConfirm
{ "order_no": "xxx" }
```

后端收到确认后，执行所有名额扣减（按字段更新规则表）：

**查询范围**（confirm 重新查询，不依赖 reject 时返回的清单）：
- A 端：查 `customer_course_accounts WHERE customer_id=订单.customer_id AND course_id=订单.course_id`（含万能账户 course_id=0）
- A 的 consumed：查 `course_account_transactions WHERE customer_id=A AND course_id IN (订单.course_id, 0) AND type IN ('enroll_consume','universal_consume','transfer_out')`，按 LIFO 确定追回哪些 session
- 转赠记录：查 `course_gift_records WHERE giver_id=A AND course_id IN (订单.course_id, 0) AND status IN ('claimed','pending')`
- 各 receiver 账户：根据 gift_records 的 receiver_id 查询

**执行扣减**（一个事务内）：
- 抹掉 A：total-=N, consumed-=自报名+代报名数, frozen-=转赠数
- 抹掉各 receiver：total/gifted/consumed/cg 对应递减
- gift_records 标记 status='revoked'
- 写操作日志

**并发控制**：confirm 期间 A 的账户可能被其他操作修改（如用户又转赠）。采用乐观锁：`UPDATE ... WHERE total_slots >= N AND ...`，affected_rows=0 则返回错误提示"账户状态已变化，请重试 reject"。

**幂等设计**：通过订单 `verification_status` 判断
- `rejected` 且未 confirm → 执行扣减，标记为 `rejected_confirmed`
- `rejected_confirmed` → 直接返回成功（幂等）
- 其他状态 → 返回 400

**失败处理**：
- 前端删报名失败 → 不调 confirm → 名额保持不变，订单状态为 `rejected`（未确认）
- 人工介入处理后，重新调 confirm

**举例验证**（A充5→A自报名1(session=2)→A代B报名1(session=2)→A转赠1给C(C未用)→A转赠2给D(D报名1(session=2),剩1未用)）：

- 初始：A(total=5,consumed=2,frozen=3,avail=0), C(total=1,gifted=1,avail=1), D(total=2,gifted=2,consumed=1,cg=1,avail=1)
- reject 返回：
  - withdraw_list: [{A,2}, {B,2}, {D,2}]
  - gift_revoke_list: [{C,101,1}, {D,102,1}, {D,103,1}]
- 前端删 A/B/D 的报名记录 → 调 confirm
- 后端扣减：
  - A: total-=5, consumed-=2, frozen-=3 → total=0, consumed=0, frozen=0
  - C: total-=1, gifted-=1 → total=0, gifted=0
  - D: total-=2, gifted-=2, consumed-=1, cg-=1 → total=0, gifted=0, consumed=0, cg=0
- **结果**：A/C/D 全归零✓，完全守恒

## Implementation Notes

1. **并发安全**：所有 `UPDATE customer_course_accounts` 必须带乐观锁条件，检查 `affected_rows == 1`
2. **事务边界**：`handlePayOrderCreateOffline` 现状无事务（行 1510 直接 insert），需新增 `conn->begin/commit` 包裹 insert order + applyOrderRecharge + 自动报名
3. **reject 追回不调 withdraw**：采用"直接扣减"模式，沿 gift 单层链（A→B，中间人透明）逐节点抹掉字段。与 withdraw 的"归还到源头"模型不同，避免中间节点残留
4. **跨服务协调（两步式）**：reject 只改状态+返回清单（withdraw_list + gift_revoke_list）；前端删完另一台服务器报名记录后调 `payOrderReclaimConfirm`，后端执行名额扣减。前端删失败则不调 confirm，名额保持不变
5. **追回顺序**：available → 自报名consumed → 代报名consumed → 转赠已消耗 → 转赠已领取未消耗 → 转赠未领取（LIFO，按时间倒序）
6. **`payOrderReclaimConfirm` 新接口**：传 order_no，后端重新查询名额分布执行扣减。幂等设计（rejected→rejected_confirmed，重复调用返回成功）
7. **新增状态 `rejected_confirmed`**：verification_status 新增值，表示 reject 已确认追回完成。无需改 entity/init_db（string 类型字段）
8. **名额同质性**：名额不分订单来源，reject 追回 N=订单.slot_quantity，从账户当前余额优先扣，不足 LIFO 追回最近消耗。多订单叠加场景也能正确处理
9. **`handlePayOrderReview` 路径变化**：approve 路径不再加名额（创建时已到账），只改状态为 `manual_verified`；reject 路径改为返回清单（不立即扣名额）
10. **状态机**：`pending_review`（创建，名额已到账）→ `manual_verified`（审核通过，无变化）/ `rejected`（审核拒绝，待追回）→ `rejected_confirmed`（追回完成）

## 影响点矩阵（开发前确认）

### 无需改动（基于账户余额，不依赖订单状态）

| 接口 | 行号 | 原因 |
| --- | --- | --- |
| `handlePayGiftCreate` | 6435 | 只检查 `transferable = total-consumed-refunded-frozen >= 1`，名额到账即可转赠 |
| `handlePayGiftClaim` | 6749 | 基于 gift_records 领取，不涉及订单 |
| `handlePayGiftRecall` | 7017 | 基于 gift_records 召回，不涉及订单 |
| `handlePayCourseAccountDeduct` | 4031 | 自报名消耗，检查账户余额，不依赖订单状态 |
| `handlePayCourseAccountTransfer` | 4527 | 代报名，检查账户余额，不依赖订单状态 |
| `handlePayCourseAccountWithdraw` | 5182 | 退课，自身逻辑不变，reject 追回不复用此接口（方案B直接扣减） |
| `handlePayOrderClose` | 1757 | 只处理 `pending_payment`，线下订单 status=paid 不受影响 |
| `handlePayRefund` | 2147 | pending_review 不可退款，保持现状 |


### 需要改动

| 接口 | 行号 | 改动内容 | 影响级别 |
| --- | --- | --- | --- |
| `handlePayOrderCreateOffline` | 1428 | 事务包裹 + 调用 applyOrderRecharge() 直接到账 + session_id>0 自动报名 | 高（核心） |
| `handlePayOrderReview` approve | 1661 | 移除账户累加逻辑，只改状态为 manual_verified | 高（核心） |
| `handlePayOrderReview` reject | 1627 | 重写为两步式追回（reject返回清单 + confirm直接扣减，不调withdraw） | 高（核心） |
| `handlePayOrderResubmit` | 3328 | **rejected 重新提交时必须调用 applyOrderRecharge() 重新到账**（当前行 3463 只改状态没到账） | 高（新发现） |
| `handlePayOrderModify` | 3615 | **pending_review 状态下锁定 slot_quantity/course_id**（当前行 3711-3735 允许改，与 Resubmit 行 3427 锁定不一致） | 中（新发现） |
| `handlePayUniversalRecharge` | 4393 | 创建时立即到账（调用 applyOrderRecharge），审核通过只改状态 | 中 |
| `payMySlots` | 6244 | 按 course_id==0 区分万能充值（已提交） | 低 |
| `handlePayOrderDelete` | 3545 | 暂不改动，但 pending_review 删除会导致名额悬空（风险标记） | 低（风险） |


### 关键新发现（plan 原文未覆盖）

**发现1：handlePayOrderResubmit 重新提交不到账（行 3463）**

- 当前逻辑：rejected → 改字段 → `verification_status = "pending_review"` → `conn->update(ord)`
- **问题**：只改状态，没有任何账户操作。需求1后 rejected 名额已被追回清零，重新提交不重新到账 → 名额永久丢失
- **修复**：rejected 重新提交时，在 `conn->update(ord)` 后调用 `applyOrderRecharge()` 重新到账

**发现2：handlePayOrderModify 对 pending_review 没锁 slot_quantity/course_id（行 3711-3735）**

- 当前逻辑：`manual_verified`/`auto_verified` 不可修改（行 3676），但 `pending_review`/`rejected` 都可改全部字段
- 对比：`handlePayOrderResubmit` 行 3427 明确锁定了 pending_review 的 total_amount/slot_quantity/customer_id/course_id
- **问题**：需求1后 pending_review 名额已到账，通过 Modify 改 slot_quantity/course_id 会导致名额不一致
- **修复**：pending_review 状态下，Modify 也锁定 slot_quantity/course_id（与 Resubmit 对齐）
- 注意：Modify 接口不处理 session_id 字段，无需锁定

## 测试脚本影响评估（开发前确认）

### 核心机制变化对测试的影响

**关键变化**：`handlePayOrderCreateOffline` 创建时立即到账（原来是审核通过才到账），`handlePayOrderReview` approve 不再累加名额。

**辅助函数影响**：所有测试脚本都通过 `create_and_review_order()` 和 `universal_recharge()` 辅助函数充值（内部连续调用 创建+审核，中间无余额检查）。改动后：创建时到账 + 审核只改状态 = 审核后余额不变。**辅助函数无需修改**。

### 现有测试脚本影响矩阵

| 脚本 | 行数 | API匹配数 | 影响级别 | 说明 |
| --- | --- | --- | --- | --- |
| `test_withdraw.sh` | 2296 | 76 | ✅ 不受影响 | 全部使用 `create_and_review_order`，审核后检查余额，到账时机变化不影响最终值 |
| `test_gift_withdraw.sh` | 1436 | 11 | ✅ 不受影响 | 同上，辅助函数连续调用，中间无余额检查 |
| `test_proxy_enroll.sh` | 749 | 15 | ✅ 不受影响 | 同上 |
| `test_direct_gift_withdraw.sh` | - | 42 | ✅ 不受影响 | 转赠相关，使用辅助函数充值 |
| `test_bug_gift_chain.sh` | - | 6 | ✅ 不受影响 | Bug 修复测试，同模式 |
| `test_bug_chain_cleanup.sh` | - | 10 | ✅ 不受影响 | 链式清理测试，同模式 |
| `test_bug2_bug7.sh` | - | 6 | ⚠️ 需验证 | 可能有审核流程断言，需检查 |
| `test_bug_new.sh` | - | 4 | ⚠️ 需验证 | 可能有审核流程断言，需检查 |
| `test_stress.sh` | - | 7 | ⚠️ 需验证 | 压力测试，需确认并发审核不受影响 |
| `test_pay_api.sh` | 3951 | 74 | ⚠️ **需重点检查** | 最完整测试套件，可能有"审核前余额=0"或"审核后到账"的独立断言 |
| `lib_test.sh` | 161 | 0 | ✅ 不受影响 | 公共库，基线校验逻辑不变 |


### 需要重点验证的测试场景

**1. `test_pay_api.sh` 中的审核流程测试**

- 检查是否有"创建订单后审核前，账户余额为0"的断言 → 会失败
- 检查是否有"审核通过后账户余额增加N"的断言 → 会失败（余额在创建时已增加）
- **应对**：修改这些断言为"创建后余额=N"和"审核后余额不变"

**2. `test_bug2_bug7.sh` / `test_bug_new.sh`**

- 这些是修复特定 bug 的测试，可能包含审核流程的精确断言
- **应对**：运行一遍看哪些失败，针对性修改

### 需要新增的测试场景

#### 场景组1：基础到账验证

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 1.1 纯充值 | A下单3名额，不传session_id | 创建后A.total=3,consumed=0,avail=3；审核通过后不变 |
| 1.2 充值+自动报名 | A下单3名额，session_id=2 | 创建后A.total=3,consumed=3,avail=0；有enroll_consume流水 |
| 1.3 万能充值 | payUniversalRecharge 3名额 | 创建后A(课程0).total=3,consumed=0,avail=3；无需审核 |

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
| 3.1 转赠未领取 | A下单3,A转赠1给B(pending) → reject | withdraw_list空；confirm后A.total=2,frozen=0 |
| 3.2 转赠已领取未消耗 | A下单3,A转赠1给B(B领取未用) → reject | gift_revoke_list=[{B}]；confirm后A.total=2,B.total=0 |
| 3.3 转赠已领取已消耗 | A下单3,A转赠1给B,B报名1(session=2) → reject | withdraw_list=[{B,2}],gift_revoke_list=[{B}]；confirm后A.total=2,B.total=0,consumed=0 |
| 3.4 转赠混合 | A下单5,A自报1(s=2)+A代B报1(s=2)+A转赠1给C(未用)+A转赠2给D(D报1(s=2),剩1) → reject | withdraw_list=[{A,2},{B,2},{D,2}]；confirm后A/C/D全归零 |

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

#### 场景组6：自动报名验证

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 6.1 session_id=0 | A下单3,不传session_id | consumed=0,无enroll_consume流水 |
| 6.2 session_id>0 | A下单3,session_id=2 | consumed=3,有enroll_consume流水,session_id=2 |
| 6.3 自动报名后手动再报 | A下单3,session=2(自动3) → A再deduct 1 | deduct失败（余额不足） |

#### 场景组7：异常恢复与幂等

| 场景 | 操作 | 验证点 |
|------|------|--------|
| 7.1 confirm重试 | A下单3→reject→confirm(网络失败)→confirm(重试) | 第二次成功，名额只扣一次 |
| 7.2 confirm中途事务失败 | 模拟confirm事务中途失败 | 事务回滚，订单保持rejected，名额不变 |
| 7.3 僵尸订单告警 | reject后1小时未confirm | 后端启动扫描到，写WARNING日志 |
| 7.4 人工触发confirm | reject后人工调confirm(带operator_id) | 成功执行扣减，标记rejected_confirmed |
| 7.5 rejected超时禁止confirm | reject后超过24小时调confirm | 返回400，需人工介入 |


### 测试策略

1. **先跑现有测试**：改动后先运行 `test_withdraw.sh`、`test_gift_withdraw.sh`、`test_proxy_enroll.sh`，确认退课/转赠/代报名链路不受影响
2. **修复 `test_pay_api.sh`**：针对性修改审核流程断言
3. **新增 `test_offline_reject.sh`**：覆盖订单拒绝追回全链路
4. **新增 `test_auto_enroll.sh`**：覆盖创建时自动报名
5. **回归测试**：全部测试脚本通过后再提交

## 已识别的问题与应对

### 问题1：`handlePayOrderDelete` 状态限制

**问题**：行 3545 限制只有 `pending_review` 或 `rejected` 状态可删除。需求1后，pending_review 状态的名额已到账，删除 pending_review 订单会导致名额悬空。
**应对方案**：先保留删除功能，后续决定是否触发追回或移除删除功能。当前阶段不改动 `handlePayOrderDelete`。

### 问题2：`handlePayOrderModify` 限制

**问题**：行 3425-3432，pending_review 状态锁定金额/客户字段。需求1后 pending_review 名额已到账，修改 slot_quantity/course_id/session_id 会导致名额不一致。
**应对方案**：pending_review 状态下，**锁定 `course_id`、`session_id`、`slot_quantity` 三个字段不可修改**（这三个字段直接影响名额）。其他字段（备注、凭证等）可正常修改。

### 问题3：退款限制冲突（重要）

**问题**：行 2231，`handlePayRefund` 限制 `verification_status == "pending_review"` 不可退款。需求1后，pending_review 状态名额已到账且用户可自由支配，但退款被禁止。这意味着：用户用了名额，财务还没审核，无法退款。
**应对方案**：保持现状（pending_review 不可退款）。业务逻辑上，审核拒绝走追回流程，不需要退款；审核通过后才能退款。这与现有设计一致，无需改动。

### 问题4：万能充值订单的 type 字段（已确认）

**问题**：`handlePayUniversalRecharge`（行 4425）`type = "recharge"`，而 `payMySlots`（行 6335）按 `type == "universal_recharge"` 区分万能充值。
**确认方案**：统一用 `type="recharge"` + `course_id==0` 区分万能充值，不需要 `universal_recharge` 类型。`payMySlots` 改为按 `course_id==0` 判断万能充值，不再依赖 `type` 字段。

### 问题5：名额是否区分万能/普通（已确认）

**确认**：名额通过 `course_id` 区分——

- `course_id = 0` → 万能账户（万能名额，可用于任意课程报名）
- `course_id > 0` → 课程账户（普通名额，只能用于该课程）
- 报名时优先用课程账户（`from_course`），不足用万能账户（`from_universal`）
- 追回时也需分别处理两个账户：查 `course_id IN (cid, 0)` 的消耗

### 问题6：reject 未确认状态的订单操作限制

**问题**：`rejected`（未 confirm）状态的订单，是否允许重新提交（resubmit）？
**应对方案**：`rejected` 未确认时不允许 resubmit（返回400提示"请先完成追回确认"）；`rejected_confirmed` 才允许 resubmit。

### 问题7：`handlePayOrderClose` 不受影响

**确认**：行 1811，`handlePayOrderClose` 只处理 `status == "pending_payment"` 的订单（小程序未支付订单）。线下订单创建时 `status="paid"`，不受影响。无需改动。

### 问题8：多订单叠加追回（已确认）

**确认**：名额同质，不区分来自哪个订单。reject 追回 N=订单.slot_quantity，从当前 available 优先扣，不足 LIFO 追回。即使账户有多个订单充值，也能正确处理。

### 问题9：confirm 失败的重试

**问题**：confirm 执行扣减时部分失败（如某个 receiver 账户异常）怎么办？
**应对方案**：confirm 在一个事务内执行所有扣减，要么全部成功要么全部回滚。失败时订单保持 `rejected` 状态，前端可重试。幂等保证重试安全。

### 问题10：万能名额订单的追回

**问题**：如果订单是万能充值（course_id=0），追回时如何处理？
**应对方案**：
- 查 gift_records：`WHERE giver_id=A AND course_id=0`
- 查 consumed：`WHERE customer_id=A AND course_id=0 AND type IN ('enroll_consume','universal_consume','transfer_out')`
- 但 receiver 用万能名额报名时，consumed_session_id 记录的是实际报名的课程 session。withdraw_list 里返回的是 receiver_id + session_id（报名时的 session），不涉及 course_id
- confirm 扣减：A 端扣万能账户（course_id=0），receiver 端扣其万能账户（course_id=0）

### 问题11：reject 与 confirm 之间的状态变化

**问题**：reject 返回清单后，confirm 之前用户又操作了（转赠/报名），导致清单与实际不符？
**应对方案**：confirm 重新查询当前状态执行扣减，不依赖 reject 时的清单。如果 available 增加了（用户退课了），多扣的 available 部分 total-=1 即可；如果 available 减少了（用户又报名了），需要追回更多 consumed。乐观锁检测并发变化，失败提示重试。

### 问题12：前端删报名成功但 confirm 未到达后端（网络丢失）

**问题**：前端调另一台服务器删报名成功，但调本服务器 confirm 时网络丢失，后端没收到。
**结果**：另一台服务器报名已删，本服务器订单还是 `rejected`（未 confirm），名额没扣。
**应对方案**：
- **前端重试**：confirm 幂等设计，前端应实现重试机制（如重试3次，间隔递增）
- **后端兜底扫描**：后端启动时 + 定时任务扫描长时间处于 `rejected`（未 confirm）的订单，记录告警日志，通知人工处理
- **超时标记**：reject 超过一定时间（如24小时）仍未 confirm，标记为 `rejected_timeout`，禁止后续 confirm（需人工介入）

### 问题13：confirm 执行中途后端宕机（事务中断）

**问题**：confirm 事务执行到一半后端挂了。
**应对方案**：MySQL 事务保证原子性——未 commit 的事务在重启后自动回滚。订单状态保持 `rejected`，前端重试 confirm 即可。**无数据不一致风险**。

### 问题14：后端重启后的僵尸订单恢复

**问题**：后端重启后，可能有订单卡在 `rejected`（未 confirm）状态，另一台服务器报名已删但本服务器名额未扣。
**应对方案**：
- 后端启动时执行 `SELECT order_no FROM orders WHERE verification_status='rejected' AND reviewed_at < DATE_SUB(NOW(), INTERVAL 1 HOUR)`
- 对查到的订单写 CROW_LOG_WARNING 告警
- 不自动 confirm（因为不确定前端是否删报名成功），仅告警通知人工处理
- 提供 admin 接口 `POST /api/payOrderReclaimConfirm`（带 operator_id）供人工触发 confirm

## Design Style

本需求不涉及 UI 改造，无需设计页面。

# Agent Extensions

<!-- 无可用扩展工具，不输出此节 -->
