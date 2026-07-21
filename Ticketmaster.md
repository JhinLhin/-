# Ticketmaster

## Understanding the Problem

用大白话说：就是一个**网上卖票的平台**。演唱会、球赛、话剧的票都在上面买。

| 需求类型                                  | 说人话         | 句式                         |
| ----------------------------------------- | -------------- | ---------------------------- |
| Functional Requirements（功能需求）       | 用户**能干啥** | "Users should be able to..." |
| Non-Functional Requirements（非功能需求） | 系统**得多牛** | "The system should..."       |

**记忆口诀**：**功能管"能做什么"，非功能管"做得多好"**。

面试一共就 35-45 分钟，你不可能把所有功能都设计一遍。所以策略是：

1. **挑出最核心的 3 个功能**，放在"above the line"（线以上，要做）
2. 其他想到的功能列出来，标明"below the line"（线以下，不做）
3. **主动问面试官**："你看这样划分 OK 吗？要不要调整？"

### Functional Requirements

Above the line（要做的 3 个）

1. **看演出**（View events）
2. **搜演出**（Search events）
3. **买票**（Book tickets）

**记忆口诀**：**看、搜、买**（三个动词，一气呵成，这就是用户用 Ticketmaster 的完整路径）。

Below the line（不做的）

- 查自己买过的票
- 管理员上架演出
- 热门票动态定价

### Non-Functional Requirements

这一块是**重点**，因为它决定了你后面选什么技术。

**1. 看/搜 要 AP，买票要 CP**

| 操作           | 优先级                         | 为啥                                         |
| -------------- | ------------------------------ | -------------------------------------------- |
| 看演出、搜演出 | **可用性优先**（Availability） | 用户看不到页面就跑了，但偶尔数据有点旧没关系 |
| 买票           | **一致性优先**（Consistency）  | **绝对不能一张票卖两次！**（double booking） |

**这就是 CAP 定理的实际应用**：同一个系统里，**不同功能可以选不同的 CAP 策略**。

**金句**：

> "We need availability for browsing but strong consistency for booking — no double booking is a hard requirement."

**2. 能扛住爆款演出**

**场景**：周杰伦演唱会开票，1000 万人同时抢一场的票。系统不能崩。

这意味着后面要考虑：**缓存、削峰、排队、分布式锁**。

**3. 搜索要快（< 500ms）**

用户在搜索框打字时，等超过半秒就会觉得卡。这意味着后面**搜索必须用 Elasticsearch** 这类专门的搜索引擎，不能直接查数据库。

**4. 读多写少（100:1）**

**100 个人来看，才有 1 个人下单**。这就告诉你：

- ✅ **必须重缓存**（Redis、CDN）
- ✅ **读写分离**（写主库，读从库）

**记忆口诀**：**读多写少 → 加缓存、读写分离**。

## The Set Up

### Planning the Approach

> "I'll build the design up sequentially, going through the functional requirements one by one. Once those are satisfied, I'll use the non-functional requirements to guide the deep dives."

**记忆口诀**：**功能驱动主干，非功能驱动深挖**。

具体来说就是：

| 阶段     | 干什么                               | 用什么指导     |
| -------- | ------------------------------------ | -------------- |
| 主体设计 | 一个功能一个功能搭出来               | **功能需求**   |
| 深挖优化 | 针对性优化（扩展性、一致性、低延迟） | **非功能需求** |

**为什么这么做？** 因为系统设计面试最容易翻车的地方就是**跑偏**——你聊着聊着就钻进某个技术细节出不来了。按功能顺序走，能保证你**主线清晰、不丢分**。

### Defining the Core Entities

**实体 = 系统里的"主要名词"**。先别管字段细节，先列出来"这个系统里有哪些东西"。

Ticketmaster 的 6 个核心实体

| 实体          | 中文   | 是什么                                             |
| ------------- | ------ | -------------------------------------------------- |
| **Event**     | 演出   | 一场具体的演出（演唱会、球赛），有日期、描述、类型 |
| **User**      | 用户   | 用 Ticketmaster 的人                               |
| **Performer** | 表演者 | 谁演的（艺人、球队、剧团）                         |
| **Venue**     | 场馆   | 在哪演（地址、容量、**座位图**）                   |
| **Ticket**    | 票     | 一张具体的票（绑定座位 + 价格 + 状态）             |
| **Booking**   | 订单   | 一次购买行为（一个用户买了哪几张票）               |

------

关键关系（一定要理清楚）

用一个例子串起来：

> **周杰伦**（Performer）在**鸟巢**（Venue）开一场**演唱会**（Event）。鸟巢有 8 万个座位，所以系统生成 **8 万张票**（Ticket）。**张三**（User）一次买了 2 张相邻的票，这就形成了一笔**订单**（Booking）。

**记忆口诀**：**谁（Performer）+ 在哪（Venue）+ 演什么（Event）+ 哪张票（Ticket）+ 谁买了（User + Booking）**。

------

几个容易被问到的细节

① 票（Ticket）是怎么来的？

**演出一创建，系统就根据场馆的座位图，给每个座位生成一张 Ticket**。

- 座位图（seat map）存在 **Venue 实体**里（一般用 JSON 结构存）
- 前端拿到座位图 + 每张票的状态（available / sold），就能渲染出那个**可视化选座界面**

② 为什么 Booking 和 Ticket 要分开？

**naive 想法**：把购买信息直接塞到 Ticket 里（比如加个 `buyer_id` 字段）就完了。

**问题在哪？** 如果一个用户**一次买 4 张票**，这 4 张票其实是**同一笔交易**——共享一个**总价、一个支付状态**。如果只用 Ticket，你就得在 4 行里重复存这些信息，很乱。

**最佳实践**：

```
Booking（订单 - 一次交易）
   ├── Ticket A  ← 关联 booking_id
   ├── Ticket B  ← 关联 booking_id
   └── 总价、支付状态都在 Booking 上
```

**记忆口诀**：**Ticket 是商品，Booking 是订单**。一个订单可以含多个商品。

### API or System Interface

## High-Level Design

### Users should be able to view events

### Users should be able to search for events

### Users should be able to book tickets to events

## Potential Deep Dives

### How do we improve the booking experience by reserving tickets?

### How is the view API going to scale to support 10s of millions of concurrent requests during popular events?

### How will the system ensure a good user experience during high-demand events with millions simultaneously booking tickets?

### How can you improve search to ensure we meet our low latency requirements?

### How can you speed up frequently repeated search queries and reduce load on our search infrastructure?