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
// 用 ErrorHandlingDeserializer 包裹实际的反序列化器
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,   ErrorHandlingDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS,   StringDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, CloudEventDeserializer.class);
// auto.offset.reset 未显式设置，沿用 Kafka 默认值 latest
// IAM 认证相关配置
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "AWS_MSK_IAM");
// ...
```

### `ConsumerConfig` 常量：类型安全的配置键

上面代码中所有 `ConsumerConfig.XXX_CONFIG` 形式的常量，都是 Apache Kafka 官方 Java 客户端库（`kafka-clients`）中 `org.apache.kafka.clients.consumer.ConsumerConfig` 类的**静态字符串字段**。

```java
// ConsumerConfig 源码（节选）
public class ConsumerConfig {
    public static final String BOOTSTRAP_SERVERS_CONFIG = "bootstrap.servers";
    public static final String GROUP_ID_CONFIG          = "group.id";
    public static final String AUTO_OFFSET_RESET_CONFIG = "auto.offset.reset";
    public static final String MAX_POLL_RECORDS_CONFIG  = "max.poll.records";
    // ...
}
```

使用常量而非直接写字符串的好处：**IDE 能补全、编译期能检查**，避免 `"bootstrap.server"`（漏了 `s`）这类拼写错误在运行时才暴露。

以下是几个常见配置项及其含义：

| 常量 | 实际键名 | 默认值 | 说明 |
|---|---|---|---|
| `BOOTSTRAP_SERVERS_CONFIG` | `bootstrap.servers` | — | Kafka 集群入口地址列表 |
| `GROUP_ID_CONFIG` | `group.id` | — | 消费组 ID（必填） |
| `KEY_DESERIALIZER_CLASS_CONFIG` | `key.deserializer` | — | 消息 Key 的反序列化器 |
| `VALUE_DESERIALIZER_CLASS_CONFIG` | `value.deserializer` | — | 消息 Value 的反序列化器 |
| `AUTO_OFFSET_RESET_CONFIG` | `auto.offset.reset` | `latest` | 消费组无历史 Offset 时的读取起点 |
| `ENABLE_AUTO_COMMIT_CONFIG` | `enable.auto.commit` | `true` | 是否自动提交 Offset（Spring Kafka 会覆盖为 `false`） |
| `MAX_POLL_RECORDS_CONFIG` | `max.poll.records` | `500` | 单次 poll 最多拉取的消息条数 |
| `MAX_POLL_INTERVAL_MS_CONFIG` | `max.poll.interval.ms` | `300000`（5 分钟） | 两次 poll 之间允许的最长间隔，超时则触发 Rebalance |

**`MAX_POLL_RECORDS_CONFIG` 与 `MAX_POLL_INTERVAL_MS_CONFIG` 需要配合调整。** 假设每条消息处理需要 1 秒，`max.poll.records=500` 时单批最多需要 500 秒，远超 `max.poll.interval.ms` 的 5 分钟上限，就会触发 Rebalance。实际项目中如果下游有较慢的 I/O 调用，通常要大幅降低 `max.poll.records`：

```java
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10);          // 每批只拉 10 条
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);  // 5 分钟（默认值，此处显式声明意图）
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

### `<String, CloudEvent>`：Java 泛型的作用

`ConsumerRecord<K, V>` 是一个泛型类，`K` 和 `V` 分别代表消息 key 和 value 的类型。当写成 `ConsumerRecord<String, CloudEvent>` 时，就相当于告诉 Java：

- `record.key()` 返回的是 `String`
- `record.value()` 返回的是 `CloudEvent`

而不是 `Object`，也不需要手动强制转换。Java **编译期**就能检查类型，避免运行时出现 `ClassCastException`。

Kafka 本身是通用消息系统，并非专为 CloudEvent 设计，所以 `ConsumerRecord` 必须对 key/value 的具体类型"一无所知"，只能用泛型而不能写死——这是泛型存在的根本原因。

**泛型只影响 `key()` 和 `value()` 的类型；`topic()`、`partition()`、`offset()` 等元数据字段和泛型无关，始终存在。**

### 泛型类型由反序列化器决定

`ConsumerRecord<String, CloudEvent>` 能成立，前提是 `buildConsumerFactory` 中配置了匹配的反序列化器。实际项目中，`StringDeserializer` 和 `CloudEventDeserializer` 都被 **`ErrorHandlingDeserializer`** 包裹：

```java
// 第一层：告诉 Kafka 客户端使用 ErrorHandlingDeserializer 作为入口
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,   ErrorHandlingDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
// 第二层：告诉 ErrorHandlingDeserializer 真正干活的是谁
props.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS,   StringDeserializer.class);
props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, CloudEventDeserializer.class);
```

`ErrorHandlingDeserializer` 是 Spring Kafka 提供的包装类，它在内部委托给真正的反序列化器，但会**捕获反序列化异常**，将失败的消息交给 `ErrorHandler`（即上一节配置的 `DefaultErrorHandler`）处理，而不是让异常直接向上抛出导致消费者线程崩溃。

Consumer 在后台线程中持续轮询 Kafka，反序列化发生在这个轮询循环内部。如果反序列化直接抛异常而没有包装层拦截，同一条损坏消息（"毒消息"，poison pill）会被无限重试，消费者永远卡在这里。`ErrorHandlingDeserializer` 捕获这个异常后，将其转换成包含原始字节的 `DeserializationException`，交给 `DefaultErrorHandler`，最终路由到 DLQ——与业务异常走同一套容错链路。

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

在实际项目中，如果消费逻辑涉及较慢的下游调用（例如同步调用外部 API），需要将 `max.poll.records` 调小（默认 500），让每批消息的总处理时间控制在超时阈值之内。两者的配合方式参见第二节「ConsumerConfig 常量」一节中的说明。

---

## 六、`KafkaProducerConfig`：配置生产者工厂与模板

与消费者对应，发送消息的一侧同样需要一个配置类。Spring Kafka 中，生产者的核心抽象是 `ProducerFactory` 和 `KafkaTemplate`。

### `ProducerFactory` 与 `KafkaTemplate` 的职责

两者的分工很清晰：

- **`ProducerFactory<String, CloudEvent>`**：生产者工厂，持有所有连接与序列化配置（bootstrap 地址、key/value 序列化器、IAM 认证等），负责创建底层 `KafkaProducer` 实例。
- **`KafkaTemplate<String, CloudEvent>`**：Spring Kafka 对底层 `KafkaProducer` 的高级封装，屏蔽了手动管理连接生命周期、序列化、flush/close 等繁琐细节，业务代码只需一行调用：

```java
kafkaTemplate.send(topic, key, cloudEvent);
```

两者的层次关系：

```
ProducerFactory
    └── 持有配置，负责创建 Producer
         └── KafkaTemplate 用它来管理 Producer
              └── 业务代码调用 kafkaTemplate.send() 发消息
```

### 序列化：无需 `ErrorHandlingSerializer` 包装

Consumer 端需要用 `ErrorHandlingDeserializer` 包裹反序列化器，Producer 端则**不需要**任何等价的包装。原因在于两端的错误处理机制根本不同：

Producer 的序列化发生在**应用代码的调用栈上**：`KafkaTemplate.send()` 是同步触发序列化的，异常会直接传播回调用方，用普通的 `try/catch` 或 `CompletableFuture.exceptionally()` 处理即可，和普通 Java 异常处理没有区别。Spring Kafka 也没有提供 `ErrorHandlingSerializer`，因为根本不需要。

从数据所有权的角度看，两端的失败性质也截然不同：

| | Consumer | Producer |
|---|---|---|
| **数据来源** | 外部系统写入，不可控 | 本服务自己构造的 CloudEvent |
| **失败性质** | 运行时数据问题（poison pill） | 通常是代码 bug（字段类型不可序列化等） |
| **发现时机** | 生产环境任意时刻 | 开发/测试阶段就会暴露 |

Producer 这边序列化失败几乎肯定是编程错误，测试阶段就能发现，不是需要 DLQ 兜底的运行时场景，所以不需要同等的包装机制。

### 为什么要分成两个 Bean？

```java
@Bean
ProducerFactory<String, CloudEvent> finxQualifiedLeadProducerFactory() { ... }

@Bean
KafkaTemplate<String, CloudEvent> finxQualifiedLeadKafkaTemplate() {
    return new KafkaTemplate<>(finxQualifiedLeadProducerFactory());
}
```

将工厂和模板拆成两个 Bean 有以下好处：

- **可测试性**：测试时可以单独 mock `ProducerFactory`，替换成假实现，不需要真实的 Kafka 集群
- **可扩展性**：若将来有第二个 Topic 需要不同配置（如不同的序列化器或认证方式），再建一个 `ProducerFactory` 和对应的 `KafkaTemplate` 即可，互不干扰
- **职责分离**：工厂管"怎么连"，Template 管"怎么发"

### 连接是懒加载的

`@Bean` 方法在 Spring 启动时会执行，但**执行 Bean 方法 ≠ 建立 Kafka 连接**。

```
Spring 启动
    │
    ▼
执行 finxQualifiedLeadProducerFactory()
    → 创建 DefaultKafkaProducerFactory 对象，把配置存入内存
    → 没有任何网络连接
    ▼
执行 finxQualifiedLeadKafkaTemplate()
    → new KafkaTemplate(factory)，拿到工厂引用
    → 依然没有任何网络连接
```

真正的连接在**第一次调用 `.send()` 时**才建立：

```java
// 这一行触发 ProducerFactory.createProducer()，底层 KafkaProducer 才发起 TCP 连接
finxQualifiedLeadKafkaTemplate.send(finxQualifiedLeadTopicName, cloudEvent);
```

`DefaultKafkaProducerFactory` 默认会**缓存并复用** Producer 实例，因此：

- 第一次 `send()` → 创建 Producer，建立连接
- 后续 `send()` → 复用已有 Producer，直接发送，无需重连

这就像手机里存了号码（工厂存了配置），但存号码时并没有打电话，只有真正点餐（调用 `.send()`）时才拨出去建立通话。

---

## 总结

这篇文章通过一个真实的 Spring Boot Kafka 服务，串联了消费者与生产者两侧的以下知识点：

| 概念 | 作用 |
|---|---|
| `kafka_bootstrap_address` | Kafka 集群入口，用于获取集群元数据 |
| IAM Role Assume | 跨账号访问 Kafka Topic 的权限方案 |
| `@EnableKafka` | 激活 Spring 对 `@KafkaListener` 的扫描 |
| `ConcurrentKafkaListenerContainerFactory` | 创建并管理消费者线程的工厂 |
| `ErrorHandlingDeserializer` | 包裹反序列化器，将反序列化异常路由到 ErrorHandler 而非阻塞消费循环 |
| `FixedBackOff` + DLQ | 重试失败后将消息转移到死信队列的容错策略 |
| Offset 提交（AckMode） | 消息处理完成后由 Spring Kafka 自动提交 Offset，保证 at-least-once |
| `auto.offset.reset` | 新消费组首次启动时的读取起点（项目未显式设置，沿用 Kafka 原生默认值 `latest`） |
| `setConcurrency` | 单实例内的并发消费线程数 |
| `ConsumerRecord` | 封装消息体和元数据的对象 |
| `groupId` + 多实例 | 通过多个服务实例并行消费同一 Topic |
| `max.poll.records` | 控制每次拉取的消息数，避免单批处理超时触发 Rebalance |
| `ProducerFactory` | 持有生产者连接配置，负责创建并缓存 `KafkaProducer` 实例（懒加载） |
| `KafkaTemplate` | 对底层 Producer 的高级封装，提供简洁的消息发送 API |

Kafka 的设计哲学是"简单的服务端，聪明的客户端"——Spring Kafka 在客户端侧封装了大量复杂的逻辑（重试、错误处理、分区分配等），让业务代码只需关注消息本身的处理。
