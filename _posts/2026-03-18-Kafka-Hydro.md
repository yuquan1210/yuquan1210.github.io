# AWS MSK 上的 Kafka Topic 是如何创建和管理的

在 REA，Kafka 是数据流传输的核心基础设施。但 Kafka Topic 的创建、权限管理、配置调优和自动化部署，远比调用一个 API 复杂得多——尤其是在企业级云环境下，涉及 AWS MSK、CloudFormation、IAM、Tiered Storage 等一系列技术。

这篇文章以 REA 内部的 **Hydro 平台**和 `incoming-lead-topic` 为具体案例，逐层拆解一个 Kafka Topic 从配置文件到正式上线的完整旅程。

---

## 一、平台背景：Hydro 是什么

**Hydro** 是 REA 内部的流数据平台，底层使用 **AWS MSK**（Managed Streaming for Apache Kafka）托管 Kafka 集群。MSK 的好处是托管了 Broker 的运维工作（EC2、网络、监控、升级），团队只需关注 Topic 和业务逻辑。

`incoming-lead-topic` 所在的集群叫 `hydro-dev`，集群配置文件位于 Hydro 仓库的：

```
hydro/hydro_env/hydro-dev/hydro-dev-cluster.yaml
```

其中 `broker_count = 3`，意味着集群由 3 个 Kafka Broker 节点组成。

### CDK → CloudFormation：基础设施的管理方式

Hydro 用 **AWS CDK（Cloud Development Kit）** 管理 AWS 资源。CDK 本质上是一个 CloudFormation 模板生成器 + 部署工具，工作流程如下：

```
CDK 代码（TypeScript）
    ↓ cdk synth
CloudFormation 模板（JSON/YAML）
    ↓ cdk deploy
CloudFormation 执行（实际创建/更新 AWS 资源）
```

从使用者角度来说，**CDK 是日常操作的工具**，CloudFormation 是幕后真正创建资源的执行层。

> 集群级别的基础设施（MSK Cluster、Broker 节点、网络配置等）直接通过 CDK 管理，不需要经过 Metacontroller（Metacontroller 是 Kubernetes 的 CRD 控制器机制，不适用于 AWS 原生资源）。Topic 的管理则走另一套流程，后文会详细介绍。

---

## 二、Topic 配置详解

Kafka Topic 通过一份 YAML manifest 来声明式地定义。下面是一个简化的 `incoming-lead-topic` 配置，以此为例逐条介绍各项配置的含义：

```yaml
hydro_env: hydro-dev
hydro_metadata:
  owner: lead-filtering
  description: "Incoming lead events"

partitions: 3
replication: 3

kafka_config:
  cleanup.policy: delete
  retention.ms: 604800000   # 7 天

permissions:
  - name: incoming-lead-readonly
    access_level: topic-read
    principals:
      - principal_type: consumer-app
        arn: arn:aws:iam::6666:role/application/lead-store-role-ap-southeast-2

retired: false
```

下面逐项拆解。

### `hydro_env`：集群环境

`hydro_env: hydro-dev` 指定了这个 Topic 属于哪个 Kafka 集群环境。服务会通过它去 AWS SSM Parameter Store 查询对应集群的参数（如集群 ARN、部署 Role ARN 等），路径格式为：

```
/kafka/hydro/hydro-dev/{cluster}/...
```

### `hydro_metadata`：标签式元数据

类似于标签（Tag），用于标注所有者、描述等信息，方便平台管理和账单归因。

---

### 分区数（`partitions`）

```yaml
partitions: 3
```

分区（Partition）是 Kafka 实现**并行处理**的核心机制。一个分区在同一时刻只能被消费者组中的一个消费者实例占用，因此：

- `partitions: 3` → 最多支持 3 个消费者实例并行处理
- **创建后不能减少**（增加分区是允许的，但会打乱按 Key 路由的一致性，需谨慎）

> 分区数也参与 `retention.bytes` 的计算：`总容量上限 ÷ 分区数 = 每分区上限`。

---

### 副本因子（`replication`）

```yaml
replication: 3
```

副本（Replica）是 Kafka 的**容错机制**——每条消息被同时存储在 3 个不同的 Broker 节点上。即使某个 Broker 宕机，数据依然完整。

关于副本和 ISR（In-Sync Replicas）的工作方式，可以参考前一篇文章 [深入浅出 Kafka：从基础概念到一致性机制](kafka-fundamentals.md) 中的介绍。

---

### 消息清理策略（`cleanup.policy`）

```yaml
kafka_config:
  cleanup.policy: delete
```

`cleanup.policy` 控制消息**超过保留时间后如何处理**，有两个选项：

| 策略 | 行为 |
|------|------|
| `delete` | 超过 `retention.ms` 或 `retention.bytes` 限制的旧消息直接删除，**适合事件流** |
| `compact` | 按消息 Key 压缩，只保留每个 Key 的**最新值**，适合"状态快照"类 Topic |

**重要：`cleanup.policy` 一旦 Topic 创建后不能修改**，Hydro 平台的 `validate_cleanup_policy_cant_change` 校验会拦截任何变更请求。因此在创建前务必确认业务需求。

---

### 消息保留容量（`retention.bytes`）

`retention.bytes` 控制**每个分区最多存储多少数据**。Hydro 平台会根据 `cleanup.policy` 和 `retention.ms` 自动补全默认值，规则如下：

| 条件 | `retention.bytes`（每分区） |
|------|-----------------------------|
| `retention.ms` < 21 天 | 约 10 GB（`10737418240`） |
| `retention.ms` ≥ 21 天（启用 Tiered Storage）| 约 300 GB（`MAX_RETENTION_BYTES_TIERED`） |

> 以 `partitions: 3`、`retention.ms` 不超过 21 天为例，集群里该 Topic 的**总数据上限 = 10 GB × 3 = 30 GB**。

---

### 分层存储（Tiered Storage）

当 `retention.ms` 超过 21 天（`MAX_RETENTION_MS_LOCAL`）时，Hydro 平台会自动启用 **Tiered Storage（分层存储）**，同时将配置补全为 `remote.storage.enable: true`。

标准 Kafka 将所有数据存储在 Broker 的本地磁盘（EBS）上，成本高、容量有限。Tiered Storage 把历史数据卸载到 **AWS S3**，大幅降低成本，并支持更长的保留时间（最长可达 2 年）。

启用 Tiered Storage 后，存储层架构如下：

```
本地热数据（Broker EBS 磁盘）
  └─ 由 local.retention.ms / local.retention.bytes 控制
  └─ 只保留最近 21 天的数据
          ↓ 超出后自动迁移
冷数据（AWS S3 远程存储）
  └─ 由 retention.ms / retention.bytes 控制总量
  └─ 支持超长保留（数月至 2 年）
```

两组配置的职责区别：

| 参数 | 控制范围 |
|------|----------|
| `local.retention.ms` / `local.retention.bytes` | 本地磁盘保留的热数据范围 |
| `retention.ms` / `retention.bytes` | 包括 S3 在内的数据总保留范围 |

---

## 三、IAM 权限模型

Kafka 的消息读写本质上是通过网络直连 MSK Broker，需要在 **AWS IAM 层面**控制谁能连接、读写哪些 Topic。

Hydro 平台通过 `permissions` 字段统一声明式地管理这套权限：

```yaml
permissions:
  - name: incoming-lead-readonly
    access_level: topic-read
    principals:
      - principal_type: consumer-app
        arn: arn:aws:iam::6666:role/application/lead-store-role-ap-southeast-2
```

### 字段说明

| 字段 | 说明 |
|------|------|
| `name` | 权限组名，平台会与集群 ID 和 Topic Hash 拼接，生成 IAM Role 名，例如 `hydro-dev-27efb0e-incoming-lead-readonly` |
| `access_level` | 可选 `topic-read`、`topic-write`、`topics-describe`，决定附加哪些 MSK 操作权限 |
| `principal_type` | `consumer-app` 或 `producer-app`，标注应用角色，配合 `access_level` 确定 IAM Policy 内容 |
| `arn` | 应用运行时使用的 IAM Role ARN，Trust Policy 会允许这个 ARN 执行 `sts:AssumeRole` |

### 生成的 IAM 资源结构

平台会自动生成对应的 IAM Role 和 Managed Policy：

```
AWS::IAM::Role
└── hydro-dev-27efb0e-incoming-lead-readonly
      │
      │  附加了
      ▼
AWS::IAM::ManagedPolicy
└── （Kafka topic 读取托管策略）
      │
      │  包含以下权限
      ▼
      ├── kafka-cluster:Connect
      ├── kafka-cluster:DescribeTopic
      └── kafka-cluster:ReadData        ← topic-read
```

如果 `access_level` 是 `topic-write`，还会额外包含：
- `kafka-cluster:WriteData`
- `kafka-cluster:AlterGroup`

### 跨账号 AssumeRole

`lead-store` 服务（AWS 账号 `6666`）和 Kafka 集群（AWS 账号 `8888`）在**不同的账号**里。

平台生成的 IAM Role（`hydro-dev-27efb0e-incoming-lead-readonly`）的 **Trust Policy** 会允许 `lead-store` 的 Role ARN 来 `sts:AssumeRole`。运行时，`lead-store` 先 Assume 这个跨账号 Role，再用临时凭证连接 MSK 读取消息。

### 顺带一提：两种 IAM Policy 类型

在 CloudFormation 模板中，IAM Policy 有两种定义方式：

| 类型 | 说明 |
|------|------|
| `AWS::IAM::Policy`（内联策略） | 与一个 Role 强绑定，Role 删除后该策略也一起删除 |
| `AWS::IAM::ManagedPolicy`（托管策略） | 独立资源，有自己的 ARN，可以被多个 Role 共享，删除 Role 不影响策略本身 |

Hydro 平台选用 `AWS::IAM::ManagedPolicy`，原因在于同一份权限策略可以被多个应用复用（例如多个消费者 Role 复用同一份 "topic-read" 策略）。

---

## 四、CloudFormation Custom Resource：为什么需要 Lambda

CloudFormation 能管理大多数 AWS 资源，但它有一个能力边界：**无法管理 Kafka 内部** 的资源，比如 Topic。

创建 MSK Kafka Topic 不是调用 `aws msk create-topic` 这样的 AWS API，而是要通过 **Kafka AdminClient 协议**直接连接到 Broker，发送 `CreateTopics` 请求——这是 Kafka 内部协议，不属于 AWS 的 API 体系。

为了解决这个问题，CloudFormation 提供了 **Custom Resource（自定义资源）** 机制：让 CloudFormation Stack 在遇到不支持的资源类型时，把事件转发给一个 **Lambda 函数**，由 Lambda 执行真正的操作，再把结果回传给 CloudFormation。

```
CloudFormation Stack
  └─ 遇到 Custom Resource（Kafka Topic）
        ↓ 发送 Create/Update/Delete 事件
  Lambda（Provider）
        ↓ 连接 MSK Broker
  Kafka AdminClient（CreateTopics / DeleteTopics）
        ↓ 操作完成后回调
  CloudFormation（资源状态 → CREATE_COMPLETE）
```

### Lambda 需要的权限

这个 Lambda 需要一个 **执行 Role** 以及对应的 `AWS::IAM::ManagedPolicy`，包括连接 MSK 集群、调用 Kafka API 等权限。

### OnEvent Lambda

CDK 框架会自动生成一个 **OnEvent Lambda**，它不包含业务逻辑，而是负责处理 CloudFormation 与 Lambda 之间的通信协议：超时处理、异步响应格式、错误上报等。实际的 Kafka API 调用逻辑写在业务 Lambda 中。

---

## 五、自助部署全流程：CoPilot + Metacontroller

前面介绍了配置和原理，现在看整个部署链路是如何工作的。

**CoPilot** 是 REA 内部的基础设施自助服务平台。所有团队通过提交 YAML manifest 来申请 AWS 资源，底层统一走这套流水线：

```
Kubernetes YAML（HydroTopic Manifest）
    ↓ kubectl apply（via aws-infra automation）
Kubernetes API
    ↓ 监听资源变动
Metacontroller
    ↓ 调用 /sync 接口
hydro-topic-service（业务验证 + 资源规划）
    ↓ 返回 desired children
rea-copilot-cdk（生成 CloudFormation 模板）
    ↓ 提交
CloudFormation
    ↓ 创建/更新
AWS 资源（MSK Topic、IAM Role/Policy、CloudWatch Alarm 等）
```

### 各组件职责

| 组件 | 职责 |
|------|------|
| **HydroTopic Manifest** | 用户提交的声明式配置（kind, metadata, spec） |
| **Metacontroller** | 监听 Kubernetes 自定义资源的变化，驱动控制循环，协调父子资源的创建/更新/删除 |
| **hydro-topic-service** | 理解业务意图，验证配置合法性，补全默认参数，计算并声明需要哪些子资源 |
| **rea-copilot-cdk** | 把技术方案翻译成 CloudFormation 模板，真正部署 AWS 资源 |

### 事件驱动的控制循环

这是一个典型的 **Operator 模式**（声明式 + 协调循环）：

1. 工程师提交 YAML manifest 到 `aws-infra`，CI/CD 自动执行 `kubectl apply` 发送到 Kubernetes
2. Metacontroller 检测到资源变化，调用 `hydro-topic-service` 的 `/sync` 接口
3. `hydro-topic-service` 验证配置、检查现有资源状态，返回"期望的子资源列表（desired children）"
4. Metacontroller 根据返回结果，调用 `rea-copilot-cdk` 执行实际部署
5. 部署完成后，资源状态更新为 `Ready`，再次触发 Metacontroller
6. `hydro-topic-service.sync()` 返回 `Ready=True`，本轮调协完成

> 任何配置变更（更新 manifest）或资源状态变动，都会重新触发这个循环，确保实际状态始终与声明状态一致。

### 一次 Topic 创建会生成哪些资源

`copilot-hydro-topic-service` 在一次部署中会创建和管理以下资源：

```
copilot-hydro-topic-service 直接管理
  ├─ MSK Kafka Topic          ← CloudFormation（via Custom Resource + Lambda）
  ├─ IAM Role + Policy        ← CloudFormation（permissions 配置对应的资源）
  └─ CloudWatch Alarm         ← CloudFormation（Topic 的可观测性告警）

可选：S3 Sink Connector
  └─ 由 copilot-hydro-connect-service 管理（依赖 HydroConnect，实现 Topic → S3 备份）
```

### 部署操作步骤

1. 在 `copilot/<env>/ap-southeast-2/hydro-topics/` 目录下添加或修改 YAML manifest
2. 合并到 `main` 分支 —— CoPilot 会自动触发部署
3. 如果 IAM Role 的 Trust Relationship 需要更新，前往 Slack 的 `#hydro-streaming-platform` 频道请求平台团队处理

---

## 六、Topic 的退役与删除

当一个 Topic 不再需要时，正确的删除方式是使用 `retired` 标志：

```yaml
retired: true
```

将 `retired` 设为 `true` 并重新提交 manifest，会触发以下删除流程：

1. 清空 Topic 中的数据
2. 删除所有关联的子资源（IAM Role、Policy、CloudWatch Alarm 等）
3. 最终删除 Topic 本身

> **注意**：直接从仓库中删除 manifest 文件只会让平台停止管理该资源，**不会触发真正的删除操作**。应先设置 `retired: true`、等资源清理完成后，再移除文件。

---

## 总结

一个 Kafka Topic 从配置文件到正式上线，涉及相当多的技术层面：

| 层面 | 技术/工具 |
|------|-----------|
| Topic 声明 | YAML Manifest（partitions、replication、cleanup.policy、retention、permissions） |
| 权限管理 | AWS IAM（ManagedPolicy、Trust Policy、跨账号 AssumeRole） |
| 创建机制 | CloudFormation Custom Resource + Lambda（Kafka AdminClient） |
| 基础设施管理 | AWS CDK → CloudFormation |
| 自助部署流程 | CoPilot + Metacontroller + hydro-topic-service + rea-copilot-cdk |
| 长期存储 | Tiered Storage（本地 EBS 热数据 + S3 冷数据） |

这套体系的核心思路是**声明式管理 + 平台统一治理**：开发者只需关心"我要什么样的 Topic"，底层的资源编排、权限配置、告警搭建全部由平台自动完成。这与前一篇介绍的 Spring Boot 消费者配置形成互补——消费者侧关注"怎么消费"，平台侧关注"资源是否就绪"。
