**把 NiFi 集群的日志和指标统一接入 CloudWatch，并建立一个集中监控 Dashboard。**

- **353262（调研阶段）** — 研究如何通过 CloudWatch **动态调整日志级别**（对应 Document 1 里 Mukul 说的**用 log4j2.xml 把 `com.philips` 设为 info、其他设为 error 来减少日志量**）。

- **353263（NiFi 日志接入）** — 在 NiFi 节点上安装/配置 CloudWatch Agent，把 NiFi 的日志发送到 CloudWatch。这就是为什么你要通过 jumpbox 连到 `Dev-cluster-nifi-node-1`（那台 m4.4xlarge，已经挂了 `CloudWatchAgentEC2Role` IAM 角色）。

  > - **1. Role (`CloudWatchAgentEC2Role`)：机器的工作证**
  >
  >   **Role 是给这台服务器（机器）用的，不是给你用的。**
  >
  >   - **它的作用**： 这台 NiFi 机器需要把日志（账本）送到 AWS CloudWatch（总行金库）。 但是 AWS 很严格，不是谁都能往金库里塞东西。 于是，你给这台机器挂了一个 **IAM Role**。这就好比给财务室的员工发了一张 **“运钞员证件”**。
  >   - **如果没有它**： CloudWatch Agent（搬运工）把日志搬到 AWS 门口，AWS 门卫会拦住它：“你是谁？你有证件吗？” Agent 拿不出证件（Role），AWS 就会拒绝接收日志 (**Access Denied**)。
  >   - **总结**：Role 解决了 **“这台机器有没有资格往 AWS 发数据”** 的问题。
  >
  >   **2. Jumpbox (跳板机)：你的防盗门钥匙**
  >
  >   **Jumpbox 是给你（操作员）用的，用来打通网络。**
  >
  >   - **它的作用**： NiFi 服务器藏在 **Private Subnet（内网）** 里，就像财务室建在地下金库，根本没有连接互联网的大门。 你在家里（公网），根本摸不到它。 **Jumpbox** 是唯一连接地面和地下金库的 **“特许通道”**。
  >   - **为什么需要它**： 因为你要去修改 `log4j2.xml`（修改记账规则）。你必须亲自进入这个房间去改文件。 既然房间没有外网门，你只能先进入 Jumpbox（地面大厅），再通过内部电梯（内网 SSH）进入 NiFi 房间。
  >   - **总结**：Jumpbox 解决了 **“你如何物理连接到这台机器”** 的问题。

- **375696（扩展 + Dashboard）** — 不仅是 NiFi，还要把 Postgres 日志、EC2 系统指标也一起导入 CloudWatch，然后在 Test 环境建一个集中 Dashboard。

- **363572（收尾决策）** — 最终确定用 CloudWatch 方案（而非 Grafana+Loki），对应 Document 1 里那张对比表的结论。

------

**为什么要通过 Jumpbox 访问 NiFi？**

NiFi 节点虽然有公网 IP（34.226.78.173），但它的安全组很可能只允许 VPC 内部或特定 IP 访问 9443 端口。

- Jumpbox（54.167.7.191）在同一个 VPC 和子网（`citus-subnet-public1-us-east-1a`）里，所以流程是：你 RDP 到 Jumpbox → 从 Jumpbox 的浏览器访问 `https://ec2-34-226-78-173...:9443/nifi`。这是标准的堡垒机模式，为了安全。

------

**你现在具体该做什么**

1. **在 NiFi 节点上配置 CloudWatch Agent** — SSH 到 `Dev-cluster-nifi-node-1`，安装并配置 agent，指定要采集的日志文件路径（NiFi 的 `nifi-app.log` 等）和目标 Log Group。
2. **创建 Log Group** — 按 Mukul 的要求，为 Dev 环境创建 NiFi 相关的 Log Group，retention 设 3 天，加上 tag。
3. **调整日志级别** — 修改 NiFi 的 `log4j2.xml`（或通过 `logback.xml`），减少非必要日志。
4. **建 CloudWatch Dashboard** — 这是 353263 和 375696 的交付物，把 NiFi 流量、错误率、EC2 指标等关键 metrics 放到一个 Dashboard 里。
5. **最终写出结论** — 在 363572 里总结为什么选 CloudWatch（成本低、运维少、团队技能匹配），对应 Document 1 的分析。

------

# CloudWatch + NiFi 监控体系搭建 — 详细操作指南

------

## 整体目标（一句话）

把 NiFi 节点上的日志文件 → 通过 CloudWatch Agent → 发送到 AWS CloudWatch → 在 CloudWatch 里创建 Dashboard 看图表 → 最终实现集中监控。

------

## 第一步：在 NiFi 节点上安装和配置 CloudWatch Agent

### 这一步到底在干什么？

你的 NiFi 跑在一台 EC2 上（`Dev-cluster-nifi-node-1`，IP `34.226.78.173`）。NiFi 在运行时会不断往本地磁盘写日志文件（比如 `/opt/nifi/logs/nifi-app.log`）。但问题是，这些日志只存在于那台 EC2 的硬盘上，你必须 SSH 进去才能看。

CloudWatch Agent 是 AWS 提供的一个小程序，装在 EC2 上之后，它会像一个"快递员"一样，持续读取你指定的本地日志文件，然后把内容实时发送到 AWS CloudWatch 服务端。这样你就不用 SSH 进机器看日志了，直接在 AWS 控制台就能看。

### 具体流程

```
┌─────────────────────────────────────────────────────┐
│  EC2: Dev-cluster-nifi-node-1                       │
│                                                     │
│  NiFi 运行中 ──写日志──▶ /opt/nifi/logs/nifi-app.log │
│                                                     │
│  CloudWatch Agent（你要安装的）                       │
│       │  持续读取上面这个日志文件                      │
│       │  按你的配置，发送到指定的 Log Group            │
│       ▼                                             │
└──── 通过网络发送到 AWS CloudWatch 服务端 ─────────────┘
                        │
                        ▼
         ┌─────────────────────────────┐
         │  AWS CloudWatch             │
         │  Log Group: /nifi/dev/app   │
         │    └─ Log Stream (按实例)    │
         │       └─ 你的日志内容都在这里 │
         └─────────────────────────────┘
```

### 操作步骤

**1. SSH 到 NiFi 节点**

你不能直接 SSH（安全组限制），所以先 RDP 到 jumpbox（54.167.7.191），然后从 jumpbox 内部 SSH 到 NiFi 节点的内网 IP：

```
你的电脑 ──RDP──▶ Jumpbox (54.167.7.191) ──SSH──▶ NiFi Node (10.0.0.172)
```

**2. 安装 CloudWatch Agent**

在 NiFi 节点上执行：

```bash
# 下载 agent 安装包
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

# 安装
sudo dpkg -i amazon-cloudwatch-agent.deb
```

**3. 创建 Agent 配置文件**

这个配置文件告诉 Agent："监控哪个日志文件"、"发送到哪个 Log Group"、"还要不要采集 CPU/内存等系统指标"。

在 `/opt/aws/amazon-cloudwatch-agent/etc/` 下创建 `amazon-cloudwatch-agent.json`：

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/opt/nifi/logs/nifi-app.log",
            "log_group_name": "/nifi/dev/app",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          },
          {
            "file_path": "/opt/nifi/logs/nifi-user.log",
            "log_group_name": "/nifi/dev/user",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          },
          {
            "file_path": "/opt/nifi/logs/nifi-bootstrap.log",
            "log_group_name": "/nifi/dev/bootstrap",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "NiFi/Dev",
    "metrics_collected": {
      "cpu": { "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"] },
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["disk_used_percent"], "resources": ["*"] }
    }
  }
}
```

**每个字段的含义：**

| 字段              | 意思                                                         |
| ----------------- | ------------------------------------------------------------ |
| `file_path`       | Agent 要监控的本地日志文件路径。NiFi 默认会写 `nifi-app.log`（主日志）、`nifi-user.log`（用户操作日志）、`nifi-bootstrap.log`（启动日志） |
| `log_group_name`  | 日志发送到 CloudWatch 后归入哪个"文件夹"。你可以自己命名，比如 `/nifi/dev/app` 表示"NiFi 的 Dev 环境的应用日志" |
| `log_stream_name` | 每个 Log Group 里面可以有多个 Stream。用 `{instance_id}` 意味着按 EC2 实例 ID 自动命名，这样如果以后有多个 NiFi 节点，日志不会混在一起 |
| `metrics` 部分    | 额外采集 EC2 的 CPU、内存、磁盘使用率等系统指标，发送到 CloudWatch Metrics，后面建 Dashboard 时可以用 |

**4. 确保 IAM 权限**

Agent 需要权限才能往 CloudWatch 写数据。你的 NiFi 节点已经绑了 `CloudWatchAgentEC2Role` 这个 IAM Role（你贴的配置里可以看到），所以这一步已经搞定了。如果没有这个 Role，Agent 发日志时会被 AWS 拒绝。

**5. 启动 Agent**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

这条命令的意思是：读取你刚写的配置文件，然后启动 Agent。启动后它会在后台持续运行，自动把日志发送到 CloudWatch。

**6. 验证是否成功**

去 AWS 控制台 → CloudWatch → Log Groups，看看有没有出现 `/nifi/dev/app` 等 Log Group，里面有没有日志内容。如果有，说明 Agent 配置成功。

------

## 第二步：创建 Log Group 并设置保留期

### 这一步到底在干什么？

上一步里 Agent 把日志发送到了 CloudWatch 的 Log Group 里。但如果你不主动管理这些 Log Group，AWS 会**永久保留**所有日志，这意味着存储费会越来越高。

Mukul 的要求是：Dev/Test/Demo 环境的日志只保留 3 天（因为这些环境的日志没有长期价值），Prod 环境的保留期需要跟架构师讨论（可能 7-30 天，然后归档到 S3）。

另外，要给每个 Log Group 打 Tag（标签），方便以后按环境、按服务筛选和计费。

### 为什么用 Terraform？

你可以在 AWS 控制台手动创建 Log Group，但问题是你有多个环境（dev/test/demo/prod），每个环境有多个 Log Group（app、user、bootstrap），手动创建容易遗漏或不一致。Terraform 是一个基础设施即代码（IaC）工具，你写一次配置，它帮你在所有环境批量创建，而且配置可以存到 Git 里做版本管理。

### Terraform 示例

```hcl
# 定义变量：哪些环境、每个环境保留几天
variable "environments" {
  default = {
    dev  = 3    # 3 天
    test = 3
    demo = 3
    prod = 30   # 生产环境先设 30 天，后续跟架构师确认
  }
}

# NiFi 会产生的日志类型
variable "log_types" {
  default = ["app", "user", "bootstrap"]
}

# 批量创建 Log Group
resource "aws_cloudwatch_log_group" "nifi_logs" {
  for_each = {
    for pair in setproduct(keys(var.environments), var.log_types) :
    "${pair[0]}-${pair[1]}" => {
      env       = pair[0]
      log_type  = pair[1]
      retention = var.environments[pair[0]]
    }
  }

  name              = "/nifi/${each.value.env}/${each.value.log_type}"
  retention_in_days = each.value.retention

  tags = {
    Environment = each.value.env
    Service     = "nifi"
    LogType     = each.value.log_type
    ManagedBy   = "terraform"
  }
}
```

**运行 Terraform 后会创建什么：**

| Log Group 名称        | 保留天数 | Tag                                              |
| --------------------- | -------- | ------------------------------------------------ |
| `/nifi/dev/app`       | 3 天     | Environment=dev, Service=nifi, LogType=app       |
| `/nifi/dev/user`      | 3 天     | Environment=dev, Service=nifi, LogType=user      |
| `/nifi/dev/bootstrap` | 3 天     | Environment=dev, Service=nifi, LogType=bootstrap |
| `/nifi/test/app`      | 3 天     | Environment=test, Service=nifi, LogType=app      |
| ...以此类推...        |          |                                                  |
| `/nifi/prod/app`      | 30 天    | Environment=prod, Service=nifi, LogType=app      |

------

## 第三步：调整日志级别（减少日志量 = 省钱）

### 这一步到底在干什么？

NiFi 是一个 Java 应用，它内部用了很多第三方库（比如 Apache Jetty、Spring、Zookeeper 等）。默认情况下，如果日志级别是 INFO，那么 NiFi 自己的代码 + 所有第三方库的代码都会输出 INFO 级别的日志。但实际上你只关心 NiFi 和你们公司（`com.philips`）的 INFO 日志，第三方库的 INFO 日志对你没用，还会产生大量数据，增加 CloudWatch 的费用。

所以 Mukul 的策略是：

```
com.philips 包      → INFO 级别（正常记录，因为这是你们自己的代码）
org.apache.nifi 包  → INFO 级别（NiFi 核心日志，需要看）
其他所有包           → ERROR 级别（只记录错误，跳过无用的 INFO 日志）
```

### 日志级别科普

```
TRACE → DEBUG → INFO → WARN → ERROR → FATAL
 最详细                                最简略

设为 INFO：会记录 INFO + WARN + ERROR + FATAL
设为 ERROR：只记录 ERROR + FATAL（大幅减少日志量）
```

### 怎么改？

找到 NiFi 的日志配置文件，通常在 `/opt/nifi/conf/logback.xml` 或 `log4j2.xml`（取决于 NiFi 版本）：

```xml
<Configuration>
  <Loggers>
    <!-- 你们公司的代码：保持 INFO -->
    <Logger name="com.philips" level="INFO" />

    <!-- NiFi 核心：保持 INFO -->
    <Logger name="org.apache.nifi" level="INFO" />

    <!-- 其他所有第三方库：只记 ERROR，大幅减少日志量 -->
    <Root level="ERROR">
      <AppenderRef ref="APP_FILE" />
    </Root>
  </Loggers>
</Configuration>
```

**改之前 vs 改之后的效果（举例）：**

| 日志来源                                    | 改之前（全 INFO）         | 改之后       |
| ------------------------------------------- | ------------------------- | ------------ |
| `com.philips.pipeline.DataProcessor`        | ✅ 记录 INFO               | ✅ 记录 INFO  |
| `org.apache.nifi.controller.FlowController` | ✅ 记录 INFO               | ✅ 记录 INFO  |
| `org.eclipse.jetty.server.Server`           | ✅ 记录 INFO（没用但很多） | ❌ 只记 ERROR |
| `org.apache.zookeeper.ClientCnxn`           | ✅ 记录 INFO（没用但很多） | ❌ 只记 ERROR |
| `org.springframework.beans.factory`         | ✅ 记录 INFO（没用但很多） | ❌ 只记 ERROR |

**省钱效果：** CloudWatch 按日志数据量收费（$0.50/GB 入库），减少无用日志可能直接砍掉 50%-70% 的日志量。

### 改完后要做什么？

重启 NiFi 使配置生效（或者如果 NiFi 支持热加载日志配置则不需要重启）。然后去 CloudWatch 观察日志量是否明显下降。

------

## 第四步：建 CloudWatch Dashboard

### 这一步到底在干什么？

前三步完成后，你的日志和系统指标已经全部进入 CloudWatch 了。但它们是"原始数据"，你需要在 CloudWatch 里创建一个 Dashboard（仪表盘），把这些数据用图表的形式展示出来。这样任何人打开这个 Dashboard 就能一目了然地看到系统的健康状况，而不需要手动去查日志或查指标。

### Dashboard 上应该放什么？

按照 ticket 375696 的要求，Dashboard 需要涵盖 NiFi + Postgres + EC2 三个层面：

```
┌─────────────────────────────────────────────────────────────┐
│  CloudWatch Dashboard: NiFi-Dev-Cluster                     │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ EC2 CPU 使用率    │  │ EC2 内存使用率    │  ← 系统层指标   │
│  │ (折线图)          │  │ (折线图)          │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ EC2 磁盘使用率    │  │ EC2 网络流量      │  ← 系统层指标   │
│  │ (折线图)          │  │ (折线图)          │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ NiFi 错误日志数量  │  │ NiFi 警告日志数量  │ ← 从日志里查询 │
│  │ (柱状图，每小时)   │  │ (柱状图，每小时)   │                │
│  └──────────────────┘  └──────────────────┘                 │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ Postgres 连接数   │  │ Postgres 磁盘 IO  │ ← 数据库层     │
│  │ (折线图)          │  │ (折线图)          │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                             │
│  ┌─────────────────────────────────────────┐                │
│  │ 最近的错误日志（实时滚动）                │ ← Logs widget  │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 两种数据来源

Dashboard 上的 widget（小部件）有两类数据来源：

**1. Metrics（指标）— 免费的 AWS 默认指标**

这些是 AWS 自动采集的，不需要你做任何配置：

- EC2 的 CPU 使用率（`CPUUtilization`）
- EC2 的网络流入/流出（`NetworkIn`/`NetworkOut`）
- EC2 的磁盘读写（`DiskReadOps`/`DiskWriteOps`）
- RDS（Postgres）的连接数、IOPS 等

加上你在第一步 Agent 里配置的自定义指标（内存、磁盘使用率百分比）。

**2. Logs Insights Query（日志查询）— 按扫描量收费**

你可以在 Dashboard 里嵌入 CloudWatch Logs Insights 查询，比如：

```sql
-- 每小时 ERROR 日志数量
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by bin(1h)
```

这就是 Mukul 说的"像写 SQL 一样查日志"，要注意控制时间范围和扫描量，因为 CloudWatch 按扫描数据量收费（$0.005/GB）。

### 费用

每个 Dashboard 每月 $3。Mukul 的策略是给 Dev、Test、Demo、Prod 各建一个，总计 $12/月。

### 怎么创建？

**方法一（手动）：** AWS 控制台 → CloudWatch → Dashboards → Create Dashboard → 添加 Widget → 选指标或日志查询。

**方法二（Terraform，推荐）：** 用 `aws_cloudwatch_dashboard` 资源，把 Dashboard 的 JSON 定义写在 Terraform 里，这样可以版本管理，也方便在多个环境复制。

### 关于已有的 Dashboard

Mukul 提到了两个已有 Dashboard：

- `NifiDashboard` — 没人用，可以删掉（省 $3/月）
- `CloudWatch-Default` — 在用，但需要跟 DevOps 团队对齐内容

------

## 第五步：最终结论文档（363572）

### 这一步到底在干什么？

ticket 363572 "Finalize the Cloudwatch Approach" 是一个决策文档，需要你把前面的调研结果整理成正式结论。核心内容就是 Document 1 里那张 CloudWatch vs Grafana+Loki 的对比表，加上你的推荐理由。

### 结论要点

**选 CloudWatch，理由：**

1. **成本合理** — Dev/Test/Demo 预估 < $50/月（日志量 < 10GB/天，3 个 Dashboard，默认指标免费）
2. **零运维开销** — AWS 全托管，不需要团队去维护 EC2、打补丁、升级 Grafana
3. **团队技能匹配** — CloudWatch 学习成本低，不需要深度 observability 经验
4. **安全合规** — 原生 IAM/KMS 集成，不需要自己搞 RBAC/TLS
5. **Grafana+Loki 的劣势** — 需要额外 2 台 EC2（$100-200/月），需要有人维护，团队目前没有这个人力

**已采取的省钱措施：**

- 关闭 Container Insights（不需要按实例级别的伸缩指标）
- 日志级别过滤（`com.philips` = INFO，其他 = ERROR，减少 50%+ 日志量）
- Log Group 保留期 3 天（非 Prod 环境）
- 日志查询规范（缩小时间范围、只查特定 Log Group、只搜特定关键词）

------

## 整体流程图总结

```
┌─────────┐    ┌───────────┐    ┌──────────────────┐    ┌──────────────┐
│ 第三步   │    │ 第一步     │    │ 第二步            │    │ 第四步        │
│ 调整日志 │───▶│ 安装Agent  │───▶│ 创建Log Group    │───▶│ 建Dashboard  │
│ 级别     │    │ 发送日志   │    │ 设保留期+Tag     │    │ 可视化展示    │
│          │    │            │    │                  │    │              │
│ 减少量   │    │ 日志上云   │    │ 控制成本+管理    │    │ 集中监控     │
└─────────┘    └───────────┘    └──────────────────┘    └──────────────┘
                                                              │
                                                              ▼
                                                     ┌──────────────┐
                                                     │ 第五步        │
                                                     │ 写结论文档    │
                                                     │ 关闭ticket   │
                                                     └──────────────┘
```

> **注意：** 第三步（调整日志级别）应该在第一步之前或同时做，因为你希望 Agent 开始发送的就是已经过滤后的日志，而不是先发一堆垃圾日志再去调。

## 验证前三步是否完成的具体检查方法

------

### 验证第一步：CloudWatch Agent 是否在运行并发送日志

#### 在 AWS Console 上检查

1. **CloudWatch → Log groups** → 搜索 `/nifi/dev/`
2. 看是否存在这三个 Log Group：
   - `/nifi/dev/app`
   - `/nifi/dev/user`
   - `/nifi/dev/bootstrap`
3. 点进去任意一个 → 看 **Log streams** 里有没有以实例 ID 命名的 stream（类似 `i-0a1b2c3d4e5f`）
4. 点进 stream → 看里面有没有**最近几分钟**的日志内容

**如果 Log Group 存在但没有最近的日志** → Agent 可能挂了或配置有误

1. 再检查自定义指标：**CloudWatch → Metrics → Custom namespaces** → 看有没有 **NiFi/Dev** 这个 namespace
2. 点进去看 `mem_used_percent`、`cpu_usage_idle`、`disk_used_percent` 有没有数据点

#### 如果需要 SSH 到机器上确认（通过 Jumpbox）

RDP 到 Jumpbox → SSH 到 NiFi 节点后执行：

```bash
# 查看 Agent 运行状态
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# 期望输出里看到 "status": "running"
# 如果是 "stopped" 说明 Agent 没启动
# 查看 Agent 自己的日志，看有没有报错
sudo tail -50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# 重点关注有没有 "Access Denied"（IAM 权限问题）
# 或 "file not found"（日志路径配错了）
# 确认配置文件存在
cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# 检查 file_path 里写的路径是否真的有日志文件
ls -la /opt/nifi/logs/
# 应该能看到 nifi-app.log、nifi-user.log、nifi-bootstrap.log
```

------

### 验证第二步：Log Group 保留期和 Tag 是否设置正确

#### 在 AWS Console 上检查

1. **CloudWatch → Log groups** → 找到 `/nifi/dev/app`
2. 看 **Retention** 列 → 应该显示 **3 days**，不是 "Never expire"
3. 对 `/nifi/dev/user` 和 `/nifi/dev/bootstrap` 做同样检查
4. 点进任意一个 Log Group → 点 **Tags** 标签页 → 确认有以下 tag：

| Key         | Expected Value         |
| ----------- | ---------------------- |
| Environment | dev                    |
| Service     | nifi                   |
| LogType     | app / user / bootstrap |
| ManagedBy   | terraform              |

**如果 Retention 是 "Never expire"** → 第二步没做完，日志会无限累积产生费用

**如果没有 Tag** → 以后没法按环境和服务筛选计费，需要补上

------

### 验证第三步：日志级别是否已调整

这一步没法光在 Console 上看配置，但可以通过**观察日志内容**来判断。

#### 方法一：在 CloudWatch 控制台查日志内容

1. **CloudWatch → Log groups → `/nifi/dev/app`** → 点进最新的 Log stream
2. 看最近的日志条目，观察日志来源包名：
   - ✅ 应该看到 `com.philips.xxx` 的 INFO 日志
   - ✅ 应该看到 `org.apache.nifi.xxx` 的 INFO 日志
   - ❌ 不应该大量出现 `org.eclipse.jetty`、`org.apache.zookeeper`、`org.springframework` 的 INFO 日志
3. 如果第三方库的 INFO 日志铺天盖地 → 日志级别没改

#### 方法二：用 Logs Insights 查询验证

1. **CloudWatch → Logs Insights**
2. 选 Log Group `/nifi/dev/app`
3. 时间范围选最近 1 小时
4. 执行这个查询：

```
fields @message
| filter @message like /INFO/
| parse @message /(?<logger>[a-zA-Z0-9_.]+)/
| stats count(*) as cnt by logger
| sort cnt desc
| limit 20
```

1. 看结果里排名靠前的 logger 是哪些：
   - 如果 `org.eclipse.jetty`、`org.apache.zookeeper` 等第三方库排名靠前且数量很大 → **第三步没做**
   - 如果 INFO 日志主要来自 `com.philips` 和 `org.apache.nifi` → **第三步已完成**

#### 方法三：SSH 到机器上直接看配置文件

```bash
# 查看日志配置
cat /opt/nifi/conf/logback.xml
# 或者
cat /opt/nifi/conf/log4j2.xml

# 重点确认：
# 1. Root level 是否设为 "ERROR"
# 2. com.philips 是否单独设为 "INFO"
# 3. org.apache.nifi 是否单独设为 "INFO"
```

------

### 汇总检查清单

| 检查项                   | 怎么查                    | 期望结果                      | 状态 |
| ------------------------ | ------------------------- | ----------------------------- | ---- |
| Agent 在运行             | SSH 到节点查 status       | `"running"`                   | ☐    |
| Log Group 存在           | Console → Log groups      | 3 个 group 都在               | ☐    |
| Log Stream 有最近数据    | 点进 Log Group 看         | 几分钟内有新日志              | ☐    |
| 自定义指标存在           | Metrics → NiFi/Dev        | 有 CPU/内存/磁盘数据点        | ☐    |
| 保留期设为 3 天          | Log Group Retention 列    | 显示 3 days                   | ☐    |
| Tag 已打                 | Log Group → Tags 标签页   | 4 个 tag 都在                 | ☐    |
| 第三方库 INFO 日志已过滤 | Logs Insights 查询        | INFO 主要来自 philips 和 nifi | ☐    |
| IAM Role 已绑定          | EC2 → 实例详情 → IAM Role | `CloudWatchAgentEC2Role`      | ☐    |

前面 8 项全部通过后，你就可以放心开始建 Dashboard 了——所有数据源都已就绪。如果任何一项没通过，先找负责对应步骤的人解决，否则你建的 Dashboard 要么没数据，要么数据不准确。

好问题。

从你文档的实际操作来看，你确实**只登录了一台 EC2**（`Dev-cluster-nifi-node-1`）。但它名字里带 "cluster" 是因为 **NiFi 本身是按集群架构设计的**，即使当前 Dev 环境可能只有这一个节点。

几种可能的情况：

**Dev 环境目前就是单节点。** 很多团队在 Dev 环境只部署一个 NiFi 节点来省钱，但命名上沿用了集群的命名规范（`Dev-cluster-nifi-node-1`），方便以后扩展时直接加 `node-2`、`node-3`。名字里的 "node-1" 本身就暗示了它是集群里的第一个节点。

**Prod 环境可能是真正的多节点集群。** 生产环境为了高可用，通常会跑 3 个或更多 NiFi 节点。你现在在 Dev 上做的配置（Agent 安装、日志级别调整）相当于先在单节点上验证流程，以后推广到 Prod 时每个节点都要重复同样的操作。

**对你建 Dashboard 的影响：** 这也是为什么 Agent 配置里 `log_stream_name` 用的是 `{instance_id}` — 如果以后有多个节点，每个节点的日志会自动进入同一个 Log Group 下不同的 Stream，不会混在一起。你建 Dashboard 时选指标也可以按 instance 筛选或聚合。

所以严格来说，你现在操作的是集群里的一个节点，叫它"NiFi 节点"比"NiFi 集群"更准确。我之前沿用了文档里的说法，确实不太严谨。



这很正常，不矛盾。你看到的情况说明：

**你通过 Jumpbox 访问的 NiFi UI，可能不是 Dev 环境的，而是另一个环境（比如 Test 或 Prod）的 NiFi 集群。** 或者这些节点在另一个 AWS 账号下。

从截图可以看到几个关键信息：

**确实是一个 4 节点的真实集群：** 3 个 CONNECTED，1 个 DISCONNECTED。其中 `ec2-44-198-154-131` 是 PRIMARY COORDINATOR（集群协调者）。这些节点从 2025 年 5 月底就启动了，已经跑了大半年。

**为什么在你的 Dev account 里找不到这 4 台 EC2：** 最可能的原因是这个 NiFi 集群部署在**另一个 AWS 账号**里。很多企业会把不同环境放在不同账号下（比如 Dev 一个账号、Test/Prod 一个账号），或者 NiFi 集群统一放在一个共享服务账号里。你的 Dev account 里的那台 `Dev-cluster-nifi-node-1` 可能是新建的、还没加入集群的节点，或者是另一套独立部署。

**建议你跟团队确认几件事：**

1. 你通过 Jumpbox 访问的这个 NiFi UI（9443 端口）到底属于哪个环境/哪个 AWS 账号？
2. 你文档里提到的 `Dev-cluster-nifi-node-1`（IP `34.226.78.173`）是否就是截图里这 4 个节点之一？你可以对比一下 IP 地址。
3. 你负责建 Dashboard 的目标范围是这 4 个节点，还是只针对你 Dev account 里的那一台？

搞清楚这个归属关系很重要，因为它直接影响你 Dashboard 里要引用哪些实例的指标、哪个账号的 CloudWatch 数据。

很好的问题，这里其实涉及三个层次：

## 日志的"存在哪"和"看在哪"

**第一层：日志的源头 — EC2 本地磁盘**

NiFi 运行时，日志实时写入这台机器的硬盘上，比如 `/opt/nifi/logs/nifi-app.log`。这是日志最原始的存在位置。问题是你必须 SSH 进机器才能看，而且磁盘空间有限，旧日志会被 NiFi 自动轮转覆盖掉。

**第二层：日志的存储 — CloudWatch Log Groups**

CloudWatch Agent 把本地日志文件的内容持续搬运到 AWS CloudWatch 服务端。日志实际存储在 **Log Groups** 里（比如 `/nifi/dev/app`）。这才是真正的"存"。它解决的问题是：不用 SSH 进机器就能查日志，多个节点的日志集中在一个地方，可以设保留期（比如 3 天后自动删除控制成本），可以用 Logs Insights 做搜索和统计。

**第三层：日志的展示 — Dashboard**

Dashboard 本身**不存任何数据**。它只是一组图表和查询的集合，每次你打开它，它会实时去 Log Groups 和 Metrics 里拉数据然后画图。你可以把它理解为一个保存好的"书签页面"。

## 用一个比喻串起来

```
NiFi 写日志到本地磁盘        →  你办公桌上的手写笔记本（原始记录，翻起来费劲）
          │
    CloudWatch Agent 搬运
          │
          ▼
CloudWatch Log Groups        →  公司的档案室（统一存放，可检索，到期自动销毁）
          │
    Dashboard 读取展示
          │
          ▼
CloudWatch Dashboard         →  档案室门口的电子屏幕（实时显示摘要，本身不存东西）
```

所以回答你的问题：**你们存的是 Log Groups 里的日志数据**，这是 CloudWatch 按量收费的部分（$0.50/GB 入库 + 存储费）。Dashboard 只是一个展示窗口，它本身每月固定 $3，不管你看多少次。保留期设 3 天就意味着 Log Groups 里的日志 3 天后自动删掉，不再占存储费用。