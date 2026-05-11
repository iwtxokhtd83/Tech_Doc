
# 内存泄漏与 GC 讲义

> 本讲义系统讲解**内存管理**与**垃圾回收**的通用原理、主流语言 / 运行时的 GC 设计、**内存泄漏**的成因与排查、性能调优实践。覆盖 Java / JavaScript (V8) / Python / Go / .NET / C / C++，以及容器化场景的内存观察。每章配"知识点 + 笔试题"。
>
> 约定：内存单位按 IEC（`KiB=1024B`）；示例工具链以 JDK 17+、Node.js 20+、Python 3.12+、Go 1.22+ 为主。

## 目录

1. [基础：内存模型与分配](#1-基础内存模型与分配)
2. [手动内存管理 vs 自动 GC](#2-手动内存管理-vs-自动-gc)
3. [GC 核心算法](#3-gc-核心算法)
4. [分代假说与分代 GC](#4-分代假说与分代-gc)
5. [JVM 内存结构与 GC](#5-jvm-内存结构与-gc)
6. [JVM GC 收集器演进](#6-jvm-gc-收集器演进)
7. [JavaScript / V8 GC](#7-javascript--v8-gc)
8. [Python 内存与 GC](#8-python-内存与-gc)
9. [Go GC](#9-go-gc)
10. [.NET / CLR GC](#10-net--clr-gc)
11. [C / C++ 与智能指针](#11-c--c-与智能指针)
12. [内存泄漏的本质与分类](#12-内存泄漏的本质与分类)
13. [常见内存泄漏场景](#13-常见内存泄漏场景)
14. [排查方法与工具](#14-排查方法与工具)
15. [堆外内存与 Native 内存](#15-堆外内存与-native-内存)
16. [容器环境内存问题](#16-容器环境内存问题)
17. [GC 调优原则与实战](#17-gc-调优原则与实战)
18. [反模式与最佳实践](#18-反模式与最佳实践)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 基础：内存模型与分配

### 1.1 进程内存布局（典型 Linux）

```
┌────────────────────┐  高地址
│       Stack        │  ← 向下增长，函数调用、局部变量
├────────────────────┤
│         ↓          │
│      (空闲)        │
│         ↑          │
├────────────────────┤
│       Heap         │  ← 向上增长，动态分配
├────────────────────┤
│       BSS          │  未初始化全局/静态
├────────────────────┤
│       Data         │  已初始化全局/静态
├────────────────────┤
│       Text         │  代码段
└────────────────────┘  低地址
```

对于运行在托管运行时（JVM、V8、CLR、CPython）里的程序，**用户看到的堆"是运行时管理的堆，不是 OS 堆**；运行时再向 OS 申请大块内存给自己分派。

### 1.2 栈（Stack）vs 堆（Heap）

| 维度 | 栈 | 堆 |
|------|----|----|
| 分配 | 编译期已知大小 | 运行时动态 |
| 速度 | 极快（寄存器 + push/pop） | 较慢（分配器算法） |
| 生命周期 | 作用域结束自动释放 | 主动释放 / GC |
| 大小 | 线程级，MB 级 | 受可用虚存限制 |
| 并发 | 每线程独立 | 共享，需同步 |

### 1.3 分配器

- **libc malloc**：ptmalloc2
- **jemalloc**：低碎片、多 arena，Redis/MongoDB/Rust 默认常用
- **tcmalloc**：Google，线程本地缓存
- **mimalloc**：Microsoft，性能好

托管运行时（JVM/V8/CLR/Go）一般**不直接用 libc malloc**，自己实现分配器（bump pointer / free list），以配合 GC。

### 1.4 虚拟内存核心概念

- **虚拟地址空间 vs 物理内存**：内核 MMU 通过页表映射
- **Page**：默认 4 KiB
- **RSS**：进程实际占用的物理内存
- **VSZ / VIRT**：虚拟地址空间
- **Shared / Private**：共享页（多进程共用）vs 私有
- **Swap**：低内存时把不活跃页换到磁盘
- **OOM Killer**：Linux 内核在内存不足时按分数杀进程

### 📝 笔试题 1-1：RSS 和 VIRT 的区别？哪个更能反映"内存占用"？

- **VIRT**：进程**虚拟地址空间**大小，包括未映射到物理内存的部分（mmap、库、预留）
- **RSS**：进程当前**实际占用的物理内存**

日常判断"吃多少内存"通常看 **RSS**；但 RSS 包含共享库份额（每进程各自统计一份）可能高估。更精确可看 **PSS**（按比例分摊共享页）。容器内存限额也基本按 RSS 类指标考察。

---

## 2. 手动内存管理 vs 自动 GC

### 2.1 手动（C / C++）

```c
void* p = malloc(1024);
// ... use p
free(p);      // 必须释放
```

**问题**：

- **内存泄漏**：忘记 free
- **double free**：释放两次
- **use-after-free**：已释放仍访问
- **悬挂指针**、**野指针**
- **栈溢出** / **缓冲区溢出**

### 2.2 半自动（C++ / Rust）

- **RAII**：资源随对象生命周期绑定
- **智能指针**：`unique_ptr` / `shared_ptr` / `weak_ptr`
- Rust **所有权 + 借用检查**：编译期保证无悬挂、无数据竞争

### 2.3 自动（GC 托管）

- 开发者只管"new"，运行时决定何时回收
- 代价：STW 暂停、吞吐开销、内存占用略高
- 收益：开发效率、内存安全

### 2.4 GC 的两大目标

- **吞吐量（Throughput）**：GC 时间 / 应用时间，越小越好
- **暂停时间（Pause）**：STW 越短越好

二者常冲突。不同 GC 算法在 **暂停 vs 吞吐 vs 内存** 三角中做取舍。

### 📝 笔试题 2-1：GC 语言就不会内存泄漏吗？

**错**。GC 只回收**不可达对象**；如果你让对象一直**可达**（放进全局集合、静态 Map、监听器没注销），GC 无能为力。**GC 泄漏** 本质是"逻辑泄漏"：你以为这个对象可以丢了，但代码里仍持有引用。

---

## 3. GC 核心算法

### 3.1 引用计数（Reference Counting）

每个对象维护一个计数，引用增减即时修改，归零立即释放。

- ✅ 实时性好，释放即时
- ❌ **循环引用**无法回收（A 引 B，B 引 A，计数都是 1）
- ❌ 多线程下 `inc/dec` 需原子或锁

代表：**CPython**（主路径）、Swift ARC、Objective-C ARC。

### 3.2 可达性分析（Tracing GC）

从 **GC Roots** 出发，沿引用遍历标记所有可达对象，其余回收。

**GC Roots 典型来源**：

- 活跃线程的栈帧局部变量
- 静态字段 / 全局
- JNI / Native 引用
- 同步锁持有对象
- 运行时系统 class / metadata

大部分现代 GC（JVM、V8、CLR、Go）都以 tracing 为核心。

### 3.3 经典算法

#### 标记-清除（Mark-Sweep）

1. 从 roots 标记所有可达对象
2. 扫描堆，未标记的释放

- ❌ 产生**碎片**

#### 标记-整理（Mark-Compact）

标记后把存活对象**向一端移动**，释放整片连续空间。

- ✅ 无碎片
- ❌ 移动成本高（需更新引用）

#### 复制（Copying / Semi-space）

堆分两半，只用一半；GC 时把存活对象拷贝到另一半，一次性清空旧区。

- ✅ 简单快速
- ❌ **浪费一半空间**
- 特别适合"大多数对象早亡"的新生代

#### 分代（Generational）

年轻代用复制、老年代用标记-整理，基于"弱分代假说"（见下章）。

#### 并发 / 增量 GC

- **STW (Stop-The-World)**：暂停全部应用线程才 GC，延迟高
- **并发**：GC 与应用线程同时运行，减少 STW
- **增量**：把 GC 工作切成多步，穿插进应用执行
- **并行**：多线程并行做 GC 工作

现代低延迟 GC（G1 / ZGC / Shenandoah）大量运用并发技术，把 STW 控制到毫秒甚至亚毫秒级。

### 3.4 三色标记（Tri-color Marking）

并发标记的核心算法：

- **白**：未访问
- **灰**：已访问但引用还没扫完
- **黑**：自己和引用都已扫完

完成后仍是白的对象 = 不可达 = 可回收。

**并发标记的坏情况**：应用线程在标记中途修改引用，可能导致"黑指向新白"而丢对象。解决：

- **写屏障（Write Barrier）**：每次引用赋值时记录，GC 能感知修改
- **读屏障（Read Barrier）**：读时做处理（ZGC 用）
- **快照（SATB）** / **增量更新**：两种"保持正确性"的思路

### 3.5 区域化堆（Region-based）

把堆分成很多相等/近似的**区域（region）**：

- 回收时**按区域**选择（Garbage-First）
- 支持不规则分代、混合回收
- 代表：**G1**、**ZGC**、**Shenandoah**

### 📝 笔试题 3-1：为什么 Python 在引用计数之外还需要循环检测 GC？

因为引用计数**无法回收循环引用**：

```python
a, b = {}, {}
a['ref'] = b
b['ref'] = a
del a, b    # 双方计数都 >0，永远不释放
```

CPython 在 `gc` 模块里用**分代 + 可达性分析**的额外检测机制，定期扫描容器对象处理循环。简单值对象不会循环，所以只对"可能含循环"的类型做检测。

---

## 4. 分代假说与分代 GC

### 4.1 弱分代假说

> **绝大多数对象生命周期很短，存活下来的少数对象会长寿。**

这是分代 GC 的理论基础。经验上：

- 请求处理过程中产生的临时对象
- 循环里的中间对象
- 短期 buffer / 包装器

这些都"朝生夕死"。

### 4.2 分代策略

- **年轻代（Young Gen / Nursery）**：多次 Minor GC，主要用复制算法
- **老年代（Old / Tenured）**：偶尔 Major/Full GC，多用标记-整理
- **晋升（Promotion）**：存活 N 次 Minor GC 的对象晋升到老年代

### 4.3 跨代引用与记忆集（Remembered Set）

**问题**：Minor GC 只扫年轻代；老年代里的对象如果指向年轻代对象，也是 root。怎么避免扫整个老年代？

**解法**：

- **Card Table（卡表）**：把老年代切成一个个 512B 左右的卡片，用一个 bit/byte 标记该卡片是否含跨代引用
- **Remembered Set**：G1 等 region 化 GC 用更精细的数据结构
- **写屏障**：应用线程修改引用时更新卡表

### 4.4 分代不是银弹

假说在某些场景不成立：

- 大对象缓存（直接老年代分配）
- 长事务 / 长周期（对象不短命）
- 流计算（很多中长生命周期 buffer）

所以 **ZGC / Shenandoah** 最初设计时不分代，后期又加回分代（ZGC 21+、Shenandoah 实验性分代）以兼顾两种场景。

### 📝 笔试题 4-1：什么样的对象**不应该**进入年轻代？

- **巨大对象**：避免在新生代复制反复拷贝，很多 GC 有"大对象阈值"直接分配到老年代（JVM `-XX:PretenureSizeThreshold`）
- **明显长寿对象**：缓存、常驻连接池

但这些策略都是**优化**而非规则；错误的预判可能拖垮老年代，让 Full GC 频繁。

---

## 5. JVM 内存结构与 GC

### 5.1 运行时数据区

```
┌───────────────── JVM ─────────────────┐
│ ┌──────────┐  ┌──────────┐            │
│ │ Stack x N│  │ PC x N   │  每线程    │
│ └──────────┘  └──────────┘            │
│ ┌───────────────────────────────┐    │
│ │            Heap               │    │
│ │  ┌────────┐  ┌──────────┐     │    │
│ │  │ Young  │  │ Old Gen  │     │    │
│ │  │ Eden S0│  │          │     │    │
│ │  │      S1│  │          │     │    │
│ │  └────────┘  └──────────┘     │    │
│ └───────────────────────────────┘    │
│ ┌───────────────────────────────┐    │
│ │  Metaspace (类元数据 Native)  │    │
│ └───────────────────────────────┘    │
│ ┌───────────────────────────────┐    │
│ │  Direct / Native / CodeCache  │    │
│ └───────────────────────────────┘    │
└───────────────────────────────────────┘
```

- **Heap**：GC 主战场
- **Metaspace**（JDK 8+ 取代 PermGen）：类元数据，**堆外**，默认不限
- **Stack**：每线程独立，存局部变量、操作数栈
- **Direct Memory**：`ByteBuffer.allocateDirect`，堆外
- **Code Cache**：JIT 编译后的机器码
- **Native Heap**：JNI、本地库

### 5.2 新生代 Eden + Survivor

```
┌─────┐ ┌───┐ ┌───┐
│Eden │ │S0 │ │S1 │
└─────┘ └───┘ └───┘
 8:1:1 (默认)
```

- 新对象分配在 **Eden**
- Minor GC：Eden + 一个 S 存活对象 → 另一个 S，清空
- 每次存活 age+1，超过阈值（默认 15）晋升到老年代
- S 放不下 → 直接晋升（提前晋升）

### 5.3 TLAB（Thread Local Allocation Buffer）

- 每线程在 Eden 中预留一小块私有内存
- 分配走 bump pointer，**无锁**
- 用完向全局 Eden 要新块
- `-XX:+UseTLAB`（默认开）

### 5.4 对象头与压缩指针

64 位 JVM 默认开启 **压缩对象指针（Compressed Oops）**：

- 对象头本来 16 字节变 12 字节（加对齐 → 16）
- 引用从 8 字节降为 4 字节
- 堆 < 32 GB 时有效
- `-XX:+UseCompressedOops`（默认开）

### 5.5 触发 Full GC 的常见原因

- 老年代空间不足 → 触发 Major / Full GC
- `System.gc()` 被调用（尽量禁用或配 `-XX:+DisableExplicitGC`）
- 方法区 / Metaspace 溢出
- CMS 并发失败（promotion failure / concurrent mode failure）
- G1 Humongous 对象过多
- 外部触发（jmap -dump 等）

### 📝 笔试题 5-1：`OutOfMemoryError: Java heap space` vs `OutOfMemoryError: Metaspace`？

- **Java heap space**：堆不足，常见于对象泄漏 / 集合膨胀 / 大对象
- **Metaspace**：类元数据不足，常见于
  - 动态生成类过多（JSP、动态代理、CGLIB）
  - 类加载器泄漏（热部署反复加载）
  - 未设 `-XX:MaxMetaspaceSize` 且持续膨胀

不同原因、不同排查路径；要看错误消息和 `jmap`/`jcmd VM.native_memory`。

---

## 6. JVM GC 收集器演进

### 6.1 一览

| 收集器 | 年代 | 特点 | 默认？ |
|--------|------|------|--------|
| **Serial** | 新 + 老 | 单线程，小堆 / 客户端 | - |
| **Parallel** | 新 + 老 | 多线程，吞吐优先（JDK 8 默认） | JDK 8 |
| **CMS** | 老 | 低停顿的并发标记清除 | 已移除（JDK 14） |
| **G1（Garbage-First）** | 全堆区域化 | 可预测停顿，分区收集 | JDK 9+ 默认 |
| **ZGC** | 全堆 | 着色指针 + 读屏障，停顿 < 1 ms；21+ 支持分代 | - |
| **Shenandoah** | 全堆 | 并发压缩，停顿低（RedHat） | - |
| **Epsilon** | 不回收 | 仅分配，不 GC，用于压测/基准 | - |

**现代生产默认**：

- JDK 8：Parallel
- JDK 11+：G1
- JDK 17+ 低延迟场景：ZGC / Shenandoah

### 6.2 Parallel GC

- 多线程并行做 Young/Old GC
- 吞吐优先，停顿时间随堆大小增长
- 适合后台批处理、吞吐 > 延迟 的场景

参数：

```
-XX:+UseParallelGC
-XX:ParallelGCThreads=N
-XX:MaxGCPauseMillis=200       # 目标停顿（仅建议）
-XX:GCTimeRatio=99             # GC 时间占比 ≤ 1/(1+ratio)
```

### 6.3 CMS（已退役）

- 并发标记清除，低停顿（STW 短）
- 不整理 → 碎片 → `promotion failure` → 突然 Full GC
- 已于 JDK 14 移除，了解即可

### 6.4 G1（Garbage-First）

- 堆切成 1-32 MB 的 **region**
- 新生代 / 老年代 / Humongous 都以 region 形式存在
- 每次只回收"收益最高"的若干 region（**Mixed GC**）
- 目标：**可预测的暂停时间**（近似）

**GC 阶段**：

1. Young GC（STW）
2. 并发标记周期（Initial Mark → Concurrent Mark → Remark → Cleanup）
3. Mixed GC（Young + 部分 Old）

参数：

```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # 停顿目标
-XX:G1HeapRegionSize=16m        # 区域大小
-XX:InitiatingHeapOccupancyPercent=45   # 触发并发标记的老年代占比
-XX:G1MixedGCCountTarget=8
```

**调优建议**：

- 先用默认，观察 GC 日志再动
- 大对象（> region/2）进入 Humongous，过多会触发 Full GC，考虑调大 region
- 停顿未达标：优先加堆、降并发压力，而非一味调参

### 6.5 ZGC

- **亚毫秒停顿**，TB 级堆
- **着色指针（Colored Pointers）**：指针高位存 GC 状态
- **读屏障（Load Barrier）**：读引用时校正
- **几乎所有工作并发完成**
- JDK 21 开始**分代 ZGC**（兼顾短命对象优化）

参数：

```
-XX:+UseZGC
-Xmx16g
-XX:+ZGenerational             # JDK 21+ 分代
```

代价：额外内存开销（指针位占用 + 记忆结构）；吞吐略低于 G1。

### 6.6 Shenandoah

- RedHat 主导，思路与 ZGC 相似（Brooks Forwarding Pointer / 读屏障）
- 停顿同样亚毫秒 - 毫秒级
- 在 OpenJDK 里是 **实验/可选**

### 6.7 GC 日志与可视化

```
-Xlog:gc*,gc+heap=info,gc+age=trace:file=gc.log:time,tags:filecount=10,filesize=100m
```

分析工具：

- **GCEasy**（在线）
- **GCViewer**（本地）
- **JClarity Censum**（商业）
- **VisualVM** / **JFR (Java Flight Recorder)**

### 6.8 堆大小设置

```
-Xms4g -Xmx4g         # 最小=最大，避免动态伸缩抖动
-Xmn1g                # 年轻代（仅 Parallel）
```

通常建议 **`-Xms = -Xmx`**，一次拿到位，避免 commit/uncommit 抖动。

容器环境：JVM 10+ 已感知 cgroup 内存限额，但仍建议 **显式 `-Xmx`**，留 20-30% 给 Metaspace、堆外、线程栈、JIT、Native。

### 📝 笔试题 6-1：什么时候选 ZGC / Shenandoah？

- 堆很大（> 16 GB）且对停顿敏感（延迟敏感的在线服务）
- 当前 G1 调不下来停顿
- 愿意接受略低吞吐换取极低停顿
- 不宜：小堆（几 GB）、吞吐优先的批处理

---

## 7. JavaScript / V8 GC

### 7.1 V8 堆结构

V8（Chrome / Node.js）使用**分代 GC**：

```
┌──────────────────── V8 Heap ────────────────────┐
│ New Space (Young) : 小，用 Scavenge 复制算法     │
│ Old Space        : 大，用 Mark-Sweep-Compact     │
│ Large Object Space: 大对象直接分配这里           │
│ Code Space       : 已编译代码                    │
│ Map Space        : 隐藏类                        │
└──────────────────────────────────────────────────┘
```

### 7.2 新生代：Scavenge

- 两半 From / To，复制算法
- Minor GC 非常快（毫秒级）
- 存活两次后晋升 Old Space

### 7.3 老生代：Mark-Sweep / Mark-Compact + Orinoco

- **Orinoco**：V8 并发/并行/增量 GC 项目名
- **Incremental Marking**：把 Marking 切成多步与应用交替
- **Concurrent Marking**：用辅助线程并发标记
- **Lazy Sweeping**：空闲时再清扫
- **Compaction**：需要时做整理

### 7.4 Node.js 内存上限

- 默认老生代约 1.4-2 GB（版本不同）
- 调大：`--max-old-space-size=4096`（MB）
- 容器化时务必设显式值，并留余量给 C++ / libuv / 缓冲

### 7.5 JS 典型泄漏源

- **闭包**持有大对象不释放
- **全局变量**（显式 / 隐式 `window.x=`）
- 定时器 `setInterval` 未 `clearInterval`
- DOM 监听未解绑（浏览器端）
- 脱离 DOM 仍被 JS 引用（detached DOM）
- **Buffer / Stream** 未消费或未关

### 7.6 工具

- **Chrome DevTools Memory** 面板：Heap Snapshot / Allocation timeline / Allocation sampling
- Node.js：`--inspect` + DevTools；`heapdump` 包；`clinic.js`
- **`v8.getHeapStatistics()`**
- **`--trace-gc`**：输出 GC 日志

### 📝 笔试题 7-1：Node.js 服务内存缓慢上涨，怎么排查？

1. `process.memoryUsage()` 看 `heapUsed` / `rss` / `external`
2. 开 `--inspect` 连接 Chrome DevTools，做 **Heap Snapshot**，对比多次快照找增长对象类型
3. 借助 **Allocation timeline** 找分配热点
4. 检查定时器、事件监听、缓存 Map、闭包是否漏清
5. 排查 `external`（C++ 绑定、Buffer）泄漏
6. 看 GC 日志 `--trace-gc` 判断是否只是堆扩张而非真正泄漏

---

## 8. Python 内存与 GC

### 8.1 CPython 内存模型

- 一切皆**对象**，堆分配
- 使用 **PyMalloc**：专属小对象分配器（<512 字节走 arena）
- 大对象直接走 libc malloc / jemalloc

### 8.2 垃圾回收机制

CPython 两套：

1. **引用计数**（主路径）：`sys.getrefcount(x)`；即时释放
2. **循环检测 GC**（`gc` 模块）：分三代，定期扫描容器对象

```python
import gc
gc.get_threshold()   # (700, 10, 10) 默认
gc.collect()          # 手动触发
gc.disable() / gc.enable()
```

分代阈值：
- `gen0` 分配次数超过阈值 → 扫描
- 存活对象晋升到 `gen1`，`gen1` 再晋升 `gen2`

### 8.3 GIL 与内存

- **GIL**（Global Interpreter Lock）：同一时刻只有一个线程执行 Python 字节码
- GC 在 GIL 下执行，相对简单安全
- 3.13 引入 **--disable-gil** 实验模式（无 GIL），GC 也需适配

### 8.4 Python 常见内存陷阱

- **循环引用** + 对象含 `__del__`：某些老版本里会阻止回收（3.4+ 已修复）
- **全局字典**累积（注册表、缓存）
- **闭包**长时间持有大对象
- 长运行服务：小对象 **arena 碎片**使 RSS 居高（典型"Python 归还内存难"）
- C 扩展泄漏（numpy、pandas 内部的 `Py_INCREF` 不配对）
- **pickle / dict** 保存很多**临时对象**没及时清

### 8.5 工具

- **`tracemalloc`**（内置）：分配栈追踪
- **`objgraph`**：对象关系图
- **`guppy3` / `heapy`**：堆快照
- **`memory_profiler`**：按行/函数的内存
- **`pympler`**：总览
- **`py-spy`**：CPU 剖析（对内存间接有用）

示例：

```python
import tracemalloc
tracemalloc.start(25)

# ... do work
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```

### 8.6 Python "RSS 降不下来"

Python 归还内存给 OS 较保守，arena 空闲时不一定能释放。缓解：

- **fork worker**：让子进程处理任务，完事退出（Gunicorn、多进程）
- 使用 **jemalloc** 替代 ptmalloc（`LD_PRELOAD=libjemalloc.so`）
- 分批处理大数据，显式 `del` + `gc.collect()`
- 真正内存大户考虑 Rust/Go/C++ 扩展

### 📝 笔试题 8-1：Python 服务 RSS 稳定上涨但 `gc.collect()` 没用，为什么？

可能不是"GC 回收不动"，而是：

- **C 扩展泄漏**（不受 Python GC 管）
- **glibc malloc 碎片**让归还 OS 困难
- 还是**逻辑引用**未释放（全局 / 缓存 / 闭包）
- 大量小对象在 **arena** 里等待整体释放

排查：`tracemalloc` 找 Python 侧增长；`ps`/`pmap` 看 RSS 结构；换 jemalloc；必要时用 valgrind / ASan 查 C 扩展。

---

## 9. Go GC

### 9.1 设计目标

- **低延迟**：STW 通常 < 1 ms
- **并发标记 + 三色 + 写屏障**
- **非分代**（历史上认为不必要，近年有讨论）
- **非整理**（不移动对象，避免指针修正）

### 9.2 GC 过程

1. **STW 启动**：设置屏障、准备根
2. **并发标记**（与应用同时执行）
3. **STW 标记终结**：处理写屏障遗留
4. **并发清扫**（边分配边清扫）

两次 STW 都非常短。

### 9.3 触发时机

Go 通过 **GOGC**（默认 100）控制堆目标：

```
下一次 GC 目标堆 = 上次 live * (1 + GOGC/100)
```

GOGC=100 表示"堆翻倍时触发"；调高降 GC 频率（代价更大内存）；调低反之。

1.19+ 起还可用 `GOMEMLIMIT` 设**软内存上限**，防 OOM：

```bash
GOGC=100 GOMEMLIMIT=8GiB ./app
```

### 9.4 Go 常见内存问题

- **goroutine 泄漏**：通道无人读/写，goroutine 卡住不退出
- **slice 引用大底层数组**：`small = big[:10]` 让整个 big 无法释放，需 `copy`
- **map 删除不降容**：删除 key 不回收 bucket，大 map 用完后应置 nil 重建
- **全局注册表**长期持有
- **闭包**捕获大对象
- **channel / context 泄漏**

### 9.5 工具

- **`runtime/pprof` + `net/http/pprof`**：heap/goroutine/allocs profile
- **`go tool pprof`** + 火焰图
- **`GODEBUG=gctrace=1`**：打印每次 GC 信息
- **`runtime.MemStats`**：代码内获取
- **trace**：`runtime/trace`，调度/GC 可视化

```go
import _ "net/http/pprof"

// http://host:port/debug/pprof/heap
// go tool pprof http://host:port/debug/pprof/heap
```

### 9.6 示例：goroutine 泄漏

```go
func leak() {
    ch := make(chan int)
    go func() {
        ch <- 1       // 永远阻塞，goroutine 泄漏
    }()
    // 没人读 ch
}
```

修法：用 `context` 传取消、设置缓冲 channel、或 `select + default`。

### 📝 笔试题 9-1：Go 程序"RSS 持续上涨但 GC 正常"，可能是什么？

- **goroutine 泄漏**：每 goroutine 占 2KB+ 栈，数百万个吃掉几 GB
- **slice/map 未释放底层数组**
- **cgo / 第三方库**分配的 C 内存不受 Go GC 管
- **pprof heap profile** 可能不完全反映（因为 sampling）
- `runtime.MemStats.HeapSys` vs `HeapReleased` 看未归还页

先看 goroutine 数（`/debug/pprof/goroutine`），再看 heap profile，最后怀疑 cgo。

---

## 10. .NET / CLR GC

### 10.1 设计特点

- **分代**：Gen0 / Gen1 / Gen2 + LOH（Large Object Heap，>= 85 KB）
- **标记-整理**（Workstation / Server GC 两种模式）
- **并发 GC（Background GC）**：Gen2 可与应用并发
- .NET 5+ 引入 **POH（Pinned Object Heap）** 专放固定对象

### 10.2 两种 GC 模式

- **Workstation GC**：单线程、低影响，适合桌面、小服务
- **Server GC**：每 CPU 一个 GC 堆 + 并行 GC，大型服务默认开

```
<ServerGarbageCollection>true</ServerGarbageCollection>
<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
```

### 10.3 LOH 注意事项

- 不整理 → 碎片
- 频繁创建大对象（图片、缓冲区）易碎
- .NET Core 3 开始可**按需整理 LOH**：`GCSettings.LargeObjectHeapCompactionMode = CompactOnce`

### 10.4 常见泄漏

- **event handler 未注销**：发布者长生命周期，订阅者泄漏
- **静态字段**持有大对象
- **IDisposable 未释放**：数据库连接 / Stream / 文件
- **Timer** 持有回调
- **WeakReference 误用**

### 10.5 工具

- **dotnet-counters** / **dotnet-dump** / **dotnet-gcdump**
- **PerfView**（经典）
- **Visual Studio Memory Profiler**
- **JetBrains dotMemory / dotTrace**

---

## 11. C / C++ 与智能指针

### 11.1 典型问题

```c
char* p = malloc(100);
...
// 漏 free(p)
```

或更糟：

```c
free(p);
// 后续代码又用 p；double free / use-after-free
```

### 11.2 C++ 现代实践

- **RAII**：资源随栈对象销毁自动释放
- **智能指针**：
  - `std::unique_ptr<T>`：独占所有权，零开销
  - `std::shared_ptr<T>`：引用计数共享
  - `std::weak_ptr<T>`：弱引用，打破循环
- **避免裸 `new / delete`**

```cpp
auto p = std::make_unique<Foo>();   // 独占
auto s = std::make_shared<Bar>(42); // 共享
std::weak_ptr<Bar> w = s;           // 弱
```

**`shared_ptr` 循环引用**仍会泄漏：

```cpp
struct Node { std::shared_ptr<Node> next; };
auto a = std::make_shared<Node>();
auto b = std::make_shared<Node>();
a->next = b; b->next = a;    // 循环，不会被销毁
```

修法：其中一方用 `weak_ptr`。

### 11.3 工具

- **Valgrind (memcheck)**：经典，慢但强
- **AddressSanitizer (ASan)**：编译时启用，近实时发现 UAF / leak / overflow
- **LeakSanitizer (LSan)**：与 ASan 集成，检测未释放
- **Massif / heaptrack**：堆分配剖析
- **Dr.Memory**（Windows）
- **Visual Studio 内存诊断**

### 11.4 Rust 的内存安全哲学

Rust 在编译期用**所有权 + 借用检查器**消除大部分内存错误：

- 不存在"野指针"
- 不存在"use-after-free"
- 不存在"double free"
- 数据竞争大部分靠类型系统解决

代价：学习曲线陡、迁移老代码成本高。适合新建系统软件、数据库、WASM 等。

### 📝 笔试题 11-1：`shared_ptr` 会内存泄漏吗？

会。两种场景：

1. **循环引用**：两个对象互相用 `shared_ptr`，引用计数永不归零。需要用 `weak_ptr` 打破循环
2. **与裸指针混用**：从同一裸指针构造两个独立 `shared_ptr` → 分别维护两套计数 → double free 或误释放

`shared_ptr` 是便利工具而非免死金牌，仍需理解所有权语义。

---

## 12. 内存泄漏的本质与分类

### 12.1 定义

**内存泄漏（Memory Leak）**：进程内存持续增长或在应当释放时未释放，导致：

- 长期运行后 OOM
- RSS 压迫同宿主其他服务
- 频繁 GC 拖慢性能
- 缓慢衰退难以察觉

### 12.2 逻辑泄漏 vs 物理泄漏

| 类型 | 说明 | 典型语言 |
|------|------|----------|
| **物理泄漏** | `malloc` 后未 `free` | C / C++ |
| **逻辑泄漏** | 对象仍可达但业务已不需 | Java / JS / Python / Go / .NET |

GC 语言里绝大多数"泄漏"是**逻辑泄漏**——你没告诉 GC 这玩意可以丢了。

### 12.3 短期泄漏 vs 长期泄漏

- **短期**：单次请求未清理，多次叠加才显现
- **长期**：启动即持有，不断累积（全局集合、缓存无上限）

### 12.4 增长速率分类

- **线性增长**：典型缓存/集合无限堆积
- **尖峰不回落**：某次请求产生大对象后不释放
- **阶梯状**：每次触发某功能就涨一格
- **震荡上行**：周期性大对象，GC 收不尽

看增长曲线能推断类型：

```
RSS
  │            ┌──── long-term leak (稳定斜线)
  │          /
  │        / 
  │   __/      ┌── step leak (每次事件抬一层)
  │   ↑        
  │─ 正常波动
  └─────────────────▶ time
```

### 📝 笔试题 12-1：如何从"内存高但不一定是泄漏"里区分？

重启后是否回到正常基线？

- **是**，且随时间**持续**上涨无上限 → 泄漏
- **是**，但在某阈值稳定下来（例如缓存上限） → 这就是**应有的内存占用**，不是泄漏
- 增长会在负载下降时回落 → **正常**堆扩张
- 只看曲线容易误判，配合 **压测 + 堆 dump 对比**更可靠

---

## 13. 常见内存泄漏场景

### 13.1 集合 / 缓存无界

典型：

```java
static Map<String, User> cache = new HashMap<>();   // 永远只加不删
```

修法：

- **上限**：`Caffeine`, `Guava` 的 `maximumSize`
- **TTL**：按访问/写入时间过期
- **WeakHashMap / WeakReference**：key 被回收时自动清理
- **LRU**：只留最近访问的

### 13.2 事件监听器 / 回调未注销

- Swing / Android Activity：生命周期不同步导致长生命周期持短生命周期
- Node.js `emitter.on(...)` 在循环里反复注册
- React `useEffect` 未清理

修法：**显式 off / removeListener**；React 返回 cleanup。

### 13.3 静态字段 / 单例持有

```java
public class Holder {
    private static final List<byte[]> BUFFER = new ArrayList<>();
    public static void add(byte[] b) { BUFFER.add(b); }
}
```

静态对象生命周期 = 整个应用；任何引用都永远不能被 GC 回收。除非必要，避免静态集合。

### 13.4 ThreadLocal 未 remove

- 每线程一份，线程池长期存活 → 值泄漏
- 手动 `threadLocal.remove()`，或用 `try-finally`
- Web 容器里 ThreadLocal + 类加载器 + 线程池 → 经典**热部署泄漏**

### 13.5 类加载器泄漏

- Tomcat 热部署时旧 WAR 的 ClassLoader 本应卸载
- 被某个静态引用 / JDBC / Log / ThreadLocal / JNI 持有
- 造成 `PermGen / Metaspace OOM`

### 13.6 大对象 / 长生命周期上下文

- HTTP 处理中把整个 session、整个请求体塞进缓存
- 日志字段里附带大对象快照（toString）

### 13.7 连接未关闭

- JDBC Connection / Statement / ResultSet 没 close
- InputStream / FileChannel 未关
- Kafka Consumer、Redis 客户端

修法：**try-with-resources** / Python `with` / Go `defer` / Rust `Drop`。

### 13.8 定时任务 / 异步任务

- `ScheduledExecutor` 提交的任务持有外层对象
- `setInterval` 未清
- Timer、CompletableFuture 链条里漏 release

### 13.9 反射 / 动态代理 / 字节码增强

- CGLIB、Javassist 生成大量类 → Metaspace 膨胀
- AOP 框架在 hot reload 下反复生成

### 13.10 字符串 / 池化的误用

- `String.intern()` 滥用 → 字符串常量池膨胀
- 大量"unique"字符串调用 `intern`

### 📝 笔试题 13-1：ThreadLocal 为什么容易泄漏？

`ThreadLocalMap` 的 **key 是 WeakReference**（ThreadLocal 本身），**value 是强引用**。当外层没有引用 ThreadLocal 时 key 能被回收，但 value 仍被 Thread 引用住。线程池里的线程长期不销毁，value 就长期泄漏。

修法：用完 `threadLocal.remove()`；并在 finally 里执行，避免异常漏清。

---

## 14. 排查方法与工具

### 14.1 通用流程

```
观察指标 → 确认泄漏 → 缩小范围 → 定位对象 → 找引用链 → 修复验证
```

1. **观察**：RSS / 堆 / GC 次数随时间曲线
2. **确认**：施加负载 → 停止负载 → 看是否回落
3. **缩小**：二分关闭功能 / 打开采样
4. **定位**：堆 dump / heap snapshot
5. **引用链**：找到"谁在持有它"
6. **修复**：代码改动 + 验证曲线

### 14.2 工具按语言

**JVM**：

- `jps` / `jcmd` / `jmap -heap / -dump` / `jstat -gc`
- **VisualVM**、**JFR + JMC**、**async-profiler**
- **Eclipse MAT**（分析 heap dump 的王者）
- **GC 日志**：`-Xlog:gc*`

**Node.js / V8**：

- `node --inspect` + Chrome DevTools（Heap Snapshot 对比）
- `clinic.js`、`0x`（火焰图）
- `heapdump` 包

**Python**：

- `tracemalloc`（内置）、`objgraph`、`pympler`、`memory_profiler`、`py-spy`

**Go**：

- `net/http/pprof`、`go tool pprof`、`runtime/trace`
- `GODEBUG=gctrace=1`

**.NET**：

- `dotnet-counters / -dump / -gcdump`、**PerfView**、**dotMemory**

**C / C++**：

- **Valgrind**、**ASan / LSan**、**heaptrack**、**Massif**

**系统层**：

- `top` / `htop`、`ps`、`free`、`smem`
- `pmap <pid>`、`cat /proc/<pid>/status`、`/smaps`
- `ftrace` / `bpftrace` / `bcc`
- `perf mem`

### 14.3 Heap Dump 对比法

**双快照（diff）** 是查泄漏的利器：

1. 系统进入稳定状态后取 snapshot A
2. 施加负载一段时间
3. 回到稳定再取 snapshot B
4. **对比** A → B 新增最多的类/保留大小
5. 查看 **GC Root 引用链**

JVM 里用 MAT 的 **Histogram / Dominator Tree / Path to GC Roots**。

### 14.4 生产环境注意

- 做 heap dump 会**暂停进程**（JVM jmap 会 STW）
- 生产做 dump 前**先摘流量**或开并发方案
- dump 很大（与堆同量级）→ 保留空间
- 脱敏后才能下载分析（可能包含 PII）

### 14.5 监控告警

- **RSS 增长率 > 阈值 / day**
- **GC 频次 & 时长**：Full GC 次数、平均 pause
- **JVM Native Memory**：NMT 输出差值
- **FD 数量 / 线程数**：间接反映资源泄漏
- **容器级 OOMKilled 计数**

### 📝 笔试题 14-1：如何优雅地在生产拿 heap dump？

- 有多实例时**先从 LB 摘流** → 再 dump → 恢复
- JVM：`jcmd <pid> GC.heap_dump /path/to.hprof`，或 `-XX:+HeapDumpOnOutOfMemoryError` 在 OOM 时自动
- 使用**低开销手段**：JFR 定期采样比全量 dump 小得多
- dump 越早越小（内存占用）；等到几十 GB 再 dump 往往来不及

---

## 15. 堆外内存与 Native 内存

### 15.1 什么是堆外

JVM / Node / Python 等运行时除了"托管堆"，还有大量**堆外**内存：

- **DirectByteBuffer**（JVM NIO）
- **Metaspace**（类元数据）
- **线程栈**（每线程 512KB-1MB）
- **Code Cache**（JIT 产物）
- **JNI / C 扩展**
- **glibc / jemalloc arena**

任何一块涨上去都会撑爆容器，但 Java `Xmx` 只管堆。

### 15.2 JVM 堆外监控

```
-XX:NativeMemoryTracking=summary   # detail
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory baseline
jcmd <pid> VM.native_memory summary.diff
```

输出能看到 Java heap / class / thread / code / GC / internal / symbol / native memory 等占用。

### 15.3 Direct Memory 泄漏

```
OutOfMemoryError: Direct buffer memory
```

常见原因：

- 依赖库（Netty、RocksDB）大量 `ByteBuffer.allocateDirect`
- 未关闭 channel / 未释放 buffer
- 参数 `-XX:MaxDirectMemorySize` 默认等于 Xmx，容器里显式设

### 15.4 JNI / C 扩展泄漏

- C 侧 `malloc` 后忘 `free`
- JVM 无法感知；堆 dump 里看不到
- 用 **NMT** + **pmap** / **jemalloc profiling** / **valgrind** 定位

### 15.5 容器 RSS 结构拆解

```
container RSS = Java heap + Metaspace + Code Cache + 线程栈 + Direct + JNI + glibc arena + 其它
```

一个常见经验：

- 容器 limit 8 GB
- `-Xmx` 建议 4-5 GB，留 3 GB 给其他
- 若设 `-Xmx=7.5g`，很容易被 OOMKilled

### 📝 笔试题 15-1：Java 容器被 OOMKilled 但 jmap 看堆还远没满？

堆外吃掉了：

- Metaspace 无上限膨胀（动态生成类）
- Direct Memory 泄漏（Netty / NIO）
- JNI / C 扩展泄漏
- 线程数爆炸（每线程栈 1MB）
- Code Cache 满

看 `NMT` + `pmap` + `/proc/<pid>/status` 综合分析；`Xmx` 只解堆层面的问题。

---

## 16. 容器环境内存问题

### 16.1 cgroup 限额与可见性

- K8s `resources.limits.memory` → cgroup `memory.max`
- 超限 → **OOMKilled**（退出码 137）
- 运行时需"感知 cgroup"才不误用全机内存

| 运行时 | cgroup 感知 |
|--------|-------------|
| OpenJDK 8 update 191+ / 11+ | `-XX:+UseContainerSupport` 默认开 |
| Node.js | 需 `--max-old-space-size` 显式设 |
| Go | 1.19+ `GOMEMLIMIT` |
| Python | 通常无感知；自行约束 |

### 16.2 Working Set Memory

K8s 统计的 "memory usage" 其实是 **working_set**：

```
working_set ≈ RSS + shmem - inactive_file
```

关键点：**活跃文件缓存会被计入**，因此 "kubectl top" 看到的数常大于 `ps` 的 RSS。

### 16.3 设置 limits 的建议

- **requests ≤ limits**
- JVM：`-Xmx` 约为 `limits` 的 60-70%
- Node.js：`--max-old-space-size` ≈ `limits` × 0.75
- 留出头部空间给 OS 缓冲、JIT、堆外、sidecar
- **不要不设 limits**：无限增长会占光节点

### 16.4 Requests 与 OOM

- requests 决定调度；limits 决定触发 OOMKill
- 节点真正紧张时，**超过 requests 的 Pod 先被驱逐**
- 设对 requests 对稳定性至关重要

### 16.5 排障 Checklist

1. 容器退出码 = 137 → 几乎肯定是 OOMKilled
2. `kubectl describe pod` → Last State / Reason
3. 节点 `dmesg` 看 OOM 记录
4. 进程内部指标：GC / heap / RSS / direct
5. 观察 K8s `container_memory_working_set_bytes`
6. 评估是 **真的堆涨** 还是 **堆外涨**
7. 修复：调代码 / 调参 / 增 limits，但不是无脑加内存

### 📝 笔试题 16-1：容器里 `top` 看到 8 GB 内存，而我的 Pod limits 只有 2 GB，为什么？

老运行时（如旧 JVM、或者没带 cgroup 感知的工具）默认**看到整机内存**。因此：

- JVM 默认 heap 可能设成机器内存 1/4，远超 limits
- 启动就可能 OOMKilled

解法：升级运行时 + 显式设置 `-Xmx`/`GOMEMLIMIT` 等，或传入 `--cpus` / `--memory` 确保工具读 cgroup。

---

## 17. GC 调优原则与实战

### 17.1 调优心法

1. **先测量再动手**：没数据就没诊断
2. **以 SLO 倒推**：先定"可接受的停顿 / 吞吐"
3. **一次一个变量**：乱改参数很难归因
4. **保留基线**：每次改动都要可回滚对比
5. **大部分时候不用调**：默认参数在 80% 场景够用
6. **加堆是最后一招**：先找泄漏和低效分配

### 17.2 JVM 常见调优步骤

1. **开 GC 日志 + JFR**，跑几天收集
2. **GCEasy / MAT** 分析：
   - Young/Old 比例
   - 晋升速率
   - Full GC 频率和原因
   - 最大停顿分位
3. **关注**：
   - 是否频繁 Full GC？→ 老年代过小 / 泄漏
   - Young GC 后立即 Full？→ 晋升失败 / 大对象
   - 停顿超目标？→ 考虑换 G1 → ZGC
4. **参数调整**：堆大小、G1 `MaxGCPauseMillis`、`InitiatingHeapOccupancyPercent`
5. **代码侧**：减少短命对象数量、对象复用、避免大对象
6. **验证**：压测对比

### 17.3 典型 JVM 参数模板

**吞吐优先（批处理）**：

```
-Xms8g -Xmx8g
-XX:+UseParallelGC
-XX:MaxGCPauseMillis=500
-XX:ParallelGCThreads=8
-XX:+PrintGCDetails
```

**在线服务（低停顿）**：

```
-Xms8g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/oom.hprof
-Xlog:gc*,gc+heap=info:file=/logs/gc.log:time,tags:filecount=10,filesize=100m
```

**超大堆低延迟**（JDK 17+）：

```
-Xms16g -Xmx16g
-XX:+UseZGC
-XX:+ZGenerational           # JDK 21+
```

### 17.4 代码层面减轻 GC 压力

- **复用对象 / 对象池**（慎用，可能适得其反）
- **减少字符串拼接**：`StringBuilder`
- **预分配集合容量**
- **用数组替代大量小对象**（数据局部性 + GC 少）
- **避免 `autoboxing`**（`List<Long>` 频繁 boxing）
- **大数据集用 `stream + flatMap`** 而非一次性 `collect(toList)`
- **懒初始化** 大静态对象

### 17.5 其它语言调优要点

- **V8 / Node**：控制 old space；大对象分批处理；避免闭包捕获大对象
- **Python**：多进程优于多线程；定期重启 worker（Gunicorn `max_requests`）；必要时换 jemalloc
- **Go**：调 `GOGC`、`GOMEMLIMIT`；优化分配（`sync.Pool`、复用 slice/byte buffer）；减少 goroutine 数
- **.NET**：启用 Server GC；LOH 碎片处理；`GC.TryStartNoGCRegion` 关键路径

### 📝 笔试题 17-1：服务 P99 偶尔抖到 2s，GC 日志显示一次 1.8s 的 Full GC，如何处理？

- **短期**：
  - 开 `-XX:+HeapDumpOnOutOfMemoryError` 先把 dump 留下
  - 适度加堆，降低 Full GC 频率
- **诊断**：
  - MAT 看是否泄漏
  - 看是否 "promotion failure"（老年代碎片）
- **换 GC**：
  - G1 → ZGC / Shenandoah，停顿降到亚毫秒
- **代码**：
  - 拆大对象、减少瞬时 burst（批处理分片）
  - 禁用 `System.gc()`
- **长期**：SLO + 持续监控，避免"偶尔抖"变"经常抖"

---

## 18. 反模式与最佳实践

### 18.1 常见反模式

❌ **无上限缓存**：`new HashMap()` 作永久缓存
❌ **静态 List 累积**：静态字段持续 add
❌ **ThreadLocal 不 remove**：线程池环境必炸
❌ **每请求 `new` 大对象**：浪费 GC
❌ **滥用 `String.intern()`**：常量池膨胀
❌ **在热点路径放 `System.gc()`**：无差别 Full GC
❌ **"宁可多不可少"设 `Xmx`**：堆大 = GC 时间长
❌ **用 `shared_ptr` 满天飞**：循环引用、开销、不如 `unique_ptr`
❌ **忽视堆外内存**：只看堆不看 native
❌ **容器无 limits**：一挂挂全机
❌ **只看平均内存不看增长率**：慢泄漏藏得深

### 18.2 最佳实践

- ✅ **缓存加上限 + TTL + 命中统计**
- ✅ **资源自动释放**（try-with-resources / with / defer / Drop）
- ✅ **事件 / 回调配对注册/解绑**
- ✅ **监控多维**：RSS、堆、GC、FD、线程、Direct
- ✅ **生产常备 heap dump 流程**
- ✅ **压测 + 长跑（soak test）** 发现慢泄漏
- ✅ **GC 日志永久开启**：小磁盘成本，大排障价值
- ✅ **"每服务 runbook"**：OOM 时按步骤定位
- ✅ **容器 limits 合理留白**：给堆外与 OS

### 18.3 心智模型

```
内存问题三问：
1. 到底涨了哪部分？（堆？堆外？FD？线程？）
2. 谁在持有它？（引用链 / allocator）
3. 业务上它应该还活着吗？（逻辑 vs 物理泄漏）
```

### 📝 笔试题 18-1：一位同学总坚持"加大 `-Xmx` 就能解决 OOM"，你怎么引导？

先反问：

1. **什么 OOM**？Heap / Metaspace / Direct？`-Xmx` 只管堆
2. **为什么堆变满**？做了对比 heap dump 吗？找到泄漏对象了吗？
3. **加大真能消除问题还是拖延**？有时候只是把 2 小时挂掉变成 2 天挂掉
4. **GC 成本**：堆越大 STW 越长，P99 可能更差
5. **容器限额**：加 `-Xmx` 也要相应调 limits，否则触发 OOMKilled

先**定位再处方**，`-Xmx` 是工具不是答案。

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 下列哪个**不是** GC Root？
A. 活跃线程栈里的局部变量
B. 静态字段
C. JNI Native 引用
D. 已经不可达但未释放的对象

<details><summary>答案</summary>D。</details>

**Q2** GC 语言不可能出现内存泄漏吗？
A. 是
B. 不是，"逻辑泄漏"仍常见
C. 只要关 `gc` 开关就不泄
D. 只有 Java 会

<details><summary>答案</summary>B。</details>

**Q3** CPython 为什么仍需循环检测 GC？
A. 兼容老代码
B. 引用计数无法回收循环引用
C. 纯属性能优化
D. 用于调试

<details><summary>答案</summary>B。</details>

**Q4** JVM 中触发 Full GC 最不常见的原因？
A. 老年代空间不足
B. Metaspace 溢出
C. Young 空间不足
D. `System.gc()` 被调用

<details><summary>答案</summary>C（Young 不足触发 Minor GC）。</details>

**Q5** 关于 ZGC，错误的是？
A. 着色指针 + 读屏障
B. 停顿亚毫秒级
C. 吞吐通常略低于 G1
D. 不支持分代

<details><summary>答案</summary>D（JDK 21+ 支持分代）。</details>

**Q6** Go 语言防止内存无限增长的关键机制？
A. 只增 GC 频率
B. 手动 `free`
C. `GOGC` + `GOMEMLIMIT` 控制堆目标与上限
D. 缓冲池

<details><summary>答案</summary>C。</details>

**Q7** Node.js 内存问题首先应看？
A. CPU 使用率
B. `process.memoryUsage()` + Heap Snapshot 对比
C. 磁盘 IO
D. 网络

<details><summary>答案</summary>B。</details>

**Q8** 哪种**不是**典型的堆外内存？
A. DirectByteBuffer
B. Metaspace
C. Young 代
D. JNI 分配的 C 内存

<details><summary>答案</summary>C。</details>

### 19.2 判断题

1. `System.gc()` 在生产代码中是好习惯。 ❌
2. ThreadLocal 用完一定要 remove。 ✅
3. Go 有 GC，所以不会 goroutine 泄漏。 ❌
4. `shared_ptr` 可防所有 C++ 内存泄漏。 ❌（循环引用 / 误用）
5. 容器内 `top` 看到的内存永远等于 cgroup 限额。 ❌
6. G1 的停顿时间目标是硬保证。 ❌（是期望，非承诺）
7. ZGC 适合所有 Java 应用。 ❌（吞吐/内存开销并非总优）
8. 内存曲线稳定回落才算"无泄漏"。 ✅

### 19.3 简答题

**Q1** 描述排查"Java 服务内存缓慢上涨"的完整流程。

1. 监控曲线：确认是单调上涨还是波动
2. GC 日志 / JFR：看 Full GC 频率、老年代占用趋势
3. 压测复现 → 取对比 heap dump
4. Eclipse MAT 分析 Dominator Tree / GC Root path
5. 排除嫌疑对象：集合 / ThreadLocal / 静态字段 / 类加载器
6. 检查堆外：NMT 基线对比、Metaspace、DirectMemory
7. 修复 + 回归：再次长跑验证曲线

**Q2** 为什么说"分代假说"是分代 GC 的基石？

分代 GC 对年轻代高频回收、老年代低频回收，若对象**不是多半早亡**，这个策略就失效：

- 对象集体长寿 → 年轻代存活多，Minor GC 效率低，频繁晋升
- 对象寿命双峰分布 → 分代两头夹
- 很多流计算、大量 buffer 的系统就会出现这种情况，需要 **ZGC/Shenandoah 非分代**或**更大堆**

**Q3** GC 暂停 vs 吞吐量的权衡，举例说明。

- **Parallel GC**：吞吐高但 STW 长，适合离线批处理
- **G1**：平衡
- **ZGC/Shenandoah**：停顿极短，吞吐略低、内存占用略高，适合延迟敏感在线服务

选型取决于业务目标：**分析/报表 → 吞吐优先**，**API/交易 → 停顿优先**。

**Q4** 你的 Go 服务 RSS 持续涨，GC 看起来正常，可能是什么？

优先怀疑：

- **goroutine 泄漏**（看 `/debug/pprof/goroutine` 数）
- **slice / map 底层数组未释放**（`big[:10]` → 整块存活）
- **cgo 分配的 C 内存不受 GC 管**
- **连接/文件句柄泄漏**（打开不关）
- **sync.Pool 对象太多**

先看 goroutine，再看 heap profile，最后看 cgo 与系统层。

### 19.4 场景分析

**Q1** 容器 limits=2GB，Java 服务设 `-Xmx=2g`，频繁 OOMKilled，你会怎么做？

- `-Xmx` 不能吃掉全部 limits；堆外（Metaspace、Direct、线程、JIT）要留空间
- 建议：`-Xmx=1.3g ~ 1.5g`，留 500-700MB 给堆外
- 开 **NMT** 观察堆外占比，定位大头
- 若代码确实需要更多堆 → 加 limits 为 3-4 GB

**Q2** 某 Node 服务每天凌晨 OOM 重启，白天正常，如何诊断？

- 凌晨有定时任务？大批量拉数据 / 分析？
- 开 `--trace-gc` + `--heap-prof` 记录当晚
- 对比白天和凌晨 heap snapshot，定位新增对象
- 通常是"一次性加载大 JSON / 大报表"未分片
- 修法：流式处理 / 分批 / 限速

**Q3** Python Web 服务内存逐步上涨但 `gc.collect()` 不降，修复思路？

- 区分：Python 对象、C 扩展、arena 碎片
- `tracemalloc` 定位 Python 侧增长点
- `pympler`、`objgraph` 找最大对象类型
- 若是分配碎片：用 **jemalloc** 替代 ptmalloc
- 若是 C 扩展：用 valgrind / ASan 查
- 多进程架构下：Gunicorn `max_requests` + `max_requests_jitter`，让 worker 定期重启保健康

---

## 📚 学习建议

1. **读经典**：
   - 《The Garbage Collection Handbook》（GC 算法圣经）
   - 《Java Performance》（Scott Oaks）
   - 《深入理解 Java 虚拟机》（周志明）
2. **亲手 dump + 分析**：每语言都做一次"制造泄漏 → 排查 → 修复"
3. **看监控**：从今天起开始关心 RSS、GC 曲线，不要等出事
4. **跨语言横向比**：写同一个服务在 JVM/Go/Node/Python 跑起来对比内存特征
5. **出现问题时别猜**：先拿数据，再动代码
6. **GC 参数不是魔法**：好代码比会调参数更重要

> 内存是 CPU 的速度慢 4-5 个数量级的存储。善待它，系统就善待你。
