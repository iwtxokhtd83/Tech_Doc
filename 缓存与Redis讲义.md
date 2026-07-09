# 缓存与 Redis 讲义

> 本讲义按两条主线组织：**缓存通用原理与工程实践** 与 **Redis 深度使用**（数据结构、持久化、复制、集群、线程模型、常见问题）。每章配"知识点 + 笔试题"。
>
> 约定：示例以 **Redis 7.x** 为主；命令大小写不敏感但本讲义按惯例用大写；默认端口 6379。

## 目录

1. [为什么需要缓存](#1-为什么需要缓存)
2. [缓存层级与位置](#2-缓存层级与位置)
3. [缓存模式与更新策略](#3-缓存模式与更新策略)
4. [淘汰策略与过期机制](#4-淘汰策略与过期机制)
5. [缓存三大经典问题](#5-缓存三大经典问题)
6. [缓存一致性](#6-缓存一致性)
7. [Redis 基础与数据结构](#7-redis-基础与数据结构)
8. [Key 与过期管理](#8-key-与过期管理)
9. [事务、Pipeline 与 Lua 脚本](#9-事务pipeline-与-lua-脚本)
10. [发布订阅、Stream 与消息](#10-发布订阅stream-与消息)
11. [持久化：RDB 与 AOF](#11-持久化rdb-与-aof)
12. [复制、哨兵与集群](#12-复制哨兵与集群)
13. [线程模型与性能](#13-线程模型与性能)
14. [内存与淘汰](#14-内存与淘汰)
15. [分布式锁与限流](#15-分布式锁与限流)
16. [常见业务模式](#16-常见业务模式)
17. [运维、监控与安全](#17-运维监控与安全)
18. [常见问题与性能调优](#18-常见问题与性能调优)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 为什么需要缓存

### 1.1 核心价值

- **降低延迟**：内存访问纳秒级，数据库查询毫秒~秒级
- **削峰与保护后端**：DB 扛不住的流量让缓存顶
- **降成本**：少建一台数据库实例，多台缓存实例
- **提升吞吐**：单 Redis 节点轻松 10w+ QPS

### 1.2 缓存适用场景

- **读多写少**的热点数据（商品详情、用户资料）
- **结果昂贵**的计算（聚合报表、推荐）
- **会话/令牌**（Session、Token、验证码）
- **去重/限流/计数**
- **排行榜、最新列表**
- **分布式锁、幂等**

### 1.3 不适合缓存的场景

- **强一致性**要求（交易结果、余额变动）
- **写极多**：命中率低，反增 DB 压力
- **数据体量过大**且访问分布均匀，没有"热点"

### 📝 笔试题 1-1：为什么缓存一般命中率要求 > 90%？

缓存 miss 时要回源 DB，若命中率太低：

- 流量放大（每次 miss 既查缓存又查 DB）
- 回源压力把 DB 打挂
- 缓存反而成**负资产**

通常 90%+ 才能显著减轻 DB；冷启动或 miss storm 要专门处理。

---

## 2. 缓存层级与位置

### 2.1 经典多层缓存

```
浏览器缓存
  │
CDN（边缘）
  │
网关/Nginx 缓存
  │
应用内存（本地缓存）
  │
分布式缓存（Redis/Memcached）
  │
DB 自身缓存（InnoDB Buffer Pool、query cache）
  │
DB
```

### 2.2 本地缓存 vs 分布式缓存

| 维度 | 本地缓存 | 分布式缓存 |
|------|----------|------------|
| 访问速度 | 极快（内存） | 快（网络 + 内存） |
| 容量 | 受单机内存限制 | 可横向扩展 |
| 一致性 | 多实例间难同步 | 集中管理 |
| 故障影响 | 单机 | 集群风险但可隔离 |
| 代表 | Caffeine / Guava / LRUCache | Redis / Memcached |

生产常组合：**本地缓存 + 分布式缓存（L1 + L2）**，热 key 命中本地，miss 回 Redis，再回源 DB。

### 2.3 一致性哈希

分布式缓存横向扩展的关键。普通取模在扩缩容时命中率骤降，**一致性哈希**只影响环上局部：

- 节点 + 数据 均哈希到 `[0, 2³²)` 环
- 顺时针找到第一个节点
- 加虚拟节点缓解数据倾斜

### 📝 笔试题 2-1：CDN 和 Redis 解决的问题有何不同？

- **CDN**：**边缘缓存静态资源**（HTML/JS/CSS/图片/视频），靠近用户降延迟、节省中心带宽
- **Redis**：**中心化数据缓存**，支撑动态查询、业务状态、锁/队列等

前者是**内容分发**视角，后者是**数据访问**视角。两者常共存。

---

## 3. 缓存模式与更新策略

### 3.1 读写模式

#### Cache-Aside（旁路缓存，最常用）

```
读：
  1. 查缓存；命中即返回
  2. miss 时查 DB；写回缓存；返回
写：
  1. 写 DB
  2. 删除缓存（不是更新）
```

**优点**：简单通用。
**缺点**：首次访问 miss；写后读一致性有窗口。

#### Read-Through / Write-Through

缓存层统一封装 DB 访问：

- **Read-Through**：读穿透——缓存层自己回源 DB
- **Write-Through**：写穿透——写时同步更新 DB + 缓存

**优点**：业务代码简单。
**缺点**：需要 SDK/中间层支持；写延迟更高。

#### Write-Behind（回写）

- 写到缓存立即返回，后台批量异步落 DB

**优点**：写极快，高吞吐。
**缺点**：**掉电/崩溃可能丢数据**；一致性复杂。适合对丢失容忍度高的场景（日志、统计）。

### 3.2 为什么写时"删"而不是"更新"缓存？

- **并发**：两线程同时写 DB 与缓存，顺序不定，可能得到旧值
- **复杂值**：缓存可能是计算结果（如聚合），不是单行
- **懒加载**：下次读时按需重建，天然避免无效缓存
- 经验：**写 DB + 删除缓存** 是"最少错"的默认选择

### 3.3 先删缓存还是先写 DB？

两种方案都有窗口可能读到旧数据：

**方案 A（Cache-Aside 标准）：先写 DB，再删缓存**

- 风险：删除失败 → 缓存永远脏
- 缓解：引入**重试**（MQ / binlog）直到删成功

**方案 B：先删缓存，再写 DB**

- 风险：删后到写 DB 之间有读请求，把**旧值**回种到缓存
- 缓解：**延迟双删**
  ```
  删缓存 → 写 DB → sleep N ms → 再删一次缓存
  ```

**推荐**：方案 A + 消息队列保证删除最终成功；一致性要求极高时结合**订阅 binlog / CDC**。

### 📝 笔试题 3-1：为什么"更新缓存"比"删除缓存"更容易出问题？

并发下两次写操作 A、B 顺序可能为：

```
A 写 DB v1
B 写 DB v2
B 更新缓存 v2
A 更新缓存 v1   ← 最终缓存 v1（旧值），与 DB v2 不一致
```

而"删除"操作是幂等的："删了就是没有"，下次读自然拉到最新值。

---

## 4. 淘汰策略与过期机制

### 4.1 常见淘汰算法

| 算法 | 淘汰对象 |
|------|----------|
| **FIFO** | 最早进入 |
| **LRU** (Least Recently Used) | 最近最少使用 |
| **LFU** (Least Frequently Used) | 最不频繁使用 |
| **ARC** (Adaptive Replacement Cache) | LRU + LFU 自适应 |
| **Random** | 随机 |
| **2Q / W-TinyLFU** | 改进 LFU，更适合热点场景（Caffeine） |

### 4.2 过期（TTL）

- **设置 TTL**：减小脏数据影响、强制刷新
- **加随机抖动**：不同 key 分散过期，避免"同一时刻雪崩"

```
ttl = base + random(jitter)
```

### 4.3 Redis 删除过期 key

- **惰性删除**：访问时发现过期才删
- **定期删除**：后台定时扫描抽样删一些
- 两者配合；内存紧张时靠**淘汰策略**兜底（见第 14 章）

### 📝 笔试题 4-1：只靠惰性删除会有什么问题？

大量**永不再访问**的过期 key 长期占内存，导致内存膨胀。Redis 所以还有定期删除与淘汰策略作为兜底。

---

## 5. 缓存三大经典问题

### 5.1 缓存穿透（Penetration）

**现象**：查询**根本不存在**的数据，缓存每次 miss，请求打到 DB。

**成因**：恶意扫描、参数被篡改、业务 bug。

**对策**：

1. **空值缓存**：对 DB 查出为空的结果也缓存（短 TTL）
2. **布隆过滤器（Bloom Filter）**：预热存在的 key；查询前先问 Bloom：**不在则直接拒绝**
3. **接口参数校验** + 网关限流

### 5.2 缓存击穿（Breakdown）

**现象**：**热点 key 过期瞬间**，大量请求同时 miss，集中回源。

**对策**：

1. **互斥锁重建**：同一 key miss 时只有一个线程回源，其他等待
   ```
   if miss:
     if tryLock(key):
       val = loadDB(); setCache(val); unlock()
     else:
       sleep; retry
   ```
2. **逻辑过期**：缓存里存 `{value, expireAt}`，永不物理过期；后台异步或懒触发重建
3. **热点探测**：对超高频 key 延长 TTL / 本地 L1 缓存

### 5.3 缓存雪崩（Avalanche）

**现象**：**大量 key 同时过期** 或 **缓存宕机**，流量涌向 DB。

**对策**：

1. **TTL 加随机抖动**
2. **多级缓存**：本地 + 分布式
3. **热点预热**：重启或发版前主动灌入
4. **限流、熔断、降级**：DB 前置保护
5. **高可用部署**：Redis Sentinel / Cluster，多副本

### 📝 笔试题 5-1：穿透 vs 击穿 vs 雪崩 区别？

- **穿透**：请求的**数据不存在**，缓存永远 miss
- **击穿**：**热点 key 过期**，高并发同时 miss
- **雪崩**：**大面积 key** 同时失效或缓存不可用

**记忆**：穿透是"无中生有"，击穿是"一个热点点破防线"，雪崩是"群体同时崩溃"。

---

## 6. 缓存一致性

### 6.1 一致性谱系

从强到弱：

```
强一致 → 读写屏障（RWBarrier）→ 读己之写 → 最终一致 → 弱一致
```

大部分缓存系统做**最终一致**：允许短窗口脏读，保证"一段时间后一致"。

### 6.2 常见不一致场景

- 写 DB 成功但**删除缓存失败**
- **先删后写**期间被读旧值回种
- **主从复制延迟**：写主、读从还没同步
- **跨地域多集群**：网络分区

### 6.3 对策

- **删缓存重试**：失败写入 MQ 延迟重试
- **订阅 binlog**（Canal / Debezium）**异步删缓存**，兜底"人肉写漏"
- **延迟双删**
- **版本号 / CAS**：缓存值带版本，读取后校验再用
- **读主**：一致性要求高的读走主 DB（带 read-your-writes 语义）
- **TTL 兜底**：即使偶尔脏，过期后自愈

### 📝 笔试题 6-1：如何保证缓存和 DB 最终一致？

组合拳：

1. 业务代码里**写 DB + 删缓存**（主路径）
2. 删除失败放 **MQ 异步重试**
3. 另一路订阅 **binlog** 做兜底刷/删
4. 给关键 key 设合理 **TTL**
5. 监控不一致率（采样对账）

---

## 7. Redis 基础与数据结构

### 7.1 Redis 是什么

- 基于**内存**的 KV 存储，支持多种数据结构
- 单进程 + **IO 多路复用**（epoll）+ 单线程命令执行（6.0+ 有多线程 IO）
- 支持**持久化**（RDB/AOF）、**复制**、**哨兵**、**集群**、**脚本**、**事务**、**发布订阅**、**流**

### 7.2 常用数据结构概览

| 类型 | 典型命令 | 底层实现 | 场景 |
|------|----------|----------|------|
| **String** | GET/SET/INCR/APPEND | SDS（简单动态字符串） / int | 缓存对象、计数 |
| **List** | LPUSH/RPUSH/LPOP/BRPOP/LRANGE | ziplist (7.x 改 listpack) / quicklist | 队列、最新列表 |
| **Hash** | HSET/HGET/HINCRBY/HGETALL | listpack / hashtable | 对象字段存储 |
| **Set** | SADD/SREM/SMEMBERS/SINTER | intset / hashtable | 去重、交并差 |
| **Sorted Set (ZSet)** | ZADD/ZRANGE/ZRANGEBYSCORE | listpack / skiplist+hashtable | 排行榜、延时队列 |
| **Bitmap** | SETBIT/GETBIT/BITCOUNT/BITOP | String | 签到、活跃标记 |
| **HyperLogLog** | PFADD/PFCOUNT | 稀疏/稠密编码 | 基数估计（UV） |
| **Geo** | GEOADD/GEODIST/GEOSEARCH | 基于 ZSet + GeoHash | 附近的人、LBS |
| **Stream** | XADD/XREAD/XREADGROUP | radix tree | 消息队列 |
| **Bitfield** | BITFIELD | String | 多计数器混存 |

### 7.3 String

```bash
SET user:42 '{"id":42,"name":"Alice"}' EX 3600 NX    # 只在不存在时设
GET user:42
APPEND key " world"
STRLEN key
INCR counter                # 原子自增
INCRBY counter 10
DECR / DECRBY
GETSET key val              # 返回旧值并设新值（推荐 SET ... GET）
SETEX key 60 val            # 带 TTL
MSET k1 v1 k2 v2            # 批量
MGET k1 k2
```

**编码**：短字符串 + 纯数字 → `int` 或 `embstr`；超过阈值 → `raw` SDS。

### 7.4 List

```bash
LPUSH queue a b c           # 左进（后进者在前）
RPUSH queue x y
LRANGE queue 0 -1           # 全部
LPOP queue / RPOP queue
BLPOP queue 10              # 阻塞弹出，10 秒超时
LLEN queue
LREM queue 0 a              # 按值删
LTRIM queue 0 999           # 只保留前 1000 条（裁剪列表）
```

### 7.5 Hash

```bash
HSET user:42 name Alice age 30 email a@x.com
HGET user:42 name
HMGET user:42 name email
HGETALL user:42
HINCRBY user:42 age 1
HDEL user:42 email
HEXISTS user:42 name
HLEN user:42
```

**何时用 Hash 而非多个 String**：同一对象的若干字段集中存，内存更紧凑，管理更便利。

### 7.6 Set

```bash
SADD tags redis nosql cache
SMEMBERS tags
SISMEMBER tags redis
SREM tags cache
SCARD tags                  # 元素数
SINTER a b                  # 交集（共同好友）
SUNION a b                  # 并集
SDIFF a b                   # 差集
SRANDMEMBER tags 2          # 随机 N 个
SPOP tags                   # 弹出随机一个
```

### 7.7 ZSet

```bash
ZADD rank 100 alice 90 bob 85 carol
ZRANGE rank 0 -1 WITHSCORES              # 升序
ZREVRANGE rank 0 9 WITHSCORES            # Top 10 降序（7.x 推 ZRANGE ... REV）
ZRANGEBYSCORE rank 90 100
ZINCRBY rank 5 bob                       # bob +5
ZRANK / ZREVRANK rank alice
ZSCORE rank alice
ZREM rank carol
ZCOUNT rank 80 100
```

**场景**：排行榜、延时队列（score 为执行时间戳，定时 `ZRANGEBYSCORE 0 now`）。

### 7.8 Bitmap

```bash
SETBIT sign:u42:2025-01 0 1       # 第 1 天签到
GETBIT sign:u42:2025-01 0
BITCOUNT sign:u42:2025-01
BITOP AND res a b                  # 多个 Bitmap 与运算
```

1 亿用户签到只需约 12MB。

### 7.9 HyperLogLog

基数估计，极省内存（12KB），误差 ~0.81%：

```bash
PFADD uv:2025-01-15 user1 user2 user3
PFCOUNT uv:2025-01-15
PFMERGE uv:2025-W3 uv:2025-01-13 uv:2025-01-14 ...
```

**适合**：UV 统计、去重计数；**不适合**精确值与列出成员。

### 7.10 Geo

```bash
GEOADD shop 121.47 31.23 shanghai 116.40 39.90 beijing
GEODIST shop shanghai beijing km
GEOSEARCH shop FROMLONLAT 121.5 31.2 BYRADIUS 200 km ASC COUNT 10
```

### 📝 笔试题 7-1：大对象属性更新，用 String 存 JSON 还是 Hash？

**优先 Hash**：

- 局部更新：`HSET user:1 age 31`，只动一个字段
- 网络传输少
- 内存编码更紧凑（小对象走 listpack）
- JSON 要读-改-写整体覆盖

缺点：Hash 值只能是字符串，嵌套结构仍要序列化。极复杂可用 **RedisJSON** 模块。

---

## 8. Key 与过期管理

### 8.1 命名规范

```
对象:ID:字段
user:42:profile
order:2025:00010023
session:<uuid>
cache:api:v1:<hash>
```

**要点**：分层清晰、可读、可 MATCH、环境前缀（`dev:` / `prd:`）。

### 8.2 TTL 操作

```bash
EXPIRE key 3600              # 秒
PEXPIRE key 60000            # 毫秒
EXPIREAT key 1700000000      # Unix 时间戳
TTL key                      # 剩余秒；-1 无期；-2 不存在
PERSIST key                  # 取消过期
```

### 8.3 避免危险命令

| 命令 | 风险 |
|------|------|
| `KEYS *` | **阻塞整个 Redis**，生产禁用 |
| `FLUSHALL` / `FLUSHDB` | 清空数据 |
| `SAVE` | 同步 RDB，阻塞 |
| `CONFIG SET` | 改动可能影响全局 |
| 大 Hash/Set 的 `HGETALL`/`SMEMBERS` | 一次返回过大 |

**生产建议**：在配置/ACL 中禁用或重命名这些命令。

### 8.4 扫描代替遍历

```bash
SCAN 0 MATCH user:* COUNT 1000
HSCAN user:42 0 COUNT 100
SSCAN tags 0
ZSCAN rank 0
```

`SCAN` 用游标分批扫，**不会阻塞**；`COUNT` 只是提示量，非严格。

### 8.5 UNLINK 与 DEL

`DEL big_key` 可能阻塞。用 `UNLINK big_key` 异步释放。

### 📝 笔试题 8-1：想定位当前哪些 key 即将过期，能用 `KEYS` 吗？

不能，线上禁用。用：

```bash
redis-cli --scan --pattern "user:*" | xargs -I{} redis-cli TTL {}
```

结合 SCAN 分批遍历，尽量避开高峰。

---

## 9. 事务、Pipeline 与 Lua 脚本

### 9.1 事务 MULTI/EXEC

```bash
MULTI
SET a 1
INCR b
EXEC
```

- `MULTI` 开启 → 命令进入队列 → `EXEC` 原子执行
- **不是数据库事务**：遇到运行时错误**不回滚**，继续执行其他命令
- 可配合 `WATCH` 做乐观锁

```bash
WATCH stock
val = GET stock
if val > 0:
  MULTI
  DECR stock
  EXEC        # 若 stock 期间被别人改过 → EXEC 返回 nil
```

### 9.2 Pipeline

客户端把**多条命令打包发送**，服务端依次执行后一次返回，大幅减少 RTT：

```python
# Python redis-py
pipe = r.pipeline(transaction=False)
for uid in user_ids:
    pipe.hgetall(f"user:{uid}")
res = pipe.execute()     # 一次网络往返
```

Pipeline ≠ 事务；`transaction=False` 可完全关闭 MULTI 语义，纯加速。

### 9.3 Lua 脚本

服务端原子执行：

```bash
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock mytoken
```

- 一次往返；**期间不会被其他命令打断**
- 适合做 check-and-set、分布式锁释放、复杂原子逻辑
- 脚本可 `SCRIPT LOAD` 得到 SHA，后续 `EVALSHA` 节省带宽

### 9.4 脚本注意事项

- 脚本里**不要写慢命令**（阻塞整个实例）
- 键必须通过 `KEYS[]` 声明，**不要**硬编码（Cluster 需要路由）
- 避免不确定性函数（`TIME` 用前传入）；Redis 7 支持 `FUNCTION`（更好的脚本管理）

### 📝 笔试题 9-1：Pipeline 和事务的区别？

- **Pipeline**：客户端优化技术，批量发送减少 RTT，**不保证原子性**
- **事务 (MULTI/EXEC)**：服务端把命令排队并**原子执行**（不被别的客户端插队），但遇到运行时错误不回滚

实际生产为性能而用 Pipeline；为原子性用 **Lua 脚本**（比 MULTI 更强）。

---

## 10. 发布订阅、Stream 与消息

### 10.1 Pub/Sub

```bash
# 订阅
SUBSCRIBE news
PSUBSCRIBE news.*

# 发布
PUBLISH news "hello"
```

- **实时广播**，消息不持久化
- 订阅者**下线期间的消息丢失**
- 集群模式下 `PUBLISH` 在所有节点广播

**不适合**：可靠消息队列。**适合**：实时通知（广播配置变更、踢人）。

### 10.2 Stream（5.0+）

更现代的消息流：

```bash
# 生产
XADD orders * user 42 amount 100        # * 自动生成 id（timestamp-seq）

# 消费（非消费者组）
XRANGE orders - +
XREAD COUNT 10 BLOCK 5000 STREAMS orders $

# 消费者组
XGROUP CREATE orders g1 0
XREADGROUP GROUP g1 consumer1 COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders g1 <id>
XPENDING orders g1                      # 未 ACK 的消息
XCLAIM                                  # 认领超时的消息
```

**特性**：

- 持久化、可回溯（消息存在流中）
- 消费者组：多消费者分担、各自游标
- ACK + 超时重投机制

### 10.3 与专业 MQ 的对比

| | Redis Pub/Sub | Redis Stream | Kafka / RabbitMQ |
|-|---------------|--------------|-------------------|
| 持久化 | ❌ | ✅ | ✅ |
| 重消费 | ❌ | ✅ | ✅ |
| 分区 | ❌ | 手动 | 原生 |
| 吞吐 | 中 | 中高 | 极高 |
| 复杂度 | 低 | 中 | 高 |

Redis Stream 适合**中等规模、低运维成本**的消息场景；超大规模仍选 Kafka。

### 📝 笔试题 10-1：Redis Pub/Sub 可以做订单通知吗？

**不建议**。订阅者离线消息即丢失，业务关键通知不能依赖它。用 **Stream**、**List + BRPOP**（简单队列）或独立 MQ（Kafka/RabbitMQ）。

---

## 11. 持久化：RDB 与 AOF

### 11.1 RDB（快照）

- 在指定条件下 fork 子进程把内存**快照**写到 `dump.rdb`
- 文件紧凑、恢复快；**丢失**上次快照以来的数据

```
save 3600 1     # 3600 秒有 1 次以上变更就 bgsave
save 300 100
save 60 10000
```

命令：

```bash
BGSAVE          # 后台快照
SAVE            # 同步，阻塞，生产禁用
LASTSAVE        # 上次成功时间戳
```

### 11.2 AOF（追加日志）

- 每条写命令追加到日志（`appendonly.aof`）
- 恢复时"重放"；比 RDB 更安全但文件大

```
appendonly yes
appendfsync everysec     # 每秒 fsync（推荐）
# always: 每条都 fsync（最安全，最慢）
# no:     交给 OS（最快，最不安全）
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**AOF 重写（BGREWRITEAOF）**：定期压缩日志为等价的最小命令集。

### 11.3 混合持久化（4.0+，推荐）

```
aof-use-rdb-preamble yes
```

AOF 文件开头是 RDB 快照，之后是增量 AOF。兼顾**恢复速度**（RDB 快）与**数据完整性**（AOF 详）。

### 11.4 选择策略

| 场景 | 建议 |
|------|------|
| 可接受几分钟丢失 | RDB |
| 数据不能丢 | AOF everysec |
| 想两者兼得 | 混合持久化 |
| 用作纯缓存 | 可关闭持久化（加快重启） |

### 11.5 注意事项

- **fork 代价**：内存很大时 fork 会阻塞毫秒~秒级
- **COW 内存**：持久化期间内存占用可能翻倍
- **磁盘抖动**：高写入 + AOF always 可能成为瓶颈
- **恢复时间**：AOF 大时启动慢，混合模式更优

### 📝 笔试题 11-1：为什么 RDB fork 子进程？性能影响是？

利用 **写时复制（COW）**：父子共享物理页，父继续响应请求；父写某页时内核复制一份新页。

影响：

- **fork 本身** 需要复制页表，内存越大越慢（大实例可达几百 ms）
- **持久化期间内存翻倍风险**（极端写入多时）
- 临界业务考虑做**只读从节点** BGSAVE，不在主节点

---

## 12. 复制、哨兵与集群

### 12.1 主从复制

```
# 从节点
replicaof 10.0.0.1 6379     # (或 slaveof)
```

- 异步复制：**主写→ Binlog → 从异步拉取**
- 从节点默认只读（`replica-read-only yes`）
- 支持级联：主 → 从 → 从的从

**全量同步**：初次或偏移量丢失 → RDB 传输 + 后续增量
**部分同步**：基于 `replication_backlog` 环形缓冲区

### 12.2 哨兵（Sentinel）

用于**主从自动故障切换**：

- 3 个以上哨兵投票检测主挂
- 选举一个从升级为主
- 客户端通过 Sentinel 发现当前主

**架构**：1 主 + N 从 + 3 哨兵。最小可用：1 主 1 从 3 哨。

### 12.3 Redis Cluster（3.0+）

- 数据分片：**16384 个 slot**，hash(key) mod 16384 → slot → 节点
- 每个节点管一部分 slot
- 内置主从 + 自动故障转移
- **无中心代理**，客户端感知拓扑（`MOVED` 重定向）

#### 命令限制

- 跨 slot 操作默认不允许（如 `MSET k1 k2` 若落不同 slot）
- **Hash Tag**：用 `{tag}` 把一组 key 放到同一 slot
  ```
  user:{42}:profile
  user:{42}:orders
  ```

#### 集群命令

```bash
CLUSTER NODES
CLUSTER INFO
CLUSTER KEYSLOT foo
CLUSTER COUNTKEYSINSLOT 7000
CLUSTER GETKEYSINSLOT 7000 10
```

### 12.4 架构选型

| 架构 | 容量 | 可用 | 场景 |
|------|------|------|------|
| 单机 | 小 | 低 | 开发测试 |
| 主从 + Sentinel | 受限于单机 | 高（自动切换） | 中小规模 |
| Cluster | 水平扩展 | 高 | 大规模 |
| 代理型（Codis / Twemproxy） | 扩展 | 看代理 | 老项目兼容 |
| 云 Redis（ElastiCache / Cloud Memorystore / 腾讯云） | 托管 | 高 | 免运维 |

### 12.5 读写分离

主写从读可以提升读吞吐，但：

- **复制延迟** → 可能读到旧值
- 需要"**读己之写**"走主
- 客户端/代理要支持

一般推荐**强一致路径走主**，**能容忍滞后路径走从**。

### 📝 笔试题 12-1：Redis Cluster 为什么是 16384 个 slot？

作者 antirez 解释：

- 心跳包中节点之间要交换**每节点负责的 slot bitmap**
- 16384 / 8 = 2KB，适合每节点数千的集群；若 65536 就 8KB，心跳太大
- 16384 对大部分集群规模（数千节点）已足够细的粒度

### 📝 笔试题 12-2：`MOVED` 与 `ASK` 的区别？

- **MOVED**：该 slot**已归属**新节点，客户端应更新本地路由表
- **ASK**：slot **正在迁移**过程，临时转发一次，不更新路由表

智能客户端同时处理这两种重定向。

---

## 13. 线程模型与性能

### 13.1 单线程？多线程？

- **命令执行仍是单线程**：保证原子性，简化数据结构
- **IO 多路复用**：`epoll` 一个线程处理成千上万连接
- **6.0+ IO 多线程**：网络读写/协议解析由多个 IO 线程并行；**命令执行仍串行**

```
io-threads 4
io-threads-do-reads yes
```

### 13.2 为什么单线程也能这么快

- 纯**内存**操作
- 精心设计的数据结构（SDS / skiplist / listpack / hashtable 自适应）
- 非阻塞 IO
- 无锁，无 CPU 上下文切换开销

### 13.3 慢命令警戒

| 危险 | 替代 |
|------|------|
| `KEYS *` | `SCAN` |
| `HGETALL huge_hash` | `HSCAN` + 分批 |
| `SMEMBERS huge_set` | `SSCAN` |
| 大 `LRANGE 0 -1` | `LRANGE` + 分页 |
| 大 `DEL big_key` | `UNLINK` |
| 复杂 Lua 脚本 | 拆小 |
| `SORT` 未带限制 | 避免 |

### 13.4 基准测试

```bash
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 1000000 -t get,set,incr
redis-benchmark -q -P 16 -t set      # -P 16 管道
```

单实例典型 QPS：10 万级别；配合 Pipeline 可到 50 万+。

### 📝 笔试题 13-1：Redis "是单线程" 的说法准确吗？

**不完全准确**。主事件循环（命令执行）单线程，但 Redis 还使用多个后台线程处理：

- **BIO**：关闭文件描述符、AOF fsync、懒释放（UNLINK/FLUSH ASYNC）
- **IO 多线程**（6.0+）：网络读写并行
- **子进程**：持久化（BGSAVE / BGREWRITEAOF）

应该说"**核心命令执行单线程**"。

---

## 14. 内存与淘汰

### 14.1 设置内存上限

```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

### 14.2 淘汰策略

| 策略 | 作用范围 | 选择依据 |
|------|----------|----------|
| `noeviction`（默认） | — | 满了就**报错**，禁止写 |
| `allkeys-lru` | 全部 key | LRU |
| `allkeys-lfu` | 全部 key | LFU（4.0+，更贴合热点）|
| `allkeys-random` | 全部 key | 随机 |
| `volatile-lru` | 有 TTL 的 key | LRU |
| `volatile-lfu` | 有 TTL 的 key | LFU |
| `volatile-random` | 有 TTL 的 key | 随机 |
| `volatile-ttl` | 有 TTL 的 key | TTL 最近 |

**选择建议**：

- 纯缓存：`allkeys-lru` 或 `allkeys-lfu`
- 带业务 TTL：`volatile-lru`
- 不容忍丢数据：`noeviction`（配合监控容量）

### 14.3 内存占用分析

```bash
INFO memory
MEMORY USAGE key                 # 某 key 占用估算
MEMORY STATS                     # 详细分解
redis-cli --bigkeys              # 扫描各类型最大 key
redis-cli --memkeys              # 按内存最大 top
redis-cli --hotkeys              # 热点 key（需 LFU）
```

### 14.4 内存优化

- 小 Hash/List/ZSet 用紧凑编码（listpack）；突破阈值后转 hashtable/skiplist
  ```
  hash-max-listpack-entries 128
  hash-max-listpack-value   64
  ```
- **短 key**：`u:42` 比 `user:42` 省
- **合适类型**：计数用 String+INCR；大量 bool 用 Bitmap；基数估计用 HyperLogLog
- **压缩对象**：业务可接受时用 MsgPack/Protobuf 再存

### 📝 笔试题 14-1：Redis 用了内存淘汰策略，为什么还是 OOM？

- 设了 `noeviction`：直接拒写
- 大 key 单次分配超过剩余内存
- 数据写入速度 >> 淘汰速度
- **内存碎片率**高（`mem_fragmentation_ratio` > 1.5）导致实际可用内存少
- 内核 OOM Killer 杀进程（`dmesg` 可见）

处理：换淘汰策略、拆大 key、增加内存、启用 `activedefrag`。

---

## 15. 分布式锁与限流

### 15.1 分布式锁：基本方案

```bash
SET lock:order:100 <uuid> NX PX 30000
```

- `NX`：不存在才设；`PX`：毫秒 TTL
- 释放锁必须校验 uuid，用 **Lua 脚本原子**：

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

### 15.2 业务侧要点

- **超时时长**：覆盖业务最长处理时间 + 缓冲；避免锁被误释放
- **续约**：长任务需后台 watchdog 定期 `PEXPIRE`（Redisson 自带）
- **可重入**：用 Hash 记录 `holder → count`
- **公平锁**：排队；用 Pub/Sub 通知
- **避免单点**：`SET NX` 在主从 + Sentinel 下可能有一致性问题（主挂但锁尚未复制到从）→ **RedLock**

### 15.3 RedLock 算法（争议）

- N 个独立 Redis 实例（通常 5 个），客户端串行加锁，超过半数成功且耗时 < TTL 即视为成功
- 学界对其在时钟漂移、GC Pause 等异常下的正确性有质疑
- **一般业务** 单实例 + Redisson 的续约已经够用；**强一致**用 Zookeeper/etcd

### 15.4 限流模式

#### 固定窗口（Fixed Window）

```bash
key = "rl:user:42:" + floor(now / 60)
INCR key
EXPIRE key 60
if count > 100: reject
```

简单，但窗口边界存在"双倍流量"问题。

#### 滑动窗口（Sliding Log / Sliding Counter）

用 ZSet 存请求时间戳：

```bash
ZADD rl:u:42 <now> <uuid>
ZREMRANGEBYSCORE rl:u:42 0 now-60000
ZCARD rl:u:42
```

更精准，但内存占用随请求数增长。

#### 令牌桶 / 漏桶（推荐）

Lua 实现：

```lua
-- KEYS[1]=key, ARGV=capacity,rate,now,requested
-- 返回 1 表示通过，0 表示拒绝
```

许多库（`redis-cell` 模块、Redisson、Sentinel）内置。

#### 漏斗限流（redis-cell 模块）

```bash
CL.THROTTLE user:42 15 30 60 1        # 突发 15，60 秒 30 个
```

### 📝 笔试题 15-1：用 `SETNX` + `EXPIRE` 两条命令实现锁的坑？

```bash
SETNX lock value    # 成功
# ... 此时进程崩溃，没来得及 EXPIRE
```

锁永远不过期，后续业务被卡住。**正确**：`SET key val NX PX 30000` 原子完成；释放锁用 Lua 校验 value 再 DEL。

---

## 16. 常见业务模式

### 16.1 排行榜

```bash
ZADD rank 100 alice
ZINCRBY rank 5 alice
ZREVRANGE rank 0 9 WITHSCORES           # Top10
ZREVRANK rank alice                     # 某人名次
```

带时间衰减：score 定期乘衰减系数，或用 Lua 定期重算。

### 16.2 延时队列

```bash
ZADD delay_q <execute_at_ms> <task_id>
# 定时 worker:
ids = ZRANGEBYSCORE delay_q 0 now LIMIT 0 100
ZREM delay_q <id>
```

对 `ZREM` + 业务执行用 **Lua 原子化**，防重复消费。

### 16.3 限速接口（滑窗 ZSet 版）

```
key = "rl:api:/login:" + userId
ZREMRANGEBYSCORE key 0 now - 60000
count = ZCARD key
if count >= 10: reject
ZADD key now <uuid>
EXPIRE key 60
```

### 16.4 Feed 流（时间线）

- **推模式**：发布时写入每个粉丝 List（`LPUSH feed:<uid>`）→ 读快；粉丝多时写放大大
- **拉模式**：读时聚合所有关注者最新帖 → 写轻；读慢
- **推拉结合**：大 V 拉、普通用户推

### 16.5 计数器

- 访问量：`INCR` / `INCRBY`，结合 TTL 或周期落库
- UV：HyperLogLog 或 Set（精确小规模）
- 多维度：`HINCRBY stats:2025-01-15 pv 1`

### 16.6 缓存预热

- 启动时批量 MSET / Pipeline 加载热点 key
- 发版/促销前用脚本遍历热点 key 并加载
- 避免穿透：空结果也缓存短 TTL

### 📝 笔试题 16-1：用 Redis 实现"用户最近浏览商品（最多 100 件，去重）"？

```
LPUSH seen:<uid> <productId>
LREM  seen:<uid> 0 <productId>       # 先去重
LPUSH seen:<uid> <productId>
LTRIM seen:<uid> 0 99
```

原子性用 Lua 脚本包住。也可用 ZSet + 时间戳作 score，自然有序去重。

---

## 17. 运维、监控与安全

### 17.1 INFO 与关键指标

```bash
INFO server
INFO clients
INFO memory
INFO stats
INFO replication
INFO commandstats
INFO latency
```

**核心指标**：

- QPS / 命中率 (`keyspace_hits / (hits + misses)`)
- 内存 / 碎片率
- 连接数 / 拒绝连接数
- 主从延迟 (`master_repl_offset - slave_repl_offset`)
- 慢日志 (`SLOWLOG GET 100`)
- 阻塞与大 key

### 17.2 慢查询日志

```bash
CONFIG SET slowlog-log-slower-than 10000      # 微秒
CONFIG SET slowlog-max-len 128
SLOWLOG GET 10
SLOWLOG RESET
```

### 17.3 Latency 监控

```bash
CONFIG SET latency-monitor-threshold 100      # 毫秒
LATENCY LATEST
LATENCY HISTORY event
LATENCY DOCTOR
```

### 17.4 常用工具

- **redis-cli** 内置：`--scan`、`--bigkeys`、`--hotkeys`、`--latency`、`--stat`
- **RedisInsight**：官方可视化
- **Prometheus + redis_exporter**：指标采集
- **Grafana 模板**：社区成熟面板
- **Arthas/Jmeter/ Locust**：压测

### 17.5 安全

- **绑定地址**：`bind 127.0.0.1 10.0.0.5`（非 0.0.0.0）
- **密码**：`requirepass`；ACL（6.0+）按用户细粒度权限
- **禁用命令**：`rename-command FLUSHALL ""` / `rename-command KEYS ""`
- **TLS**：`tls-port 6379`（6.0+）
- **防火墙/安全组**：最小开放
- **不暴露公网**：很多数据泄露事件源于公网无密码 Redis

### 17.6 ACL 示例（6.0+）

```
ACL SETUSER alice on >pa55word ~cache:* +get +set +mget +mset
```

- `on`：启用；`>`：密码；`~`：key 模式；`+/-`：允许/禁止命令

### 📝 笔试题 17-1：线上 Redis 突然变慢，排查思路？

1. `SLOWLOG GET 100`：找慢命令
2. `INFO commandstats`：哪些命令耗时高
3. `redis-cli --bigkeys`：大 key
4. `LATENCY DOCTOR`：系统建议
5. 持久化：是否正在 BGSAVE/AOF 重写（`rdb_bgsave_in_progress`）
6. 内存：碎片率、是否接近 maxmemory
7. 系统：CPU、IO、Swap（Redis 最怕 swap）
8. 网络：客户端数、连接异常、带宽
9. 复制：主从同步延迟

---

## 18. 常见问题与性能调优

### 18.1 大 key

- 定义：单 key 很大（String > 10KB，集合元素 > 5000）
- 危害：网络/命令阻塞、持久化慢、迁移慢、内存碎片
- 处理：
  - **拆分**（大 Hash → 多个小 Hash）
  - `UNLINK` 而非 `DEL`
  - 定期扫描发现（`--bigkeys`）

### 18.2 热 key

- 单 key QPS 过高，单节点瓶颈
- 发现：`--hotkeys`（需 LFU）、监控旁路采样
- 处理：
  - 客户端**本地缓存**
  - **分片**（打散为 N 个 key，读时随机聚合）
  - 热点复制（多节点副本，客户端轮询）

### 18.3 BIGKEY 预防清单

- 列表长度、集合元素数加业务上限
- 不把**永久累加数据**放一个 key（日志/全量历史）
- 监控告警：按类型和大小阈值

### 18.4 Pipeline + 批量

- 读写多 key 统一用 Pipeline，减小网络开销
- `MGET` / `MSET` / `HMGET` 批量 API
- Cluster 下批量要处理**跨 slot**（要么 hash tag，要么分组 pipeline）

### 18.5 连接与 I/O

- 客户端用**连接池**，避免短连接
- `client-output-buffer-limit` 防止慢客户端撑爆内存
- `tcp-keepalive 60`

### 18.6 Swap 必须关

```bash
vm.swappiness = 0                # Linux
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

Redis 一旦进入 swap，延迟直接从微秒到数毫秒/秒。

### 📝 笔试题 18-1：一次发现某热点商品详情 key 的 QPS 是其他商品的 1000 倍，如何优化？

1. **本地缓存 L1**（Caffeine）：绝大部分请求不再打 Redis
2. **多副本**：`hotkey:1`、`hotkey:2` ... 随机读，一致性可接受
3. **拆分字段**：大 Hash 按访问场景拆
4. **异步刷新**：逻辑过期避免集中重建
5. **限流**：极端情况下保护后端
6. **监控热 key**：提前发现，避免被动

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 关于 Cache-Aside，以下错误的是？
A. 读：miss 则回源 DB 并写缓存
B. 写：更新 DB + 删除缓存
C. 写：更新 DB + 更新缓存（更保险）
D. 删除失败需要重试机制

<details><summary>答案</summary>C。"更新缓存"在并发下易出脏数据。</details>

**Q2** 缓存击穿描述为？
A. 不存在的 key 被反复查询
B. 大量 key 同一时刻过期
C. 单个热点 key 过期瞬间并发回源
D. Redis 实例宕机

<details><summary>答案</summary>C。</details>

**Q3** 以下哪条命令会阻塞 Redis 数秒？
A. `GET key`  B. `KEYS *`（百万 key 下）  C. `INCR k`  D. `EXPIRE k 60`

<details><summary>答案</summary>B。</details>

**Q4** Redis 保证原子性的最佳手段？
A. Pipeline
B. MULTI/EXEC 且中途失败回滚
C. Lua 脚本
D. 单线程使客户端天然安全

<details><summary>答案</summary>C。MULTI/EXEC 不回滚运行时错误。</details>

**Q5** `SETNX` + `EXPIRE` 实现分布式锁的主要问题？
A. 非原子，前者成功后进程崩溃则锁永驻
B. 性能太差
C. 不支持集群
D. 无法设置值

<details><summary>答案</summary>A。改用 `SET key val NX PX ttl`。</details>

**Q6** Redis Cluster 的 slot 数是？
A. 1024  B. 4096  C. 16384  D. 65536

<details><summary>答案</summary>C。</details>

**Q7** 下列哪项**不属于** Redis 数据结构？
A. ZSet  B. HyperLogLog  C. Bloom Filter（内置）  D. Stream

<details><summary>答案</summary>C。原生没有 Bloom Filter，RedisBloom 模块才有。</details>

**Q8** 关于 RDB 与 AOF：
A. AOF 文件总是比 RDB 小
B. 混合持久化结合 RDB 快照 + AOF 增量
C. RDB 不能在后台生成
D. AOF `everysec` 会严重影响性能

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. `KEYS *` 在生产环境可以放心使用。 ❌
2. Redis 单线程所以一定不会出现并发问题。 ❌（客户端层仍可能，多命令组合需要原子）
3. Pub/Sub 是可靠消息队列。 ❌
4. Stream 支持消费者组与 ACK。 ✅
5. 一致性哈希的目的是在扩缩容时最小化 rehash 影响。 ✅
6. HyperLogLog 会精确返回基数。 ❌（估计值，误差 ~0.81%）
7. `UNLINK` 比 `DEL` 更适合删除大 key。 ✅
8. Redis 必须开启 Swap 才能跑起来。 ❌（恰相反，生产要关 swap）

### 19.3 简答题

**Q1** 如何防止缓存穿透、击穿、雪崩？分别一条最关键的对策。

- **穿透**：布隆过滤器 + 空值短缓存
- **击穿**：互斥锁重建 / 逻辑过期
- **雪崩**：TTL 加随机抖动 + 多级缓存 + 限流熔断

**Q2** 订单扣库存的分布式锁如何做？

```python
token = str(uuid4())
ok = r.set(f"lock:sku:{sku}", token, nx=True, px=30000)
if not ok:
    raise Busy()
try:
    # 扣减逻辑
finally:
    r.eval(RELEASE_SCRIPT, 1, f"lock:sku:{sku}", token)
```

长事务用 Redisson 或 watchdog 自动续期。

**Q3** 主从 Redis，如何保证"读己之写"一致性？

- 写后立即读 **强制走主**
- 或用 `WAIT numreplicas timeout` 等待至少 N 个从节点同步完成再返回
- 对强一致路径不做读写分离

**Q4** 如何为一个 1 亿行历史订单列表做缓存？

- 单 key 存全量不现实 → 按维度切分（按用户、按日期）
- 热门用户：`HSET order_cnt:<uid>` 或 ZSet 存最近 N 条
- 冷数据不缓存，走 DB + 索引
- 查询频繁聚合：预聚合结果缓存
- 分页优化：游标分页，而非 `LRANGE 0 -1`

### 19.4 实操题

**Q1** 用 Redis 实现签到与"连续签到天数"，给出命令。

```
# 签到（第 D 天）
SETBIT sign:u:42:202501 (D-1) 1

# 当月签到次数
BITCOUNT sign:u:42:202501

# 连续签到天数（从今天起向前）：
# 伪代码：循环 GETBIT 往前数，遇 0 停止
```

**Q2** 实现一个简单的 API 限流：每 IP 每分钟最多 100 次。

```lua
-- KEYS[1]=rl:ip:1.2.3.4
-- ARGV[1]=limit, ARGV[2]=window_sec
local c = redis.call('INCR', KEYS[1])
if c == 1 then
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
end
if c > tonumber(ARGV[1]) then return 0 end
return 1
```

**Q3** 用 ZSet 实现延迟任务队列：添加任务、消费任务。

```
# 添加
ZADD delay_q 1700000000000 task123

# 消费（worker 轮询）
ids = ZRANGEBYSCORE delay_q 0 now LIMIT 0 20
for id in ids:
    if ZREM delay_q id == 1:
        process(id)
```

`ZRANGEBYSCORE` + `ZREM` 用 Lua 原子化防重复。

**Q4** 使用 Pipeline 批量获取 1000 个用户资料。

```python
pipe = r.pipeline(transaction=False)
for uid in uids:
    pipe.hgetall(f"user:{uid}")
profiles = pipe.execute()
```

---

## 📚 学习建议
0. **动手实践**：在本机装单机redis或搭建一个简单的redis集群，通过redis-cli练习其指令，使用一门编程语言写一个redis client连接你的redis并写代码管理缓存(查询、设置缓存、更新缓存、删除缓存)
1. **吃透数据结构**：每种类型的适用场景胜过所有高级特性
2. **搞清编码**：listpack / hashtable / skiplist 的切换阈值影响内存与性能
3. **读源码**：antirez 注释详尽；`t_string.c`、`t_hash.c`、`dict.c` 值得一读
4. **练手场景**：排行榜、延时队列、分布式锁、限流——能手写就能应对 90% 面试
5. **运维视角**：监控、慢查询、大 key、持久化、复制延迟比单纯 API 更重要
6. **官方文档**：<https://redis.io/docs/> 与 <https://redis.io/commands/> 权威详尽

> 祝你的缓存命中率常绿、QPS 常高。
