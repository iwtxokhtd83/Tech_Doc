
# Event Loop 讲义

> 本讲义系统讲解 **事件循环（Event Loop）** 的原理、不同运行时的实现差异（浏览器 / Node.js / Python asyncio / Rust tokio）、调度模型（宏任务/微任务 / Phase / Runnable / Fiber）、以及由此引发的常见陷阱与排障方法。每章配"知识点 + 笔试题"。
>
> 约定：JavaScript 以 Chrome V8 / Node.js 20+ 为主；Python 以 asyncio (3.12+) 为主；原理章节保持运行时中立。

## 目录

1. [为什么需要 Event Loop](#1-为什么需要-event-loop)
2. [核心模型：单线程 + 非阻塞 IO](#2-核心模型单线程--非阻塞-io)
3. [操作系统层：多路复用基础](#3-操作系统层多路复用基础)
4. [浏览器 Event Loop（HTML 规范）](#4-浏览器-event-loophtml-规范)
5. [微任务与宏任务](#5-微任务与宏任务)
6. [渲染时机与 rAF / rIC](#6-渲染时机与-raf--ric)
7. [Node.js Event Loop 与 libuv](#7-nodejs-event-loop-与-libuv)
8. [Node.js 六阶段详解](#8-nodejs-六阶段详解)
9. [process.nextTick vs Promise.then](#9-processnexttick-vs-promisethen)
10. [Worker / Web Worker / Worker Thread](#10-worker--web-worker--worker-thread)
11. [Python asyncio 事件循环](#11-python-asyncio-事件循环)
12. [其他语言运行时：Rust/Go/Swift](#12-其他语言运行时rustgoswift)
13. [异步编程范式对比](#13-异步编程范式对比)
14. [典型陷阱与反模式](#14-典型陷阱与反模式)
15. [排障与观测](#15-排障与观测)
16. [性能影响与优化](#16-性能影响与优化)
17. [面试高频代码题](#17-面试高频代码题)
18. [心智模型总结](#18-心智模型总结)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 为什么需要 Event Loop

### 1.1 问题起点

早期网络 / GUI 程序一个经典问题：**等 IO 时整个程序都卡住**。

```
read socket  ← 阻塞 10ms
render UI    ← 等待
handle click ← 等待
```

如果用"一连接一线程"模式：

- **大量线程**成本高（栈内存、上下文切换）
- 锁与并发 bug 增加
- 线程再多也只能到万级

**C10K 问题**：单机能同时服务多少客户端？纯线程模型扛不住。

### 1.2 Event Loop 的解法

单个或少量线程循环检查"**哪些 IO 就绪、哪些定时器到期、哪些回调可跑**"，立即处理，不浪费 CPU 在等待上。

核心思想：

- **IO 不阻塞**：用 epoll/kqueue/IOCP 等多路复用
- **任务以回调形式排队**：就绪一个处理一个
- **单线程串行执行回调**：无锁、无数据竞争

### 1.3 普适性

Event Loop 不是 JS 专属：

- **浏览器 JS**：HTML spec
- **Node.js**：libuv
- **Python**：asyncio
- **Rust**：tokio / async-std
- **C#**：TaskScheduler
- **Swift**：DispatchQueue / async
- **游戏引擎**：Unity / Unreal 主循环
- **GUI 框架**：Qt / GTK / WinForms
- **数据库客户端、消息队列客户端** 底层多半是事件循环

### 📝 笔试题 1-1：Event Loop 和多线程相比，解决了什么问题？

用**少量线程**服务海量并发 IO：

- 不再按"1 连接 1 线程"
- 没有线程切换 / 锁的开销
- 单线程内天然无数据竞争
- 代价：**一个任务卡住整个循环** —— 所以不能让 CPU 密集 / 阻塞 IO 污染主线程，需要 worker 或线程池卸载

---

## 2. 核心模型：单线程 + 非阻塞 IO

### 2.1 伪代码

```
while (true) {
    tasks = dequeueReady()
    for task in tasks:
        run(task)                    // 执行回调

    if (hasTimersDue()) runDueTimers()

    readyIO = pollIO(timeoutUntilNext())  // 非阻塞/短超时
    for event in readyIO:
        scheduleCallback(event)
}
```

关键词：**单线程轮转，每次只做一件事**。

### 2.2 不是"循环**查**事件"而是"循环**等**事件"

OS 提供的 `epoll_wait` / `kqueue` / `IOCP` 可以**带超时阻塞等待**；没有事件时 **CPU 睡眠**；有事件到就唤醒。

所以事件循环在"空闲"时其实是在内核里睡，**不是 busy-loop**。

### 2.3 可运行任务（Runnable）

任何被调度到事件循环的都是 "Runnable"：

- 用户回调（`setTimeout`、`addEventListener`、`Promise.then`）
- IO 就绪通知
- 定时器到期
- IPC / 消息
- 主动投递的微任务

### 2.4 "一次只跑一个"的含义

- 每个 Runnable 运行到**显式 await / return** 为止，不可抢占
- 所以**长任务** = 所有其他事件延迟
- 这是"单线程并发"和"多线程并行"的根本差别

### 📝 笔试题 2-1："单线程"和"非阻塞 IO"同时成立为什么不矛盾？

- **单线程**说的是 JS / Python 协程侧只有一条执行流
- **非阻塞 IO**说的是 OS 侧内核用多路复用代你监视一堆 FD
- 事件循环的线程去 `epoll_wait` 睡着等通知
- 回调上面的代码当然是单线程串行执行，不代表整条进程什么都不在做

两者配合才是高并发的基础。

---

## 3. 操作系统层：多路复用基础

### 3.1 select / poll / epoll / kqueue / IOCP

- **select**：老，最多 1024 fd，O(n) 扫描
- **poll**：支持任意 fd 但仍 O(n)
- **epoll**（Linux）：事件驱动、O(1) 通知、边缘 / 水平触发
- **kqueue**（BSD / macOS）：类似 epoll
- **IOCP**（Windows）：完成端口，**Proactor 模型**
- **io_uring**（Linux 5.x+）：全新提交/完成环，支持真正的异步 IO

多数事件循环库（libuv、tokio-mio、asyncio-selectors）对上述做了统一抽象。

### 3.2 Reactor vs Proactor

- **Reactor**（epoll/kqueue）：OS 通知 **"你可以读/写了"**，事件循环主动读写
- **Proactor**（IOCP/io_uring）：OS 通知 **"我已经帮你读/写完了"**，直接拿结果

多数 Unix 事件循环是 Reactor；Windows IOCP 是 Proactor；libuv 在 Windows 上用 IOCP，Unix 上用 Reactor + 线程池模拟。

### 3.3 LT vs ET

- **Level-Triggered（水平）**：只要 fd 处于可读状态就反复通知
- **Edge-Triggered（边缘）**：状态变化时通知一次，必须一次读干净

epoll 默认 LT；高性能场景（Nginx）用 ET + 非阻塞循环读。Node.js 通常 LT。

### 3.4 线程池卸载

Reactor 只管"可读/可写"，但有些操作 **本身就是阻塞的**：

- 文件 IO（磁盘的 `read/write` 在 Linux 没有真正异步）
- DNS 查询
- `crypto.pbkdf2` 等 CPU 密集

**策略**：把它们丢到**内部线程池**执行，完成后把结果回到事件循环线程排队。libuv 默认 4 个线程 (`UV_THREADPOOL_SIZE`)。

### 📝 笔试题 3-1：Node.js 号称非阻塞，为什么读文件还要线程池？

Linux 的磁盘 IO 读写 **没有真正的异步系统调用**（io_uring 之前），只能同步阻塞。libuv 在内部用线程池 **模拟异步**：主循环发 JS 回调"丢"给线程池，完成后通过 uv_async 通知主循环。所以 Node 对文件 IO 的"非阻塞"是假象级的并发，不是真正的 OS 异步。

---

## 4. 浏览器 Event Loop（HTML 规范）

### 4.1 核心概念

HTML 规范定义：

- **Task Queue（任务队列 / 宏任务）**：每个来源一个队列（Timer、Network、UI 事件、postMessage…）
- **Microtask Queue（微任务队列）**：处理 Promise、MutationObserver、queueMicrotask
- **Rendering Steps**：合适时机执行布局和绘制

### 4.2 每轮（tick）的流程

简化版：

```
loop:
  1. 从某个任务队列取 **1 个 task**（宏任务）并执行到底
  2. 执行**整个** microtask 队列，直到清空
     （期间新产生的 microtask 也要处理）
  3. 必要时进行渲染（Style → Layout → Paint → Composite）
  4. 进入下一轮
```

### 4.3 宏任务来源

典型：

- `setTimeout` / `setInterval`
- `MessageChannel` / `postMessage`
- UI 事件（click、input）
- Fetch / XHR 的 load 事件
- `<script>` 加载完成
- Worker 消息

### 4.4 微任务来源

- `Promise.then / catch / finally`
- `queueMicrotask(fn)`
- `MutationObserver`
- `process.nextTick`（Node 特有，非浏览器；见第 9 章）

### 4.5 关键口诀

- 一次 tick 只跑 **一个**宏任务
- 跑完后**清空所有** microtask（深度无上限）
- 浏览器可能在此处再做渲染

### 4.6 经典执行顺序例

```js
console.log("A");
setTimeout(() => console.log("B"));     // 宏
Promise.resolve().then(() => console.log("C")); // 微
queueMicrotask(() => console.log("D")); // 微
console.log("E");
// 输出：A E C D B
```

解释：

1. 同步 "A E"
2. 当前宏任务（即 script）结束 → 清空微任务 "C D"
3. 取下一个宏任务（setTimeout）→ "B"

### 📝 笔试题 4-1：为什么 `Promise.then` 比 `setTimeout 0` 先执行？

因为 `then` 回调进微任务队列，当前宏任务完成后会**一次性清空**所有微任务；`setTimeout` 回调进宏任务队列，要等**下一轮**才跑。**微任务优先级高于同期宏任务**。

---

## 5. 微任务与宏任务

### 5.1 定义的"精确"视角

- **Task（宏任务）**：一个"从头到尾跑到底"的工作单元，来自某个任务源
- **Microtask（微任务）**：属于"上一任务的收尾工作"，本质上是把这些工作**追加到当前任务末尾**

重点：**微任务不是一种新宏任务**，而是"当前任务执行完之前"继续跑的。

### 5.2 微任务循环（清空过程）

```
while (microtaskQueue 非空) {
    task = microtaskQueue.shift()
    run(task)          // 期间可能再入队微任务
}
```

因此微任务链能**无限延长**！这种"永不让出"模式会让页面卡死：

```js
function infinite() {
    Promise.resolve().then(infinite);
}
infinite();             // 浏览器卡死但不会 setTimeout 切走
```

### 5.3 对比宏任务

| 维度 | 宏任务 | 微任务 |
|------|--------|--------|
| 何时跑 | 每轮 tick 取一个 | 当前任务完成后清空 |
| 能否插队 | 不能，必须等下轮 | 能（优先于下一个宏任务） |
| 典型来源 | setTimeout / fetch / UI | Promise / queueMicrotask |
| 渲染是否可能插入之间 | 可以 | **不会**（微任务清空前不渲染） |
| 对用户感知 | 较慢 | 即时 |

### 5.4 "微任务不让渲染"的坑

```js
button.onclick = () => {
    state = "loading";
    renderLoadingUI();          // DOM 改了但还没画
    Promise.resolve().then(heavyWork);  // 微任务
    // heavyWork 跑完才有机会渲染 → 看不到 loading
};
```

解法：把 `heavyWork` 放到 **宏任务**（`setTimeout(..., 0)` / `MessageChannel` / `requestIdleCallback`）里。

### 5.5 `queueMicrotask` 的用途

- 显式把某工作推迟到"当前同步代码后、下一个宏任务前"
- 比 `Promise.resolve().then()` 更语义化、更轻量
- 用于**批量 DOM 读写**、状态同步

### 📝 笔试题 5-1：下面输出？

```js
console.log(1);
setTimeout(() => {
    console.log(2);
    Promise.resolve().then(() => console.log(3));
});
Promise.resolve().then(() => {
    console.log(4);
    setTimeout(() => console.log(5));
});
console.log(6);
```

<details><summary>答案</summary>

```
1 6 4 2 3 5
```

执行过程：

1. 主 script 执行 → 1 6；微任务排：then-4；宏任务排：timer-2
2. 清空微任务 → 4；产生新宏任务 timer-5
3. 下一轮宏任务 → timer-2 → 2；微任务排：then-3 → 清空 → 3
4. 再下一轮 → timer-5 → 5
</details>

---

## 6. 渲染时机与 rAF / rIC

### 6.1 浏览器的 1 帧

典型 60Hz ≈ **16.67ms / 帧**。一帧里浏览器想做：

```
|← 输入事件 ←|← 其它宏任务 ←|← rAF 回调 ←|← 布局 ←|← 绘制 ←|← 合成 ←| …空闲 (rIC) … |
```

### 6.2 `requestAnimationFrame`（rAF）

- 下一次**重绘前**执行
- 频率跟随显示器刷新（60/120/144 Hz）
- 常用于动画：帧同步更流畅
- **不等于** `setTimeout(fn, 16)`，后者与刷新不同步

### 6.3 `requestIdleCallback`（rIC）

- 浏览器**空闲**时执行，可传 `{ timeout }`
- 给回调一个 `deadline.timeRemaining()`，可多次让出
- 适合"不紧急的任务"：预取、日志上报

### 6.4 关系图

```
宏任务 → 微任务清空 → rAF 回调 → Layout → Paint → Composite → [空闲] → rIC
```

> 规范细节随版本略有演进，但相对顺序稳定。

### 6.5 `MessageChannel` 常用技巧

`setTimeout(fn, 0)` 在浏览器有最小延迟（1 / 4 ms 不等），`MessageChannel` 更快：

```js
const { port1, port2 } = new MessageChannel();
port2.onmessage = () => doWork();
port1.postMessage(null);        // 立刻发，下一 tick 触发
```

React Scheduler、大量库用它实现"下个 macrotask 立刻执行"。

### 📝 笔试题 6-1：为什么动画建议用 `rAF` 而非 `setInterval`？

- rAF 与显卡刷新同步，动画平滑（无撕裂）
- 页面隐藏时 rAF **自动暂停**，省电
- setInterval 固定间隔 vs 显卡刷新不匹配，容易掉帧或重复帧
- 真动画也应避免改动会触发 Layout 的属性，用 `transform / opacity`

---

## 7. Node.js Event Loop 与 libuv

### 7.1 架构层次

```
应用 JS (V8)
      │
   Node 内置模块 (fs, net, http, crypto, …)
      │
   C++ 绑定
      │
   libuv（事件循环 + 线程池 + 文件/网络/定时器抽象）
      │
   OS (epoll / kqueue / IOCP)
```

### 7.2 libuv 视角的循环

libuv 提供 **uv_loop**，每次 `uv_run` 做一轮：

```
uv_run() {
  update_time()
  run_timers()
  run_pending()       // 上轮未处理的 IO callback
  run_idle()
  run_prepare()
  uv_io_poll(timeout) // epoll_wait
  run_check()
  run_close_cb()
}
```

Node 的"六阶段"就是这段代码面向 JS 开发者的简化表述。

### 7.3 与浏览器 Event Loop 的差异

- Node 没有"渲染阶段"
- 多阶段队列（timers / pending / poll / check / close）
- `setImmediate` / `process.nextTick` 是 Node 专属
- 微任务清空时机：**每次从 C++ 回到 JS 结束**都会清微任务（不再只在宏任务之间）
- 默认单事件循环（**主线程**），CPU 任务要用 `worker_threads`

### 7.4 为何"回到 JS 结束就清微任务"

Node 11+ 对齐浏览器行为，使得：

```
setImmediate 回调 → 自动清微任务
setTimeout 回调  → 自动清微任务
```

而老版本（10 及之前）在同一阶段的多个回调**跑完才清**，会产生微妙的顺序差异。升级后更贴近浏览器直觉。

### 📝 笔试题 7-1：Node 跑一遍"同步 JS"后是直接 exit 吗？

只要**还有定时器 / 打开的 FD / HTTP server / Interval / 未完成 Promise**，libuv 就认为有"活"要干，循环继续。当 **refcount 归零** 时，`uv_run` 返回，进程才退出。

---

## 8. Node.js 六阶段详解

### 8.1 阶段一览

```
┌──────────────────────── Event Loop ────────────────────────┐
│  timers        : setTimeout / setInterval 到期回调          │
│  pending       : TCP/UDP 错误等上轮遗留 IO 回调             │
│  idle / prepare: 内部用                                      │
│  poll          : 取 IO 事件，如没定时器可在此阻塞           │
│  check         : setImmediate 回调                           │
│  close         : 'close' 事件（socket.on('close'))           │
└─────────────────────────────────────────────────────────────┘

每两阶段之间 / 每次 JS 退出前：
  → 清空 process.nextTick 队列
  → 清空 Promise 微任务
```

### 8.2 timers

- `setTimeout(cb, ms)` 和 `setInterval(cb, ms)` 的到期回调在此跑
- `ms` 是**最小**延迟，不保证精确
- 如果超过 1 ms 的回调会被"挤"到下一轮

### 8.3 pending callbacks

- 例如 TCP `ECONNREFUSED` 这类错误可能在此阶段跑

### 8.4 poll（核心）

- 计算出等待时间：若 timer/immediate 还没到，可睡这么久
- 执行 IO callback
- 如果 `setImmediate` 有活，**不会**阻塞 poll
- 一般花时间最多的就是这里

### 8.5 check

- **`setImmediate`** 回调专属阶段
- 保证在 poll 之后立即执行，区别于 timers

### 8.6 close callbacks

- 关闭资源时触发的 `'close'` 事件

### 8.7 setTimeout 0 vs setImmediate

主脚本里执行：

```js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
```

顺序**不确定**：取决于循环启动那一刻 timer 是否已"到期"。

但在 **IO 回调中**：

```js
fs.readFile(__filename, () => {
    setTimeout(() => console.log("timeout"), 0);
    setImmediate(() => console.log("immediate"));
});
```

结果**总是**：

```
immediate
timeout
```

因为 IO 回调在 poll 阶段跑完后，接下来就是 check 阶段 → `setImmediate` 先运行，而 timer 要等下一轮 timers。

### 📝 笔试题 8-1：`setImmediate` 和 `setTimeout(fn, 0)` 哪个更"立即"？

- **在 IO 回调里**：`setImmediate` 一定更早（下一个阶段 check）
- **在其它位置（主脚本、微任务）**：**不确定**，实测常常 setTimeout 先（取决于事件循环刚启动时是否已超过最小时延）

想立刻排下一个 tick：**IO 回调里用 setImmediate**，其它地方 `queueMicrotask` / `process.nextTick` 更快。

---

## 9. process.nextTick vs Promise.then

### 9.1 两者都是"微任务级"

但在 Node 里它们有**不同的队列**：

- `process.nextTick` → **nextTick queue**
- `Promise.then / queueMicrotask` → **Promise microtask queue**

### 9.2 执行顺序

> **nextTick 优先于 Promise**

Node 每次从 C++ 回到用户 JS 结束时：

1. 清空 **nextTick 队列**
2. 清空 **Promise 微任务队列**

例：

```js
setTimeout(() => {
    console.log("timeout");
    Promise.resolve().then(() => console.log("promise in timeout"));
    process.nextTick(() => console.log("nextTick in timeout"));
});
// 输出：
// timeout
// nextTick in timeout
// promise in timeout
```

### 9.3 优先级高也意味着危险

`process.nextTick` 会阻塞后续的任何任务（包括 IO）：

```js
function recurse() {
    process.nextTick(recurse);
}
recurse();
// IO 再也跑不到，服务器假死
```

官方建议**默认用** `queueMicrotask` / `Promise.then`，只有在**必须先于 Promise 微任务**时再用 `nextTick`。

### 9.4 经典用例

- 错误抛出延后（`EventEmitter` 内部让 error 同步抛→异步通知）
- 保证回调在 **当前操作完全结束后**触发，给消费者一致的异步行为

### 📝 笔试题 9-1：输出？

```js
setImmediate(() => console.log("immediate"));
process.nextTick(() => console.log("nextTick"));
Promise.resolve().then(() => console.log("promise"));
console.log("sync");
```

<details><summary>答案</summary>

```
sync
nextTick
promise
immediate
```

同步先；从 JS 退出后 nextTick 队列优先于 Promise；check 阶段跑 setImmediate。
</details>

---

## 10. Worker / Web Worker / Worker Thread

### 10.1 事件循环不够用时怎么办

**CPU 密集**或**必须阻塞**的任务不能进主事件循环。解法：

- 浏览器：**Web Worker** / **Service Worker** / **SharedWorker**
- Node.js：**worker_threads** / `child_process`
- Python：`concurrent.futures` / `multiprocessing`
- Rust：tokio multi-thread runtime / rayon

### 10.2 Web Worker

- 独立 JS 线程，**独立事件循环**
- 通过 `postMessage` 传**结构化克隆**（或 Transferable）
- 不能直接访问 DOM
- 适合：加解密、大数据解析、图像处理、复杂计算

```js
// main.js
const w = new Worker('./worker.js');
w.postMessage({ task: 'heavy', data });
w.onmessage = (e) => console.log('result', e.data);

// worker.js
onmessage = (e) => {
    const r = heavyCompute(e.data);
    postMessage(r);
};
```

### 10.3 Node.js worker_threads

```js
const { Worker } = require('node:worker_threads');
new Worker('./heavy.js', { workerData: { n: 1e9 } });
```

- 每个 worker 有自己的 V8 实例、事件循环
- 通过 `MessageChannel` 通信
- 支持 **Atomics / SharedArrayBuffer** 共享内存
- 比 `child_process` 轻量，适合 CPU 密集；IO 密集仍推主线程异步

### 10.4 关键结论

**单事件循环擅长 IO 密集，不擅长 CPU 密集**。

- 加密一个大 JSON / 解压大文件 → worker
- 排序 100 万条 → worker
- 读 10k 个 URL → 主线程异步就够了

### 📝 笔试题 10-1：主线程一次循环 10 秒做完 CPU 任务，Web Worker 能加速吗？

视任务可并行程度：

- 纯顺序计算：移到 Worker 不会加速，但**不再卡主线程**，页面仍响应
- 可并行：多 Worker 分片加速，近似线性（Amdahl 定律约束）
- **真正目标通常是"释放主线程"**，而非总时长减半

---

## 11. Python asyncio 事件循环

### 11.1 基本模型

- `asyncio.run(main())` 启动事件循环
- `async def` 定义协程；`await` 让出执行权
- 底层依赖 **selectors**（epoll/kqueue/IOCP）

```python
import asyncio

async def main():
    await asyncio.gather(task("a", 1), task("b", 2))

async def task(name, delay):
    await asyncio.sleep(delay)
    print(name)

asyncio.run(main())
```

### 11.2 协程调度

- `await` 把协程挂起，把"等待的 future"注册到事件循环
- 事件循环在 `selectors.select()` 等多路复用
- 就绪后恢复协程，从 `await` 继续

### 11.3 Task / Future

- **coroutine**：函数调用后得到的对象，**本身不会执行**，需被驱动
- **Task**：包了一层可被事件循环调度的对象
- **Future**：低层占位，表示"未完成计算"

`asyncio.create_task(coro())` 把协程丢给事件循环并发执行。

### 11.4 阻塞调用破坏事件循环

```python
async def bad():
    time.sleep(1)        # ❌ 会卡整个 loop
    response = requests.get(url)   # ❌ 同步阻塞库
```

正确：

- 用 `await asyncio.sleep(1)`
- 用 `aiohttp` / `httpx.AsyncClient`
- 只能用同步库时：`await asyncio.to_thread(func, args)` 丢线程池

### 11.5 常用 API

```python
await asyncio.gather(a(), b(), c())        # 并发
await asyncio.wait_for(task, timeout=5)    # 带超时
await asyncio.shield(task)                 # 保护不被取消
async with asyncio.TaskGroup() as tg:      # Python 3.11+
    tg.create_task(a())
    tg.create_task(b())
```

### 11.6 循环实现选择

- 默认 **SelectorEventLoop**（各平台不同 selector）
- **uvloop**：基于 libuv 的替代，性能显著提升，Sanic / FastAPI 常用：

```python
import uvloop; uvloop.install()
```

### 11.7 GIL 的影响

- Python 的 GIL 让"多线程 CPU 并行"不成立
- 但**异步 IO 仍然并发**（IO 期间释放 GIL）
- 3.13+ 有实验性 no-GIL 构建（PEP 703）

### 📝 笔试题 11-1：Python 协程中 `time.sleep(1)` 会发生什么？

**会完全阻塞事件循环 1 秒**。所有协程都无法推进，网络请求、定时任务全停。必须用 `await asyncio.sleep(1)`。同样 `requests`、同步 DB 驱动等都要替换为异步版或用 `to_thread` 卸载。

---

## 12. 其他语言运行时：Rust/Go/Swift

### 12.1 Rust (tokio)

- `async fn` + `.await`
- 编译时生成状态机（零成本抽象）
- 需要一个**运行时（runtime）** 驱动：
  - `tokio`：多线程工作窃取 + 每 worker 独立事件循环
  - `async-std` / `smol`：更轻量
- IO 底层基于 `mio`（epoll/kqueue/IOCP 抽象）

示例：

```rust
#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(fetch("x"), fetch("y"));
}
```

特色：

- 编译期保证"无数据竞争"
- `Send + Sync` 区分跨线程安全
- **阻塞函数** 放 `tokio::task::spawn_blocking`

### 12.2 Go

Go 用 **goroutine**（由 runtime 调度的用户态线程）和 **GMP 模型**（Goroutine / Machine / Processor），**不是传统 Event Loop 模型**：

- 你写的看起来是同步阻塞代码
- runtime 在 syscall 可阻塞时把 goroutine 移开，让其它 goroutine 用这个 OS 线程
- 对程序员 **透明的异步**

但底层 netpoller 仍使用 epoll/kqueue。Go 选择了更高级的抽象把"事件循环"内化进 runtime。

### 12.3 Swift / 苹果生态

- 传统 **Grand Central Dispatch (GCD)**：串行/并行队列 + 闭包
- Swift Concurrency (5.5+)：`async`/`await`、Actor、Task
- 应用主线程有 **RunLoop**（UI 事件循环）

### 12.4 GUI / 游戏引擎

- Qt：`QEventLoop` 支持嵌套
- GTK：`GMainLoop`
- Unity / Unreal：帧循环 + 消息派发
- 都与 IO 事件循环同源，结构相似

### 📝 笔试题 12-1：Go 没有 `await`，凭什么说它"异步"？

Go 的 **runtime 调度器**处理了一切。当 goroutine 里发生 `net.Read` 会调 `read` 系统调用：

- 如能立即完成：直接返回
- 若被阻塞：runtime 把该 goroutine 挂起，执行其它 goroutine；等底层 netpoller（epoll）通知该 fd 可读时再恢复

程序员感觉是"同步"，实际是"抢占式调度 + Reactor"合力的效果。

---

## 13. 异步编程范式对比

| 范式 | 代表 | 心智 |
|------|------|------|
| **回调** | 早期 Node API | 层层嵌套（callback hell） |
| **Promise / Future** | ES6、Java Future、.NET Task | 链式 `.then`，组合方便 |
| **async/await** | JS / Python / C# / Rust | 同步风格写异步 |
| **Reactive** | RxJS、Reactor、Akka Streams | 数据流 + 背压 |
| **协程 + Channel** | Go、Kotlin、Elixir | 消息传递 |
| **Actor** | Erlang、Akka | 状态隔离 + 邮箱 |
| **Fiber / 绿色线程** | Java Loom、Ruby Fiber | 同步 API 表象 + 运行时调度 |

### 13.1 Promise / Future 组合

```js
Promise.all([a(), b()])             // 全部
Promise.race([a(), b()])            // 任一先返回
Promise.allSettled([a(), b()])      // 结果均带 status
Promise.any([a(), b()])             // 任一成功；全失败抛 AggregateError
```

### 13.2 async/await 本质

```js
async function f() {
    const a = await g();
    return a + 1;
}
```

等价于：

```js
function f() {
    return g().then(a => a + 1);
}
```

`await` 就是"暂停协程，注册 `then`"。

### 13.3 背压（Backpressure）

异步中生产快于消费时必须有限流：

- Node Streams 的 `highWaterMark`
- Reactive Streams 标准 `request(n)`
- Go channel 带缓冲上限
- 手动：Semaphore / Queue 限制并发

### 📝 笔试题 13-1：`await` 会"阻塞"事件循环吗？

**不会**。`await expr` 把当前协程挂起，把控制权**交回事件循环**处理其它任务；等到 `expr` resolve，再恢复该协程。它"阻塞的是协程自身"，对进程/线程不阻塞。

只有**同步阻塞调用**（`readFileSync`、`time.sleep`、`while(true){}`）才真正阻塞事件循环。

---

## 14. 典型陷阱与反模式

### 14.1 长任务阻塞主线程

```js
button.onclick = () => {
    for (let i = 0; i < 1e8; i++) sum += i;
    // 期间页面全卡
};
```

**对策**：拆任务 / Web Worker / `requestIdleCallback`。

### 14.2 `setTimeout(fn, 0)` ≠ 立即

浏览器最小延迟（嵌套 4 次以上强制 4ms）；Node 也有最小 1ms。**不要依赖"立刻执行"**。

### 14.3 微任务饥饿宏任务

```js
function loop() { Promise.resolve().then(loop); }
loop();
```

微任务无限递归会让：UI 不更新、`setTimeout` 永不触发、浏览器可能终止页面。

### 14.4 同步阻塞 API 混入

```js
const data = fs.readFileSync(big);   // Node，阻塞
time.sleep(2);                       // Python 协程
```

在事件循环里放同步阻塞 = 自挖墙脚。

### 14.5 循环里 await 丢失并发

```js
for (const u of urls) {
    await fetch(u);      // 串行
}
// 想并发：
await Promise.all(urls.map(fetch));
```

### 14.6 未处理的 Promise 拒绝

```
UnhandledPromiseRejection
```

现代 Node 会**终止进程**（`--unhandled-rejections=strict`）。要 `.catch(...)` 或 `try/await`。

### 14.7 错误的 try/catch

```js
try {
    setTimeout(() => { throw new Error(); }, 0);  // 捕不到
} catch (e) {}
```

异步异常必须在**回调自己内部**捕获，或用 `process.on('uncaughtException')` 兜底（谨慎）。

### 14.8 Node `process.nextTick` 递归

见第 9 章—饿死 IO。

### 14.9 async 函数返回 undefined

```js
async function fn() { setTimeout(...); }    // 忘返回
await fn();                                 // 实际没等 setTimeout
```

记得返回 Promise 或把 setTimeout 包成 Promise。

### 📝 笔试题 14-1：`forEach` 里用 `await` 为什么失效？

`Array.prototype.forEach` 不识别返回的 Promise，回调函数的 `await` 对外层不起作用——外面的代码会立即继续执行，不会等任何一条完成。

修法：用 `for...of`（串行）或 `Promise.all(map(...))`（并发）。

---

## 15. 排障与观测

### 15.1 浏览器

- **Performance 面板**：火焰图看主线程长任务
- **Long Tasks API**：
  ```js
  new PerformanceObserver(list => {
      list.getEntries().forEach(e => console.log(e.duration));
  }).observe({ type: 'longtask', buffered: true });
  ```
- **INP** / **FID** 核心 Web Vitals 反映响应度

### 15.2 Node.js

- `node --inspect` + Chrome DevTools → Performance / Memory
- `process.monitorEventLoopDelay`（Node 10+）测循环延迟
- `perf_hooks`：高精度测量
- **clinic.js**：doctor / flame / bubbleprof
- `--trace-warnings`、`--async-stack-traces`

示例：

```js
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
    console.log('p99 ms', (h.percentile(99) / 1e6).toFixed(2));
    h.reset();
}, 1000);
```

### 15.3 Python

- `asyncio.get_event_loop().slow_callback_duration = 0.1` → 记录超过 100ms 的回调
- `loop.set_debug(True)` 开调试模式（发现未 await 协程、同步阻塞）
- `aiomonitor` / `aiodebug` 第三方
- `py-spy` 采样火焰图

### 15.4 常见症状

- **CPU 100% 页面不响应** → 主线程长任务
- **Node 响应延迟抖动** → 某回调阻塞 / 某 GC / 线程池耗尽
- **Python 忽然全停**：某协程调用同步库
- **打点显示"定时器滞后"**：事件循环被更高优先级塞满

### 📝 笔试题 15-1：如何定位 Node "请求变慢"的问题？

1. 测 **event loop lag**（`monitorEventLoopDelay`）：是 CPU/阻塞导致循环卡顿？
2. 看 **GC 日志**：大对象 / 频繁 GC？
3. 看 **IO / DB**：下游慢？
4. 看 **线程池**：`UV_THREADPOOL_SIZE`（默认 4）是否被 DNS / crypto / fs 打满？
5. clinic.js 出火焰图定位热点
6. 必要时把 CPU 任务挪到 worker_threads

---

## 16. 性能影响与优化

### 16.1 事件循环的核心成本

- **单个任务过长** → 所有其它事件卡
- **任务数过多** → 队列堆积，延迟
- **微任务过多** → 卡住渲染 / IO

### 16.2 优化思路

- **拆任务**：长计算切片（每片 < 50ms）
- **批量合并**：多次 DOM 写合并（`requestAnimationFrame`）
- **异步化 + 并发**：`Promise.all`
- **让出控制**：`await null` / `queueMicrotask` / `setImmediate`
- **卸载到 Worker**：CPU 密集
- **调整 libuv 线程池**：`UV_THREADPOOL_SIZE`（对 fs/crypto/DNS 有效）

### 16.3 "让出"技巧示例

```js
async function chunk(items, work, slice = 50) {
    for (let i = 0; i < items.length; i++) {
        work(items[i]);
        if (i % slice === 0) await new Promise(r => setTimeout(r));
        // 或 queueMicrotask / MessageChannel
    }
}
```

React 18 调度器就是做这件事：把 render 切片，交给 Scheduler 决定何时继续。

### 16.4 "调度优先级"的工业实践

- 浏览器提案 **Prioritized Task Scheduling API**（`scheduler.postTask`）：`user-blocking` / `user-visible` / `background`
- React 并发模式：transition / priority
- Android Looper：按 Handler 优先级

### 📝 笔试题 16-1：30,000 条数据渲染导致页面卡死，如何优化？

- **虚拟列表**：只渲染视口内，库如 `react-window`
- **分片**：每帧处理 100 条 + `requestAnimationFrame`
- **Web Worker** 预处理（过滤、排序）
- **骨架屏 + 懒加载**：先让用户看到部分
- **DOM 操作批量化 + 使用 `DocumentFragment`**
- **避免 Reflow**：批量读写分离

---

## 17. 面试高频代码题

### 17.1 顺序判断

```js
async function async1() {
    console.log('A1 start');
    await async2();
    console.log('A1 end');
}
async function async2() {
    console.log('A2');
}
console.log('script start');
setTimeout(() => console.log('timeout'), 0);
async1();
new Promise(resolve => {
    console.log('promise1');
    resolve();
}).then(() => console.log('promise2'));
console.log('script end');
```

输出：

```
script start
A1 start
A2
promise1
script end
A1 end
promise2
timeout
```

### 17.2 并发限制器

```js
async function parallelLimit(tasks, limit) {
    const results = [];
    const executing = new Set();
    for (const [i, task] of tasks.entries()) {
        const p = Promise.resolve().then(() => task());
        results.push(p);
        executing.add(p);
        p.finally(() => executing.delete(p));
        if (executing.size >= limit) await Promise.race(executing);
    }
    return Promise.all(results);
}
```

### 17.3 自实现 sleep / nextTick

```js
const sleep = ms => new Promise(r => setTimeout(r, ms));
const nextTick = () => new Promise(r => queueMicrotask(r));
```

### 17.4 异步重试 + 退避

```js
async function retry(fn, { max = 3, base = 200, factor = 2 } = {}) {
    let attempt = 0;
    while (true) {
        try { return await fn(); }
        catch (e) {
            if (++attempt >= max) throw e;
            const delay = base * Math.pow(factor, attempt) + Math.random() * 100;
            await new Promise(r => setTimeout(r, delay));
        }
    }
}
```

### 17.5 AbortController 级联取消

```js
const ctrl = new AbortController();
const res = await fetch(url, { signal: ctrl.signal });
// 某处：
setTimeout(() => ctrl.abort('timeout'), 3000);
```

### 17.6 async generator 限流流

```js
async function* throttled(stream, rate) {
    for await (const item of stream) {
        yield item;
        await new Promise(r => setTimeout(r, 1000 / rate));
    }
}
```

### 📝 笔试题 17-1：解释下面 Node 代码输出

```js
setImmediate(() => console.log(1));
Promise.resolve().then(() => console.log(2));
process.nextTick(() => console.log(3));
setTimeout(() => console.log(4), 0);
console.log(5);
```

<details><summary>答案</summary>

```
5
3
2
4
1
```

- 5：同步
- 3：JS 结束后清 nextTick
- 2：清 Promise 微任务
- 4：下一轮 timers
- 1：再下一轮 check

timers 和 setImmediate 的先后在**无 IO 的主脚本**里不完全确定，但 Node 一般让 timers 先。
</details>

---

## 18. 心智模型总结

### 18.1 一句话版本

**Event Loop = 一条循环流水线，每次取一个"待办事项"跑，跑完把收尾工作（微任务）处理干净，必要时渲染/IO，再取下一个。**

### 18.2 三条核心纪律

1. **不要阻塞事件循环**：CPU 任务卸载，别 `sleep`
2. **不要让微任务饥饿宏任务**：别搞自调用的 Promise 链
3. **拆长任务**：> 50ms 就是长任务，影响响应

### 18.3 四类选手

| 角色 | 来源 | 何时跑 |
|------|------|--------|
| 宏任务 | setTimeout / IO / UI | 一轮一个 |
| 微任务 | Promise / queueMicrotask | 每个宏任务结束后全部清空 |
| nextTick（Node） | process.nextTick | 高于 Promise 微任务 |
| Idle | requestIdleCallback | 空闲时 |

### 18.4 浏览器 vs Node 对照

| 维度 | 浏览器 | Node |
|------|--------|------|
| 引擎 | V8 + Blink | V8 + libuv |
| 渲染 | 有 | 无 |
| 微任务 | Promise / queueMicrotask / MutationObserver | + process.nextTick |
| 宏任务 | 通用队列 + UI | 六阶段队列 |
| IO 模型 | Reactor | Reactor (Unix) / Proactor (Win) |
| 线程扩展 | Web Worker | worker_threads |

### 18.5 各语言共同模式

- **单线程事件循环** + **线程池卸载阻塞**
- **async/await** 用状态机实现
- **多路复用**下沉到 OS（epoll/kqueue/IOCP）
- 真正多核靠 worker / goroutine / task pool

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 下列**不属于**微任务的是？
A. Promise.then  B. queueMicrotask  C. setTimeout(fn,0)  D. MutationObserver

<details><summary>答案</summary>C。</details>

**Q2** Node.js 六阶段里 setImmediate 在哪个阶段？
A. timers  B. poll  C. check  D. close

<details><summary>答案</summary>C。</details>

**Q3** process.nextTick 的特点？
A. 跟 Promise.then 完全等价
B. 优先于 Promise 微任务
C. 属于宏任务
D. 与 setImmediate 同阶段

<details><summary>答案</summary>B。</details>

**Q4** 浏览器一帧里哪个最先发生？
A. Paint
B. 微任务清空
C. rAF 回调
D. rIC

<details><summary>答案</summary>B（每个宏任务结束后先清微任务）。</details>

**Q5** `setTimeout(fn, 0)` 可以"立即"执行吗？
A. 一定立即
B. 至少下一轮宏任务，且有最小延迟
C. 比 Promise.then 快
D. 与 process.nextTick 等价

<details><summary>答案</summary>B。</details>

**Q6** Python 里 `time.sleep(1)` 用在协程里会？
A. 正常让出
B. 完全阻塞事件循环
C. 与 await asyncio.sleep 等价
D. 仅阻塞当前协程

<details><summary>答案</summary>B。</details>

**Q7** Node 的 libuv 为什么需要线程池？
A. CPU 密集加速
B. 磁盘 IO 无真正异步接口，用线程模拟；也用于 DNS / crypto
C. 只为 console.log
D. 用于 GC

<details><summary>答案</summary>B。</details>

**Q8** 下列关于 `Worker` 说法错误的是？
A. 独立事件循环
B. 可直接访问 DOM
C. 通过 postMessage 通信
D. 适合 CPU 密集任务

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. Event Loop 只存在于 JavaScript。 ❌
2. 微任务清空期间浏览器会渲染。 ❌
3. await 会阻塞整个线程。 ❌
4. setTimeout(fn, 0) 和 setImmediate 等价。 ❌
5. Node `process.nextTick` 可以饿死 IO。 ✅
6. Go 的异步用事件循环暴露给开发者。 ❌（runtime 隐藏）
7. 长任务超过 50ms 会影响 INP。 ✅
8. `Promise.all` 会串行执行任务。 ❌

### 19.3 简答题

**Q1** 简述浏览器 Event Loop 一轮的完整流程。

```
取一个宏任务 → 执行 → 清空全部微任务（含新增）
→ 必要渲染（Style/Layout/Paint/Composite） → 下一轮
```

**Q2** Node.js 为什么引入 `process.nextTick`？它和 Promise.then 的先后？

为了让 EventEmitter 等 API 能把"同步抛错误"改成"异步通知"而不依赖 Promise。优先级高于 Promise 微任务，所以 Node 在每次 JS 出栈时先清 nextTick 队列，再清 Promise 队列。

**Q3** 如何检测事件循环是否被阻塞？

- 浏览器：Performance 面板长任务、PerformanceObserver('longtask')
- Node：`monitorEventLoopDelay` 测延迟分位
- Python：`loop.set_debug(True)` + `slow_callback_duration`
- 应用层：埋点对比"预期时延 vs 实际"

**Q4** 为什么不能在 async 函数里 `time.sleep` / `readFileSync`？

这些调用会在 OS 线程上真正阻塞；事件循环此时无法处理其它任务，导致所有并发协程全部停滞。正确：用异步版（`asyncio.sleep` / `fs.promises.readFile`）或 `to_thread` / worker 卸载。

### 19.4 编程题

**Q1** 写一个 `nextMacrotask()`，让调用者可以等到下一个宏任务再继续（浏览器）。

```js
function nextMacrotask() {
    return new Promise(resolve => {
        const { port1, port2 } = new MessageChannel();
        port1.onmessage = () => resolve();
        port2.postMessage(null);
    });
}
// await nextMacrotask();
```

**Q2** 实现一个能限制并发的 `fetchAll(urls, concurrency)`。

```js
async function fetchAll(urls, concurrency = 5) {
    const ret = [];
    let i = 0;
    async function worker() {
        while (i < urls.length) {
            const idx = i++;
            ret[idx] = await fetch(urls[idx]).then(r => r.text());
        }
    }
    await Promise.all(Array.from({ length: concurrency }, worker));
    return ret;
}
```

**Q3** 分析并修正：

```js
async function run() {
    const ids = [1, 2, 3];
    ids.forEach(async (id) => {
        await process(id);
    });
    console.log('done');
}
```

问题：`forEach` 不等内部 `await`，`'done'` 在所有 process 完成前就打印。修正：

```js
async function run() {
    for (const id of [1, 2, 3]) await process(id);    // 串行
    // 或并发:
    // await Promise.all([1,2,3].map(process));
    console.log('done');
}
```

**Q4** 用 Python 写一个并发抓 100 个 URL、最大并发 10、带超时 3s、失败自动重试 2 次的函数。

```python
import asyncio, aiohttp
from asyncio import Semaphore

async def fetch_one(session, url, sem, timeout=3, retries=2):
    for attempt in range(retries + 1):
        try:
            async with sem, session.get(url, timeout=timeout) as r:
                return await r.text()
        except Exception as e:
            if attempt == retries:
                raise
            await asyncio.sleep(0.2 * (2 ** attempt))

async def fetch_all(urls, concurrency=10):
    sem = Semaphore(concurrency)
    async with aiohttp.ClientSession() as session:
        return await asyncio.gather(
            *[fetch_one(session, u, sem) for u in urls],
            return_exceptions=True
        )
```

---

## 📚 学习建议

1. **看动画演示**：Philip Roberts "What the heck is the event loop anyway?" 演讲是入门经典
2. **读规范**：
   - HTML spec: [Event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop)
   - Node.js 文档: `docs/guides/event-loop-timers-and-nexttick`
3. **看 libuv / asyncio / tokio 源码**：真正让"抽象落地"
4. **做一次火焰图分析**：用 Chrome Performance / py-spy / async-profiler
5. **写一个迷你事件循环**：100 行 JS / Python 就能跑（队列 + setImmediate 模拟）
6. **跨语言比较**：同一业务同时用 Node / Python / Go 写，感受各运行时模型差异

> 理解 Event Loop = 写出不卡、不崩、不饿的高并发异步代码的前提。
