# 课程培训平台 - 产品需求文档 (PRD)

**版本**: v1.5

**日期**: 2026-04-22

**状态**: 修订稿

------

## 1. 产品概述

### 1.1 产品定位

面向职业技能培训的 B2C 课程平台，支持微信小程序端用户购课、签到，以及后台管理系统进行课程运营和学员管理。

### 1.2 目标用户

- **学员**：通过微信小程序下单、查看订单、参与培训的 C 端用户
- **讲师**：授课并在后台查看自己课期学员与签到情况的专业人士
- **运营人员**：负责课程上下架、课期安排、客户资料维护、签到管理、座位管理的工作人员
- **业务员**：负责客户对接、课程销售、线下订单录入、客户绑定和业绩跟进的工作人员，仅管理自己绑定的客户数据
- **财务人员**：负责订单核销、退款处理、财务对账的工作人员
- **管理员**：拥有系统全部权限，管理账号、分配权限

   ------

## 2. 用户角色与权限

   | 角色         | 权限范围                                                     |
   | :----------- | :----------------------------------------------------------- |
   | **学员**     | 浏览课程、下单支付、查看我的订单、查看已生效课期、扫码签到、发起转赠、发起退款申请 |
   | **讲师**     | 查看自己授课的课期、查看课期学员列表、查看签到情况           |
   | **运营人员** | 客户资料维护、课程/课期管理、签到管理、会场座位管理、客户课期查询 |
   | **业务员**   | 客户管理（仅自己绑定的客户）、课程推荐、线下创建订单、查看个人业绩统计、查看自己客户的课期信息 |
   | **财务人员** | 订单核销、退款处理、财务报表查看                             |
   | **管理员**   | 全部权限 + 账号管理（创建/禁用工作人员账号）、权限分配（可为非管理员用户设置角色如财务、业务员等）、数据导出 |

   ------

## 3. 功能模块

### 3.1 微信小程序（C 端）

#### 3.1.1 用户注册

   - 微信用户首次进入小程序，自动完成注册
   - 注册信息：微信 OpenID、昵称、头像、注册时间
   
#### 3.1.2 课程浏览

   - 课程列表：展示所有上架课程（名称、简介、封面、价格区间）
   - 课程详情：课程介绍、课期列表、价格、名额情况
   
#### 3.1.3 课期购买

   - 选择课期 → 确认订单 → 微信支付 → 订单进入待核销
   - 支付成功仅代表订单状态变为「已支付」，需财务核销通过后才创建客户课期记录与名额明细
   - **名额转赠功能（小程序）**：仅已核销且仍有效的名额可转赠，选择系统内客户作为受赠人，转赠立即生效
   - 课期退款：按订单维度申请，需财务审核；仅已核销、未转赠、未签到、未分配座位的名额可退款

#### 3.1.4 我的订单

   - 展示订单列表（按创建时间倒序）
   - 订单状态：待支付、待核销、已生效、已拒绝、退款中、已退款、已关闭
   - 支持查看订单支付时间、核销进度、拒绝原因、退款进度
   
#### 3.1.5 我的课期

   - 仅展示已核销生效且当前仍持有有效名额的课期列表（按时间倒序）
   - 课期状态：待上课、进行中、已完成
   - 签到入口：课期详情页展示专属签到二维码，仅对当前有效名额开放
   - **转赠管理**：可查看自己转赠给他人的记录，以及他人转赠给自己的记录
   
#### 3.1.6 扫码签到

   **流程设计（双二维码防代签机制）：**

```mermaid

   flowchart LR
       A[工作人员<br/>展示课期二维码] --> B[学员端<br/>扫码识别]
       B --> C[核销端<br/>生成个人码]
       C --> D[工作人员扫码<br/>完成核销]

       style A fill:#e1f5ff
       style B fill:#e8f5e9
       style C fill:#fff3e0
       style D fill:#fce4ec
```

1. 工作人员在后台生成课期签到二维码
2. 学员使用小程序扫描课期二维码，服务端为其锁定一个可签到的有效名额，生成个人专属签到二维码（含 registration_slot_id、学员 ID、时间戳）
3. 工作人员使用核销端扫描学员个人二维码，完成签到

------

### 3.2 后台管理系统（B 端）

#### 3.2.1 客户管理

- **客户列表**：展示所有注册客户（微信昵称、头像、注册时间、购买次数）
- **客户详情**：基本信息、订单记录、已生效课期、签到记录、**绑定业务员**
- **手动添加**：运营人员可手动录入客户（用于线下转化场景）
- **业务员绑定**：每个客户可绑定一个业务员，用于业绩统计

#### 3.2.2 课程管理

**课程（Course）**

- 字段：课程 ID、课程名称、课程简介、封面图、创建人、创建时间、状态（上架/下架）、归属类别、排序优先级
- 操作：新增、编辑、上架、下架、删除（无关联课期时）

**课期（Session）**

- 字段：课期 ID、所属课程、课期名称、讲师、授课时间、地点、名额、报名人数、课程价格、状态、创建人、创建时间
- 操作：新增、编辑、取消、删除（无报名记录时）

**课期状态流转：**

```mermaid
stateDiagram-v2
    [*] --> 未开始 : 创建课期
    未开始 --> 报名中 : 手动发布
    报名中 --> 已截止 : 手动截止
    报名中 --> 已截止 : 自动（开课1小时前）
    报名中 --> 进行中 : 自动（到达开课时间）
    已截止 --> 进行中 : 自动（到达开课时间）
    进行中 --> 已结束 : 自动（授课完成）
    未开始 --> 已取消 : 手动取消
    报名中 --> 已取消 : 手动取消

    note right of 未开始
        可编辑所有字段
    end note
    
    note right of 报名中
        学员可购买
    end note
```

- **未开始**：课期创建后的初始状态，可编辑所有字段
- **报名中**：学员可下单，运营人员可手动设为「已截止」
- **已截止**：停止报名，自动或手动触发
- **进行中**：到达授课时间自动切换
- **已结束**：授课完成（自动）
- **已取消**：手动取消，已购学员需线下处理（财务核销）

#### 3.2.3 签到管理

- **签到二维码生成**：为每个课期生成唯一签到二维码
- **签到记录**：查看课期签到列表（学员、名额、签到时间）
- **手动签到**：运营人员可为特殊情况按名额手动标记签到

#### 3.2.4 会场座位

- **分组管理**：为课期创建座位分组（如 1 组、2 组、3 组 或 A 组、B 组、C 组）
- **学员分配**：工作人员按已生效名额将学员分配到指定分组
- **转赠学员分组**：转赠的学员可与其赠送人分配在同一分组，便于现场组织
- **用途**：现场组织、分组讨论、座位引导

#### 3.2.5 课程充值

- **充值**：客户对指定课期预存名额，支持小程序充值和业务员线下创建
- **名额计算**：充值金额 ÷ 课期单价 = 名额数量
- **使用场景**：企业团购、家长为孩子预存课时
- **统一订单模式**：购课和充值都走 Order 表，全部需财务核销后生效
- **业务员线下充值**：业务员为客户手动创建充值订单，直接标记为已支付（现金/转账已收），等待财务核销
- **财务核销**：所有订单（购课/充值）支付成功后状态为已支付，财务人员核销后变为已核销，同时创建报名汇总和名额明细
- **退款处理**：仅支持退订单下仍归属于当前客户、未转赠、未签到、未分配座位的剩余名额，已转赠部分不可退款

#### 3.2.6 客户课期管理（培训报名）

- **报名汇总（Registration）**：按客户 + 课期 + 业绩归属业务员聚合的报名记录
- **名额明细（RegistrationSlot）**：每个已核销名额一条记录，作为签到、座位、转赠、退款的最小操作单元
- **状态**：Registration = 有效 / 已清空 / 已取消；RegistrationSlot = 有效 / 已签到 / 已退款 / 已取消
- **业务员业绩**：按名额明细的根业务员归属统计，转赠不改变业绩归属
- SessionSegment 创建：课期创建时默认生成一个「全场次」，工作人员可再添加其他场次（如上午场、下午场）

#### 3.2.7 账号管理（仅管理员）

- **工作人员列表**：讲师、业务员、财务人员账号管理
- **角色分配**：创建账号时指定角色，管理员可随时调整非管理员用户的权限
- **状态控制**：启用 / 禁用账号

------

## 4. 数据模型

### 4.1 核心实体关系

```mermaid

erDiagram
    Customer ||--o{ Order : "下单"
    Session ||--o{ Order : "购买"
    Order ||--o{ Payment : "支付"
    Customer ||--o{ Registration : "拥有报名汇总"
    Session ||--o{ Registration : "被报名"
    Registration ||--o{ RegistrationSlot : "包含名额"
    Order ||--o{ RegistrationSlot : "核销产生名额"
    Course ||--o{ Session : "包含"
    User ||--o{ Session : "创建"
    User ||--o{ Session : "授课"
    Session ||--o{ SeatGroup : "分组"
    SeatGroup ||--o{ SeatAssignment : "分组座位"
    RegistrationSlot ||--o{ SeatAssignment : "座位分配"
    Session ||--o{ SessionSegment : "包含场次"
    RegistrationSlot ||--o{ CheckIn : "产生签到"
    SessionSegment ||--o{ CheckIn : "场次签到"
    User ||--o{ CheckIn : "核销"
    User ||--o{ Order : "核销"
    Customer }|--|| User : "绑定业务员"
    Registration }|--|| User : "业绩归属"
    Registration ||--o{ GiftRecord : "转赠记录"
    Customer ||--o{ GiftRecord : "作为受赠人"
    Order ||--o{ Refund : "退款记录"

    Order {
        string id PK "订单ID"
        string order_no UK "订单号"
        enum type "类型：购课/充值"
        enum source "来源：小程序/线下"
        string customer_id FK "客户ID"
        string session_id FK "课期ID"
        int slots "名额数量"
        decimal amount "订单金额"
        enum status "状态：待支付/已支付/已关闭/已核销/已拒绝/退款中/已退款"
        string verified_by FK "核销人ID"
        datetime verified_at "核销时间"
        datetime expire_at "支付截止时间"
        datetime paid_at "支付时间"
        string created_by_ip "下单IP"
        string created_by FK "创建人ID（线下充值时）"
        datetime created_at "创建时间"
    }
    
    Payment {
        string id PK "支付ID"
        string order_id FK "订单ID"
        string channel_trade_no UK "渠道交易号"
        string channel_prepay_id "渠道预支付ID"
        enum channel "支付渠道：wechat"
        enum status "状态：待确认/成功/失败"
        string callback_raw "回调报文摘要"
        datetime callback_at "回调时间"
        datetime created_at "创建时间"
    }
    
    Customer {
        string id PK "客户ID"
        string wx_openid UK "微信OpenID"
        string nickname "昵称"
        string avatar "头像"
        string sales_agent_id FK "绑定业务员ID"
        datetime created_at "注册时间"
    }
    
    Registration {
        string id PK "记录ID"
        string customer_id FK "客户ID"
        string session_id FK "课期ID"
        string sales_agent_id FK "业绩归属业务员ID（转赠后不变）"
        int total_slots "当前归属名额总数"
        int active_slots "当前可用名额数"
        enum status "状态：有效/已清空/已取消"
        datetime created_at "报名时间"
    }

    RegistrationSlot {
        string id PK "名额ID"
        string root_order_id FK "来源订单ID"
        string registration_id FK "当前归属报名汇总ID"
        string session_id FK "课期ID"
        string owner_customer_id FK "当前持有人ID"
        string sales_agent_id FK "业绩归属业务员ID"
        enum source "来源：购课/线下/转赠"
        enum status "状态：有效/已签到/已退款/已取消"
        datetime created_at "创建时间"
    }
    
    Session {
        string id PK "课期ID"
        string course_id FK "课程ID"
        string name "课期名称"
        string instructor_id FK "讲师ID"
        datetime start_time "授课时间"
        string location "授课地点"
        int capacity "名额上限"
        int enrolled_count "已生效名额数（有效+已签到）"
        decimal price "价格"
        enum status "状态"
        int version "乐观锁版本号"
        string created_by FK "创建人ID"
        datetime created_at "创建时间"
    }
    
    Course {
        string id PK "课程ID"
        string name "课程名称"
        string description "简介"
        string cover_image "封面图"
        enum status "状态：上架/下架"
        string category_id FK "分类ID"
        int sort_order "排序优先级"
        string created_by FK "创建人ID"
        datetime created_at "创建时间"
    }

    CourseCategory {
        string id PK "分类ID"
        string name "分类名称"
        string parent_id FK "父分类ID"
        int sort_order "排序"
        datetime created_at "创建时间"
    }
    
    User {
        string id PK "用户ID"
        string name "姓名"
        string phone UK "手机号"
        enum role "角色：讲师/运营/业务员/财务/管理员"
        enum status "状态：启用/禁用"
        datetime created_at "创建时间"
    }
    
    CheckIn {
        string id PK "签到ID"
        string registration_slot_id FK "名额ID"
        string session_id FK "课期ID"
        string session_segment_id FK "场次ID"
        string customer_id FK "客户ID"
        datetime checked_in_at "签到时间"
        string checked_in_by FK "核销人员ID"
    }
    
    
    SeatGroup {
        string id PK "分组ID"
        string session_id FK "课期ID"
        string name "分组名称"
        int capacity "分组容量"
        datetime created_at "创建时间"
    }

    SeatAssignment {
        string id PK "分配ID"
        string registration_slot_id FK "名额ID"
        string seat_group_id FK "分组ID"
        datetime assigned_at "分配时间"
        string assigned_by FK "分配人ID"
    }

    SessionSegment {
        string id PK "场次ID"
        string session_id FK "课期ID"
        string name "场次名称"
        datetime start_time "开始时间"
        datetime end_time "结束时间"
        datetime created_at "创建时间"
    }

    GiftRecord {
        string id PK "转赠记录ID"
        string from_registration_id FK "赠与人报名记录ID"
        string to_registration_id FK "受赠人报名记录ID"
        string from_customer_id FK "赠与人ID"
        string to_customer_id FK "受赠人ID"
        int slots "转赠名额数量"
        string slot_ids_snapshot "本次转赠名额ID快照"
        datetime created_at "转赠时间"
    }

    Refund {
        string id PK "退款记录ID"
        string order_id FK "关联订单ID"
        decimal amount "退款金额"
        int refunded_slots "退款名额数量"
        enum type "类型：全额/部分"
        enum channel "渠道：原路退回/线下处理"
        string slot_ids_snapshot "本次退款名额ID快照"
        string reason "退款原因"
        string handled_by FK "处理人ID"
        datetime created_at "退款时间"
    }
```

### 4.2 字段定义

**Customer（客户）**

| 字段           | 类型     | 说明          |
| :------------- | :------- | :------------ |
| id             | string   | 客户 ID       |
| wx_openid      | string   | 微信 OpenID   |
| nickname       | string   | 微信昵称      |
| avatar         | string   | 头像 URL      |
| sales_agent_id | string   | 绑定业务员 ID |
| created_at     | datetime | 注册时间      |

**Course（课程）**

| 字段        | 类型     | 说明            |
| :---------- | :------- | :-------------- |
| id          | string   | 课程 ID         |
| name        | string   | 课程名称        |
| description | text     | 课程简介        |
| cover_image | string   | 封面图 URL      |
| status      | enum     | 状态：上架/下架 |
| category_id | string   | 分类 ID         |
| sort_order  | int      | 排序优先级      |
| created_by  | string   | 创建人 ID       |
| created_at  | datetime | 创建时间        |

**CourseCategory（课程分类）**

| 字段       | 类型     | 说明                  |
| :--------- | :------- | :-------------------- |
| id         | string   | 分类 ID               |
| name       | string   | 分类名称              |
| parent_id  | string   | 父分类 ID（支持层级） |
| sort_order | int      | 排序序号              |
| created_at | datetime | 创建时间              |

**Order（订单）**

| 字段                | 类型     | 说明                              |
| :------------------ | :------- | :-------------------------------- |
| id                  | string   | 订单 ID                           |
| order_no            | string   | 业务订单号（唯一）                |
| type                | enum     | 类型：购课/充值                   |
| source              | enum     | 来源：小程序/线下                 |
| customer_id         | string   | 客户 ID                           |
| session_id          | string   | 课期 ID                           |
| slots               | int      | 名额数量（充值时 > 1）            |
| amount              | decimal  | 订单金额                          |
| status              | enum     | 状态：待支付/已支付/已关闭/已核销/已拒绝/退款中/已退款 |
| verified_by         | string   | 核销人 ID（已核销/已拒绝时填充）  |
| verified_at         | datetime | 核销时间                          |
| expire_at           | datetime | 支付截止时间                      |
| paid_at             | datetime | 实际支付时间                      |
| created_by_ip       | string   | 下单来源 IP                       |
| created_by          | string   | 创建人 ID（线下充值时）           |
| created_at          | datetime | 创建时间                          |

**Payment（支付流水）**

| 字段              | 类型     | 说明                   |
| :---------------- | :------- | :--------------------- |
| id                | string   | 支付流水 ID            |
| order_id          | string   | 订单 ID                |
| channel_trade_no  | string   | 微信交易号（唯一）     |
| channel_prepay_id | string   | 微信预支付单号         |
| channel           | enum     | 渠道：wechat           |
| status            | enum     | 状态：待确认/成功/失败 |
| callback_raw      | text     | 支付回调关键报文摘要   |
| callback_at       | datetime | 回调时间               |
| created_at        | datetime | 创建时间               |

**Session（课期）**

| 字段             | 类型       | 说明                         |
| :------------- | :------- | :------------------------- |
| id             | string   | 课期 ID                      |
| course_id      | string   | 所属课程 ID                    |
| name           | string   | 课期名称（如：第 1 期）              |
| instructor_id  | string   | 讲师 ID                      |
| start_time     | datetime | 授课时间                       |
| location       | string   | 授课地点                       |
| capacity       | int      | 名额上限                       |
| enrolled_count | int      | 已生效名额数（有效+已签到）     |
| price          | decimal  | 课程价格（元）                    |
| status         | enum     | 状态：未开始/报名中/已截止/进行中/已结束/已取消 |
| version        | int      | 乐观锁版本号（并发扣减名额）             |
| created_by     | string   | 创建人 ID                     |
| created_at     | datetime | 创建时间                       |

**Registration（客户课期/报名记录）**

| 字段           | 类型     | 说明                                    |
| :------------- | :------- | :-------------------------------------- |
| id             | string   | 记录 ID                                 |
| customer_id    | string   | 客户 ID                                 |
| session_id     | string   | 课期 ID                                 |
| sales_agent_id | string   | 业绩归属业务员 ID（转赠后不变）         |
| total_slots    | int      | 当前归属名额总数                         |
| active_slots   | int      | 当前可用名额数                           |
| status         | enum     | 状态：有效/已清空/已取消                 |
| created_at     | datetime | 报名时间                                |

**RegistrationSlot（报名名额明细）**

| 字段             | 类型     | 说明                               |
| :--------------- | :------- | :--------------------------------- |
| id               | string   | 名额 ID                            |
| root_order_id    | string   | 来源订单 ID                        |
| registration_id  | string   | 当前归属报名汇总 ID                |
| session_id       | string   | 课期 ID                            |
| owner_customer_id| string   | 当前持有人 ID                      |
| sales_agent_id   | string   | 业绩归属业务员 ID                  |
| source           | enum     | 来源：购课/线下/转赠               |
| status           | enum     | 状态：有效/已签到/已退款/已取消    |
| created_at       | datetime | 创建时间                           |

**CheckIn（签到记录）**

| 字段                | 类型     | 说明                    |
| :------------------ | :------- | :---------------------- |
| id                  | string   | 签到 ID                 |
| registration_slot_id| string   | 名额 ID                 |
| session_id          | string   | 课期 ID                 |
| session_segment_id  | string   | 场次 ID（多场签到支持） |
| customer_id         | string   | 客户 ID                 |
| checked_in_at       | datetime | 签到时间                |
| checked_in_by       | string   | 核销人员 ID             |

**SeatAssignment（座位分配）**

| 字段            | 类型     | 说明        |
| :-------------- | :------- | :---------- |
| id              | string   | 分配 ID     |
| registration_slot_id | string | 名额 ID   |
| seat_group_id   | string   | 分组 ID     |
| assigned_at     | datetime | 分配时间    |
| assigned_by     | string   | 分配人 ID   |

**SessionSegment（课期场次）**

| 字段       | 类型     | 说明     |
| :--------- | :------- | :------- |
| id         | string   | 场次 ID  |
| session_id | string   | 课期 ID  |
| name       | string   | 场次名称 |
| start_time | datetime | 开始时间 |
| end_time   | datetime | 结束时间 |
| created_at | datetime | 创建时间 |

**GiftRecord（转赠记录）**

| 字段                | 类型     | 说明                    |
| :------------------ | :------- | :---------------------- |
| id                  | string   | 转赠记录 ID             |
| from_registration_id | string  | 赠与人报名记录 ID       |
| to_registration_id  | string   | 受赠人报名记录 ID       |
| from_customer_id    | string   | 赠与人 ID               |
| to_customer_id      | string   | 受赠人 ID               |
| slots               | int      | 转赠名额数量            |
| slot_ids_snapshot   | string   | 本次转赠名额ID快照      |
| created_at          | datetime | 转赠时间                |

**Refund（退款记录）**

| 字段                | 类型     | 说明                    |
| :------------------ | :------- | :---------------------- |
| id                  | string   | 退款记录 ID             |
| order_id            | string   | 关联订单 ID             |
| amount              | decimal  | 退款金额                |
| refunded_slots      | int      | 退款名额数量            |
| type                | enum     | 类型：全额/部分         |
| channel             | enum     | 渠道：原路退回/线下处理 |
| slot_ids_snapshot   | string   | 本次退款名额ID快照      |
| reason              | string   | 退款原因                |
| handled_by          | string   | 处理人 ID               |
| created_at          | datetime | 退款时间                |

**SeatGroup（会场座位分组）**

| 字段       | 类型     | 说明                               |
| :--------- | :------- | :--------------------------------- |
| id         | string   | 分组 ID                            |
| session_id | string   | 课期 ID                            |
| name       | string   | 分组名称（如：1 组、2 组、A 组、B 组） |
| capacity   | int      | 分组容量                           |
| created_at | datetime | 创建时间                           |

### 4.3 关键约束与索引建议

- 唯一约束：`Registration(customer_id, session_id, sales_agent_id)`，同一客户在同一课期、同一业绩归属下仅保留一条有效报名汇总
- 唯一约束：`CheckIn(registration_slot_id, session_segment_id)`，防止同一名额同一场次重复签到
- 签到状态以 `CheckIn` 和 `RegistrationSlot.status` 为准，`Registration` 只维护汇总数量
- 唯一约束：`Order(order_no)`、`Payment(channel_trade_no)`，保障交易幂等。
- 索引：`Session(status, start_time)`，用于课期列表和状态任务扫描。
- 索引：`Registration(session_id, status)`，用于课期汇总查询。
- 索引：`RegistrationSlot(session_id, status)`，用于签到和座位分配加载名额名单。
- 索引：`RegistrationSlot(owner_customer_id, status)`，用于小程序我的课期和转赠校验。
- 索引：`Order(customer_id, created_at)`，用于用户订单查询和对账。
- 索引：`Registration(sales_agent_id, created_at)`，用于业务员业绩统计。
- 数据一致性：`Session.enrolled_count`、`Registration` 汇总、`RegistrationSlot` 明细需放在同一事务中提交。

------

## 5. 关键业务流程

### 5.1 购课流程

```mermaid
flowchart TD
    A[学员浏览课程] --> B[选择课期并提交订单]
    B --> C[服务端校验课期状态]
    C --> D{状态=报名中 且名额>0?}
    D -->|否| E[返回不可购买]
    D -->|是| F[创建待支付订单<br/>15分钟过期]
    F --> G[调用微信预下单]
    G --> H[学员发起支付]
    H --> I{支付回调/主动查询}
    I -->|成功| J[更新订单为已支付]
    J --> K[状态: 已支付]
    K --> L[财务核销]
    L -->|通过| M[事务: 查找或创建 Registration]
    M --> N[事务: 批量创建 RegistrationSlot]
    N --> O[事务: 重算 Registration 汇总与 enrolled_count]
    O --> P[状态: 已核销]
    I -->|失败| Q[Payment 记为失败<br/>Order 保持待支付]
    I -->|超时| R[关闭订单]
    L -->|拒绝| S[状态: 已拒绝]
    P --> T[返回购买成功]

    style A fill:#e3f2fd
    style P fill:#e8f5e9
    style T fill:#e8f5e9
    style E fill:#ffebee
    style R fill:#ffebee
    style S fill:#ffebee
```

**异常处理：**

- 并发超卖：创建订单时只校验可售状态，财务核销时按名额做最终校验并落明细。
- 重复回调：按 `order_no` 和 `channel_trade_no` 幂等处理，重复通知直接返回成功。
- 支付失败：仅记录在 `Payment.status`，订单保持待支付并允许用户重试，直到超时关闭。
- 支付超时：超过 `expire_at` 自动关闭订单，避免脏订单长期占用资源。
- 核销拒绝：财务拒绝后订单状态为"已拒绝"
  - 线下订单：业务员可修改信息后重新提交核销申请
  - 小程序订单：财务手动发起退款（原路退回），订单关闭

### 5.2 订单状态流转

```mermaid
stateDiagram-v2
    [*] --> 待支付 : 创建订单（小程序）
    [*] --> 已支付 : 创建订单（线下）
    待支付 --> 已支付 : 支付成功
    待支付 --> 已关闭 : 用户取消/超时关闭
    已支付 --> 已核销 : 财务审核通过
    已支付 --> 已拒绝 : 财务审核拒绝
    已拒绝 --> 已支付 : 线下订单修改重提
    已拒绝 --> 退款中 : 发起退款
    已核销 --> 退款中 : 发起退款
    退款中 --> 已退款 : 退款完成
    退款中 --> 已关闭 : 全额退款关闭
    已退款 --> [*]
    已关闭 --> [*]

    note right of 待支付
        15分钟支付超时
        自动关闭订单
        支付失败仅记录Payment
    end note

    note right of 已支付
        等待财务核销
        核销后名额生效
    end note

    note right of 已拒绝
        线下：可修改重提
        小程序：退款关闭
    end note

    note right of 退款中
        财务处理退款
        支持部分退款
    end note
```

**订单状态说明：**

| 状态 | 说明 |
| :--- | :--- |
| **待支付** | 订单创建，等待用户支付（仅小程序订单） |
| **已支付** | 支付成功，等待财务核销 |
| **已关闭** | 订单关闭（取消/超时/全额退款） |
| **已核销** | 财务审核通过，报名汇总与名额明细生效 |
| **已拒绝** | 财务审核拒绝，等待处理 |
| **退款中** | 退款申请处理中 |
| **已退款** | 部分退款完成，订单仍保留已生效部分 |

**退款独立记录：**

| 字段 | 说明 |
| :--- | :--- |
| id | 退款记录ID |
| order_id | 关联订单ID |
| amount | 退款金额 |
| type | 类型：全额/部分 |
| channel | 渠道：原路退回/线下处理 |
| reason | 退款原因 |
| handled_by | 处理人ID |
| created_at | 退款时间 |

**退款规则：**
- 未核销订单：直接退款，订单关闭，不产生报名数据
- 已核销订单全额退款：退款该订单下全部可退款名额，订单关闭
- 已核销订单部分退款：仅退款该订单下部分可退款名额，订单变为"已退款"
- 线下订单退款：财务手动处理，记录日志

### 5.3 签到流程

```mermaid
flowchart TD
    subgraph 课前准备
        A[工作人员生成<br/>课期签到二维码] --> B[展示于会场入口]
    end

    subgraph 签到过程
        C[学员打开微信] --> D[扫描课期二维码]
        D --> E[生成短时效个人签到码<br/>包含 registration_slot_id + nonce]
        E --> F[工作人员使用核销端扫码]
        F --> G{校验有效性}

        G -->|未报名| H[提示未报名]
        G -->|已签到| I[提示重复签到]
        G -->|课期不符| J[提示课期错误]
        G -->|码过期/伪造| M[提示二维码无效]
        G -->|有效| K[事务: 写入签到记录]
        K --> L[返回签到成功]
    end

    B -.-> C

    style H fill:#ffebee
    style I fill:#ffebee
    style J fill:#ffebee
    style M fill:#ffebee
    style L fill:#e8f5e9
```

**校验规则：**

- 学员必须持有该课期至少一个 `status=有效` 的名额
- 每个名额每场次只能签到一次，同一课期不同场次可重复签到
- 核销时服务端锁定一个有效名额并写入签到记录，同时将该名额状态更新为「已签到」
- 个人签到码有效期可配置（默认 120 秒），核销后立即失效，防截图转发
- 不限制签到时间（课前课后均可，由现场灵活控制）

### 5.4 充值流程（统一订单模式）

```mermaid
flowchart TD
    A[客户提出充值需求] --> B{充值方式}
    B -->|小程序在线充值| C[创建充值订单<br/>type=充值,source=小程序]
    B -->|业务员线下对接| D[业务员创建充值订单<br/>type=充值,source=线下]
    C --> E[调用微信支付]
    E --> F{支付结果}
    F -->|成功| G[订单状态: 已支付]
    F -->|失败| H[Payment 记为失败<br/>Order 保持待支付]
    F -->|超时| I[关闭订单]
    D --> J[订单状态: 已支付<br/>业务员标记已收款]
    G --> K[等待财务核销]
    J --> K
    K --> L[财务人员核销]
    L -->|通过| M[事务: 查找或创建 Registration]
    M --> N[事务: 批量创建 RegistrationSlot]
    N --> O[事务: 重算 Registration 汇总与 enrolled_count]
    O --> P[状态: 已核销]
    L -->|拒绝| R[状态: 已拒绝]
    P --> Q[充值完成]

    style A fill:#e3f2fd
    style Q fill:#e8f5e9
    style P fill:#e8f5e9
    style H fill:#ffebee
    style I fill:#ffebee
    style R fill:#ffebee
```

### 5.5 名额转赠流程

**转赠规则：**
- 核销后才能转赠（Order.status = 已核销）
- 支持部分转赠、多次转赠（不限制次数）
- 仅 `RegistrationSlot.status = 有效` 且未分配座位的名额可转赠
- 业绩始终归属最开始的业务员（Registration.sales_agent_id 和 RegistrationSlot.sales_agent_id 不变）
- 转赠立即生效，受赠人无需操作

```mermaid
flowchart TD
    subgraph 核销后创建报名数据
        A[财务核销订单] --> B[创建/更新 Registration<br/>并生成 RegistrationSlot]
    end

    subgraph 转赠流程
        C[学员发起转赠] --> D{校验: 已核销?<br/>名额均为有效?<br/>无座位分配?<br/>数量充足?}
        D -->|校验失败| E[返回不能转赠]
        D -->|校验通过| F[输入转赠数量<br/>选择受赠人]
        F --> G[查找或创建受赠人 Registration]
        G --> H[更新选中 RegistrationSlot 的归属<br/>registration_id / owner_customer_id]
        H --> I[重算原/新 Registration 汇总]
        I --> J[创建 GiftRecord<br/>slots=转赠数量 + slot_ids_snapshot]
        J --> K[转赠完成]
    end

    B -.-> C

    style A fill:#e3f2fd
    style B fill:#e3f2fd
    style K fill:#e8f5e9
    style E fill:#ffebee
```

**转赠链支持：**
- 允许无限次转赠（A→B→C→D...）
- 每次转赠都创建新的 GiftRecord，必要时为受赠人创建新的 Registration
- 所有转赠产生的 Registration 和 RegistrationSlot 业绩都归属最开始的业务员
- `active_slots=0` 且 `total_slots=0` 的 Registration 标记为「已清空」，不在小程序展示，但保留审计记录

### 5.6 退款流程

**退款规则：**
- 退款入口：小程序用户申请 或 后台工作人员发起
- 已核销订单可退款，但仅针对该订单下当前仍归属于申请人、未转赠、未签到、未分配座位的名额
- 退款金额按可退款名额数量 × 原始单价计算
- 转赠后仅支持退剩余未转赠部分（如购买5个名额，转赠2个，只可退3个）
- 线下订单退款由工作人员处理，记录日志即可

```mermaid
flowchart TD
    A[用户申请/工作人员发起退款] --> B[财务审核]
    B -->|拒绝| C[退款拒绝]
    B -->|通过| D{订单是否已核销?}
    
    D -->|否| E[直接关闭订单]
    D -->|是| F[筛选可退款名额<br/>root_order_id=当前订单<br/>且 status=有效<br/>且未分配座位]
    F --> G{可退款名额>0?}
    G -->|否| H[返回不可退款]
    G -->|是| I[按可退款名额计算退款金额]
    I --> J{订单类型}
    J -->|小程序订单| K[原路退回支付金额]
    J -->|线下订单| L[工作人员处理退款]
    K --> M[将对应 RegistrationSlot 标记为已退款]
    L --> M
    M --> N[重算 Registration 汇总与 Session.enrolled_count]
    N --> O{是否全额退款?}
    O -->|是| P[订单关闭]
    O -->|否| Q[订单状态=已退款]
    
    style C fill:#ffebee
    style P fill:#e8f5e9
    style Q fill:#e8f5e9
```

**退款处理细节：**
- 已核销订单退款：仅标记对应 RegistrationSlot 为已退款，并重算 Registration 汇总与 Session.enrolled_count
- 未核销订单退款：直接关闭订单，无需恢复名额
- 部分退款（转赠后）：按剩余可退款名额数量计算退款金额
- 线下订单：财务手动处理，系统记录退款日志

## 6. 接口规范（关键接口）

### 6.1 小程序端

| 接口                  | 方法 | 说明                         |
| :-------------------- | :--- | :--------------------------- |
| /api/courses          | GET  | 获取课程列表                 |
| /api/courses/:id      | GET  | 获取课程详情                 |
| /api/sessions/:id     | GET  | 获取课期详情                 |
| /api/orders           | POST | 创建订单                     |
| /api/orders/mine      | GET  | 获取我的订单                 |
| /api/payments         | POST | 发起微信支付                 |
| /api/registrations    | GET  | 获取我的已生效课期汇总       |
| /api/checkin/qrcode   | GET  | 获取课期签到二维码（扫码用） |
| /api/checkin/generate | POST | 生成个人签到码并锁定名额     |
| /api/gifts            | POST | 创建转赠记录                 |
| /api/orders/:id/refund| POST | 发起退款申请                 |

### 6.2 管理后台

| 接口                            | 方法 | 说明               |
| :------------------------------ | :--- | :----------------- |
| /api/admin/courses              | CRUD | 课程管理           |
| /api/admin/sessions             | CRUD | 课期管理           |
| /api/admin/customers            | CRUD | 客户管理           |
| /api/admin/registrations        | CRUD | 客户课期汇总管理   |
| /api/admin/registration-slots   | GET  | 名额明细查询       |
| /api/admin/checkin/qrcode       | GET  | 生成课期签到二维码 |
| /api/admin/checkin/verify       | POST | 扫码核销           |
| /api/admin/orders               | POST | 创建充值订单（业务员） |
| /api/admin/orders/:id/verify    | POST | 订单核销（财务）   |
| /api/admin/orders/:id/refund    | POST | 订单退款（财务）   |
| /api/admin/refunds              | GET  | 退款记录查询       |
| /api/admin/seat-groups          | CRUD | 会场座位分组       |
| /api/admin/users/permissions    | PUT  | 权限分配（管理员） |

------

## 7. 非功能需求

### 7.1 性能要求

- 课程列表加载 < 1s
- 支付流程完成 < 3s
- 签到核销响应 < 500ms

### 7.2 安全要求

- 微信支付签名验证
- 管理后台接口 JWT 认证
- 敏感操作（充值、取消课期）需二次确认
- 财务核销操作需独立权限控制

### 7.3 兼容性

- 微信小程序：基础库 2.19.0+
- 管理后台：Chrome 90+, Safari 14+

------

## 8. 附录

### 8.1 术语表

| 术语   | 说明                                              |
| :----- | :------------------------------------------------ |
| 课程   | 抽象的课程类型，如「Python 入门」                 |
| 课期   | 课程的具体实例，如「Python 入门 -2026 年 4 月班」 |
| 客户   | 在小程序注册的用户                                |
| 报名汇总 | 客户在课期下的聚合报名记录，按客户+课期+业绩归属维度汇总 |
| 名额   | 已核销后生成的最小业务单元，用于签到、座位、转赠、退款 |
| 订单   | 购课或充值的统一订单，支付成功后进入待核销，核销后才生效 |
| 业务员 | 负责客户销售的工作人员，与客户和报名记录绑定      |
| 转赠   | 将购买的课期名额转让给其他人的功能（立即生效）    |
| 退款   | 订单退款，支持全额退款和部分退款（转赠后）        |
| 退款记录 | 独立记录每笔退款，关联原订单                      |

------

**文档变更记录**

| 版本 | 日期       | 变更内容                                                     | 作者 |
| :--- | :--------- | :----------------------------------------------------------- | :--- |
| v1.5 | 2026-04-22 | 1. 统一“支付成功后待核销、核销后才生效”的口径 2. 拆分运营与业务员角色 3. 新增我的订单模块 4. 引入 RegistrationSlot 作为签到/座位/转赠/退款的最小单元 5. 修正订单支付失败、退款、转赠、报名汇总等状态规则 6. 补充相关接口与约束 | -    |
| v1.0 | 2026-04-19 | 初稿完成                                                     | -    |
| v1.4 | 2026-04-22 | 1. 完善转赠流程：支持部分转赠、多次转赠、转赠链，转赠立即生效 2. Registration 新增 slots 字段记录名额数量 3. GiftRecord 简化为日志记录 4. 明确转赠校验规则（已签到/已分配座位不能转赠）5. 明确业绩归属（始终归属最开始的业务员）6. 明确 SessionSegment 创建规则（默认全场次）7. 明确签到码有效期可配置（默认120秒）8. 明确退款流程（全额/部分退款，线下退款记录）9. 明确核销拒绝后处理流程 10. 补充退款接口 | -    |
| v1.3 | 2026-04-21 | 1. 统一订单模式：购课和充值都走 Order 表 2. 所有订单需财务核销 3. 删除 Recharge 表，功能合并至 Order 表 | -    |
| v1.2 | 2026-04-21 | 1. 明确业务员权限范围（仅管理自己绑定的客户数据）2. 支持课期多场签到（新增 SessionSegment 表）3. 新增转赠记录表 GiftRecord 4. 新增座位分配表 SeatAssignment 5. 新增课程分类表 CourseCategory 6. 明确退款需财务审核 7. 签到状态以 CheckIn 记录为准 8. 充值绑定到具体课期 | -    |
| v1.1 | 2026-04-20 | 1. 增加管理员权限分配功能 2. 增加业务员绑定和业绩统计 3. 优化会场座位分组和名额转赠功能 4. 增加充值财务核销流程 | -    |

### 8.2 设计决策汇总

以下是在需求确认过程中明确的关键决策：

**转赠机制**
- 转赠时机：核销后才能转赠（Order.status = 已核销）
- 转赠方式：立即生效，受赠人无需操作，仅记录 GiftRecord 日志
- 支持部分转赠、多次转赠，不限制转赠链长度（A→B→C→D...）
- 仅有效且未分配座位的名额可转赠，转赠以 RegistrationSlot 为最小单位
- 业绩始终归属最开始的业务员（Registration.sales_agent_id、RegistrationSlot.sales_agent_id 不变）
- `active_slots=0` 且 `total_slots=0` 的 Registration 标记为「已清空」，不在小程序展示

**签到机制**
- SessionSegment：课期创建时默认生成一个「全场次」，工作人员可再添加其他场次
- 签到码有效期：可配置，默认 120 秒
- 签到校验：唯一约束 CheckIn(registration_slot_id, session_segment_id) 防止同一名额重复签到

**座位分配**
- 工作人员按已生效名额将学员分配到指定分组
- 转赠的学员可与其赠送人分配在同一分组

**退款机制**
- 退款入口：小程序用户申请 或 后台工作人员发起
- 退款状态：退款中 → 已退款（部分）/ 已关闭（全额）
- 已核销订单可退款，但仅处理该订单下可退款的 RegistrationSlot
- 退款金额按可退款名额数量 × 原始单价计算
- 转赠后仅支持退剩余未转赠部分（如购买5个名额，转赠2个，只可退3个）
- 退款独立记录，支持多次部分退款
- 线下订单退款由工作人员处理，记录日志即可

**已拒绝处理**
- 线下订单：业务员可修改信息后重新提交核销申请
- 小程序订单：财务手动发起退款（原路退回），订单关闭

**线下充值**
- 业务员为客户手动创建充值订单，直接标记为已支付（现金/转账已收），等待财务核销
- 客户无需在线支付，订单直接进入财务核销流程
