# Getting Started with AWS

> # AWS 全球基础设施 🌍
>
> ## 核心概念：三层结构
>
> ------
>
> ## 1️⃣ Region（区域）
>
> - 就是一个**地理大区**，比如 `us-east-1`（美东）、`eu-west-3`（欧洲）
> - 本质是**一堆数据中心的集合**
> - 绝大多数 AWS 服务是 **Region 级别**的（换了区就是另一套）
>
> ### 🧠 如何选 Region？考试最爱考！背这4个字：**法近服价**
>
> | 字     | 含义       | 记忆                        |
> | ------ | ---------- | --------------------------- |
> | **法** | 法律合规   | 数据不能出境就选本国 Region |
> | **近** | 离用户近   | 延迟低，用户体验好          |
> | **服** | 服务可用性 | 新功能不是每个区都有        |
> | **价** | 价格       | 同服务不同区价格不一样      |
>
> ------
>
> ## 2️⃣ Availability Zone / AZ（可用区）
>
> - 每个 Region 有 **3~6 个 AZ**（考试默认记 **最少3个**）
> - 命名规则：`ap-southeast-2a`、`2b`、`2c`（后面加小写字母）
> - 每个 AZ = **一个或多个独立数据中心**，有独立的电力和网络
> - AZ 之间：**物理隔离**（一个挂了不影响另一个）但用**高速低延迟网络**连着
>
> ### 🧠 一句话记住：
>
> > **Region 是城市，AZ 是城市里互相隔离的仓库，仓库间有专线相连**
>
> ------
>
> ## 3️⃣ Edge Location（边缘节点）
>
> - **400+ 个**，分布在 90+ 城市、40+ 国家
> - 不跑计算，专门**缓存内容、加速分发**（CDN）
> - 对应服务：**CloudFront**（内容分发网络）
>
> ### 🧠 类比：
>
> > Edge Location = 菜鸟驿站，东西提前放在你家附近，取的时候快
>
> ------
>
> ## 4️⃣ Global vs Region 服务 ⚠️ 必背！
>
> ### 🌐 Global 服务（不属于任何一个区）
>
> | 服务           | 干什么的     |
> | -------------- | ------------ |
> | **IAM**        | 账号权限管理 |
> | **Route 53**   | DNS 域名解析 |
> | **CloudFront** | CDN 内容分发 |
> | **WAF**        | Web 防火墙   |

# AWS Identity & Access Management (AWS IAM)

> # IAM 身份与权限管理 🔐
>
> ## 核心类比：把 AWS 账号想象成一家公司
>
> ```
> Root Account   →  公司老板（有所有权限，平时别用！）
> Users          →  员工
> Groups         →  部门（研发部、财务部）
> Policies       →  工作权限清单（你能做什么、不能做什么）
> Roles          →  临时工牌（给机器用的，不是给人用的）
> ```
>
> ------
>
> ## 1️⃣ Users & Groups
>
> - **Root账号**：创建AWS时自动生成，**平时锁起来别碰**，更别共享
> - **User**：对应一个真实的人，可以不属于任何组，也可以属于**多个组**
> - **Group**：只能装 User，**不能套娃**（组里不能放组）
>
> ### 🧠 记忆口诀：
>
> > **组只装人不装组，一个人可以进多组**
>
> ------
>
> ## 2️⃣ Policies（权限策略）= JSON 通行证
>
> 给用户或组贴一张"**你能干啥**"的清单，格式是 JSON。
>
> ```json
> {
>    "Version":"2012-10-17",
>    "Statement":[
>       {
>          "Effect":"Allow",
>          "Action":"ec2:Describe*",
>          "Resource":"*"
>       },
>       {
>          "Effect":"Allow",
>          "Action":"elasticloadbalancing:Describe*",
>          "Resource":"*"
>       },
>       {
>          "Effect":"Allow",
>          "Action":[
>             "cloudwatch:ListMetrics",
>             "cloudwatch:GetMetricStatistics",
>             "cloudwatch:Describe*"
>          ],
>          "Resource":"*"
>       }
>    ]
> }
> ```
>
> ### 核心原则：**最小权限原则**
>
> > 需要啥给啥，别多给 — 就像实习生只给他能用的门卡
>
> ### Policy 结构背这张表：
>
> | 字段          | 必填？ | 干啥的                | 记忆                 |
> | ------------- | ------ | --------------------- | -------------------- |
> | **Version**   | ✅      | 固定写 `"2012-10-17"` | 就背这个日期         |
> | **Statement** | ✅      | 权限规则的列表        | 主体内容             |
> | **Effect**    | ✅      | `Allow` 或 `Deny`     | 放行 or 拒绝         |
> | **Action**    | ✅      | 能做什么操作          | 动词（s3:GetObject） |
> | **Resource**  | ✅      | 对哪个资源生效        | 名词（哪个桶）       |
> | **Principal** | ➖      | 这条规则给谁用        | 人/账号/角色         |
> | **Condition** | ➖      | 附加条件              | 可选细化             |
> | **Sid / Id**  | ➖      | 备注名字              | 可忽略               |
>
> ### 🧠 必填5个字：**版声效动资**（Version/Statement/Effect/Action/Resource）
>
> ------
>
> ## 3️⃣ Policy 继承关系
>
> ```
> Alice 在 Dev组 + Audit组
>          ↓
> 她同时继承两个组的所有权限
>          ↓
> 如果她还有单独的 Policy，也会叠加
> ```
>
> > **权限 = 所有附加 Policy 的并集**（但 Deny 永远优先于 Allow）
>
> ------
>
> ## 4️⃣ 密码策略 & MFA
>
> ### 密码策略能设置：
>
> - 最短长度
> - 必须含大写/小写/数字/特殊字符
> - 允许用户自己改密码
> - 定期过期强制更换
> - 禁止重复用旧密码
>
> ### MFA = 密码 + 设备，两把锁🔑🔑
>
> | 设备类型     | 例子                        | 特点                    |
> | ------------ | --------------------------- | ----------------------- |
> | 虚拟MFA      | Google Authenticator、Authy | 手机App，一个设备多账号 |
> | 硬件U2F      | YubiKey                     | 插USB的实体钥匙         |
> | 硬件Key Fob  | Gemalto提供                 | 专用硬件令牌            |
> | GovCloud专用 | SurePassID提供              | 美国政府云专用          |
>
> ### 🧠 考试重点记：
>
> > **MFA = 密码被盗也没用**，因为还需要你手里的设备
>
> ------
>
> ## 5️⃣ Roles（角色）= 给机器/服务用的临时工牌
>
> **人用 User，服务用 Role！**
>
> ```
> EC2 要访问 S3？  →  给 EC2 绑一个有 S3 权限的 Role
> Lambda 要写数据库？ →  给 Lambda 绑对应 Role
> ```
>
> > 类比：保安巡逻时临时换上"可以进机房"的工牌，用完归还
>
> ### 常见 Role 场景：
>
> - **EC2 Instance Role**
> - **Lambda Function Role**
> - **CloudFormation Role**
>
> ------
>
> ## 6️⃣ 访问 AWS 的三种方式
>
> | 方式           | 用什么              | 适合谁      |
> | -------------- | ------------------- | ----------- |
> | 控制台（网页） | 用户名 + 密码 + MFA | 人          |
> | CLI（命令行）  | Access Key          | 开发者/脚本 |
> | SDK（代码里）  | Access Key          | 应用程序    |
>
> > **Access Key = Access Key ID + Secret Access Key**，就是CLI/SDK的"账号密码"，**别分享、别泄露**
>
> ------
>
> ## 7️⃣ 审计工具
>
> | 工具                       | 级别   | 干啥用                                         |
> | -------------------------- | ------ | ---------------------------------------------- |
> | **IAM Credentials Report** | 账号级 | 列出所有用户和他们的凭证状态                   |
> | **IAM Access Advisor**     | 用户级 | 看某个用户上次用了哪些服务权限，帮你瘦身Policy |
>
> ------
>
> ## ✅ 终极速记卡
>
> ```
> 👤 User    = 真实的人
> 👥 Group   = 部门（只装人）
> 📋 Policy  = JSON权限单（最小权限原则）
> 🎭 Role    = 服务用的临时权限
> 🔐 MFA     = 两把锁更安全
> ⌨️  CLI/SDK = 用Access Key，不用密码
> 🔍 审计    = Credentials Report + Access Advisor
> ```
>
> ### 🧠 最佳实践口诀：**根不用，人对人，组管权，MFA开，角色给服务，密钥别外传**
>
> ------
>
> 继续发下一章！📩

# Amazon EC2 – Basics

# Amazon EC2 – Associate

# Amazon EC2 – Instance Storage

# High Availability & Scalability

# RDS, Aurora & ElastiCache

# Amazon Route 53

# Classic Solutions Architecture

# Amazon S3

# Amazon S3 – Advanced

# Amazon S3 – Security

# CloudFront & Global Accelerator

# AWS Storage Extras

# AWS Integration & Messaging

# Containers on AWS

# Serverless Overview

# Serverless Architectures

# Databases in AWS

# Data & Analytics

# Machine Learning

# AWS Monitoring, Audit & Performance

# Advanced Identity in AWS

# AWS Security & Encryption

# Amazon VPC

# Disaster Recovery & Migrations

# More Solutions Architecture

# Other Services

# White Papers & Architectures