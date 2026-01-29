下面是一份「**新手一次做对**」的完整指南，按你这次真实环境写死（**us-east-1 / 账号 983105217597 / 两台 EC2**），并把我们一路踩过的坑全部总结在后面。照着做，你今天 **2026-01-28（美东）18:00 会自动 Stop**，明天 **2026-01-29 06:00 会自动 Start**。

------

# 指南：EventBridge Scheduler → SSM Automation 每天定时 Stop/Start 两台 EC2（Tag 安全阀）

## 目标

- 每天 **18:00（America/New_York）自动 Stop**
- 每天 **06:00（America/New_York）自动 Start**
- 只允许操作打了 Tag 的实例（防误伤）

## 你的固定信息

- Region：`us-east-1`
- Account：`983105217597`
- 实例：
  - `i-0c2df004c0753c15f`
  - `i-06d24e17dcb5bfff7`
- Tag（必须）：`AutoSchedule = OfficeHours`

------

# Step 0：给两台 EC2 打 Tag（必须）

1. EC2 Console → Instances
2. 勾选两台实例 → 下方 **Tags** → **Manage tags**
3. 添加：
   - Key：`AutoSchedule`
   - Value：`OfficeHours`
   - `（AutoStartStop, OfficeHours）`
4. Save

> 后面我们的权限策略会写成：**只有带这个 Tag 的实例才能 Start/Stop**（安全阀）。

------

# Step 1：创建 Automation 执行角色（给 SSM Automation 用）

这个角色是让 **SSM Automation runbook** 代你调用 EC2 API 的。官方 `AWS-StopEC2Instance`/`AWS-StartEC2Instance` runbook 参数里就支持 `AutomationAssumeRole` + `InstanceId`。 ([AWS Documentation](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-aws-stopec2instance.html))

## 1.1 创建 Role

IAM → Roles → Create role

- Trusted entity：**AWS service**
- Service：**Systems Manager / SSM**
- Role name：`AutomationEc2StartStopRole`

## 1.2 给 Role 加 Inline Policy（你这次最终“正确版”）

IAM → Roles → `AutomationEc2StartStopRole` → Permissions → Add permissions → Create inline policy → JSON 粘贴：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeInstances",
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    },
    {
      "Sid": "DescribeInstanceStatus",
      "Effect": "Allow",
      "Action": "ec2:DescribeInstanceStatus",
      "Resource": "*"
    },
    {
      "Sid": "StartStopOnlyTagged",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "arn:aws:ec2:us-east-1:983105217597:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/AutoSchedule": "OfficeHours"
        }
      }
    }
  ]
}
```

✅ 关键点：一定要有 `ec2:DescribeInstanceStatus`，否则 stop/start runbook 在“等待实例到达目标状态”时会 403（我们就踩过这个坑）。

------

# Step 2：手动冒烟测试（强烈建议，5 分钟定位权限问题）

Systems Manager → Automation → **Execute runbook**

- Stop 用：`AWS-StopEC2Instance`
- Start 用：`AWS-StartEC2Instance`

填参数：

- `InstanceId`：选/填你的实例 ID（两台）
- `AutomationAssumeRole`：选择 `AutomationEc2StartStopRole` 的 ARN

> 注意：实例不需要是 “Managed instance”。`AWS-StopEC2Instance` 的本质是调用 EC2 API 去停机，不要求 SSM agent 托管。 ([AWS Documentation](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-aws-stopec2instance.html))

✅ 通过标准：

- Automation execution = Success
- EC2 状态按预期变成 stopped / running

------

# Step 3：创建 Scheduler 执行角色（给 EventBridge Scheduler 用）

为什么还要第二个 Role？

- Scheduler 需要先 **assume 一个执行角色**，再去调用 `ssm:StartAutomationExecution`
- 并且因为你在参数里传了 `AutomationAssumeRole`，Scheduler 还需要 `iam:PassRole` 把它“交给”SSM Automation

官方文档明确 Scheduler 的信任主体是 `scheduler.amazonaws.com`。 ([AWS Documentation](https://docs.aws.amazon.com/scheduler/latest/UserGuide/setting-up.html))

## 3.1 创建 Role（用 Custom trust policy）

IAM → Roles → Create role → 选择 **Custom trust policy**，粘贴：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "scheduler.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Role name：`EventBridgeScheduler-StartAutomationRole`

## 3.2 给它加 Inline Policy（Scheduler 触发 Automation 必备）

IAM → Roles → `EventBridgeScheduler-StartAutomationRole` → Permissions → Create inline policy → JSON：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartAutomation",
      "Effect": "Allow",
      "Action": "ssm:StartAutomationExecution",
      "Resource": "*"
    },
    {
      "Sid": "PassAutomationRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::983105217597:role/AutomationEc2StartStopRole"
    }
  ]
}
```

**下面这样就无需分别id了：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "StartAutomation",
      "Effect": "Allow",
      "Action": "ssm:StartAutomationExecution",
      "Resource": "*"
    },
    {
      "Sid": "PassAutomationAssumeRoleToSSM",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::983105217597:role/AutomationEc2StartStopRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "ssm.amazonaws.com"
        }
      }
    },
    {
      "Sid": "ResolveTargetsByTag",
      "Effect": "Allow",
      "Action": "tag:GetResources",
      "Resource": "*"
    }
  ]
}

```

为什么要这三段：

1. Scheduler 需要 `ssm:StartAutomationExecution` 才能“发起 Automation”。（这是你报错的动作）
2. 你在参数里传了 `AutomationAssumeRole`，所以 Scheduler 还必须能 `iam:PassRole` 把这个 role 交给 SSM。
3. 你现在用 **Targets=tag:** 来选实例，SSM 在展开目标时会用到 **Tagging API** 能力，缺了就可能出现你这种“AccessDenied/unknown error”。`tag:GetResources` 是最常见的补齐项。

> 为啥我这里 `Resource:"*"`？
>  因为 `AWS-StartEC2Instance / AWS-StopEC2Instance` 是 AWS-owned 文档，ARN 写法经常让人踩坑（account 是 `aws` 或空），你现在目标是“**一次成功不出错**”，先用 `*` 最稳。等稳定后再做 least privilege。

------

# Step 4：创建 Stop Schedule（每天 18:00）

EventBridge → Scheduler → Create schedule

## 4.1 基本设置

- Name：`ec2-stop-1800`
- Schedule pattern：Cron
- Cron：`cron(0 18 * * ? *)`
- Timezone：`America/New_York`
- Flexible time window：Off

## 4.2 Target（Systems Manager → StartAutomationExecution）

Target service：Systems Manager
Operation：StartAutomationExecution
Input JSON 粘贴（你两台实例已填好）：

### Stop（每天 18:00）

```
{
  "DocumentName": "AWS-StopEC2Instance",
  "Parameters": {
    "AutomationAssumeRole": [
      "arn:aws:iam::983105217597:role/AutomationEc2StartStopRole"
    ]
  },
  "TargetParameterName": "InstanceId",
  "Targets": [
    {
      "Key": "tag:AutoSchedule",
      "Values": ["OfficeHours"]
    }
  ]
}
```

### 

## 4.3 Permissions

- Execution role：**Use existing role**
- 选择：`EventBridgeScheduler-StartAutomationRole`

Create schedule。

------

# Step 5：创建 Start Schedule（每天 06:00）

同样流程再建一个：

## 5.1 基本设置

- Name：`ec2-start-0600`
- Cron：`cron(0 6 * * ? *)`
- Timezone：`America/New_York`
- Flexible time window：Off

## 5.2 Target Input JSON

### Start（每天 06:00）

```
{
  "DocumentName": "AWS-StartEC2Instance",
  "Parameters": {
    "AutomationAssumeRole": [
      "arn:aws:iam::983105217597:role/AutomationEc2StartStopRole"
    ]
  },
  "TargetParameterName": "InstanceId",
  "Targets": [
    {
      "Key": "tag:AutoSchedule",
      "Values": ["OfficeHours"]
    }
  ]
}
```

## 5.3 Permissions

同样选：`EventBridgeScheduler-StartAutomationRole`

Create schedule。

------

# Step 6：上线验收（你只需要看两处）

## 6.1 Scheduler 侧

EventBridge Scheduler → 打开两个 schedule，确认：

- State = Enabled
- Timezone = America/New_York
- Next invocation time 正确
  （今天 2026-01-28 18:00 会触发 stop；明天 2026-01-29 06:00 会触发 start）

## 6.2 Automation 侧

Systems Manager → Automation → Executions
到点会出现新执行记录；失败会给出明确原因（权限/参数/Tag）。

------

# 我们这次踩过的坑与“正确修复方式”

1. **Automation 页面列表空 / 选不到实例**

- 原因：你用的 picker 默认只显示 “managed instances”
- 修复：切换显示 all 或直接手动填 `InstanceId`（runbook 不要求 managed）

1. **Stop 执行失败：缺 `ec2:DescribeInstanceStatus`**

- 报错：403 UnauthorizedOperation ec2:DescribeInstanceStatus
- 修复：在 `AutomationEc2StartStopRole` 加 `ec2:DescribeInstanceStatus`（Resource 必须 `*`）
  （这是你这次的关键 bug）

1. **Scheduler “Create new role for this schedule” 一直灰掉**

- 原因：当前账号/权限不允许控制台自动创建 IAM role
- 修复：手动创建 `EventBridgeScheduler-StartAutomationRole`（trust = `scheduler.amazonaws.com`） ([AWS Documentation](https://docs.aws.amazon.com/scheduler/latest/UserGuide/setting-up.html))

1. **误选了 `AmazonEventBridgeApiDestinationsServiceRolePolicy`**

- 问题：那是给 API destinations 的，不是我们要的权限集合
- 修复：删掉它，改成我们写的两条：`ssm:StartAutomationExecution` + `iam:PassRole`

1. **“Tag-based 为啥还要填 InstanceId？”**

- 解释：官方 runbook 参数就是 `InstanceId`（必填），Tag-based 是用来做权限安全阀：**传错 ID 也停不掉没打 Tag 的机器**。 ([AWS Documentation](https://docs.aws.amazon.com/systems-manager-automation-runbooks/latest/userguide/automation-aws-stopec2instance.html))

1. **时区/弹性窗口导致“不按你以为的时间跑”**

- 修复：Timezone 选 `America/New_York`；Flexible time window 选 Off

------

# 可选增强：只在工作日启停（更省钱）

- 工作日 18:00 Stop：`cron(0 18 ? * MON-FRI *)`
- 工作日 06:00 Start：`cron(0 6 ? * MON-FRI *)`

------

# 如何“关掉/暂停”自动化（省钱或停止测试）

EventBridge → Scheduler → 找到两个 schedule：

- 直接 **Disable**（推荐，保留配置）
- 或 Delete（彻底删除）

------

如果你愿意，我还可以把这份指南再压缩成一页「交付版 checklist」（每一步 1 行、适合贴到工单/文档里）。

## 总体 Flow（每天会发生什么）

### 整个动作 Flow 是这样的：

1. **Scheduler** 醒了（它身上背着 **角色 2**，所以它有权说话）。
2. **Scheduler** 对 **SSM** 喊话：“嘿！去执行关机脚本！”
3. **关键动作**：**Scheduler** 顺手把 **角色 1** 递给 **SSM**，说：“你本来没权限，**拿着这个（角色 1）** 去关机！”
   - *(这就是 `iam:PassRole` 的发生时刻)*
4. **SSM** 接过 **角色 1**，此时它拥有了关机权限。
5. **SSM** 转身把 EC2 关了。

### 假设 1：只给 Scheduler 权限，SSM 裸奔

> **你的想法**：Scheduler 自己带着“关机权限”，直接喊 SSM：“你去把机器关了！”

- **现实打击（技术上行不通）**：

  - **API 的限制**：Scheduler 只是一个**触发器**。它的工作在调用 API `ssm:StartAutomationExecution` 的那一瞬间就结束了（不到 0.1 秒）。
  - **异步执行**：SSM 收到命令后，可能过 10 秒才开始跑脚本，这时候 Scheduler 早就“下班”或者去干别的了。
  - **结果**：当 SSM 真正要去关 EC2 时，它发现自己没权限。它也不能“隔空”去借 Scheduler 的权限，因为 Scheduler 的进程早就结束了。

  **结论：干活的人（SSM）必须自己手里有权限。**

------

### 假设 2：只给 SSM 权限（绑定死），Scheduler 只管喊

> **你的想法**：把“关机权限”焊死在 SSM 身上。Scheduler 只要有权喊“开始”就行了，不用传什么 Role。

- **现实打击（安全性爆炸）**：

  - 如果 SSM 本身默认就拥有“关机权限”，那么**任何人**（比如一个刚入职的实习生），只要他有权调用 SSM（比如看日志），他可能随手写一个脚本就能利用 SSM 的默认权限把生产环境删了。
  - 这叫做**权限越权（Privilege Escalation）**。
  - AWS 的设计是：SSM 是一个**雇佣兵**，它默认是个白板，没有任何权限。**谁雇佣它，谁就得给它武器**。这样能确保“只有经过授权的人（Scheduler）才能让 SSM 拿到武器”。

  **结论：SSM 不能默认自带权限，必须每次任务动态“被授予”权限。**

------

### 假设 3：两人都有权限，但不需要 PassRole（不用传递）

> **你的想法**：Scheduler 有权喊话，SSM 有权关机。Scheduler 喊一声，SSM 自己掏出自己的 Role 去干活。为什么非要 Scheduler 做“递交”这个多此一举的动作？

- **现实打击（这是 AWS 最精妙的安全设计）**：

  - 想象一下，你的 AWS 账号里有两个 Role：

    1. `AdminRole`（能删库跑路）。
    2. `WorkerRole`（只能关机）。

  - 如果不需要 `PassRole` 检查，我在 Scheduler 的参数里偷偷写上：`"AutomationAssumeRole": "AdminRole"`。

  - SSM 就会傻乎乎地用 AdminRole 去执行我的恶意代码。

  - **PassRole 的作用是“安检”**： 当 Scheduler 想让 SSM 使用某个 Role 时，IAM 会立刻拦截检查：

    > “Wait！Scheduler 你小子只不过是个定时的，你有资格支配 `AdminRole` 吗？我看你策略里没有 `iam:PassRole` 且资源是 `AdminRole`，**你没资格碰这个高级 Role！拒绝！**”

  **结论：PassRole 是为了防止低权限的 Scheduler 偷用高权限的 Role 去干坏事。**

------

### 终极总结：为什么要这么麻烦？

因为 AWS 遵循的是 **“责任链”** 机制：

1. **Scheduler (发起人)**：必须证明自己有权**指派任务**（`ssm:StartAutomation`），并且有权**动用指定的工具**（`iam:PassRole`）。
2. **Role (工具/钥匙)**：必须明确写着**允许被 SSM 捡起来用**（Trust Policy: Allow ssm to Assume）。
3. **SSM (执行者)**：必须在运行时拿到 Role，才能证明自己是奉命行事。

**如果少了任何一环，要么流程跑不通（技术问题），要么会被黑客钻空子（安全问题）。**

### 第一部分：Role 和 Inline Policy 的关系

你可以把 **Role** 想象成一张 **“工牌”**，而 **Inline Policy** 是写在工牌背面的 **“授权清单”**。

- **Role (工牌)**：决定了 **“谁”** 能戴它（Trust Policy）。
- **Inline Policy (清单)**：决定了戴上它的人 **“能干什么”**（Permissions）。

如果一个 Role 没有 Policy，它就是一张废纸；如果有 Role 但没有 Trust Policy，谁都戴不上它。

#### 1. 角色一：实干家 (AutomationEc2StartStopRole)

- **它的身份 (Trust Policy)**：只允许 **SSM Service** (`ssm.amazonaws.com`) 佩戴。
- **它的清单 (Inline Policy)**：
  - **关键条目 1**：`ec2:StopInstances` / `ec2:StartInstances`
    - *意义*：这是核心动作。没有这一行，SSM 跑到 EC2 门口会被踢回来（Access Denied）。
  - **关键条目 2**：`ec2:DescribeInstanceStatus`
    - *意义*：这是“眼睛”。SSM 关机后，脚本里有一步是 `WaitForInstanceStopped`。如果没有这个权限，SSM 就瞎了，不知道到底关没关成功，导致脚本超时报错。
  - **关键条目 3**：`Condition: .../AutoSchedule = OfficeHours`
    - *意义*：这是“瞄准镜”。限制了这张工牌**只能**关带有这个 Tag 的机器，防止误关了隔壁的 Database。

#### 2. 角色二：发令员 (EventBridgeSchedulerRole)

- **它的身份 (Trust Policy)**：只允许 **Scheduler Service** (`scheduler.amazonaws.com`) 佩戴。
- **它的清单 (Inline Policy)**：这是最复杂的，有三行，缺一不可：
  - **条目 1**：`ssm:StartAutomationExecution`
    - *意义*：这是 **“按按钮的权限”**。Scheduler 到点醒来，想去按 SSM 的启动按钮，IAM 会查：“你有权按这个按钮吗？” 有了这一行，才能按。
  - **条目 2**：`iam:PassRole` (Resource: AutomationEc2StartStopRole)
    - *意义*：这是 **“递交钥匙的权限”**。Scheduler 按按钮的同时，要把“实干家工牌”塞进去。IAM 会查：“这工牌是你的吗？你有权把它给别人吗？” 有了这一行，才能给。
  - **条目 3**：`tag:GetResources`
    - *意义*：这是 **“找目标的权限”**。你在 Scheduler 里设置了 target 是 `Tag:AutoSchedule=OfficeHours`。Scheduler 触发的一瞬间，需要先去 AWS 的资源库里搜：“谁有这个 Tag？”。没有这个权限，它就搜不到机器，任务直接失败。

### 总结图

```json
[时间到了!]
      │
      ▼
(EventBridge Scheduler) —— 拥有 Role 2 (传令官)
      │
      ├── 1. 我有权调用 SSM 吗？ (ssm:StartAutomationExecution) ✅ Yes
      ├── 2. 我能找到目标机器吗？ (tag:GetResources) ✅ Yes
      └── 3. 我能把 Role 1 递给 SSM 吗？ (iam:PassRole) ✅ Yes
            │
            ▼
    (SSM Automation) —— 拿到 Role 1 (实干家)
            │
            ├── 1. 我有权看机器状态吗？ (ec2:Describe...) ✅ Yes
            ├── 2. 只有带 Tag 的我才动吗？ (Condition) ✅ Yes
            └── 3. 关机！ (ec2:StopInstances) ✅ Yes
                  │
                  ▼
          [EC2 Instance Stopped]
```

把它们串起来，整个链路是这样的：

1. **Role 2 (发令员)** 赋予了 Scheduler 调用 API 的能力。
2. **Target (StartAutomationExecution)** 是 Scheduler 调用的具体 API 动作。
3. **Input JSON** 是这次调用的参数，里面夹带了 **Role 1 (实干家)**。
4. **SSM** 收到 API 请求，取出 **Role 1**，戴上它，获得 **Inline Policy 1** 的权限，最终关掉了 **EC2**。

------

**时间**：

- Timezone：`America/New_York`
- Stop：`cron(0 18 * * ? *)`
- Start：`cron(0 6 * * ? *)`
- Flexible time window：Off

- **验收位置**：
  - SSM → Automation → Executions（看成功/失败原因）
  - EventBridge Scheduler → schedule 详情（看 next invocation / 最近一次触发）

如果你之后想把它改成“仅工作日启停”，或者想加第三台实例，只要动 **cron** 或 **tag** 就行，不需要再碰 JSON 或 IAM。