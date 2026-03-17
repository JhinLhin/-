# Ticketmaster 系统设计 — 面试标准答案

> **记忆口诀：「看搜订，锁座排搜缓」** 功能三件事：**看**事件、**搜**事件、**订**票 深潜五件事：**锁**票机制、**座**位实时更新、**排**队系统、**搜**索优化、**缓**存扩展

------

## 第零步：面试开场白

> "Ticketmaster is an online platform for purchasing tickets to concerts, sports events, and live entertainment. Let me start by clarifying the requirements."

------

## 第一步：需求澄清（Requirements）

### 功能需求（Functional Requirements）

| #    | 需求                           | 英文面试说法                                       |
| ---- | ------------------------------ | -------------------------------------------------- |
| ✅1   | 用户能查看事件详情（含座位图） | Users should be able to **view events**            |
| ✅2   | 用户能搜索事件                 | Users should be able to **search for events**      |
| ✅3   | 用户能购买门票                 | Users should be able to **book tickets** to events |
| ❌    | 用户查看已购票订单             | Below the line                                     |
| ❌    | 管理员创建事件                 | Below the line                                     |
| ❌    | 热门事件动态定价               | Below the line                                     |

### 非功能需求（Non-Functional Requirements）

功能需求回答的是：

> “系统要做什么？”

非功能需求回答的是：

> “系统做这些事时，最重要的质量要求是什么？”

| #    | 需求                                                | 英文面试说法                                                 |
| ---- | --------------------------------------------------- | ------------------------------------------------------------ |
| ✅1   | 查看/搜索偏重可用性，订票偏重一致性（不能重复售票） | Prioritize **availability** for viewing/search, **consistency** for booking (no double booking) |
| ✅2   | 能承受热门事件的极端并发（千万用户抢同一事件）      | Handle **high throughput** for popular events (10M users, one event) |
| ✅3   | 搜索低延迟（< 500ms）                               | **Low latency** search (< 500ms)                             |
| ✅4   | 读多写少（100:1）                                   | **Read-heavy** system (100:1 read-to-write ratio)            |

- **看页面、搜活动**：数据稍微旧一点可以接受，但系统不能挂

- **买票**：绝对不能卖重票，所以一致性最重要

  - 如果同一张票卖给两个人，那就是系统级事故

- **热门事件的极端并发**

  - 比如一场超热门演唱会开票：

    - 几百万甚至上千万用户同时涌入
    - 大量人同时刷新页面
    - 大量人同时搜索
    - 大量人同时抢同一批座位

    注意，这不是普通流量高一点，而是：

    > **很多人同时争抢同一批资源**

    这比普通高并发更难。

    为什么更难？

    因为普通电商里，大家可能买不同商品。
     而这里，所有人都盯着同一场活动、甚至同一片座区。

    这就意味着压力会集中打在：

    - 同一个 event page
    - 同一个 event search result
    - 同一个 booking path
    - 同一批 ticket rows / seat states

    所以这题天然会需要考虑：

    - 缓存
    - 水平扩展
    - 限流
    - 排队
    - 锁票

- **搜索要低延迟**

  - 这个非功能需求会直接决定你后面的技术选型。

- **系统是读多写少（100:1）**

  - 看活动页的人很多
  - 搜索的人很多
  - 真正走到买票支付的人，相对少得多
  - 影响一：读路径必须特别优化
  - 影响二：写路径可以更“重”

**CAP 决策**：

> **什么是 CAP？** 想象一个分布式系统是一个连锁餐厅。C（一致性）= 每家店菜单必须一样；A（可用性）= 每家店必须开门营业；P（分区容忍）= 即使店之间电话断了也得运转。CAP 定理说你最多只能保证其中两个。

- **查看/搜索路径**：选 AP（可用性优先）。用户看到的座位信息稍有延迟可以接受，但页面必须能打开。
- **订票路径**：选 CP（一致性优先）。绝对不能两个人买到同一张票，宁可暂时拒绝请求。

------

## 第二步：核心实体（Core Entities）

先找核心业务对象，再找它们之间的关系，再决定哪些状态要单独建模。

把现实业务拆成适合计算机管理的数据对象。

| 实体          | 说明               | 关键字段                                                     |
| ------------- | ------------------ | ------------------------------------------------------------ |
| **Event**     | 一场演出/比赛      | eventId, name, date, description, venueId, performerId       |
| **Venue**     | 场馆               | venueId, name, address, capacity, **seatMap**（座位布局JSON） |
| **Performer** | 表演者/队伍        | performerId, name, description                               |
| **Ticket**    | 一个座位对应一张票 | ticketId, eventId, section, row, seatNumber, price, **status**, reservedUntil |
| **Booking**   | 一次购买交易       | bookingId, userId, ticketIds[], totalPrice, status           |
| **User**      | 用户               | userId, name, email                                          |

- Event 是整个业务的聚合中心。

  - 很多东西都围绕它展开：
    - 查看活动详情
    - 搜索活动
    - 某活动下有哪些票
    - 某活动座位图长什么样
    - 某活动的抢票高峰

- Venue这里最关键的不是 name 和 address，
   而是这个：**seatMap**。

  - 很多场景下用户是要看座位图选座的。

  - 所以系统必须知道：

    - 有哪些 section（区域）
    - 每个 section 有哪些 row（排）
    - 每排有哪些 seat（座位号）
    - 这些座位在前端怎么画出来

    这就是 `seatMap` 的作用。这样前端才能渲染出一个可点击的座位图。

- **为什么 Venue 要和 Event 分开？**

  - 同一个 Venue 可以承办很多个 Event。

  - 如果你把 Venue 信息直接写死在 Event 里，会出现很多重复数据：

    - 场馆地址重复
    - 容量重复
    - 座位布局重复

    所以更合理的做法是：

    - Venue 存“场馆本身的信息”
    - Event 只引用 `venueId`

- 为什么 Performer 也要单独建模？

  - 因为一个表演者可能参加很多活动。
  - 如果你把 performer 信息直接塞进每个 Event 里，也会产生大量重复。

- `Ticket` 代表的是：

  - 某场活动中的一个可售卖票单位。一个座位 = 一张 Ticket
  - 系统真正卖的不是“Event”，而是：**Event 下面的一张张具体票。**

- Booking：一次订单/一次购买交易

  - 用户一次购买行为形成的订单。

**为什么 Ticket 和 Booking 要分开？**

> 想象你去超市，购物车里放了3件商品（= 3张 Ticket），结账时生成一张收据（= 1个 Booking）。一个 Booking 可以包含多张 Ticket，这样方便一次性付款、一次性退款。
>
> 在 Ticketmaster 里：
>
> - 3 张具体座位票 = 多个 Ticket
> - 一次付款生成的订单 = 一个 Booking

**Ticket 的 status 字段（三态模型）**：

```
available → reserved → booked
    ↑           |
    └───────────┘  （超时自动释放）
```

- available表示这张票当前没人占、可以买。

- reserved表示这张票暂时被某个用户锁住了，但还没最终成交。因为真实用户买票不是点击就瞬间完成。

  - 中间还有：

    - 选座
    - 填支付信息
    - 支付处理

    这个过程可能几分钟。

    如果没有 `reserved`，那就会出现：

    - 用户 A 正在付款
    - 用户 B 同时也能看到这票可买
    - 两人都以为自己买到了

  - 因为 reserved 不能无限持续。所以需要一个过期时间

  - 所以：

    > `status = reserved` 解决“当前有人占”
    >  `reservedUntil` 解决“占到什么时候”

- booked 表示已经最终卖出，不能再卖。

`User` 比较直观：

- 谁创建了 Booking
- 谁正在 reserve 哪些 Ticket
- 谁最终买到了票

**seatMap 是什么？**

> 场馆的座位布局数据，类似电影院的座位图。存成 JSON，里面记录了每个区域（section）、每排（row）、每个座位号（seat）的坐标位置。客户端拿到这个数据后渲染出可点击的交互式座位图，再结合每张 Ticket 的 status 来显示哪些座位可选。

------

## 第三步：API 设计

### 3.1 查看事件

```
GET /events/:eventId → Event & Venue & Performer & Ticket[]
```

- 返回事件详情 + 场馆信息 + 表演者信息 + 该事件所有票（用于渲染座位图）

### 3.2 搜索事件

```
GET /events/search?keyword={keyword}&start={start_date}&end={end_date}&pageSize={size}&page={page}
→ Event[]
```

- 支持关键词、日期范围、分页

### 3.3 预订门票（后续会拆成两步）

```
POST /bookings/:eventId → bookingId
Body: { "ticketIds": string[], "paymentDetails": ... }
```

> **面试金句**："This is a simple starting point. As we get into the deep dives, I'll evolve this into a two-step process — first reserve, then confirm payment."

**后续演化后的两步 API**（深潜阶段才展开讲）：

```
第一步：POST /bookings/reserve → { bookingId, expiresAt }
       Body: { "eventId": "...", "ticketIds": ["..."] }
       
第二步：POST /bookings/:bookingId/confirm → { status: "confirmed" }
       Body: { "paymentToken": "tok_xxx" }
```

------

## 第四步：高层设计（High-Level Design）

> **面试金句**："Now let me build the design sequentially, one functional requirement at a time."

### 4.1 查看事件（View Events）

**流程**：

```
用户浏览器
   │
   │  GET /events/:eventId
   ▼
API Gateway ──→ Event Service ──→ Events DB (PostgreSQL)
                                   (Event / Venue / Performer / Ticket 表)
```

**各组件解释**：

**API Gateway（API 网关）**

> **类比**：想象酒店前台。所有客人（请求）都先到前台，前台负责验身份（认证）、指路（路由）、限流（rate limiting）。
>
> 作用：统一入口，处理认证、限流、日志，然后把请求分发到对应的微服务。

**Event Service（事件服务）**

> 负责处理查看事件的请求。收到请求后去数据库查 Event、Venue、Performer 信息，以及该事件的所有 Ticket（含状态），一起返回给客户端。

**Events DB（PostgreSQL）**

> 存储 Event、Venue、Performer 表。选 PostgreSQL 因为我们后面订票需要 ACID 事务。

**什么是 ACID？**

> 想象银行转账：A 给 B 转 100 元。ACID 保证：
>
> - **A**tomicity（原子性）：要么转成功、要么完全不转，不会出现 A 扣了钱但 B 没收到
> - **C**onsistency（一致性）：转账前后，总金额不变
> - **I**solation（隔离性）：两笔转账同时发生时互不干扰
> - **D**urability（持久性）：转账成功后，即使断电数据也不丢

### 4.2 搜索事件（Search Events）

**流程**：

```
用户浏览器
   │
   │  GET /events/search?keyword=Taylor
   ▼
Load Balancer ──→ API Gateway ──→ Search Service ──→ Events DB
```

> **面试金句**："Let me start with the simplest approach — querying the database directly. We'll optimize this in the deep dives."

**Load Balancer（负载均衡器）**

> **类比**：超市收银台排队引导员。有5个收银台，引导员会把新来的顾客引到最短的队伍。
>
> 作用：把流量均匀分配到多个服务器实例，防止某台服务器被压垮。常用算法：Round Robin（轮流分配）或 Least Connections（分给当前连接最少的）。

**当前方案的问题**：直接用 SQL `LIKE '%Taylor%'` 搜索需要全表扫描，数据量大时非常慢。→ 深潜阶段优化。

### 4.3 预订门票（Book Tickets）

**朴素方案流程**：

```
用户浏览器
   │
   │  POST /bookings/:eventId
   ▼
API Gateway ──→ Booking Service ──→ Events DB (Booking/Ticket 表)
                     │
                     └──→ Payment Processor (Stripe)
```

**新增组件**：

**Booking Service（预订服务）**

> 核心职责：处理订票逻辑。收到请求后：
>
> 1. 开启数据库事务
> 2. 检查票是否可用
> 3. 更新票状态为 "booked"
> 4. 创建 Booking 记录
> 5. 调用 Stripe 处理支付
> 6. 提交事务

**Payment Processor — Stripe**

> 第三方支付服务。我们不直接处理信用卡号（PCI 合规要求），而是让客户端用 Stripe.js 把卡号转成 token，我们只处理 token。

**数据库事务保证不会双重售票**

> **什么是数据库事务？** 想象你和朋友同时抢最后一件商品。事务就像一个"原子操作包"——查库存+扣库存+生成订单，这三步要么全成功、要么全回滚。不可能出现两个人都扣了库存但只有一件商品的情况。

**为什么多个服务共享一个数据库？**

> **面试金句**："The 'database per service' rule isn't a hard-and-fast rule. Here, a shared database makes sense because the data is tightly coupled — bookings need tickets, which need events — and we need ACID transactions across them. Splitting would add complexity for no real benefit."

**当前方案的问题**：

> 用户可能花 5 分钟填支付信息，填完发现票被别人买了。体验极差。→ 深潜1解决。

------

## 第五步：深潜（Deep Dives）

> **面试金句**："Now that we've satisfied the functional requirements, let me address the non-functional requirements through deep dives."

------

### 深潜 1：锁票机制 — 如何预留座位？

> **面试金句**："The current design has a fundamental UX problem — users can spend minutes filling payment details only to find the ticket is gone. Let me walk through how to solve this with ticket reservation."

#### ❌ 坏方案：数据库长锁（Long-running DB Locks）

**思路**：用 PostgreSQL 的 `SELECT FOR UPDATE` 锁住票的那一行，直到用户付款完成才释放。

**什么是 SELECT FOR UPDATE？**

> 类比：你在图书馆看到一本想借的书，你把手按在上面说"这是我的"，别人想拿就得等你放手。`SELECT FOR UPDATE` 就是在读取一行数据的同时"按住"它，别人想改这行就得排队。

**为什么是坏方案？**

> 数据库锁设计初衷是毫秒级的短操作。让它锁 5-10 分钟（用户填表的时间）就像你在图书馆按着一本书不放 10 分钟——后面排队的人全卡住了。具体问题：
>
> 1. **资源浪费**：数据库连接被长时间占用，连接池耗尽
> 2. **死锁风险**：多个长事务交叉锁定不同行，可能互相等待永远解不开
> 3. **崩溃风险**：如果应用崩溃，锁处于不确定状态
> 4. **扩展性差**：热门事件时成千上万用户同时锁行，数据库撑不住

#### ⚠️ 中等方案：状态字段 + 过期时间（Status Field + Expiration）

**思路**：给 Ticket 加 `status`（available/reserved/booked）和 `reservedUntil` 字段。用短事务更新状态，不需要长锁。

**预订流程**：

```sql
BEGIN TRANSACTION;
  -- 检查票是否可用（available），或者虽是 reserved 但已过期
  SELECT * FROM tickets 
  WHERE ticketId = 'xxx' 
    AND (status = 'available' OR (status = 'reserved' AND reservedUntil < NOW()))
  FOR UPDATE;  -- 短锁，只在这个事务内
  
  -- 更新为 reserved，设置 10 分钟过期
  UPDATE tickets 
  SET status = 'reserved', reservedUntil = NOW() + INTERVAL '10 minutes'
  WHERE ticketId = 'xxx';
COMMIT;
```

**关键洞察**：

> **面试金句**："The status of any given ticket is the combination of two attributes: whether it's available OR whether it's been reserved but the reservation has expired."

> 这意味着我们不需要等锁释放才能把票给下一个人。只要 `reservedUntil` 过期了，下一个用户的事务就能直接拿到这张票。

**过期票怎么释放？**

选项 A：Cron Job 定时扫描 → 有延迟，不够即时

选项 B（更好）：**不需要主动释放！** 每次有人想订票时，SQL 条件本身就包含了 `reservedUntil < NOW()` 的检查。过期的 reserved 票自动变得"可用"。可以额外跑一个 cron job 清理数据让表更整洁，但系统行为不依赖它。

**优缺点**：

| 优点                           | 缺点                                                 |
| ------------------------------ | ---------------------------------------------------- |
| 不需要额外基础设施             | 读查询需要同时过滤 status 和时间，稍慢               |
| 事务极短，不会阻塞             | 数据库里会有已过期但 status 还是 reserved 的"脏数据" |
| 即使 cron 失败也不影响核心逻辑 | —                                                    |

#### ✅ 最优方案：Redis 分布式锁 + TTL

**什么是 Redis？**

> 类比：Redis 是一块超级快的白板。数据全在内存里（不像数据库写到硬盘），所以读写极快（微秒级）。适合存临时的、需要快速访问的数据。

**什么是 TTL（Time To Live）？**

> 类比：便利贴有一个自动消失功能。你贴上去写"这个座位是 Alice 的"，设定 10 分钟后自动消失。不需要任何人来撕它。Redis 的 TTL 就是这个——到时间自动删除 key。

**什么是分布式锁？**

> 类比：多个收银台共享一个"占位牌"系统。无论你在哪个收银台操作，都能看到哪些座位已经被占了。Redis 作为一个独立的共享服务，所有 Booking Service 实例都连它，所以能实现跨实例的协调。

**具体机制**：

```
Redis Key:   lock:ticket:{ticketId}
Redis Value: {userId}
TTL:         10 分钟

命令：SET lock:ticket:T123 user456 NX EX 600
```

- `NX`：只在 key 不存在时才设置（原子操作，保证只有一个人能抢到）
- `EX 600`：600 秒后自动过期

**什么是原子操作？**

> 类比：你和朋友同时伸手拿桌上最后一块蛋糕。原子操作保证只有一只手能拿到——不会出现两只手同时碰到的情况。Redis 的 `SET NX` 就是原子的，即使一百万个请求同时到达，也只有一个能成功设置 key。

**Ticket 表简化为两态**：

```
available → booked  （只有这两个状态）
```

reserved 状态完全由 Redis 管理，不在数据库里。

#### 完整预订流程（最终版）

```
用户点击座位
   │
   │ ① POST /bookings/reserve { eventId, ticketIds }
   ▼
API Gateway → Booking Service
   │
   │ ② 尝试在 Redis 获取锁：SET lock:ticket:T123 user456 NX EX 600
   │   （如果锁已存在 → 返回 "seat unavailable"）
   │
   │ ③ 锁定成功 → 在 DB 创建 Booking 记录 (status: in-progress)
   │
   │ ④ 返回 { bookingId, expiresAt } → 客户端显示支付页面 + 倒计时
   │
   │   ... 用户填写支付信息 ...
   │   （如果用户放弃 → 10分钟后 Redis TTL 自动释放锁，票自动可用）
   │
   │ ⑤ 客户端用 Stripe.js 把信用卡号 tokenize → 得到 paymentToken
   │   POST /bookings/:bookingId/confirm { paymentToken }
   │
   │ ⑥ Booking Service 创建 Stripe PaymentIntent
   │
   │ ⑦ Stripe 处理支付 → 通过 Webhook 通知我们成功
   │
   │ ⑧ Webhook handler 收到通知：
   │   - 开启 DB 事务
   │   - 更新 Ticket status → "booked"
   │   - 更新 Booking status → "confirmed"
   │   - 提交事务
   │   - 删除 Redis 锁（提前释放，不等 TTL）
   │
   ▼
完成！
```

**什么是 Webhook？**

> 类比：你在餐厅点了外卖，留了电话号码。做好了餐厅主动打给你（webhook），而不是你每分钟打一次电话问"好了没？"（polling）。Stripe 支付完成后主动调用我们的一个 URL 来通知结果。

**什么是 Tokenize？**

> 类比：你把信用卡交给一个可信的中间人（Stripe），中间人给你一个"兑换券"（token）。你把兑换券给商家，商家用兑换券找中间人要钱。商家全程没碰到你的信用卡号。这就是 PCI 合规的标准做法。

**什么是幂等性（Idempotency）？**

> 类比：你按电梯按钮，按一次和按十次效果一样——电梯都会来。Webhook handler 必须是幂等的，因为 Stripe 可能重试发送同一个通知。我们用 bookingId 作为幂等键，处理前先检查 Booking 状态：如果已经是 confirmed 就直接返回成功，不会重复扣款。

#### 边界情况：TTL 过期但支付正在处理

> 场景：用户 A 锁票 10 分钟，第 10 分钟 TTL 过期，第 10.5 分钟用户 B 抢到锁，第 11 分钟用户 A 的支付完成了。

> **解决方案**：数据库事务用 OCC（乐观并发控制）保底。只有一个人的写操作能成功，另一个会失败。失败的那个自动通过 Stripe 退款。同时，把 TTL 设得宽裕一些，并在支付发起时延长锁时间，尽量避免这种情况。

**什么是 OCC（Optimistic Concurrency Control，乐观并发控制）？**

> 类比：你和朋友同时编辑一个 Google Doc 的同一行。乐观并发 = 先让你们都编辑，提交时检查谁先提交的——后提交的那个被告知"内容已被修改，请重试"。实现方式通常是在行上加一个 `version` 字段，更新时检查 version 是否变了。

#### 座位图怎么显示 reserved 状态？

> Redis 锁住的票需要在座位图上显示为不可用。两种方案：
>
> 1. **查 Redis**：Event Service 返回数据时额外查一次 Redis，获取该事件所有被锁的 ticketId（用 Redis Set `event:{eventId}:reserved` 维护）
> 2. **Write-through**：获取锁时同时在数据库写一个 reserved 状态，以 Redis TTL 为准，定期清理数据库里过期的 reserved

------

### 深潜 2：查看接口如何扩展到千万级并发？

> **面试金句**："Event pages get absolutely hammered when tickets go on sale. Let me discuss how to handle this extreme read load through caching, horizontal scaling, and load balancing."

#### 缓存策略（Caching）

**什么是缓存？**

> 类比：你经常查某个电话号码，与其每次翻电话簿（数据库），不如写在手边的便签上（缓存）。下次直接看便签，快100倍。

**缓存什么？**

> 事件详情、表演者信息、场馆信息——这些数据很少变化，读取频率极高，非常适合缓存。

**缓存策略：Read-Through Cache**

```
客户端请求 → 先查 Redis 缓存
              │
              ├─ 命中 → 直接返回（超快）
              │
              └─ 未命中 → 查数据库 → 写入缓存 → 返回
```

> **类比**：你问前台"会议室在哪"。如果前台记得（缓存命中），直接告诉你。如果不记得（缓存未命中），前台翻一下目录（查数据库），记住答案（写入缓存），然后告诉你。

**缓存 Key 设计**：`event:{eventId}` → 事件完整对象（JSON）

**TTL 策略**：

| 数据类型   | TTL     | 原因     |
| ---------- | ------- | -------- |
| 场馆信息   | 24小时  | 几乎不变 |
| 表演者信息 | 12小时  | 很少变   |
| 事件详情   | 1-6小时 | 偶尔更新 |
| 票务可用性 | 30-60秒 | 频繁变化 |

**缓存失效（Cache Invalidation）**：

> 当事件信息更新时（比如改了日期），通过数据库触发器通知缓存系统，删除对应的缓存条目。下次请求时会自动从数据库重新加载。

#### 水平扩展（Horizontal Scaling）

**什么是水平扩展？**

> 类比：餐厅太忙了，你不是让一个厨师加班（垂直扩展 = 升级单台服务器），而是多雇几个厨师（水平扩展 = 增加服务器数量）。

> Event Service 是**无状态的**（每个请求独立处理，不依赖服务器本地数据），所以可以随时增加实例数量。配合 Load Balancer 把流量均匀分配到各实例。

#### 扩展后的查看架构

```
用户 → CDN（静态资源） 
     → Load Balancer → API Gateway → Event Service (×N 个实例) → Redis 缓存 → PostgreSQL
```

**CDN（Content Delivery Network，内容分发网络）**

> 类比：你在北京，要看一个存在美国服务器上的图片。没有 CDN 要跨太平洋取数据，很慢。有了 CDN，图片提前缓存到北京的服务器上，你直接从近处拿，快很多。CDN 适合缓存静态资源（图片、CSS、JS）和不常变化的 API 响应。

------

### 深潜 3：热门事件下如何保证用户体验？

> **面试金句**："For extremely popular events, even with SSE updates, the seat map will fill up instantly. The real solution isn't just technical — it's rethinking the user flow."

#### ⚠️ 中等方案：SSE 实时推送座位状态

**什么是 SSE（Server-Sent Events）？**

> 类比：你在看直播比分。不是你每秒刷新页面（polling），而是服务器主动把最新比分推送到你的浏览器（SSE）。SSE 是单向的——只有服务器向客户端发数据。

**机制**：当一张票被 reserved 或 booked 时，Booking Service 通过 SSE 推送更新给所有正在看该事件座位图的客户端，客户端实时把对应座位标灰。

**问题**：Taylor Swift 级别的事件，座位几秒内全部被锁定，用户体验是一片座位瞬间变灰，手忙脚乱点不到。

#### ✅ 最优方案：虚拟排队系统（Virtual Waiting Queue）

> **面试金句**："The mark of a senior/staff engineer is solving business problems by thinking outside the box of presumed constraints."

**思路**：在用户进入座位选择页面**之前**，先排队。就像演唱会门口排队入场。

**技术实现**：

```
用户请求进入订票页
   │
   │ ① 加入排队队列（Redis Sorted Set，用时间戳排序）
   │
   │ ② 建立 SSE 连接，推送排队位置和预估等待时间
   │
   │   ... 等待 ...
   │
   │ ③ 轮到你了！SSE 推送"已放行"通知
   │   同时在 Redis 的 admitted:{eventId} Set 中添加你的 sessionId（带 TTL）
   │
   │ ④ 用户进入座位选择页面，正常选座、锁票、支付
   │   Booking Service 验证 sessionId 在 admitted Set 中才允许操作
   │
   ▼
```

**什么是 Redis Sorted Set？**

> 类比：一个自动排序的排队名单。每个人进来时记录一个分数（这里用时间戳），名单自动按分数从小到大排列。你可以随时查"我排第几"。

**放行策略**：

> 根据已预订票数或固定时间间隔，从队列前端放行一批用户。比如每卖出100张票就放行下一批50人。这样保证在线选座的人数始终可控，系统不会过载。

**为什么这比纯技术方案好？**

> 不是技术更强，而是**体验更好**：用户知道自己在排队、知道前面有多少人、有预估时间，远比进去一看全被抢了要好。这是 Ticketmaster 和航空公司的真实做法。

------

### 深潜 4：如何优化搜索以满足低延迟要求？

> **面试金句**："Our current search does a full table scan with LIKE queries, which is way too slow. Let me walk through the optimization path."

#### ❌ 当前问题

```sql
SELECT * FROM Events WHERE name LIKE '%Taylor%' OR description LIKE '%Taylor%'
-- 全表扫描！数据量大时几秒甚至几十秒
```

**什么是全表扫描？**

> 类比：在一本没有目录的书里找某个词，只能从第一页翻到最后一页。

#### ⚠️ 基础方案：数据库索引（Database Indexes）

**什么是索引？**

> 类比：书的目录。有了目录，你不用翻完整本书，直接查目录就知道第几页有你要的内容。数据库索引把特定列的值映射到对应行的位置，大幅加速查询。

**做法**：在 event name、event date、performer name、venue location 等常用搜索字段上建索引。

**局限**：标准索引对部分匹配（"Taylor" 而不是完整的 "Taylor Swift"）效果有限，仍可能需要 `LIKE`。

#### ⚠️ 进阶方案：全文索引（Full-Text Index）

> PostgreSQL 内建全文搜索，使用 `tsvector`（文本向量化）和 `GIN` 索引。

**什么是 tsvector 和 GIN？**

> 类比：`tsvector` 把一段文字拆成独立的关键词列表（"Taylor Swift Concert" → ["taylor", "swift", "concert"]）。`GIN`（Generalized Inverted Index）就像一个反向电话簿——不是按人名找电话，而是按电话号码找人名。对于搜索来说就是按关键词找包含它的所有文档。

**局限**：比标准索引占更多存储，维护成本更高，不支持模糊搜索（typo 容错）。

#### ✅ 最优方案：Elasticsearch

**什么是 Elasticsearch？**

> 类比：一个超级智能的图书馆管理员。普通数据库像按书架号找书；Elasticsearch 像有一个管理员记住了每本书里的每个关键词出现在哪些书里，你说任何一个词，他秒级给你所有相关的书。

**核心原理：倒排索引（Inverted Index）**

> 普通索引：文档 → 包含哪些词 倒排索引：词 → 出现在哪些文档里

```
正向：文档1 = "Taylor Swift Concert in NYC"
      文档2 = "NYC Jazz Festival"

倒排：
  "taylor"  → [文档1]
  "swift"   → [文档1]
  "concert" → [文档1]
  "nyc"     → [文档1, 文档2]
  "jazz"    → [文档2]
  "festival"→ [文档2]

搜索 "NYC concert" → 取交集 → 文档1 ✅
```

**如何保持数据同步？用 CDC（Change Data Capture）**

**什么是 CDC？**

> 类比：你有一个主笔记本（PostgreSQL）和一个速查卡片盒（Elasticsearch）。CDC 就像一个自动抄写员，每当主笔记本有改动（增/删/改），抄写员立刻把变化同步到卡片盒。技术上，CDC 监听数据库的变更日志（WAL），实时捕获变化并推送到 Elasticsearch。

```
PostgreSQL ──CDC（如 Debezium）──→ Kafka ──→ Elasticsearch
```

**Elasticsearch 额外能力：模糊搜索（Fuzzy Search）**

> 用户输入 "Tayler Swift"（拼错了），Elasticsearch 能自动容错，仍然返回 "Taylor Swift" 的结果。纯 SQL 做不到这一点。

#### 搜索架构更新

```
用户搜索请求 → API Gateway → Search Service → Elasticsearch
                                                    ↑
                              PostgreSQL ──CDC──→ 数据同步
```

------

### 深潜 5：如何用缓存加速频繁搜索？

> **面试金句**："To further reduce latency and load on our search infrastructure, we can cache frequent search results."

#### 方案 A：Redis 缓存搜索结果

```json
{
  "key": "search:keyword=Taylor Swift&start=2026-01-01&end=2026-12-31",
  "value": "[event1, event2, event3]",
  "ttl": 86400
}
```

- **缓存 Key 设计**：用搜索参数的组合作为 key，保证相同搜索命中同一个缓存
- **TTL**：根据数据变化频率设置（如24小时）
- **失效策略**：新事件发布或事件信息更新时，通过 cache tag 批量失效相关缓存

#### 方案 B：利用 Elasticsearch 内建缓存

> Elasticsearch 自带两层缓存：
>
> - **Query Cache**（分片级别）：缓存 filter 查询结果
> - **Request Cache**（分片级别）：缓存完整搜索响应（特别适合聚合查询）
>
> 系统会自适应地缓存最频繁的查询。

#### 方案 C：CDN 缓存搜索结果

> 如果搜索结果不是个性化的（所有人搜 "Taylor Swift" 结果一样），可以把搜索结果缓存在 CDN 上。用户请求直接从最近的 CDN 节点返回，不需要打到后端。

------

## 最终架构总览

```
                          ┌─── CDN（静态资源 + 搜索缓存）
                          │
用户 ──→ Load Balancer ──→ API Gateway
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        Event Service   Search Service   Booking Service
              │               │               │
              ▼               ▼               ├──→ Redis 分布式锁（TTL 锁票）
        Redis 缓存      Elasticsearch        ├──→ Redis 排队队列
              │               ↑               │
              ▼               │               ▼
           PostgreSQL ───CDC──┘          Stripe（支付）
           (Event/Venue/Performer/Ticket/Booking)
```

**SSE 连接**：Booking Service → 客户端（座位状态实时推送 + 排队位置推送）

------

## 快速参考卡

### 记忆口诀

**「看搜订，锁座排搜缓」**

| 口诀 | 含义         | 对应设计                           |
| ---- | ------------ | ---------------------------------- |
| 看   | 查看事件     | Event Service + Events DB          |
| 搜   | 搜索事件     | Search Service + SQL               |
| 订   | 预订门票     | Booking Service + Stripe + DB 事务 |
| 锁   | 锁票预留     | Redis 分布式锁 + TTL               |
| 座   | 座位实时更新 | SSE 推送                           |
| 排   | 排队系统     | Redis Sorted Set + SSE             |
| 搜   | 搜索优化     | Elasticsearch + CDC                |
| 缓   | 缓存扩展     | Redis 缓存 + CDN                   |

### 核心组件一句话速记

| 组件            | 一句话                                         |
| --------------- | ---------------------------------------------- |
| API Gateway     | 所有请求的统一入口，负责认证、限流、路由       |
| Event Service   | 读取事件/场馆/表演者信息，无状态，可水平扩展   |
| Search Service  | 接受搜索请求，查 Elasticsearch                 |
| Booking Service | 处理锁票 → 支付 → 确认的完整购票流程           |
| Redis 缓存      | 缓存高频读取的事件数据，减轻数据库压力         |
| Redis 分布式锁  | 用 TTL 临时锁定座位，自动过期释放              |
| Redis 排队队列  | Sorted Set 实现虚拟排队，控制进入订票页的人数  |
| Elasticsearch   | 倒排索引实现快速全文搜索 + 模糊搜索            |
| CDC (Debezium)  | 实时同步 PostgreSQL 变更到 Elasticsearch       |
| PostgreSQL      | 存储所有核心数据，提供 ACID 事务保证不重复售票 |
| Stripe          | 第三方支付，我们只处理 token 不碰信用卡号      |
| CDN             | 缓存静态资源和非个性化搜索结果到用户附近       |
| SSE             | 服务器单向推送座位变化和排队位置给客户端       |

### 高频面试追问 Q&A

**Q: 为什么用 Redis 锁而不是数据库锁来预留座位？**

> A: Database locks are designed for millisecond operations. Holding a lock for 5-10 minutes while a user fills payment details strains database resources, risks deadlocks, and doesn't scale. Redis gives us automatic key expiration built-in, and in-memory operations are extremely fast under high concurrency.

**Q: 如果 Redis 锁挂了怎么办？**

> A: We never have a double booking because the database uses OCC as a safety net. The degraded experience is that users might fill payment details only to find the ticket was taken — bad UX, but not a correctness issue. This is better than the cron approach where all tickets might appear unavailable.

**Q: 为什么不用 WebSocket 而用 SSE？**

> A: SSE is simpler and sufficient here because we only need server-to-client communication — pushing seat updates and queue position. WebSocket would be overkill since the client doesn't need to send real-time data to the server.

**Q: 为什么多个服务共享同一个数据库？**

> A: The "database per service" rule isn't a hard-and-fast rule. Here the data is tightly coupled — bookings reference tickets which reference events. We need ACID transactions across them for consistency. Splitting would add complexity without real benefit.

**Q: Elasticsearch 数据同步有延迟怎么办？**

> A: CDC provides near-real-time sync, typically sub-second. For our use case, a slight delay in search results is acceptable since we prioritize availability for the search path. The booking path still goes through PostgreSQL for consistency.

**Q: 如果一个用户要订多张票怎么办？**

> A: Acquire Redis locks sequentially per ticket. If any lock fails, release all previously acquired locks and return an error. For atomicity, you can use a Redis Lua script to make multi-lock acquisition atomic if the tickets hash to the same Redis node.

**Q: 虚拟排队的放行策略是什么？**

> A: Dequeue users based on tickets booked or fixed time intervals. For example, release the next batch of 50 users every time 100 tickets are sold. This keeps the number of concurrent seat-selectors manageable and prevents system overload.

------

### 面试流程时间分配建议（35-45分钟）

| 阶段       | 时间       | 内容                                  |
| ---------- | ---------- | ------------------------------------- |
| 需求澄清   | 3-5 分钟   | 功能/非功能需求 + CAP 决策            |
| 实体 + API | 3-5 分钟   | 6 个实体 + 3 个 API                   |
| 高层设计   | 10-12 分钟 | 逐个功能：查看 → 搜索 → 订票          |
| 深潜       | 15-20 分钟 | 通常覆盖 2-3 个（面试官选或自己主导） |
| Q&A        | 3-5 分钟   | 面试官追问                            |

> **面试金句（开始深潜时）**："Now that we've covered the functional requirements, I'd like to dive into a few areas that address our non-functional requirements. I think ticket reservation, scaling reads, and search optimization are the most impactful. Would you like to start with any of these, or shall I pick?"