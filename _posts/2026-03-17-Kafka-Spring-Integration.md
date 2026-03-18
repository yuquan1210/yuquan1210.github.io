# Kafka 在 Spring Boot 中的实践：从配置到消费

上一篇文章介绍了 Kafka 的核心概念——Topic、Partition、Consumer Group、Offset 等。这篇文章我们换个角度，结合一个真实的 Spring Boot 服务，看看 Kafka 在实际项目中是如何配置和使用的。

---

## 一、Kafka 集群连接配置

在部署配置文件中，通常需要提供以下几个 Kafka 相关的参数：

### `kafka_bootstrap_address`：入口地址列表

Bootstrap address 是 Kafka 集群的入口地址列表，格式类似：

```
broker1.kafka.example.com:9092,broker2.kafka.example.com:9092,broker3.kafka.example.com:9092
```

它的工作方式如下：

1. 应用启动时，从这个列表中**任选一个地址**发起连接
2. 连接成功后，拿到整个集群的元数据（哪个 Broker 负责哪个 Topic 的哪个 Partition）
3. 之后客户端根据元数据，**直接连接到目标 Broker** 进行读写

所以这个列表并不需要包含集群中的所有 Broker，写 2-3 个地址即可——只要其中一个能连通，就不会影响功能。集群的 Broker 数量通常在 CloudFormation 等基础设施配置中定义，Bootstrap address 可以通过以下命令获取：

```bash
aws kafka get-bootstrap-brokers --cluster-arn <MSK_CLUSTER_ARN>
```

### `incoming_lead_topic_name`：Topic 名称

指定消费哪个 Topic，直接用 Topic 的名字字符串即可。

### `incoming_lead_topic_role_arn`：跨账号 IAM 授权

这个配置涉及**跨 AWS 账号的权限问题**。

假设 Kafka 集群属于账号 A（`8888`），消费者服务属于账号 B（`6666`）。账号 B 的服务没有直接读取账号 A 的 Kafka Topic 的权限，解决方案是使用 **IAM Role Assume（角色借用）**：

- 账号 A 创建一个 IAM Role，声明允许账号 B 的服务"借用"（assume）这个 Role
- 账号 B 的服务运行时，借用账号 A 的这个 Role，获得对应的读权限

`incoming_lead_topic_role_arn` 就是这个属于账号 A 的 IAM Role 的 ARN。

### 深入：服务自身 Role 的两层授权

以 `lead-store` 的 IAM Role 为例，解释 assumeRole 如何工作：

```yaml
# iam-role-stack.yml（节选）

# --- 第一层：信任策略（Trust Policy）---
# 回答：谁可以 assume 这个 Role？
AssumeRolePolicyDocument:
  Statement:
    - Effect: Allow
      Principal:
        Service: ecs-tasks.amazonaws.com   # ECS Tasks 服务可以 assume 这个 Role
      Action: sts:AssumeRole

# --- 第二层：权限策略（Permission Policy）---
# 回答：这个 Role 自身能做什么？
- Effect: Allow
  Action:
    - sts:AssumeRole                       # 这个 Role 可以去 assume 其他 Role
  Resource: '*'
```

这两处 `sts:AssumeRole` 虽然动作名称相同，但含义完全不同：

| | 信任策略（Trust Policy）| 权限策略（Permission Policy）|
|---|---|---|
| **位置** | `AssumeRolePolicyDocument` | Role 附加的 IAM Policy |
| **方向** | 谁能 assume **这个** Role | 这个 Role 能 assume **哪些** Role |
| **主体** | ECS Tasks 服务 | 这个 Role 本身 |
| **作用** | 使 ECS Task 可以使用此 Role | 使此 Role 可以扮演其他 Role |

**信任策略**是前提条件：它授权 AWS ECS 服务在启动 Task 时，以这个 `lead-store` Role 的身份运行，从而让 Task 内的应用程序获得该 Role 所拥有的所有权限（读 KMS、写 SQS、推 CloudWatch 指标等）。

**权限策略中的 `sts:AssumeRole`** 是跨账号访问的钥匙：正因为这条授权，`lead-store` 服务才能在运行时调用 `sts:AssumeRole` API，临时借用账号 A 的 Kafka Topic Role，获取该 Topic 的读取凭证。

两者协同，构成了完整的跨账号访问链路：

```
AWS ECS 服务（启动 Task）
  └─ [信任策略] → assume lead-store Role
       └─ lead-store 应用（运行中）
            └─ [权限策略 sts:AssumeRole] → assume 账号 A 的 Kafka Topic Role
                 └─ 获取临时凭证 → 读取账号 A 的 Kafka Topic
```

在hydro系统中，hydro-dev账号配置 trusy policy 允许 lead-store （临时）使用 hydro-dev 拥有的role，lead-store 自身也需要设置 Permission Policy 允许自己去临时使用他人的role

---

## 二、`KafkaConsumerConfig`：配置消费者工厂

Spring Kafka 通过一个配置类来设置 Kafka 消费者的各种行为。一个典型的配置类长这样：

```java
@EnableKafka
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, CloudEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, CloudEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(buildConsumerFactory());
        factory.setCommonErrorHandler(buildErrorHandler());
        return factory;
    }

    private ConsumerFactory<String, CloudEvent> buildConsumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
        // ... 其他配置，包括 IAM 认证配置
        return new DefaultKafkaConsumerFactory<>(props);
    }

    private CommonErrorHandler buildErrorHandler() {
        // 见下一节
    }
}
```

### `@EnableKafka`

这个注解加在 `@Configuration` 类上，告诉 Spring：**去扫描所有标注了 `@KafkaListener` 的方法**，并为它们创建对应的监听容器。没有这个注解，`@KafkaListener` 不会生效。

### `ConcurrentKafkaListenerContainerFactory`

这是 Spring Kafka 提供的监听容器工厂。它创建的容器内部持有**消费者线程**，会持续不断地从 Kafka 拉取消息，然后交给 `@KafkaListener` 方法处理。

名字里的 **Concurrent** 表示这个容器支持在内部启动多个并发线程来消费消息。默认配置下只有 1 个线程。一个重要约束是：**同一时刻，一个分区只能被同一个消费者线程拉取**，不允许多个线程同时消费同一个分区（这与 Consumer Group 的设计保持一致）。

通过 `setConcurrency` 可以调整单个实例内的并发消费线程数，每个线程对应一个独立的 Kafka Consumer：

```java
factory.setConcurrency(3);  // 单个实例内启动 3 个消费线程，各自负责不同分区
```

这相当于在一台机器上模拟了多个消费者实例的效果。同样地，线程数也不应超过 Topic 的分区数，多余的线程会空闲等待。

### `buildConsumerFactory`

`ConsumerFactory` 的配置以 `Map<String, Object>` 的形式传入，以键值对方式列出各项配置：

```java
Map<String, Object> props = new HashMap<>();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);  // 集群地址
props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);                     // 消费组 ID
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, CloudEventDeserializer.class);
// auto.offset.reset 未显式设置，沿用 Kafka 默认值 latest
// IAM 认证相关配置
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "AWS_MSK_IAM");
// ...
```

### `auto.offset.reset`：新消费组首次启动从哪里读

当一个消费组**第一次启动**（或者其 Offset 记录被删除）时，Kafka 上没有这个组的历史消费记录，不知道该从哪里开始读。`auto.offset.reset` 决定了这种情况下的行为：

| 值 | 行为 |
|---|---|
| `earliest` | 从该分区**最早**的消息开始读，确保不漏任何历史消息 |
| `latest` | 从该分区**最新**的位置开始读，只消费启动后新进来的消息 |

**项目中的实际值：`latest`（Kafka 客户端原生默认值）**

与 `enable.auto.commit` 不同，Spring Kafka **不会覆盖**这个配置。项目中没有显式设置它，因此直接沿用 Kafka 客户端的原生默认值 `latest`：当某个 consumer group 第一次消费某个 topic（没有已提交的 offset）时，从最新的消息开始消费，不会读取历史消息。

如果需要新消费组从头消费，需要在 `KafkaConsumerConfig` 中**显式配置**：

```java
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
```

> 注意：这个配置只在消费组**没有已提交 Offset** 时生效。一旦该消费组提交过 Offset，重启后会从上次的 Offset 继续，`auto.offset.reset` 不再起作用。

### Bean 的生命周期

这些 `@Bean` 方法在 **Spring 容器初始化阶段**执行，创建好 `ConsumerFactory` 和 `ContainerFactory`。Kafka 的实际连接和监听，则是在**应用启动后**才开始建立的。

---

## 三、错误处理：重试与死信队列

消费消息时，业务处理难免会出现异常，比如下游服务不可用、消息格式非预期等。这时候需要一个合理的错误处理策略。

### 重试配置：`FixedBackOff`

```java
private CommonErrorHandler buildErrorHandler() {
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(
        this::recover,
        new FixedBackOff(1000L, 1L)  // 间隔 1 秒，最多重试 1 次
    );
    return errorHandler;
}
```

`FixedBackOff(interval, maxAttempts)` 定义了重试策略：

- 第一个参数 `1000L`：重试间隔，单位毫秒（1 秒）
- 第二个参数 `1L`：最大重试次数（重试 1 次，加上第一次共尝试 2 次）

### 重试耗尽后：`recover` 回调

重试次数耗尽后仍然失败，Spring Kafka 会调用 `recover` 回调方法。这个方法决定如何处理"彻底失败"的消息：

```
消息消费失败
│
├─ FixedBackOff：等 1 秒，重试 1 次
│
└─ 还是失败 → 执行 recover 回调：
        │
        ├─ 是反序列化异常？→ 把原始字节转成字符串
        │
        └─ 是业务处理异常？→ 把 CloudEvent 序列化成 JSON
        │
        └─ 发送到 SQS 死信队列（DLQ）
                │
                └─ SQS 也失败了？→ 上报 CloudWatch 指标
```

**死信队列（Dead Letter Queue，DLQ）** 是处理"毒消息"的标准做法：把无法被正常处理的消息转移到一个专门的队列里，防止它阻塞正常的消费流程。后续可以人工检查、修复后重新处理。

### Offset 何时提交

Kafka 客户端原生的 `enable.auto.commit` 默认值是 `true`，但 Spring Kafka 会**主动将其覆盖为 `false`**，由框架自己控制 Offset 的提交时机，而不是让 Kafka 客户端定期自动提交。

原因在于：Spring Kafka 用自己的 `AckMode`（默认 `BATCH`）来管理 offset 提交，如果 Kafka 客户端的自动提交也在运行，两者会冲突，所以 Spring Kafka 在内部主动接管了这个行为。即使项目代码里没有显式设置 `enable.auto.commit=false`，Spring Kafka 也会自动处理。

默认行为（`AckMode.BATCH`）：

```
消费者拉取一批消息
│
├─ 逐条调用 @KafkaListener 方法处理
│    ├─ 处理成功 → 继续下一条
│    └─ 抛出异常 → 交给 ErrorHandler（重试、DLQ）
│
└─ 这批消息全部处理完（含 ErrorHandler 兜底处理完）→ 提交 Offset
```

这意味着：
- 消息处理成功后，Offset 才会被提交——这是 at-least-once（至少一次）语义的保证
- 如果消费者在提交前崩溃，重启后会从上一次的 Offset 重新消费，消息可能被重复处理
- 因此，`@KafkaListener` 里的业务逻辑应当尽量设计成**幂等的**（重复执行不会产生副作用）

### `containerFactory` 与 `@KafkaListener`

构建好的 `ContainerFactory` 需要在 `@KafkaListener` 注解上被引用：

```java
@KafkaListener(
    topics = "${incoming_lead_topic_name}",
    groupId = "${kafka_group_id}",
    containerFactory = "kafkaListenerContainerFactory"
)
public void consume(ConsumerRecord<String, CloudEvent> record) {
    // 业务处理逻辑
}
```

`containerFactory` 属性告诉 Spring：用哪个 `ContainerFactory` Bean 来为这个监听方法创建容器。如果不指定，Spring 会查找名为 `kafkaListenerContainerFactory` 的默认 Bean；如果找不到，启动时会直接报错。

---

## 四、`@KafkaListener` 与 `ConsumerRecord`

消费方法接收的参数类型是 `ConsumerRecord`：

```java
public void consume(ConsumerRecord<String, CloudEvent> record) {
    String key = record.key();
    CloudEvent event = record.value();
    int partition = record.partition();
    long offset = record.offset();
    String topic = record.topic();
}
```

`ConsumerRecord` 不只包含消息体，还包含**消息的完整元数据**：

| 字段 | 说明 |
|---|---|
| `topic()` | 消息所在的 Topic |
| `partition()` | 消息所在的分区编号 |
| `offset()` | 消息在分区中的 Offset（唯一位置） |
| `key()` | 消息的 Key |
| `value()` | 消息体（业务数据） |
| `timestamp()` | 消息的时间戳 |

---

## 五、`groupId`：消费组与多实例部署

### 消费组的作用

`groupId` 指定这个消费者属于哪个**消费者组**。同一个消费组内：

- 每个分区在同一时刻只能被**一个消费者**处理
- 整个组共同维护对各分区消费进度（Offset）的追踪
- 同一条消息在同一个组内**只会被消费一次**

### 跨机器的消费组

`groupId` 并不局限于单台机器。在生产环境中，一个服务通常会部署多个实例（比如多个 ECS Task）：

```
Topic: incoming-leads（4 个分区）

┌─ ECS Task 1（Consumer A）──── 消费分区 0、1
│
└─ ECS Task 2（Consumer B）──── 消费分区 2、3

两个实例共用同一个 groupId = "lead-store-consumer-group"
→ 并行消费，互不重叠
```

这就是 Kafka 实现**水平扩展消费能力**的方式：增加服务实例数，Kafka 自动将分区分配给新加入的消费者，消费能力线性提升。

> 注意：消费者实例数不应超过分区数。多余的消费者实例会处于空闲状态，因为没有可分配的分区了。

### 留意 `max.poll.interval.ms`

Kafka 要求消费者必须在 `max.poll.interval.ms` 时间内（默认 5 分钟）完成当前批次的处理并发起下一次 poll。如果超时，Kafka 会认为这个消费者"失联"，触发 **Rebalance**，将它负责的分区转移给组内其他消费者。

在实际项目中，如果消费逻辑涉及较慢的下游调用（例如同步调用外部 API），需要结合 `max.poll.records`（每次 poll 拉取的最大消息数，默认 500）一起调整，避免单次处理耗时过长：

```java
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);           // 每次只拉取 10 条，减少单批处理时间
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);  // 最长允许 5 分钟（默认值）
```

---

## 总结

这篇文章通过一个真实的 Spring Boot Kafka 消费者服务，串联了以下知识点：

| 概念 | 作用 |
|---|---|
| `kafka_bootstrap_address` | Kafka 集群入口，用于获取集群元数据 |
| IAM Role Assume | 跨账号访问 Kafka Topic 的权限方案 |
| `@EnableKafka` | 激活 Spring 对 `@KafkaListener` 的扫描 |
| `ConcurrentKafkaListenerContainerFactory` | 创建并管理消费者线程的工厂 |
| `FixedBackOff` + DLQ | 重试失败后将消息转移到死信队列的容错策略 |
| Offset 提交（AckMode） | 消息处理完成后由 Spring Kafka 自动提交 Offset，保证 at-least-once |
| `auto.offset.reset` | 新消费组首次启动时的读取起点（项目未显式设置，沿用 Kafka 原生默认值 `latest`） |
| `setConcurrency` | 单实例内的并发消费线程数 |
| `ConsumerRecord` | 封装消息体和元数据的对象 |
| `groupId` + 多实例 | 通过多个服务实例并行消费同一 Topic |
| `max.poll.records` | 控制每次拉取的消息数，避免单批处理超时触发 Rebalance |

Kafka 的设计哲学是"简单的服务端，聪明的客户端"——Spring Kafka 在客户端侧封装了大量复杂的逻辑（重试、错误处理、分区分配等），让业务代码只需关注消息本身的处理。
