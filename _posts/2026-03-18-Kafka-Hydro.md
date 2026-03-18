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

HydroTopic Manifest 是一个标准的 Kubernetes CRD（自定义资源定义）。团队通过声明式的 YAML 定义一个 Kafka Topic 的全部属性，`hydro-topic-service` 读取后验证、补全，并驱动下游资源创建。以下是 `incoming-lead-topic-dev` 的真实 manifest：

```yaml
apiVersion: service.copilot/v1
kind: HydroTopic
metadata:
  name: incoming-lead-topic-dev
  namespace: aws-account-6666   # namespace 对应该 Topic 所属的 AWS 账号 ID
spec:
  hydro_env: hydro-dev
  hydro_metadata:
    owner: finx-lead-lords
    personal_information: personal-information
  topic_name:
    domain: lead-engine
    product: incoming-lead
    topic_type: pii
    environment: dev
  partitions: 3
  kafka_config:
    cleanup.policy: delete
    retention.ms: 259200000   # 3 天
  permissions:
    incoming-lead-readonly:
      access_level:
        - topic-read
      principals:
        lead-store:
          principal_type: consumer-app
          arn: arn:aws:iam::6666:role/application/lead-store-role-ap-southeast-2
    incoming-lead-readwrite:
      access_level:
        - topic-read
        - topic-write
      principals:
        enquiry-api:
          principal_type: producer-app
          arn: arn:aws:iam::6666:role/application/enquiry-api-role-ap-southeast-2
        incoming-lead-replay:
          principal_type: producer-app
          arn: arn:aws:iam::6666:role/application/incoming-lead-replay-role-ap-southeast-2
```

下面逐项拆解。

### `hydro_env`：集群环境

`hydro_env: hydro-dev` 指定了这个 Topic 属于哪个 Kafka 集群环境。服务会通过它去 AWS SSM Parameter Store 查询对应集群的参数（如集群 ARN、部署 Role ARN 等），路径格式为：

```
/kafka/hydro/hydro-dev/{cluster}/...
```

### `hydro_metadata`：元数据与数据治理标签

`hydro_metadata` 不只是描述性字段，它的内容会被服务解析，自动生成一套 `data-asset:*` 数据治理标签，附加到 CloudFormation Stack 和所有下游资源上：

| 字段 | 作用 |
|------|------|
| `owner` | 映射为 `data-asset:technical_owner`，标注资源归属团队 |
| `personal_information: personal-information` | Topic 含个人信息（PII），触发 `data-asset:contains_pi: yes`、`data-asset:pi_category: basic_personal_information` |
| `personal_information: none-personal-information` | Topic 不含个人信息，`data-asset:contains_pi: no` |

此外，`hydro_env` 中含 `dev` 字样时，服务会自动补全 `data-asset:development_lifecycle: development`，区分数据资产的环境归属。

---

### `topic_name`：Topic 全名的推导规则

Topic 名称通过 `topic_name` 下的多个字段**拼接推导**，而非直接写死一个字符串：

| 字段 | 值 | 说明 |
|------|----|------|
| `domain` | `lead-engine` | 数据域 |
| `product` | `incoming-lead` | 产品/数据集名称 |
| `topic_type` | `pii` | 数据类型标记，各域有各自允许的合法值 |
| `environment` | `dev` | 环境 |

拼接规则为 `{domain}.{product}.{topic_type}.{environment}`（`null` 字段直接跳过），最终 Topic 全名为：

```
lead-engine.incoming-lead.pii.dev
```

这个命名规范保证了 Topic 名称的**一致性和可读性**，运维人员一眼就能识别数据域、内容和环境归属。`topic_type` 的合法值由各域的 `policy_config.yaml` 定义（例如 `pii` 是 `lead-engine` 域特别允许的值）。

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

Hydro 平台将副本因子**硬编码为 3**，不对用户开放配置——这个参数不会出现在 manifest 中，由 `hydro-topic-service` 在构建 CloudFormation 参数时自动注入。

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

### 平台自动补全的其他配置

除了 `retention.bytes` 和 Tiered Storage 相关参数，`hydro-topic-service` 在将配置下发给 Kafka 前，还会自动补全以下参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max.message.bytes` | `5242880`（5 MB） | 单条消息的大小上限 |
| `message.timestamp.before.max.ms` | `315400000000`（约 10 年） | 消息时间戳允许早于当前时间的最大范围 |
| `message.timestamp.after.max.ms` | `86400000`（24 小时） | 消息时间戳允许晚于当前时间的最大范围，防止客户端时钟偏差 |

未启用 Tiered Storage 时，`local.retention.ms` 和 `local.retention.bytes` 会被设为 `-2`，这是 MSK 的**占位值**，表示该参数不适用，不会生效。

---

## 三、IAM 权限模型

Kafka 的消息读写本质上是通过网络直连 MSK Broker，需要在 **AWS IAM 层面**控制谁能连接、读写哪些 Topic。

Hydro 平台通过 `permissions` 字段统一声明式地管理这套权限。`incoming-lead-topic` 配置了两个权限组：

```yaml
permissions:
  incoming-lead-readonly:         # 权限组名（只读）
    access_level:
      - topic-read
    principals:
      lead-store:
        principal_type: consumer-app
        arn: arn:aws:iam::6666:role/application/lead-store-role-ap-southeast-2
  incoming-lead-readwrite:        # 权限组名（读写）
    access_level:
      - topic-read
      - topic-write
    principals:
      enquiry-api:
        principal_type: producer-app
        arn: arn:aws:iam::6666:role/application/enquiry-api-role-ap-southeast-2
      incoming-lead-replay:
        principal_type: producer-app
        arn: arn:aws:iam::6666:role/application/incoming-lead-replay-role-ap-southeast-2
```

`incoming-lead-readonly` 供消费者 `lead-store` 使用（只读）；`incoming-lead-readwrite` 供两个生产者 `enquiry-api` 和 `incoming-lead-replay` 使用（读写）。生产者同时需要 `topic-read` 是因为 Kafka 生产者在事务、幂等写入等操作中也需要读取 Topic 的元数据。

### 字段说明

| 字段 | 说明 |
|------|------|
| 权限组名（如 `incoming-lead-readonly`） | 平台会与集群 ID 和 Topic 名哈希拼接，生成最终 IAM Role 名 |
| `access_level` | 可选 `topic-read`、`topic-write`、`topics-describe`，决定附加哪些 MSK 操作权限 |
| `principal_type` | `consumer-app` 或 `producer-app`，标注应用角色，配合 `access_level` 确定 IAM Policy 内容 |
| `arn` | 应用运行时使用的 IAM Role ARN，Trust Policy 会允许这个 ARN 执行 `sts:AssumeRole` |

### IAM Role 命名规则

生成的 IAM Role 名遵循固定格式：

```
{cluster_id}-{sha256(topic_name)[:7]}-{permission_group_name}
```

以 `incoming-lead-readonly` 为例：
- `cluster_id` = `hydro-dev`
- `sha256("lead-engine.incoming-lead.pii.dev")[:7]` = `27efb0e`
- 最终 IAM Role 名 = `hydro-dev-27efb0e-incoming-lead-readonly`

CloudFormation Stack 的命名规则与此类似：`{cluster_id}-topic-{sha256(topic_name)[:7]}`，即 `hydro-dev-topic-27efb0e`。

使用 Topic 全名的哈希前缀，是为了在 IAM 命名长度限制内保证唯一性，同时让 IAM Role、Stack 等资源在 AWS Console 里保持关联可查。

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

`lead-store` 服务运行在 AWS 账号 `6666`（业务账号），而 Kafka 集群和相关 IAM 资源由 Hydro 平台账号（`8888`）管理，两者在**不同的账号**里。

CloudFormation 部署时，生成的 IAM Role（如 `hydro-dev-27efb0e-incoming-lead-readonly`）被创建在 Hydro 平台账号内，其 **Trust Policy** 允许 `arn:aws:iam::6666:role/application/lead-store-role-ap-southeast-2` 执行 `sts:AssumeRole`。运行时，`lead-store` 先 Assume 这个跨账号 Role 获得临时凭证，再用该凭证连接 MSK 读取消息。

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

以 `incoming-lead-topic-dev` 为例，`hydro-topic-service` 驱动 CDK 合成的 CloudFormation Stack（名为 `hydro-dev-topic-27efb0e`）包含以下资源：

```
CloudFormation Stack: hydro-dev-topic-27efb0e
  ├─ Custom::Kafka_Topic         × 1   ← 实际 Kafka Topic（via Lambda + AdminClient）
  ├─ AWS::IAM::Role              × 3   ← Lambda 执行角色 × 1 + 权限访问角色 × 2
  ├─ AWS::IAM::Policy            × 2   ← Lambda 执行策略（内联，随角色删除）
  ├─ AWS::IAM::ManagedPolicy     × 1   ← Topic 读写权限策略（托管，可被多 Role 共享）
  └─ AWS::Lambda::Function       × 2   ← Provider Lambda（业务逻辑）+ OnEvent Lambda（CF 协议层）

可选：HydroConnect（S3 备份）
  └─ 由 copilot-hydro-connect-service 管理
  └─ 仅在 hydro-prod 环境启用（由平台常量 S3_BACKUP_ALLOWED_ENVS 控制）
  └─ hydro-dev 环境不会生成此子资源
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
| Topic 声明 | Kubernetes CRD（HydroTopic Manifest）：partitions、cleanup.policy、retention、permissions |
| Topic 命名 | 多字段拼接推导：`{domain}.{product}.{topic_type}.{environment}` |
| 数据治理标签 | `hydro_metadata` → `data-asset:*` 标签（PII 声明、环境、所有者等自动生成） |
| 权限管理 | AWS IAM（ManagedPolicy、Trust Policy、跨账号 AssumeRole、Topic 名哈希命名） |
| 创建机制 | CloudFormation Custom Resource + Lambda（Kafka AdminClient） |
| 基础设施管理 | AWS CDK → CloudFormation |
| 自助部署流程 | CoPilot + Metacontroller + hydro-topic-service + rea-copilot-cdk |
| 长期存储 | Tiered Storage（本地 EBS 热数据 + S3 冷数据，仅 hydro-prod 支持 S3 备份） |

这套体系的核心思路是**声明式管理 + 平台统一治理**：开发者只需关心"我要什么样的 Topic"，底层的资源编排、权限配置、告警搭建全部由平台自动完成。这与前一篇介绍的 Spring Boot 消费者配置形成互补——消费者侧关注"怎么消费"，平台侧关注"资源是否就绪"。
