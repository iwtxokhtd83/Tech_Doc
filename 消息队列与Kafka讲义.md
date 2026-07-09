# 消息队列与 Kafka 讲义

> 本讲义按两条主线组织：**消息队列通用原理与工程实践** 与 **Kafka 深度使用**（架构、存储、生产/消费、事务、运维、监控）。每章配"知识点 + 笔试题"。
>
> 约定：Kafka 示例基于 **3.x / KRaft 模式**；核心 API 基于官方 Java Client，思路可迁移到 Python/Go。

## 目录

1. [为什么需要消息队列](#1-为什么需要消息队列)
2. [核心模型与术语](#2-核心模型与术语)
3. [投递语义与可靠性](#3-投递语义与可靠性)
4. [顺序、幂等与去重](#4-顺序幂等与去重)
5. [消息堆积与背压](#5-消息堆积与背压)
6. [主流 MQ 对比与选型](#6-主流-mq-对比与选型)
7. [Kafka 基础与架构](#7-kafka-基础与架构)
8. [Topic、Partition 与 Replica](#8-topicpartition-与-replica)
9. [存储与日志机制](#9-存储与日志机制)
10. [Producer 深入](#10-producer-深入)
11. [Consumer 与消费组](#11-consumer-与消费组)
12. [事务与 Exactly-Once](#12-事务与-exactly-once)
13. [高可用：ISR、Leader 选举](#13-高可用isrleader-选举)
14. [Kafka 生态：Streams / Connect / Schema Registry](#14-kafka-生态streams--connect--schema-registry)
15. [监控、运维与常用命令](#15-监控运维与常用命令)
16. [常见业务模式与实战](#16-常见业务模式与实战)
17. [性能调优](#17-性能调优)
18. [综合笔试练习](#18-综合笔试练习)

---

## 1. 为什么需要消息队列

### 1.1 核心价值

- **异步解耦**：生产者不需等消费者处理完成即可返回
- **削峰填谷**：突发流量缓冲到队列，消费端按能力拉取
- **广播与订阅**：一条消息多方消费，扩展方便
- **跨系统集成**：不同语言、不同生命周期的服务通过消息通信
- **最终一致**：替代分布式事务的常用手段

### 1.2 典型场景

- 订单创建 → 扣减库存 / 发通知 / 生成发票（异步并行）
- 用户行为日志 → 实时数仓、推荐、风控
- 业务变更广播 → 缓存失效、搜索索引更新
- 流量削峰 → 秒杀、抢购、限时活动
- 分布式任务调度、延时任务
- 微服务之间的事件驱动（EDA）

### 1.3 带来的代价

- **复杂度**：多了一个中间件要运维、监控
- **一致性问题**：异步后状态暂时不一致
- **排查更难**：调用链跨越 MQ，需要 trace
- **消息可靠性**：丢失 / 重复 / 乱序 要专门设计
- **幂等设计压力**：消费者必须考虑重复消息

### 📝 笔试题 1-1：什么时候**不应该**引入 MQ？

- 强同步场景（用户等响应，不能异步）
- 链路短、两服务紧耦合、延迟敏感
- 团队没有 MQ 运维经验，故障会成为放大器
- 并发不高、流量平稳，直连足够

---

## 2. 核心模型与术语

### 2.1 两种经典模型

#### 点对点（Queue）

```
Producer → [Queue] → Consumer (只有一个消费)
```

- 每条消息被**一个**消费者消费
- 典型：任务分发
- 代表：RabbitMQ 默认队列、SQS

#### 发布订阅（Topic）

```
Producer → [Topic] → Consumer A
                   → Consumer B
                   → Consumer C
```

- 每条消息被**所有订阅者**消费
- 典型：事件广播
- 代表：Kafka Topic、Redis Pub/Sub、RabbitMQ Fanout Exchange

Kafka 通过 **Consumer Group** 统一表达了两种模型：

- 组内竞争消费 = 点对点
- 组间独立消费 = 发布订阅

### 2.2 通用术语

| 术语 | 含义 |
|------|------|
| **Broker** | MQ 服务器节点 |
| **Producer** | 生产者 |
| **Consumer** | 消费者 |
| **Topic** | 主题，逻辑分类 |
| **Partition / Queue** | 物理分片 |
| **Offset** | 消息在分区中的位置 |
| **ACK** | 消费者确认已处理 |
| **DLQ (Dead Letter Queue)** | 死信队列，处理失败转储 |
| **Retry** | 重试队列或重试机制 |
| **Backpressure** | 背压，下游慢时上游降速 |

### 📝 笔试题 2-1：RabbitMQ 的 Exchange 和 Kafka 的 Topic 有何异同？

- **相同**：都是消息的入口、负责路由
- **不同**：
  - RabbitMQ Exchange **按路由键**把消息投递到 Queue，消息存 Queue 里
  - Kafka Topic **就是**物理分片的日志，生产者按分区键直接写入分区

RabbitMQ 的路由更灵活（direct/fanout/topic/headers），Kafka 的模型更简单直接但吞吐高。

---

## 3. 投递语义与可靠性

### 3.1 三种语义

| 语义 | 保证 | 代价 |
|------|------|------|
| **At most once** | 至多一次，可能丢 | 最简单，极低延迟 |
| **At least once** | 至少一次，可能重 | 主流选择，需**业务幂等** |
| **Exactly once** | 恰好一次 | 代价较高，需端到端协议 |

### 3.2 消息的生命周期与可能丢失点

```
Producer --①--> Broker --②--> Consumer --③--> (处理)
```

- **① 发送阶段**：生产者发后未收到确认就崩溃
- **② 存储阶段**：Broker 宕机、未落盘副本
- **③ 消费阶段**：消费者 ACK 后才崩溃会重放；ACK 前崩溃会丢

### 3.3 防止丢失的通用原则

- **生产端**：
  - 开启确认机制（`acks=all` / Publisher Confirm）
  - 失败重试 + 退避
  - 本地消息表 + 定时补偿（事务消息）
- **Broker 端**：
  - 多副本（副本数 ≥ 3）
  - 持久化（刷盘策略）
  - 跨机架 / 可用区分布
- **消费端**：
  - **先处理后 ACK**
  - 失败不 ACK，让 MQ 重投
  - 配合幂等避免重复副作用

### 3.4 exactly-once 的本质

没有魔法。做法是**至少一次投递 + 幂等消费**：

- Kafka 的 EOS = 幂等 Producer + 事务 + Consumer 的 `read_committed` + offset 提交放入事务
- RabbitMQ 传统方案：**生产者确认 + 事务消息 + 幂等消费**

### 📝 笔试题 3-1：说说 `at-least-once` 下如何保证业务不重复扣款？

**业务幂等**：

1. 客户端生成唯一 `request_id`
2. 服务端处理前**查表**：该 id 是否已处理
3. 未处理 → 扣款 + 记录 id（同一事务）
4. 已处理 → 直接返回上次结果

也可以用 Redis `SETNX` 做分布式幂等令牌，或数据库唯一索引。

---

## 4. 顺序、幂等与去重

### 4.1 顺序消息

**全局顺序**几乎不可能在高吞吐下保证，常用**局部顺序**：

- Kafka：同一个 Key → 同一分区 → 同一消费者 → 有序
- RabbitMQ：单队列 + 单消费者 = 顺序；或 consistent-hash 路由
- RocketMQ：MessageQueueSelector 按业务键选队列

**顺序消息的代价**：

- 单分区单消费者 → 无法水平扩展
- 消费失败不能跳过（一跳就乱序）→ 阻塞
- 实践建议：**业务维度分区**（如按 userId），允许分区内严格有序、分区间并行

### 4.2 幂等消费

**手段**：

- **唯一键 + 去重表**：消息带 `msgId`，消费时 `INSERT ... ON DUPLICATE KEY` 或唯一索引
- **版本号 / CAS**：业务状态机只接受比当前版本高的更新
- **Redis 去重**：`SETNX dedup:<msgId> 1 EX 86400`
- **Bloom Filter**：海量去重的近似方案

**要点**：

- **幂等键**要稳定（不要用时间戳）
- 去重窗口覆盖 MQ 最长重投周期
- 业务有自然幂等（`SET x=v`）优先用

### 4.3 死信队列（DLQ）

处理失败 N 次后转入 DLQ：

```
Normal Queue → (fail N times) → DLQ → 人工/后台处理
```

- 避免毒消息（poison message）无限重试阻塞队列
- Kafka 无原生 DLQ，需业务实现（失败后投递到 `topic.DLT`）
- RabbitMQ 内置死信交换机
- RocketMQ 重试 16 次后自动进入 `%DLQ%xxx`

### 📝 笔试题 4-1：Kafka 如何保证同一订单号的消息严格有序？

1. 生产时把 `order_id` 作为消息 Key
2. Kafka 默认按 `hash(key) % partitions` 路由，同 key 落到同一分区
3. 同一分区由消费组内一个 consumer 独占消费
4. 消费者单线程处理（或按 key 再分发给内部线程池）
5. 不能在消费失败时跳过，需要阻塞重试或进入 DLQ 再人工处理

---

## 5. 消息堆积与背压

### 5.1 成因

- 消费速度 < 生产速度（突发高峰、下游慢、消费者挂掉）
- 消息很大或处理逻辑慢
- 消费者数量不够

### 5.2 后果

- Broker 磁盘/内存占用上涨
- 延迟飙升（用户看到几十分钟的滞后）
- 后续流量全部卡在队列里

### 5.3 应对策略

**预防**：

- 监控 Lag（消费滞后）、告警
- 合理设置 TTL / 保留时长
- 压测估算消费能力

**出现堆积时**：

1. **扩容消费者**：Kafka 增加 consumer 到分区数上限；超过需扩分区
2. **并发消费**：消费者内部多线程并行处理消息（注意顺序与幂等）
3. **分级降级**：将大消息 / 非关键消息降级或丢弃
4. **紧急迁移**：临时把积压消息转到其他 topic/队列，用专用消费者处理
5. **批处理**：批量拉取 + 批量写 DB
6. **限流生产**：防止继续堆积

### 5.4 背压机制

- 拉模式（Kafka）：消费者控制节奏，天然带背压
- 推模式（RabbitMQ）：通过 `prefetch count` 限制未 ACK 数量
- Reactive Streams（Flink/RxJava）有标准 `request(n)` 协议

### 📝 笔试题 5-1：Kafka 分区有 32 个，消费者加到 64 个能提速吗？

**不能**。同一消费组内，每个分区只能被**一个**消费者独占，超过分区数的消费者将**空闲**。想提速：

- 增加分区（需评估 key 分布、顺序性）
- 单消费者内部用线程池并行处理（需处理 offset 提交与顺序）

---

## 6. 主流 MQ 对比与选型

| 产品 | 模型 | 吞吐 | 延迟 | 顺序 | 事务 | 优势 | 典型场景 |
|------|------|------|------|------|------|------|----------|
| **Kafka** | 日志型 | 极高（百万+ msg/s） | 中（ms） | 分区内 | ✅（EOS） | 吞吐王、生态丰富 | 日志、流处理、CDC |
| **RabbitMQ** | Broker 路由 | 中（万级） | 低（μs-ms） | 单队列 | ✅ | 路由灵活、协议成熟（AMQP） | 业务消息、任务分发 |
| **RocketMQ** | 主题 + 队列 | 高（十万级） | 低 | ✅ | ✅（事务消息） | 阿里开源、金融级 | 订单、事务消息 |
| **ActiveMQ / Artemis** | JMS | 中 | 低 | ✅ | ✅ | JMS 规范兼容 | 传统企业 |
| **Pulsar** | 日志 + 队列 | 高 | 低 | 分区内 | ✅ | 计算存储分离、多租户 | 云原生、多集群 |
| **NATS / JetStream** | Pub/Sub | 极高 | 极低（μs） | 可选 | ✅（JetStream） | 轻量、简单 | 微服务通信、IoT |
| **Redis Stream** | 流 | 高 | 极低 | 分区内 | ✅（XACK） | 免增中间件 | 小规模、内置场景 |
| **AWS SQS/SNS/Kinesis** | 托管 | 高 | 中 | FIFO 可选 | 有限 | 免运维 | 云上 |

**选型直觉**：

- **大数据 / 流处理 / 日志汇聚** → **Kafka**
- **业务路由复杂、事务消息** → RabbitMQ / RocketMQ
- **需要云原生、跨集群、多租户** → Pulsar
- **微服务轻量通信** → NATS
- **已经有 Redis 且规模小** → Redis Stream

---

## 7. Kafka 基础与架构

### 7.1 Kafka 是什么

一个分布式、高吞吐、持久化的**发布订阅消息系统 + 流处理平台**。核心是一份分区化、副本化的**持久化日志**。

### 7.2 架构

```
┌──────────┐       ┌──────────────────────────┐      ┌──────────┐
│ Producer │──────▶│   Kafka Cluster          │◀─────│ Consumer │
└──────────┘       │   ┌────────┐ ┌────────┐  │      └──────────┘
                   │   │Broker 1│ │Broker 2│  │      Consumer Group
                   │   └────────┘ └────────┘  │
                   │   ┌────────┐ ┌────────┐  │
                   │   │Broker 3│ │Broker 4│  │
                   │   └────────┘ └────────┘  │
                   └──────────────────────────┘
                          │
                          ▼
                  ZooKeeper 或 KRaft（元数据）
```

### 7.3 核心角色

- **Broker**：Kafka 节点，承担存储 + 服务
- **Producer**：生产者
- **Consumer**：消费者，组织为 **Consumer Group**
- **Topic**：逻辑主题
- **Partition**：主题的物理分片
- **Replica**：分区副本，一 Leader + N Follower
- **Controller**：负责管理分区 Leader 选举、副本迁移的 Broker 角色
- **Coordinator**：消费组协调者（分配分区、提交 offset）
- **ZooKeeper / KRaft**：集群元数据（KRaft 是 Kafka 3.3+ 的原生替代）

### 7.4 KRaft 与 ZooKeeper

- **ZooKeeper**：历史元数据存储，维护 Controller 选举、topic 元数据
- **KRaft (KIP-500)**：Kafka 3.3+ 可完全替代 ZK，使用 Raft 协议维护元数据，**部署更简单、扩展性更好**
- 新集群建议直接 KRaft；生产已用 ZK 可平滑迁移

### 📝 笔试题 7-1：Kafka 为什么需要 Controller？

Controller 是集群的"大脑"：

- 维护所有 topic/partition 的 Leader 与 ISR
- Broker 宕机时触发 Leader 重新选举
- 处理副本创建、迁移、删除

只有一个 Broker 同时担任 Controller，Controller 挂了会重新选举（ZK watch 或 KRaft 选举）。

---

## 8. Topic、Partition 与 Replica

### 8.1 Topic

逻辑概念，按业务命名：`orders`, `user.events`, `logs.access`。

### 8.2 Partition

- Topic 被切成若干个 Partition，**分区是并行度的基本单位**
- 每个分区是**一个严格有序、不可变的日志**
- 消息写入 Partition 时分配一个**单调递增的 offset**
- 分区数决定**最大消费并发度**（一个分区只能被组内一个消费者消费）

### 8.3 分区选择

生产者写入时，Kafka 按下列顺序决定目标分区：

1. **显式指定 `partition`**
2. **根据 Key** 计算 `hash(key) % partitions`（DefaultPartitioner / Murmur2）
3. **无 Key** 时使用 **Sticky Partitioner**（2.4+）—— 批次内固定分区，提高批量效率

### 8.4 Replica（副本）

```
Topic: orders, partition 0
  Leader:    Broker 1
  Followers: Broker 2, Broker 3
```

- 生产/消费都**只与 Leader 交互**
- Follower 主动拉取 Leader 的日志，追上后加入 **ISR**
- **min.insync.replicas**：写入需要至少 N 个副本（含 Leader）确认，否则拒写

### 8.5 ISR（In-Sync Replicas）

ISR = 与 Leader "同步" 的副本集合。Follower 如果长时间追不上：

- `replica.lag.time.max.ms`（默认 30s）：超过此时间未追上 → 被踢出 ISR
- Leader 选举**只从 ISR 中**选（默认），保证不丢数据

### 8.6 分区数选择

- 更多分区 → 更高并行度，但也带来：
  - 更多 FD / 内存
  - Leader 选举时间变长
  - 端到端延迟可能升高
  - Rebalance 时间变长
- 经验：单 Broker 每秒数千到数万写入为宜；分区总数几百到几千
- **分区数只能增加不能减少**（会导致 key 分布变化）

### 📝 笔试题 8-1：`replication.factor=3`，`min.insync.replicas=2` 的含义？

- 每个分区有 **3 个副本**
- 生产者 `acks=all` 时，**至少 2 个副本**（含 Leader）写入成功才算成功
- 若 ISR 降到 1（两个 Follower 挂了），生产者收到 `NotEnoughReplicasException`，拒写保证数据安全
- 推荐经典配置：3 副本 + `min.insync.replicas=2`，容忍一台 Broker 宕机

---

## 9. 存储与日志机制

### 9.1 日志结构

每个分区对应一个物理目录：

```
/var/kafka/logs/orders-0/
  00000000000000000000.log
  00000000000000000000.index
  00000000000000000000.timeindex
  00000000000123456789.log
  00000000000123456789.index
  ...
```

- **Segment**：日志被切分为多个段文件（默认 1GB 或 7 天切一次）
- **.log**：实际消息
- **.index** / **.timeindex**：offset → 物理位置 / 时间戳 → offset 的稀疏索引

### 9.2 顺序写 + Page Cache

Kafka 的高吞吐来源：

- **顺序追加**：磁盘顺序写能到 600MB/s
- **Page Cache**：写入先进内核缓存，异步刷盘；读多数也命中 cache
- **零拷贝 (sendfile)**：消费时直接从 Page Cache 到 socket，不经用户态
- **批量**：Producer 攒 batch，Broker 压缩整批存储

### 9.3 保留策略

- **按时间**：`retention.ms`，默认 7 天
- **按大小**：`retention.bytes`，每分区
- 两者取"谁先到"

### 9.4 压缩（Compaction）

```
cleanup.policy=compact
```

**日志压缩**：同一 Key 只保留最新 value，旧值被清理。

- 适合：存储"实体最新状态"的 topic（如 `user-profiles`）
- 经典用法：Kafka Streams 的 **changelog topic**
- `delete` 和 `compact` 可组合：`cleanup.policy=compact,delete`

### 9.5 刷盘策略

默认**依赖 OS 异步刷盘 + 副本冗余**保证可靠性：

- `log.flush.interval.messages` / `log.flush.interval.ms`（默认极大，意味几乎不主动刷）
- **不建议**手动开启同步刷盘，Kafka 的可靠性模型基于复制而非单机刷盘

### 📝 笔试题 9-1：Kafka 为什么这么快？

- **顺序写磁盘**：避免随机 IO
- **批量 + 压缩**：减少网络和磁盘写入
- **零拷贝 (sendfile)**：省去用户态拷贝
- **Page Cache**：充分利用 OS 文件缓存
- **分区并行**：水平扩展
- **拉模式 + 批量消费**：消费端高效

---

## 10. Producer 深入

### 10.1 最小示例（Java）

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer",   "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");
props.put("enable.idempotence", "true");
props.put("retries", Integer.MAX_VALUE);
props.put("linger.ms", 20);
props.put("batch.size", 32 * 1024);
props.put("compression.type", "lz4");

try (KafkaProducer<String,String> p = new KafkaProducer<>(props)) {
    for (int i = 0; i < 1000; i++) {
        ProducerRecord<String,String> rec =
            new ProducerRecord<>("orders", "order-" + i, "{\"id\":" + i + "}");
        p.send(rec, (md, ex) -> {
            if (ex != null) ex.printStackTrace();
            else System.out.printf("ok p=%d off=%d%n", md.partition(), md.offset());
        });
    }
}
```

### 10.2 acks 语义

| acks | 含义 | 可靠性 | 吞吐 |
|------|------|--------|------|
| `0` | 不等 Broker 确认 | 最低，可能丢 | 最高 |
| `1` | Leader 写入即 ACK | Leader 挂了可能丢 | 高 |
| `all` / `-1` | Leader + ISR 达到 `min.insync.replicas` | 最高 | 中 |

生产用 `acks=all`，**一切讨论吞吐都建立在"已经能保证不丢数据"基础上**。

### 10.3 幂等 Producer（0.11+）

```
enable.idempotence=true
```

- Broker 为每个 `(Producer ID, Partition)` 维护序列号
- 同一消息重发只写一次（去重窗口内）
- **防止生产端网络重试导致的重复**
- 开启后，要求 `acks=all`、`retries>0`、`max.in.flight.requests.per.connection <= 5`

### 10.4 批量与压缩

| 参数 | 含义 |
|------|------|
| `linger.ms` | 攒批最长等待时间 |
| `batch.size` | 单个分区批大小（字节）|
| `compression.type` | `none`/`gzip`/`snappy`/`lz4`/`zstd` |
| `buffer.memory` | Producer 端缓冲总大小 |

压缩算法选择：

- **lz4**：速度+压缩率综合最佳（默认推荐）
- **zstd**：压缩率更高，CPU 稍重
- **snappy**：老项目常见
- **gzip**：压缩率高但 CPU 开销大，不推荐

### 10.5 发送模式

- **Fire-and-forget**：`send()` 不管结果，最快也最不安全
- **异步回调**：`send(record, callback)`，生产常用
- **同步**：`send(...).get()`，确保逐条成功，低吞吐

### 10.6 自定义分区器

```java
public class MyPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int n = cluster.partitionCountForTopic(topic);
        // 按业务规则路由
        if (key.toString().startsWith("VIP")) return 0;
        return Math.abs(key.hashCode()) % n;
    }
}
props.put("partitioner.class", "com.x.MyPartitioner");
```

### 📝 笔试题 10-1：Producer 开启 idempotence 后，还能保证顺序吗？

可以。幂等 Producer 要求 `max.in.flight.requests.per.connection <= 5`，Broker 端按序列号校验；乱序会被拒绝重发，最终落盘有序。**前提**：同一分区。

---

## 11. Consumer 与消费组

### 11.1 Consumer Group 模型

```
Topic: orders (partition 0,1,2,3)

Group A:
  consumer-a1 → partition 0,1
  consumer-a2 → partition 2,3    (组内 partition 均分)

Group B:
  consumer-b1 → partition 0,1,2,3 (组间独立)
```

- 同组内：一个 partition 只被一个 consumer 消费
- 组间：互不影响，各自消费全部消息

### 11.2 最小示例（Java）

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092");
props.put("group.id", "order-consumer");
props.put("enable.auto.commit", "false");
props.put("auto.offset.reset", "earliest");      // latest / earliest / none
props.put("key.deserializer",   StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
props.put("max.poll.records", 500);

try (KafkaConsumer<String,String> c = new KafkaConsumer<>(props)) {
    c.subscribe(List.of("orders"));
    while (true) {
        ConsumerRecords<String,String> recs = c.poll(Duration.ofMillis(500));
        for (var r : recs) {
            process(r);
        }
        c.commitSync();                           // 手动提交
    }
}
```

### 11.3 位移（Offset）

- **Current offset**：下一条要读的位置
- **Committed offset**：已提交到 Broker 的位置（`__consumer_offsets` topic 保存）
- 重启或 rebalance 后从 committed offset 恢复

### 11.4 offset 提交方式

```java
// 自动提交（不推荐生产）
enable.auto.commit=true
auto.commit.interval.ms=5000       // 每 5s 提交一次 → 可能重复

// 手动同步
consumer.commitSync();

// 手动异步（更快，失败时回调重试）
consumer.commitAsync((offsets, ex) -> { ... });

// 按分区细粒度
consumer.commitSync(Map.of(tp, new OffsetAndMetadata(offset + 1)));
```

**先处理后提交**是保证 at-least-once 的关键。

### 11.5 Rebalance

- 触发：组成员变化（加入/退出）、订阅变更、分区数变化
- **Stop-the-world**：旧 Rebalance 全组暂停
- **Cooperative Rebalance**（2.4+）：增量式，仅迁移受影响分区
- 长处理要设好 `max.poll.interval.ms`，否则可能被判定离组

```
max.poll.interval.ms = 300000            # 两次 poll 间最大间隔
session.timeout.ms   = 10000             # 组内心跳超时
heartbeat.interval.ms= 3000              # 心跳频率
```

### 11.6 消费模式

- **自动分配**（常用）：`subscribe(topics)` + Coordinator 分配分区
- **手动分配**：`assign(partitions)`，完全自主，不参与组协调（适合外部协调）

### 11.7 并发消费的做法

Kafka 消费本身是**单线程 poll**，如果处理慢：

- **增加分区 + 增加消费者实例**
- **消费者内部线程池**：poll 后按 key hash 分发到 worker（注意保 offset 与顺序）
- **批量处理**：一次 poll 几百条，合并写 DB / 调下游

### 📝 笔试题 11-1：Consumer 开了自动提交还会丢消息吗？

会。自动提交是**定时**（例如 5s）提交最新 offset，若：

1. poll 了一批消息
2. 还未处理完，自动提交时间到，offset 被提交
3. 进程崩溃 → 重启后从已提交位置继续 → **这批消息被跳过**

所以生产建议 `enable.auto.commit=false`，业务处理后 `commitSync()`。

### 📝 笔试题 11-2：为什么同一消费组内，消费者数超过分区数就多余？

Kafka 把分区独占分配给消费者，同组内**一个分区一个消费者**。分区数是上限：多余的消费者拿不到分区，处于空闲状态，等其他消费者下线才接管。想提速必须增加分区。

---

## 12. 事务与 Exactly-Once

### 12.1 幂等 vs 事务

- **幂等 Producer**：解决**单分区**同一批重发去重
- **事务 Producer**：跨多分区、多 topic 的**原子写**；与 Consumer 配合实现 **Exactly-Once Semantics (EOS)**

### 12.2 事务 API

```java
props.put("transactional.id", "tx-producer-1");
props.put("enable.idempotence", "true");
props.put("acks", "all");

KafkaProducer<String,String> p = new KafkaProducer<>(props);
p.initTransactions();

try {
    p.beginTransaction();
    p.send(new ProducerRecord<>("orders",  "k", "v1"));
    p.send(new ProducerRecord<>("payments","k", "v2"));

    // 关键：把消费 offset 也放入事务（Consume-Process-Produce 原子）
    p.sendOffsetsToTransaction(offsets, consumerGroupMetadata);

    p.commitTransaction();
} catch (Exception e) {
    p.abortTransaction();
}
```

### 12.3 消费端配合

```
isolation.level = read_committed
```

消费者只读到**已提交**事务中的消息，未提交/已中止的消息跳过。

### 12.4 EOS 的前提

- 生产端：`enable.idempotence=true` + 事务
- Broker：副本配置正确
- 消费端：`read_committed` + 把 offset 提交也包在事务里
- **下游必须是 Kafka 或幂等外部系统**（对外部系统写不在事务覆盖范围内）

### 12.5 事务代价

- 每笔事务额外开销（事务协调器、提交标记）
- 不适合高频短事务场景
- 典型用途：**流式处理中 Kafka→Kafka 管道**（Kafka Streams 默认启用）

### 📝 笔试题 12-1：Kafka 的 Exactly-Once 能延伸到 MySQL 吗？

**不能直接**。事务只覆盖 Kafka 内部；写到外部 MySQL 不在事务里，仍可能因崩溃重复。方案：

- 写外部时用**业务幂等**（唯一键、version）
- 使用 **Kafka Connect + 支持 EOS 的 Sink**（如 Debezium 到部分数据库）
- 用 **Outbox Pattern** 把业务写 + 消息写放在同一 DB 事务

---

## 13. 高可用：ISR、Leader 选举

### 13.1 ISR 动态维护

- Follower 不断拉 Leader 的日志
- 追上就在 ISR；落后超过阈值就踢出
- 恢复后再加入

### 13.2 Leader 选举

- 由 Controller 发起
- **默认只在 ISR 里选**（`unclean.leader.election.enable=false`）
- `unclean.leader.election.enable=true` 允许选非 ISR 成员，可用性↑ 但**会丢数据**

### 13.3 写入高可用配置

经典组合：

```
replication.factor        = 3
min.insync.replicas       = 2
unclean.leader.election   = false
acks                       = all
retries                    = Integer.MAX_VALUE
enable.idempotence        = true
```

容忍 1 台 Broker 宕机不丢消息，不阻塞写入。

### 13.4 机架感知

`broker.rack=rack1` 让副本尽量跨机架分布。跨 AZ 分布更好，但延迟会升高。

### 📝 笔试题 13-1：`unclean.leader.election.enable=true` 的风险？

允许一个**落后**的副本成为 Leader。已提交到旧 Leader 但未同步到该副本的消息会被**覆盖/丢失**。只有在"要可用不要数据"的场景（如日志）才开，金融/订单绝不能开。

---

## 14. Kafka 生态：Streams / Connect / Schema Registry

### 14.1 Kafka Streams

JVM 库（非独立集群），在应用内做**流处理**：

```java
StreamsBuilder b = new StreamsBuilder();
KStream<String, Order> orders = b.stream("orders");

orders
  .filter((k, v) -> v.amount > 100)
  .groupByKey()
  .count()
  .toStream()
  .to("order-counts");

new KafkaStreams(b.build(), config).start();
```

特性：

- 内置 KTable（表语义）、窗口、join、聚合
- EOS、可恢复、无需额外调度器
- 比 Flink 轻量，适合**中小规模流处理**

### 14.2 Kafka Connect

数据集成框架，用于**Source**（外部→Kafka）和 **Sink**（Kafka→外部）：

- Source 常见：MySQL binlog (Debezium)、文件、HTTP
- Sink 常见：Elasticsearch、HDFS/S3、JDBC、Redis
- 分布式模式：多 worker 负载均衡、自动故障转移
- 配置驱动，无需编码

```json
{
  "name": "mysql-source",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.server.id": "184054",
    "database.server.name": "inventory",
    "table.include.list": "inventory.orders",
    "database.history.kafka.topic": "schema-changes.inventory"
  }
}
```

### 14.3 Schema Registry

存储 Avro / Protobuf / JSON Schema 定义，保证生产/消费的**数据契约**兼容：

- **兼容模式**：BACKWARD / FORWARD / FULL / NONE
- 避免新字段导致老消费者崩溃
- 生产者/消费者用对应 Serializer 自动按 schema 编解码

### 14.4 ksqlDB

基于 Kafka Streams 的 SQL 引擎：

```sql
CREATE STREAM orders_high AS
  SELECT * FROM orders WHERE amount > 100;
```

### 14.5 生态与周边

- **MirrorMaker 2**：跨集群复制（多活、灾备）
- **Cruise Control**：自动分区再均衡
- **Conduktor / AKHQ / Kafdrop**：可视化管理
- **Confluent Platform**：Confluent 公司的企业版，含 Control Center、RBAC 等

---

## 15. 监控、运维与常用命令

### 15.1 常用 CLI

```bash
# 查看 topic
kafka-topics.sh --bootstrap-server b1:9092 --list
kafka-topics.sh --bootstrap-server b1:9092 --describe --topic orders

# 创建
kafka-topics.sh --bootstrap-server b1:9092 --create \
  --topic orders --partitions 12 --replication-factor 3 \
  --config min.insync.replicas=2 --config retention.ms=604800000

# 修改分区（只增不减）
kafka-topics.sh --bootstrap-server b1:9092 --alter --topic orders --partitions 24

# 生产 / 消费测试
kafka-console-producer.sh --bootstrap-server b1:9092 --topic orders
kafka-console-consumer.sh --bootstrap-server b1:9092 --topic orders --from-beginning

# 消费组
kafka-consumer-groups.sh --bootstrap-server b1:9092 --list
kafka-consumer-groups.sh --bootstrap-server b1:9092 --describe --group order-consumer
# 重置 offset（谨慎）
kafka-consumer-groups.sh --bootstrap-server b1:9092 --group g --topic orders \
  --reset-offsets --to-earliest --execute

# 副本再平衡
kafka-reassign-partitions.sh ...

# 性能测试
kafka-producer-perf-test.sh --topic test --num-records 1000000 \
  --record-size 1024 --throughput -1 \
  --producer-props bootstrap.servers=b1:9092 acks=all
kafka-consumer-perf-test.sh --bootstrap-server b1:9092 --topic test --messages 1000000
```

### 15.2 监控指标（JMX / Prometheus）

Broker 层：

- **UnderReplicatedPartitions**：>0 表示有分区 Follower 掉队
- **OfflinePartitionsCount**：>0 表示分区无 Leader，写入会失败
- **ActiveControllerCount**：集群内应只有 1
- **RequestHandlerAvgIdlePercent** / **NetworkProcessorAvgIdlePercent**：idle 高 = 资源富裕
- **BytesInPerSec** / **BytesOutPerSec** / **MessagesInPerSec**

Topic/Partition 层：

- **LogEndOffset** / **LogStartOffset**
- **ConsumerLag**（核心告警指标）

Producer / Consumer 层：

- Producer：`record-send-rate` / `record-error-rate` / `request-latency-avg`
- Consumer：`records-consumed-rate` / `fetch-latency-avg` / `commit-latency-avg` / lag

### 15.3 Lag 监控

```bash
kafka-consumer-groups.sh --describe --group order-consumer
```

输出 `CURRENT-OFFSET` / `LOG-END-OFFSET` / `LAG`。

生产级方案：

- **Burrow**（LinkedIn）
- **kafka-lag-exporter** + Prometheus/Grafana
- Confluent Control Center

### 15.4 常见运维操作

- **扩容**：加 Broker → 用 `kafka-reassign-partitions` 迁分区
- **升级**：滚动升级，先升级 Broker（兼容旧协议），再升级客户端
- **下线 Broker**：先把其上分区 Leader 迁走
- **紧急回滚 offset**：`--reset-offsets` 到指定时间/earliest/latest

### 📝 笔试题 15-1：线上告警"UnderReplicatedPartitions=5"，怎么办？

1. `kafka-topics --describe`：找出哪些分区 ISR 缺失
2. 查 Broker 是否宕机 / 网络断 / 磁盘满
3. 查该 Broker 的错误日志：`server.log`、`controller.log`
4. 若 Broker 恢复，Follower 会自动追赶并回到 ISR
5. 若副本数严重不足，可临时调大 `num.replica.fetchers` 加速追赶
6. 数据丢失风险：确保未触发 `unclean.leader.election`

---

## 16. 常见业务模式与实战

### 16.1 事件溯源 / CQRS

- 所有状态变更以事件形式写入 Kafka
- 消费者重放事件构建物化视图
- 天然的审计、可回溯、多视图

### 16.2 CDC（Change Data Capture）

- **Debezium** 捕获 MySQL binlog → Kafka
- 下游消费：**更新缓存 / 更新搜索索引 / 数据仓库**
- 解决"双写一致"问题（只写 DB，由 CDC 保证派生数据）

### 16.3 日志聚合

- 应用日志 → Filebeat / Fluentd → Kafka → ES / HDFS
- 削峰、解耦、高吞吐

### 16.4 流处理 Pipeline

```
Kafka Source → Flink/Streams 处理 → Kafka Sink → 下游
```

- 聚合、join、窗口计算
- 结合 state store 做有状态流处理

### 16.5 分布式事务：Outbox + Kafka

```
业务操作：
  BEGIN
    UPDATE orders SET status=paid
    INSERT INTO outbox (topic, key, payload, ...) VALUES (...)
  COMMIT

独立 Publisher：
  从 outbox 表读未发消息 → 发到 Kafka → 删除/标记已发
```

- 业务 DB 事务保证"状态变更与事件记录"原子
- Publisher 做 at-least-once 投递
- 消费端幂等

### 16.6 延时 / 定时任务

Kafka 原生**无**延时消息。常见方案：

- **按延时桶分 topic**（5s/30s/1min/...）
- 外部调度（Redis ZSet / 定时器）+ Kafka 执行
- RocketMQ 内置延时级别更方便

### 16.7 订阅 + 事件驱动架构

```
订单服务 → OrderCreated → Kafka
                          ├─ 库存服务消费扣库存
                          ├─ 优惠服务消费核销
                          ├─ 通知服务发短信
                          └─ 数仓消费入库
```

服务间仅依赖事件契约，解耦彻底。

### 📝 笔试题 16-1：订单创建成功但消息发送失败，怎么办？

**不要在业务事务中同步发 Kafka**（Kafka 宕机会让业务失败）。使用 **Outbox Pattern**：

1. 业务 DB 事务中插入 `outbox` 表
2. 独立 Publisher 后台读 outbox → 发 Kafka → 更新/删除 outbox 记录
3. Publisher 可用 **Debezium 直接读 outbox 表 binlog**，更简洁

这样业务事务与消息投递解耦，可靠性由 outbox + 重试保证。

---

## 17. 性能调优

### 17.1 Broker 侧

| 参数 | 建议 |
|------|------|
| `num.network.threads` | CPU 核数，默认 3 起步 |
| `num.io.threads` | 2 × CPU，默认 8 |
| `num.replica.fetchers` | 2-4，提高副本同步 |
| `socket.send/receive.buffer.bytes` | 100KB-1MB |
| `log.segment.bytes` | 1GB 默认合理 |
| `log.retention.*` | 按业务 |
| `log.cleaner.threads` | compact topic 多时调大 |
| JVM Heap | 5-8GB，**其他交给 Page Cache** |

### 17.2 OS 与硬件

- **SSD / NVMe**：I/O 延迟决定性能
- **独立磁盘**：日志目录独占磁盘 / RAID 10
- **关闭 swap**：`vm.swappiness=1`
- **文件描述符**：`ulimit -n 1048576`
- **网络**：千兆/万兆，RSS / RPS 调优
- **JVM GC**：G1 默认即可，关注 STW 时间

### 17.3 Producer 侧

- **加大 batch + linger**：提高吞吐
  ```
  linger.ms=20          batch.size=64KB
  ```
- **开启压缩**：`lz4` / `zstd`
- **异步发送 + 回调**
- **连接池**：一个 Producer 实例多线程共享（线程安全）

### 17.4 Consumer 侧

- **poll 粒度 + 批处理**：`max.poll.records` 调到合适
- **业务耗时 < `max.poll.interval.ms`**：否则被踢出组
- 内部 **线程池并行**，按 key 分派保顺序
- 批量下游写：攒够一批再写 DB

### 17.5 Topic 设计

- **合适分区数**：并行度 / 吞吐目标 / 消费者数
- **合理 key**：决定分区均衡与顺序
- **消息大小 ≤ 1MB**：默认 `message.max.bytes=1MB`，超大消息伤吞吐
- **拒绝超大消息**：对象存 S3，Kafka 只传指针

### 17.6 网络拓扑

- **生产者/消费者尽量同机房**
- **跨集群复制** → MirrorMaker 2 / Replicator

### 📝 笔试题 17-1：Producer 吞吐上不去，排查方向？

- `linger.ms` 是否太低（0 则不攒批）
- `batch.size` 是否太小
- 是否开了压缩
- `acks=all` 的 ISR 写入慢：看 Broker `RequestQueueTimeMs` / `ProduceTotalTimeMs`
- 网络是否瓶颈
- Producer 实例数太少（复用同一实例 + 多线程）
- 目标 topic 分区数是否够
- JVM GC 停顿是否频繁

---

## 18. 综合笔试练习

### 18.1 选择题

**Q1** Kafka 一个分区在同一消费组内最多有几个消费者消费？
A. 1  B. 2  C. 等于副本数  D. 不限

<details><summary>答案</summary>A。</details>

**Q2** 下列关于 `acks` 的描述错误的是？
A. `acks=0` 吞吐最高
B. `acks=1` Leader 写入即 ACK
C. `acks=all` 需要所有副本确认
D. `acks=all` 实际等待 `min.insync.replicas` 个确认

<details><summary>答案</summary>C。"所有副本"不准确，是 ISR 中的副本。</details>

**Q3** 能严格保证消息"恰好一次"端到端的是？
A. 幂等 Producer 就够了
B. 事务 Producer + `read_committed` + 下游 Kafka 或幂等系统
C. 消费者手动 commit 就行
D. `acks=all` 就够了

<details><summary>答案</summary>B。</details>

**Q4** 下列哪项**不是** Kafka 快的原因？
A. 顺序写
B. Page Cache
C. 零拷贝
D. 强一致同步写磁盘

<details><summary>答案</summary>D。</details>

**Q5** 下列能保证同一订单号消息顺序的是？
A. 消费者加锁
B. 用订单号作为 Key，Kafka 默认按 Key 哈希到同一分区
C. 用 `acks=all`
D. 开启事务

<details><summary>答案</summary>B。</details>

**Q6** `min.insync.replicas=2`、`replication.factor=3` 的 Broker 集群，挂 2 台会怎样？
A. 消费者继续消费
B. 生产者 `acks=all` 时报 `NotEnoughReplicasException`
C. 集群完全不可用
D. Kafka 自动降级为 `acks=1`

<details><summary>答案</summary>B。生产会拒写以保一致；消费可继续。</details>

**Q7** 关于 Rebalance，正确的是？
A. 新消费者加入时触发
B. 分区数增加时触发
C. 组成员心跳超时时触发
D. 以上都是

<details><summary>答案</summary>D。</details>

**Q8** 对比 Kafka 与 RabbitMQ，错误的是？
A. Kafka 吞吐更高
B. RabbitMQ 路由更灵活
C. Kafka 消费者天然可以多次重放消息
D. RabbitMQ 的 Topic 模型和 Kafka 完全相同

<details><summary>答案</summary>D。</details>

### 18.2 判断题

1. 消息队列可以完全解决分布式事务问题。 ❌（只是最终一致工具）
2. Kafka 分区数可以随意增加和减少。 ❌（只能加）
3. 幂等 Producer 能解决跨分区原子写。 ❌（那是事务）
4. 同一消费组内增加消费者数量总能提升吞吐。 ❌（受分区数限制）
5. `auto.offset.reset=earliest` 意味着每次启动都从头消费。 ❌（只是没有 committed offset 时的策略）
6. Kafka 依赖 ZooKeeper 是强制的。 ❌（3.3+ 可用 KRaft）
7. 开启压缩能同时减少网络和磁盘占用。 ✅
8. 死信队列可以避免毒消息阻塞主队列。 ✅

### 18.3 简答题

**Q1** Kafka 的 ISR 机制是什么？为什么重要？

ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。只有 ISR 里的副本才有资格成为新 Leader，保证了**不丢数据的同时提供快速切换**。`acks=all` 等待的是 ISR（且不少于 `min.insync.replicas`）副本确认。

**Q2** 说说消息堆积的应对策略。

- 扩容消费者到分区数上限
- 增加分区并同步扩消费者
- 消费者并发（线程池）
- 批量处理、降级非关键逻辑
- 紧急把积压转移到临时 topic 专用消费
- 长期：监控 Lag + 合理容量规划

**Q3** 如何在 Kafka 中实现延时消息？

原生不支持，常见方案：

- 分级 topic（5s/30s/1min/5min/...），到时被专用消费者转发
- 外部调度（Redis ZSet / 定时任务）时间到再投到 Kafka
- 业务量大推荐用 RocketMQ 或 Pulsar 的原生延时

**Q4** 生产者如何保证不丢消息？

- `acks=all` + `min.insync.replicas≥2` + `replication.factor≥3`
- 开启 `enable.idempotence=true`
- `retries=Integer.MAX_VALUE` + 合理退避
- 同步发送或监听 callback，失败不忽略
- 配合 **Outbox Pattern** 保证业务 DB 与消息原子

**Q5** 消费者如何保证不丢消息？

- 关闭 `enable.auto.commit`
- **先处理后提交** offset
- 处理失败不提交，下次重新拉取
- 业务幂等防重复
- 处理时间 < `max.poll.interval.ms`

### 18.4 场景设计题

**Q1** 设计一个"订单 → 库存扣减 → 发货通知"的事件驱动系统。

```
订单服务 (DB tx: orders + outbox)
     │
     ▼ outbox publisher / Debezium
Kafka topic: orders.created   (partition by order_id, replication 3)
     │
     ├─ 库存服务消费（group: inventory）
     │    幂等：唯一键 order_id，已扣则跳过
     │    成功/失败 → Kafka topic: inventory.result
     │
     └─ 通知服务消费（group: notification）
          按地址库/渠道路由
```

- EOS 可启用但对写外部 DB 仍需业务幂等
- 失败进入 DLT（`orders.created.DLT`），人工/定时重处理
- 全链路带 `trace_id` 便于排查

**Q2** 某 Kafka 集群 3 台 Broker，突然一台磁盘坏了，请说出应对步骤。

1. 告警：监控 `UnderReplicatedPartitions` 飙升
2. 判断：故障 Broker 上只有 Leader/Follower 缺失，ISR 降级但服务继续
3. 用 `kafka-topics --describe` 确认受影响分区
4. 在坏节点下线（`kafka-server-stop`），Controller 会迁移 Leader
5. 恢复硬件后重启 Broker，Follower 自动追赶 ISR
6. 观察 ISR 恢复、Lag 清零
7. 复盘：副本放置、机架感知、容量、告警阈值

**Q3** 生产发现某 topic 消费 lag 持续增长，诊断思路？

- `consumer-groups --describe` 看哪个分区 lag 高
- 查消费者实例的处理耗时 / GC / 线程池
- 下游依赖（DB/HTTP）是否慢
- 消息是否变大 / 变多（生产端）
- 是否发生 Rebalance 频繁
- 分区数是否够
- 处置：扩消费者、扩分区、并发处理、临时降级

**Q4** 生产环境发现消息重复消费严重，可能的原因？

- 自动提交 + 处理失败导致 offset 已提交但业务未完成（应该手动提交）
- Rebalance 频繁：`max.poll.interval.ms` 太小或处理超时
- Producer 重试 + 未开启幂等
- 业务没做幂等
- offset 提交滞后于处理（先提交后处理的反模式）

修复：

- 手动提交 offset 放在处理成功后
- 调大 `max.poll.interval.ms`，缩小每次 poll 数量
- Producer 启用 `enable.idempotence`
- 消费端引入幂等表 / 唯一索引

---

## 📚 学习建议

1. **先搞懂分区与副本**：Kafka 90% 的概念围绕它们
2. **亲手搭一个本地集群**：`docker-compose` 起 3 个 Broker + KRaft，写 demo 感受参数，使用一门编程语言连接Kafka，进行消息的读写。
3. **读官方文档**：`kafka.apache.org/documentation/` 是权威源头
4. **看源码 / KIP**：每个新特性都有 KIP（Kafka Improvement Proposal），思路清晰
5. **监控先行**：Lag / ISR / 请求延迟的 Grafana 面板装好再上生产
6. **熟读 Confluent 博客**：大量最佳实践、踩坑分享

> 祝你的消息不丢、不重、不乱、不卡。
