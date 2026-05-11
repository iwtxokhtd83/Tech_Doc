
# Chrome 开发者工具使用讲义

> 本讲义系统介绍 Chrome DevTools（同样适用于 Edge / Brave / Arc 等基于 Chromium 的浏览器）的常用面板、排障方法与高级技巧。面向前端工程师、后端开发、QA 与排障人员。每章配"知识点 + 笔试题"。
>
> 约定：基于 Chrome 稳定版（~120+）；快捷键标注 Windows/Linux（括号内 macOS）。

## 目录

1. [打开方式与基础操作](#1-打开方式与基础操作)
2. [DevTools 面板全景](#2-devtools-面板全景)
3. [Elements：DOM 与样式](#3-elementsdom-与样式)
4. [Console：日志与交互](#4-console日志与交互)
5. [Sources：源码与断点调试](#5-sources源码与断点调试)
6. [Network：网络请求排查](#6-network网络请求排查)
7. [Performance：运行时性能](#7-performance运行时性能)
8. [Memory：内存泄漏排查](#8-memory内存泄漏排查)
9. [Application：存储与 PWA](#9-application存储与-pwa)
10. [Lighthouse 与性能审计](#10-lighthouse-与性能审计)
11. [Security：HTTPS 与证书](#11-securityhttps-与证书)
12. [响应式设计与设备模拟](#12-响应式设计与设备模拟)
13. [Coverage / Rendering / Animations 等小面板](#13-coverage--rendering--animations-等小面板)
14. [命令菜单与快捷键](#14-命令菜单与快捷键)
15. [远程调试（Android / Node）](#15-远程调试android--node)
16. [Workspaces 与实时编辑](#16-workspaces-与实时编辑)
17. [排障实战 Playbook](#17-排障实战-playbook)
18. [常见问题与技巧](#18-常见问题与技巧)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 打开方式与基础操作

### 1.1 打开 DevTools

| 方式 | 快捷键 |
|------|--------|
| 菜单 | ⋮ → 更多工具 → 开发者工具 |
| 快捷键 | **F12**  或  **Ctrl+Shift+I**（macOS: **⌘+Option+I**） |
| 右键"检查" | `Ctrl+Shift+C`（**⌘+Shift+C**）直接进入元素选取模式 |
| Console 直达 | `Ctrl+Shift+J`（**⌘+Option+J**） |

### 1.2 DevTools 停靠位置

右上角"三点菜单"可选：

- **右侧 / 左侧 / 底部**：同窗口内
- **单独窗口**：独立窗口（宽屏刚需）
- **关闭**：`Ctrl+Shift+D`（**⌘+Shift+D**）切换位置

### 1.3 基础心法

1. **F12 先打开，再刷新**：很多数据只在 DevTools 打开时才录制（Network、Performance）
2. **Disable cache** 先勾上：调试阶段避免"改了没生效"
3. **Preserve log** 谨慎：排链路问题要开，排其他问题反而噪音
4. **每个面板都有"筛选 + 搜索 + 详情"三板斧**

### 1.4 快捷键总表（DevTools 内部）

| 操作 | Win / Linux | macOS |
|------|-------------|-------|
| 打开 DevTools | F12 / Ctrl+Shift+I | ⌘+Option+I |
| 元素选取 | Ctrl+Shift+C | ⌘+Shift+C |
| Console | Ctrl+Shift+J | ⌘+Option+J |
| 切换停靠 | Ctrl+Shift+D | ⌘+Shift+D |
| **命令菜单**（强力） | Ctrl+Shift+P | ⌘+Shift+P |
| Sources 搜索文件 | Ctrl+P | ⌘+P |
| 全局文本搜索 | Ctrl+Shift+F | ⌘+Option+F |
| Console 清空 | Ctrl+L | ⌘+K |
| 放大 / 缩小 DevTools | Ctrl+Shift++ / - | ⌘+Shift++ / - |
| 刷新忽略缓存 | Ctrl+Shift+R | ⌘+Shift+R |

### 📝 笔试题 1-1：为什么"打开 DevTools 前发生的网络请求看不到"？

Network 面板**只录制 DevTools 打开期间**的请求。解决：

- 开 **Preserve log** 可保留跨页面导航的历史
- 或在打开 DevTools 后 **刷新页面 (F5)** 重新走一遍请求
- 如果是定时触发的，静候期待时刻

---

## 2. DevTools 面板全景

顶栏主要面板（按调试场景选择）：

| 面板 | 场景 |
|------|------|
| **Elements** | 看 / 改 DOM、CSS；布局调试 |
| **Console** | 打日志、跑表达式、错误信息 |
| **Sources** | 断点、单步、源码 |
| **Network** | 请求、状态、时间瀑布 |
| **Performance** | 性能录制、火焰图 |
| **Memory** | 堆快照、分配追踪 |
| **Application** | LocalStorage / Cookie / Service Worker / IndexedDB |
| **Security** | HTTPS / 证书 |
| **Lighthouse** | 综合审计报告 |
| **Recorder** | 录制交互为脚本 |
| **侧栏 + / 更多** | Coverage、Animations、Rendering、Sensors、Issues |

不同场景下的常用组合：

- **首屏慢** → Network + Performance + Lighthouse
- **交互卡顿** → Performance + Rendering
- **CSS 布局问题** → Elements + Layout/Flex/Grid 徽标
- **JS 报错 / 异常** → Console + Sources（黑盒化）
- **内存上涨** → Memory + Performance（Memory 选项）
- **401 / CORS** → Network + Issues
- **PWA / 缓存** → Application + Service Workers

### 📝 笔试题 2-1：排查"页面首屏很慢"通常先打开哪几个面板？

- **Network**：看首屏关键资源请求大小/耗时/瀑布
- **Performance**：录制 Load 过程，看主线程、长任务、渲染
- **Lighthouse**：一键综合报告（LCP / FCP / TBT / CLS）
- 辅助：**Coverage** 发现未用 JS/CSS；**Rendering** 查看 Layout Shift

---

## 3. Elements：DOM 与样式

### 3.1 面板布局

```
┌────────── Elements ──────────┐
│ DOM 树（左）                  │
├──────────────────────────────┤
│ Styles / Computed / Layout   │
│ Event Listeners / DOM Break  │
│ Properties / Accessibility   │
└──────────────────────────────┘
```

### 3.2 选取元素

- **Ctrl+Shift+C**（⌘+Shift+C）进入 pick 模式，点页面即选中
- `$0`：当前选中的元素（Console 可用）
- `$1…$4`：最近选过的前几个

### 3.3 实时修改 DOM / CSS

- 双击标签 / 属性 / 文本直接编辑
- 右键 **Edit as HTML**：整段改写
- **Styles 面板**：
  - 勾选 / 取消规则
  - 直接改值，`↑↓`微调数值（`Shift+↑↓` 10 倍、`Alt+↑↓` 0.1）
  - 新加属性：点击空白处输入
- **Filter** 框快速找某 CSS 属性
- **`:hov`** 按钮：强制触发 `hover / focus / active / visited / focus-within`

### 3.4 布局可视化

- **Layout 选项卡**：Flex / Grid 徽标一键开启线框
- Hover DOM 树时页面**高亮盒模型**：margin / border / padding / content
- Computed 面板看**最终生效的值**与继承路径

### 3.5 DOM 断点

在 DOM 节点右键 → **Break on**：

- **subtree modifications**
- **attribute modifications**
- **node removal**

命中后跳转到修改它的 JS 代码——**定位"是谁改了我的 DOM"** 神器。

### 3.6 Event Listeners / Accessibility

- 查看元素绑定的所有事件
- 点击可跳到处理函数
- Accessibility 面板看 ARIA 树、对比度、键盘可达

### 3.7 实用 Console 命令（结合 Elements）

```js
$0                     // 当前选中元素
$$("li")               // document.querySelectorAll 简写
copy($0.outerHTML)     // 复制到剪贴板
getEventListeners($0)  // 查看事件监听
monitorEvents($0, "click")     // 监控事件
```

### 📝 笔试题 3-1：CSS 改了不生效怎么排查？

1. **Styles 面板**看：是不是被**划掉**（被更高优先级规则覆盖）
2. **Computed 面板**看最终值 + 来源文件
3. 优先级链：!important > 行内 > 选择器特异性 > 来源顺序
4. 是否在**媒体查询**外、是否选择器拼写错
5. **Disable cache** 或强刷避免旧样式
6. 是否被 **CSS-in-JS** 运行时覆盖、或被 JS 动态设置 `style`

---

## 4. Console：日志与交互

### 4.1 基本用法

- 面板下方可输入任意 JS 表达式并立即求值
- 上下箭头可翻历史
- `Ctrl+L`（⌘+K）清屏
- 右键 → **Save as** 可导出全部日志

### 4.2 常用 API

```js
console.log / info / warn / error / debug
console.table(rows)                         // 数组/对象表格
console.group("label"); ... console.groupEnd();
console.count("hit")                        // 同 label 累计计数
console.time("t"); ... console.timeEnd("t");// 计时
console.trace()                             // 打印调用栈
console.assert(cond, "msg")                 // 条件失败才打印
console.dir($0)                             // 展开对象
console.profile("p") / profileEnd()         // 走 Performance
```

### 4.3 日志筛选

- 顶部下拉：All / Errors / Warnings / Info / Verbose
- 左侧 **Default levels** 勾选
- Filter 框支持：
  - 文本
  - `/regex/`
  - `-exclude`
  - `url:foo.js`

### 4.4 Live Expressions（顶部 👁）

钉住一个表达式，页面任何改动都实时重算。适合观察某变量（如 `performance.memory.usedJSHeapSize`）。

### 4.5 Snippets（Sources → Snippets）

常用脚本片段存起来反复跑：如下载当前页所有图片、批量模拟点击、提取表格等。

### 4.6 Preserve log / Hide network

- **Preserve log**：导航后仍保留历史（排查登录后跳转问题必开）
- **Hide network**：专注 JS 异常

### 4.7 常见陷阱

- `console.log(obj)` 打印的是**实时引用**，展开时可能已被改动；要取快照用 `JSON.parse(JSON.stringify(obj))` 或 `structuredClone(obj)`
- **Top 上下文**：默认执行在顶层 frame；选 iframe 要切换上下文下拉

### 📝 笔试题 4-1：同页面有 iframe，为什么 `document.querySelector` 查不到其中的按钮？

Console 默认上下文是**顶层页面**，iframe 是独立文档。解决：

- 顶栏 **JavaScript context 下拉**切换到目标 iframe
- 或 Elements 里选中 iframe 里的元素后自动切换
- 也可以 `document.querySelector('iframe').contentDocument.querySelector('button')`，但跨源会被拦截

---

## 5. Sources：源码与断点调试

### 5.1 面板结构

```
[Page / Filesystem / Overrides / Snippets]  ←  左侧
[打开的源文件]                              ←  中间
[Watch / Breakpoints / Scope / Call Stack]  ←  右侧
```

### 5.2 打开文件

- `Ctrl+P`（⌘+P）按名字快速打开
- `Ctrl+Shift+F`（⌘+Option+F）全项目搜文本
- **Pretty-print {}** 对压缩代码格式化
- **Source map** 自动加载时可看到原始 TS / SCSS / Vue / JSX

### 5.3 断点类型

在代码行号旁点击或右键：

- **普通断点（Line-of-code）**
- **条件断点**：`userId === 42`
- **Logpoint**（日志断点，不暂停，只输出）—— **调试利器**，比 `console.log` 还方便：`userId, status`
- **DOM 断点**（Elements 面板设置）
- **XHR/fetch 断点**：URL 模式命中即停
- **Event Listener 断点**：按事件类型（click、keydown、load 等）
- **Exception 断点**：勾选 Pause on exceptions / Pause on caught exceptions

### 5.4 单步控制

| 操作 | 快捷键 |
|------|--------|
| Resume | F8 |
| Step over | F10 |
| Step into | F11 |
| Step out | Shift+F11 |
| 跳到下一个函数调用 | Ctrl+\`*\`（⌘+\`） |
| 禁用所有断点 | Ctrl+F8 |

### 5.5 Scope / Call Stack / Watch

- **Scope**：当前作用域的本地 / 闭包 / 全局变量
- **Call Stack**：调用链，点某帧跳到对应代码
- **Watch**：常驻表达式，每次暂停刷新值

### 5.6 黑盒化（Ignore List）

不想单步进入三方库（React、jQuery、webpack runtime）：

- 右键文件 → **Add script to ignore list**
- Settings → **Ignore List** 配正则：`node_modules|vendor`

### 5.7 Source map 排障

TS/SCSS 映射不对时：

- 打开 DevTools 下方 **Issues**（或 Console）看 sourcemap 警告
- 确认构建产物包含 `.map` 文件
- `//# sourceMappingURL=...` 路径可达
- 跨域 map 需要正确的 CORS

### 5.8 Workspace（见第 16 章）

把本地目录接到 Sources，改了直接落盘——做小修改 / Hot patch 方便。

### 📝 笔试题 5-1：如何只在特定 URL 的 XHR 被触发时暂停？

Sources 面板右侧 → **XHR/fetch Breakpoints** → 加一行，输入部分 URL（如 `/api/orders`）。之后凡是请求 URL 含此字符串的 XHR / fetch 发起前都会命中断点，可看到调用栈定位代码。

### 📝 笔试题 5-2：老项目构建后全部是压缩代码，怎么高效断点？

1. **Pretty-print**（{}) 先美化
2. 确保 source map 生成并在生产可访问，或使用 **Private source map**（只线下映射）
3. 用 **Logpoint** 替代乱撒 `console.log`
4. 用 **Ignore List** 排除第三方
5. 必要时用 **Overrides**（第 16 章）临时改线上 JS 本地版来调试

---

## 6. Network：网络请求排查

### 6.1 面板结构

顶部工具栏：

- 🔴 录制开关
- 🚫 清空
- **Preserve log** / **Disable cache**
- **Throttling**：限速（Fast 3G / Slow 3G / Offline / 自定义）
- **Filter**：文本 / `-status-code:200` / `domain:` / `method:` / `has-response-header:` 等
- 类型过滤：Fetch/XHR / JS / CSS / Img / Media / Font / Doc / WS / Wasm / Manifest / Other
- **Group by frame**

### 6.2 瀑布图（Waterfall）

每条请求显示分段耗时：

```
Queueing → Stalled → DNS → Initial connection → SSL → Request sent → Waiting (TTFB) → Content Download
```

常见问题定位：

- **Queueing 长**：同域连接数限制（HTTP/1.1 一般 6）
- **Stalled 长**：代理 / 本地问题
- **DNS 长**：DNS 解析慢
- **Initial connection 长**：TCP 握手慢
- **SSL 长**：TLS 握手慢，考虑 0-RTT / 会话复用
- **Waiting (TTFB) 长**：**服务端慢**
- **Content Download 长**：包大 / 带宽不足

### 6.3 请求详情

点击一条请求：

- **Headers**：请求 / 响应头、状态码、远端地址
- **Payload**：form / query / JSON body
- **Preview**：图像、JSON 格式化
- **Response**：原始
- **Initiator**：**是谁发起的**（带调用栈，可跳到源码）
- **Timing**：分段耗时
- **Cookies**
- **Response Cookies**

### 6.4 Filter 语法例

```
status-code:500
method:POST
mime-type:image/png
domain:api.example.com
has-response-header:cache-control
larger-than:100k
```

### 6.5 复制与重放请求

- 右键 → **Copy**：
  - `Copy as cURL`（最常用，可直接贴终端重现）
  - `Copy as fetch`（贴 Console 复跑）
  - `Copy as PowerShell`
- **Replay XHR**：原样重发（身份保留）
- **Edit and Replay**（通过插件或 Network → **Override**）修改再发

### 6.6 限速与离线

- Throttling 模拟弱网 / 离线
- 用来复现移动用户体验
- **Service Worker 离线**：看 PWA 是否正常 fallback

### 6.7 WebSocket / SSE 观察

- 筛 **WS**：每条双击可看 frame 方向（↑ ↓）、opcode、payload
- **EventStream** 类型筛 SSE

### 6.8 大请求 / 慢请求

- **Sort by Size / Time**
- **Requests with blocking time** 透过 **Issues** 提示
- **HTTP/2 / HTTP/3**：Protocol 列查看（可能隐藏，右键 Add column）

### 6.9 CORS / 跨域错误

典型错误：

```
Access to fetch at 'https://api' from origin 'https://web' has been blocked
by CORS policy: No 'Access-Control-Allow-Origin' header is present...
```

排查：

1. 响应头有无 `Access-Control-Allow-Origin`
2. 带 cookie 时 `credentials: include` + `Allow-Credentials: true` + 明确的 Origin（不能 `*`）
3. 预检 OPTIONS 是否返回 2xx 与必要头
4. 网关 / CDN 是否拦截了 OPTIONS

### 📝 笔试题 6-1：请求变红（标黄），如何快速定位"慢在哪"？

1. **Timing 选项卡**看分段：TTFB 慢是**服务端慢**；Content Download 慢是**下载慢**
2. **Status** 4xx/5xx 看错误码
3. **Initiator**：谁触发的？是否重复触发
4. **Response** 看大小 / 压缩
5. 结合 **Throttling**：切 Fast 3G 看是否稳定复现

结合 **Network 瀑布 + Timing + 服务端 trace_id** 才能端到端定位。

---

## 7. Performance：运行时性能

### 7.1 适用场景

- 首屏加载慢
- 交互卡顿（点击/滚动延迟）
- 动画掉帧
- 定位长任务（long task）

### 7.2 录制流程

1. 打开 Performance 面板
2. 点 🔴 录制 → 触发操作 → 停止
3. 或 🔄 **Reload and record**：录制整个加载过程

建议：**录制时长 5-15 秒**，过长文件太大。

### 7.3 报告结构（自上而下）

- **Timeline 概览**：FPS 绿条、CPU 堆积条、Network、Heap
- **Frames**：每一帧耗时（掉帧红色）
- **Interactions**：用户交互事件
- **Main 主线程火焰图**：函数调用栈耗时
- **Timings**：LCP、FCP、DCL、L（load）等标记
- **GPU / Raster / Compositor**：渲染流水线
- **Memory**（勾上"Memory"录制才有）

### 7.4 关键概念

- **Long Task**：单次主线程任务 > 50ms，严重影响响应度
- **Rendering pipeline**：JS → Style → Layout → Paint → Composite
- **重排（Layout / Reflow）** vs **重绘（Paint）**：重排更贵
- **Scripting** 时间：JS 执行
- **Rendering / Painting**：浏览器排版绘制
- **System / Idle**：GC、空闲

### 7.5 常见优化方向

- JS 密集 → 拆任务 / Web Worker / `requestIdleCallback`
- 大量 DOM 更新 → 批量写 / `DocumentFragment`
- 反复读写样式 → **读写分离**避免强制同步布局
- 滚动卡 → 被动事件 `passive: true`、`will-change`、避免 `scroll` 内重算
- 第三方脚本阻塞 → 异步 / defer / 分割

### 7.6 Web Vitals（核心指标）

- **LCP（Largest Contentful Paint）**：最大内容绘制，好 ≤ 2.5s
- **INP（Interaction to Next Paint）**：交互响应，好 ≤ 200ms（取代了 FID）
- **CLS（Cumulative Layout Shift）**：累计布局偏移，好 ≤ 0.1
- **FCP（First Contentful Paint）**：首次内容绘制
- **TTFB**：首字节时间

Performance 面板 + Lighthouse 里都能看到。

### 7.7 Performance Insights（新面板）

Chrome 新增的 **Insights** 面板：AI 式直接给出"你最该优化什么 + 对应火焰图位置"，比读原始火焰图友好很多。

### 📝 笔试题 7-1：滚动页面掉帧，从哪几点排查？

1. **Performance 录制** 滚动过程，找**红色长帧**
2. 查看**哪些任务占用主线程**：通常是 `scroll` 事件 handler 做太多事
3. 看是否触发 **Layout / Paint** 过多
4. **被动事件**：监听 `scroll/touchmove` 用 `{ passive: true }`
5. 图像懒加载 / 虚拟列表
6. GPU 加速：`will-change: transform` / `transform3d` 让合成
7. 避免 `box-shadow`、`filter` 等昂贵绘制

---

## 8. Memory：内存泄漏排查

### 8.1 三种采样模式

- **Heap snapshot**：某一时刻全堆快照
- **Allocation instrumentation on timeline**：按时间序标注分配
- **Allocation sampling**：低开销，适合长时间采样

### 8.2 基础流程

1. 打开 Memory 面板
2. **Take snapshot**（记为 A）
3. 执行可能泄漏的操作若干次
4. 触发 GC（左上垃圾桶图标）
5. **Take snapshot**（记为 B）
6. 顶部选 **Comparison**：对比 A vs B，按 **#Delta** 倒序
7. 找出"只增不减"的对象类型

### 8.3 关键列含义

- **Shallow Size**：对象自身占用
- **Retained Size**：释放它能释放的总内存
- **Distance**：到 GC Root 的距离
- **Retainers**：谁在持有它 —— 定位 **引用链** 的关键

右键对象 → **Reveal in Summary view** / **Store as global variable** 可进一步 Console 操作。

### 8.4 常见泄漏源（浏览器侧）

- **Detached DOM nodes**：DOM 已从树上移除，但仍被 JS 引用 → 在快照里筛 `Detached`
- **闭包捕获大对象**
- **全局变量 / `window.xxx`**
- **监听器未解绑**：事件、`requestAnimationFrame`、`setInterval`
- **Map / Set 无界增长**
- **Canvas / WebGL 纹理未释放**

### 8.5 Performance + Memory 组合

Performance 面板勾选 **Memory** 选项录制：

- JS heap、Nodes、Listeners、Documents 随时间变化
- 锯齿形正常；**单调上涨** = 可疑泄漏

### 8.6 Node.js 侧用法

- `node --inspect` + Chrome `chrome://inspect`
- 直接用浏览器 DevTools 调 Node 的 Memory / Console / Sources
- 详见第 15 章

### 📝 笔试题 8-1：如何找出 "detached DOM"？

1. Memory 面板 → **Take snapshot**
2. 顶部 Class filter 输入 `Detached`
3. 出来的就是脱离 DOM 但还活着的节点
4. 展开 **Retainers** 看是谁在引用（常见是某个全局 Map、React 状态、定时器）

修法：及时解绑事件、在组件卸载时清除 ref / 定时器。

---

## 9. Application：存储与 PWA

### 9.1 存储类

- **Cookies**：按域名查看、编辑、删除
- **LocalStorage / SessionStorage**：键值
- **IndexedDB**：结构化存储，能浏览 object stores
- **Cache Storage**（Service Worker 缓存）
- **Storage 总览**：用量、按类别清理

### 9.2 Service Worker

- 查看已注册 SW、状态、update 触发
- **Offline**、**Update on reload**、**Bypass for network** 选项
- **Push / Sync** 手动触发
- 调试 SW 代码：Sources 面板有独立 thread

### 9.3 PWA 审计

- **Manifest**：看 icon / theme_color / display / start_url
- Lighthouse 中 PWA 类别评分
- 安装测试：地址栏"安装"按钮

### 9.4 Clear Storage

**Clear site data** 一键清本站所有缓存 / Cookie / SW —— **调缓存问题神器**。

### 9.5 Frames

- 左侧树列出所有 iframe
- 查看 CSP 报告、跨源隔离状态、Origin Trials

### 📝 笔试题 9-1：页面 Service Worker 更新不生效怎么办？

- Application → Service Workers 勾 **Update on reload**（仅开发）
- 点 **Unregister** 彻底清除
- **Clear site data** 清所有
- 检查 SW 脚本的 HTTP 缓存 header：常见坑是 SW 脚本本身被强缓存，推荐 `Cache-Control: no-cache`

---

## 10. Lighthouse 与性能审计

### 10.1 评估类别

- **Performance**：基于 Web Vitals 等
- **Accessibility**：WCAG 相关
- **Best Practices**：HTTPS / console error / 图像尺寸等
- **SEO**
- **PWA**

### 10.2 运行模式

- **Navigation**（默认）：冷加载
- **Timespan**：一段交互范围
- **Snapshot**：当前 UI 状态

### 10.3 建议使用方式

- **隐身模式**跑，排除扩展干扰
- **网络 / CPU 节流**默认"Simulated slow 4G"，结果更贴近移动端
- 多跑几次取中位数
- 关注 **分数背后的具体建议项**，而非只看总分

### 10.4 与 PageSpeed Insights / Core Web Vitals

- Lighthouse 是**实验室**数据
- CrUX（Chrome 用户体验报告）是**真实用户**数据
- 两者常不一致——实验室干净，真实用户网络 / 设备差异大

### 📝 笔试题 10-1：Lighthouse 分 95 但 CrUX 报告极差，为什么？

- 实验室模拟的是典型中端设备 + Simulated 4G
- 真实用户可能使用老设备 / 弱网 / 大量扩展 / 广告
- 测试页 vs 用户访问的真实入口页 可能不同
- 首屏资源路由在 CDN 中可能缓存命中率不均
- 以 **CrUX** 为准，Lighthouse 作为改进指引

---

## 11. Security：HTTPS 与证书

- 左侧 Main origin 展示**证书**、TLS 版本、Mixed content
- **Reissue Certificate** 等详情链接
- 页面有 HTTP 资源 → **Mixed content** 警告
- **Certificate Viewer**：查看完整证书链

**典型用法**：定位"为什么地址栏变灰 / 不安全"，通常是：

- 证书过期 / 自签
- 混合内容（HTTPS 页面引用 HTTP 图片 / 脚本）
- HSTS 但证书链不全

---

## 12. 响应式设计与设备模拟

### 12.1 Device Toolbar

- `Ctrl+Shift+M`（⌘+Shift+M）切换
- 顶部选择设备型号或自定义尺寸
- **DPR**（Device Pixel Ratio）、横竖屏、用户代理字符串
- **Show media queries** 看断点
- **Throttle network / CPU**

### 12.2 调试移动端

- 真机调试见第 15 章（ADB）
- 模拟可覆盖 UA / 触摸事件 / 地理位置 / 语言
- Sensors 面板：**Geolocation**、**Orientation**、**Idle emulation**、**Locale**、**Timezone**

### 📝 笔试题 12-1：为什么模拟 iPhone 但行为和真机不一致？

- 只模拟 **视口 + DPR + UA**，**不等于 Safari 引擎**
- Safari 专有 bug / WebKit 特有行为在 Chrome 模拟器里无法发现
- 真机测试仍必须（远程调试 iOS 需 Safari / Web Inspector）

---

## 13. Coverage / Rendering / Animations 等小面板

### 13.1 Coverage

- 菜单 → More tools → Coverage
- 录制一次用户操作，看每个 JS / CSS 文件有多少比例未使用
- 用于砍 bundle 体积、找死代码

### 13.2 Rendering

- **Paint flashing**：高亮每次重绘区域，看哪里频繁重绘
- **Layout Shift Regions**：显示 CLS 发生位置
- **Frame Rendering Stats**：叠加 FPS
- **Highlight ad frames** / **Core Web Vitals** 叠加
- **Emulate CSS media**：`prefers-color-scheme: dark`、`prefers-reduced-motion`

### 13.3 Animations

- 录制页面上的动画
- 可暂停 / 减速 / 拉进度条
- 定位"哪里有没必要的动画"

### 13.4 Recorder

- 录制一串用户操作为可回放脚本
- 可导出成 **Puppeteer / Playwright** 脚本基础
- 性能录制 / 回归测试起步方便

### 13.5 Issues

- DevTools 集中展示**浏览器检测到的问题**：
  - Cookies SameSite 警告
  - CORS / COEP / CORP 警报
  - Deprecated API
  - Mixed content
- 相比零散的 Console 警告更结构化

### 13.6 CSS Overview（实验性）

- 扫描整个页面 CSS 统计：颜色、字体、未用媒体查询等
- 做全站设计统一时有用

### 📝 笔试题 13-1：怎么找到"滚动时哪块反复重绘"？

- Rendering 面板打开 **Paint flashing**
- 滚动页面 → 看绿色闪烁区域
- 频繁闪且超出预期范围 = 过度重绘
- 配合 Performance 录制看对应 Paint 事件

---

## 14. 命令菜单与快捷键

### 14.1 命令菜单 Command Palette

- `Ctrl+Shift+P`（⌘+Shift+P）
- 输入关键词，支持所有 DevTools 操作：
  - `Show coverage` / `Show animations` / `Show rendering`
  - `Capture screenshot` / `Capture full size screenshot`
  - `Disable JavaScript`
  - `Show network request blocking`
  - `Dark theme` / `Light theme`
- 不记得在哪 → **先 Ctrl+Shift+P**

### 14.2 截图

- Full size 截全文
- Node 截某个元素

### 14.3 Dark/Light 主题

Settings（⚙️）→ Preferences → Theme。

### 14.4 Experiments

Settings → Experiments 打开可实验功能（新面板、旧功能开关）。

### 📝 笔试题 14-1：有哪些操作建议"先打开命令菜单"再搜关键词？

- 截屏、Coverage、Animations、Rendering、Sensors、Network request blocking、CSS Overview、禁 JS、清站点数据等；基本**非主流但很强的面板**都能 `Ctrl+Shift+P` 直接打开，省去翻菜单。

---

## 15. 远程调试（Android / Node）

### 15.1 Android 真机调试

1. 手机打开**开发者模式**，启用 **USB 调试**
2. 连电脑，允许调试授权
3. Chrome 打开 `chrome://inspect`
4. 看到设备 + 远端页面 → 点 **Inspect**
5. 出来的 DevTools 就是真机的，DOM / Network / Performance 都能用
6. Port forwarding 可把本机端口映射到手机

### 15.2 Node.js 调试

- `node --inspect app.js` 监听 9229
- `node --inspect-brk` 在首行就断
- Chrome `chrome://inspect` → **Open dedicated DevTools for Node**
- DevTools 里的 Memory、Profiler、Console、Sources 全部可用

### 15.3 VS Code 里用

VS Code 也可以连 `--inspect`；与 Chrome DevTools 等效，按习惯选。

### 📝 笔试题 15-1：移动端 H5 有 bug，线下难复现，如何处理？

- 真机 **远程调试**（Android）走 USB + `chrome://inspect`
- iOS 用 Safari 的 Web Inspector（不是 Chrome）
- 没有真机时用 **BrowserStack** / **Sauce Labs**
- 打 **结构化日志** 加 trace_id 发回日志服务
- 线上埋点 Web Vitals、错误监控（Sentry / 阿里 ARMS）

---

## 16. Workspaces 与实时编辑

### 16.1 Workspaces（本地文件夹）

- Sources → Filesystem → Add folder to workspace
- 选本地项目根目录，授权
- Chrome 把在线 URL 和本地文件**映射**
- 在 DevTools 里修改立即写入本地文件（保存到磁盘）
- 等同"用 DevTools 做轻量 IDE"

### 16.2 Overrides（线上代码本地覆盖）

- Sources → Overrides → Select folder
- 任意线上 JS/CSS 修改后会保存到本地覆盖副本
- 下次加载同 URL 时用本地版
- **调线上 bug 的"脏改快验"** 工具

常见场景：

- 线上调试 bug 时临时改 JS 看能否修
- 生产 A/B 对比自定义 CSS
- 注意：**不上传到服务器**，仅你本地生效

### 16.3 Snippets

- Sources → Snippets → New snippet
- 保存常用脚本反复跑
- 例如批量提取表格数据、劫持某函数打日志

### 📝 笔试题 16-1：Overrides 和 Workspaces 的区别？

- **Workspaces**：把 DevTools 当 IDE，改的是**本地真实源码**
- **Overrides**：对任意 URL 做本地临时覆盖，**不碰服务器**

前者用于开发项目；后者用于调试/验证他人代码或生产问题。

---

## 17. 排障实战 Playbook

### 17.1 "页面白屏"

1. **Console**：有无红色错误？堆栈指向哪个脚本
2. **Network**：关键 JS/CSS 是否 4xx / 5xx / 被拦截？
3. **Application → Clear site data**：排除旧缓存
4. **Security**：证书 / 混合内容？
5. **Sources**：该脚本有无语法错误（parse error）
6. 检查 `<script>` 顺序与 `defer/async`
7. 禁用扩展、隐身模式复测

### 17.2 "接口 401 / 403"

- Network → 请求 Headers：
  - Cookie 是否携带？
  - Authorization 是否正确
- 检查 **SameSite** / **Secure** 属性
- OPTIONS 预检是否 2xx
- `Application → Cookies` 看是否被清掉
- 跨域：看响应头 `Access-Control-*`

### 17.3 "我的改动没生效"

- **Disable cache** 勾上再刷
- 硬刷：`Ctrl+Shift+R`
- Service Worker 干扰 → Application → Unregister
- CDN 缓存 → 用 `?v=timestamp` 或后端加 cache-busting
- 打开真实加载的源码 URL（Network → Response）确认是否新代码

### 17.4 "卡在某个 3 秒请求"

- **Network → Timing**：看是 TTFB 还是 Download 慢
- TTFB 慢 → 后端 trace
- Download 慢 → 包太大 / 网络差
- **Large** 列筛过滤大资源
- 同域并发被排队？看 Queued/Stalled

### 17.5 "内存越来越大最后崩溃"

- 参考第 8 章：snapshot 对比 + detached DOM
- 看 `performance.memory.usedJSHeapSize` 变化
- 检查未解绑监听 / 未清定时器
- Node 服务单独远程调试

### 17.6 "移动端 iOS 打开错乱，桌面 Chrome 正常"

- Safari 专有行为 / 兼容性差异
- 用 **Safari Web Inspector + iPhone 真机** 调试（不是 Chrome）
- Can I use 查特性兼容
- 开 BrowserStack / Sauce Labs

### 17.7 "某个 event 触发后才出错"

- Sources → Event Listener Breakpoints 勾 `click` / `submit` 等
- 触发事件后自动进入断点链路
- Call Stack 向下看，定位引发错误的代码

### 17.8 "老板说这个按钮被谁加的？"

- Inspect 按钮 → Elements 面板 → **Event Listeners**
- 看绑定的脚本位置，点击跳到源码
- 或设 DOM 断点 **attribute modifications** 等下次修改被抓

### 📝 笔试题 17-1：用户报 "图片没加载"，如何一步步定位？

1. **Network** → 筛 Img → 看失败请求（红条）
2. 看 Status：**403** 权限 / **404** 路径错 / **Blocked** CORS / 协议
3. 看 Response Headers、Origin、Referer
4. `Preview` 看是否返回内容但 MIME 错导致不渲染
5. **Console** 有无警告（Mixed content、CSP）
6. **Application → Clear site data** 排除本地缓存旧 404
7. 真机不同网络复测

---

## 18. 常见问题与技巧

### 18.1 常被忽视的小功能

- **持久化网络面板图表**：Ctrl+R 刷新后保留时间线（Preserve log）
- **Request blocking**：命令菜单 `Show network request blocking`，输入 URL 模式阻止加载，排查"没这个 JS 会怎样"
- **Animations 面板**：动画慢放、暂停
- **Dashboards 里的 Performance Monitor**（命令菜单 `Show performance monitor`）：实时 CPU / JS heap / DOM nodes / Listeners
- **Protocol monitor**：看 DevTools 自己与浏览器的 CDP 消息（用于分析 DevTools 本身）
- **CSS Overview** / **3D View**：大型页面分析

### 18.2 调试生产线上页面

- **Overrides** 临时改 JS/CSS
- **Disable JavaScript**（命令菜单）快速看"无 JS 时什么样"
- **Throttling** 模拟弱网 / CPU
- 清缓存 / 禁扩展 / 隐身模式

### 18.3 输出格式化 JSON

- Console：`console.table(rows)`
- Network 的 Response 页右下 `{}` 按钮 Pretty-print

### 18.4 复制 curl 复现

复制 curl 后直接加 `-v` 看完整协议交互；配合服务端 trace_id 定位。

### 18.5 无痕排障套路

1. 进隐身模式：`Ctrl+Shift+N`
2. 打开 DevTools，**Preserve log** + **Disable cache**
3. 关闭扩展
4. 复现问题
5. 比对普通窗口看差异是否扩展 / 缓存引起

### 18.6 使用 Copy full CSS selector / XPath

DOM 树右键节点 → **Copy → Copy selector / Copy XPath / Copy JS path**。自动化测试写 locator 时省心。

### 18.7 Chrome Flags 与实验

- `chrome://flags`：浏览器级实验开关
- DevTools → Settings → Experiments：DevTools 级
- 新功能很多先在这里孵化

### 📝 笔试题 18-1：怎么在不改服务器的情况下临时阻断某个第三方脚本？

- **Network request blocking**（命令菜单开启）
- 加 URL 模式，比如 `ads.js`
- 刷新页面 → 该脚本被拦截 → 观察效果
- 排查用，不要在生产脚本里依赖这种方式

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 打开 DevTools 后看不到已经发生的请求，应该？
A. 重装 Chrome
B. 勾 Preserve log 后刷新页面
C. 升级 DevTools
D. 关掉 Disable cache

<details><summary>答案</summary>B。</details>

**Q2** 下列最适合查"谁修改了我的 DOM"？
A. Console 打日志
B. Memory 面板
C. Elements → 右键 Break on → subtree/attribute/removal
D. Performance 录制

<details><summary>答案</summary>C。</details>

**Q3** Logpoint 的好处是？
A. 暂停执行
B. 不暂停，按格式打印，比 console.log 方便
C. 影响性能
D. 仅支持 Chrome Canary

<details><summary>答案</summary>B。</details>

**Q4** 关于 Overrides：
A. 会被服务器存档
B. 会污染真实用户
C. 本地覆盖线上资源，仅自己生效
D. 只支持 CSS

<details><summary>答案</summary>C。</details>

**Q5** 想排查"是哪个事件处理函数导致的错误"，应设？
A. XHR 断点
B. Event Listener 断点
C. DOM 断点
D. 条件断点

<details><summary>答案</summary>B。</details>

**Q6** 下列哪项可以**模拟弱网**？
A. Network Throttling
B. Performance 录制
C. Lighthouse
D. 所有选项都支持，Network 最直接

<details><summary>答案</summary>D。</details>

**Q7** Lighthouse 分数和真实用户数据差距大时，应以什么为准？
A. Lighthouse
B. CrUX（真实用户）
C. 自己感觉
D. 平均

<details><summary>答案</summary>B。</details>

**Q8** 命令菜单的快捷键是？
A. Ctrl+P
B. Ctrl+Shift+P
C. Ctrl+Shift+F
D. F12

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. F12 打开 DevTools 前发生的网络请求也能看到。 ❌
2. Console 执行会改变页面状态。 ✅
3. Performance 能回看 CPU 火焰图。 ✅
4. Memory 面板的 Comparison 能帮忙找泄漏。 ✅
5. Service Worker 更新可以 DevTools 强制触发。 ✅
6. Emulate 设备 = 在真机运行。 ❌
7. Ignore List 能避免单步进入第三方库。 ✅
8. 隐身模式下所有扩展都被禁用。 默认是（可放开特权）。 ✅（默认）

### 19.3 简答题

**Q1** 用户说"点击按钮没反应"，如何排查？

1. Console 看是否有 JS 错误导致 handler 没注册
2. Elements 选按钮 → Event Listeners 看有无绑 click
3. 设 Event Listener Breakpoints → click，点击验证
4. 检查 `preventDefault` 是否被上层框架吃掉
5. 检查是否被遮盖元素挡住（`pointer-events`）
6. 看是否 SPA 路由守卫拦截

**Q2** 首屏 LCP > 4s，如何改进？

- Lighthouse 看 LCP 元素
- Network 看关键资源：大小 / 顺序 / 压缩
- **preload** 关键字体 / 图；**preconnect** 关键域
- 图片：WebP / AVIF；`<img loading=lazy>` 但首屏图不要 lazy
- 拆包 + 路由级 code splitting
- CDN + HTTP/2 / HTTP/3
- SSR / 静态化

**Q3** 怎么让 DevTools 陪你定位偶发性 bug？

- **Preserve log** 开、Console 保留
- **Performance** 提前开始录制
- Sources **Pause on exceptions**
- 监控 `window.addEventListener('error'/'unhandledrejection')` 收集
- 复现后立刻 **Copy as cURL** 保留请求
- `chrome://net-export` 捕获网络栈全部细节

### 19.4 案例

**场景 A**：用户抱怨"页面有时候会突然卡 2 秒"。

- 打开 Performance，`Reload and record` 录一段
- 观察 **Main 轴**有没有 > 200ms 的红色长任务
- 看任务内部调用栈：是第三方广告？大 JSON 解析？同步布局？
- Rendering 打开 **Paint flashing** / **Layout Shift Regions** 排除渲染抖动
- 给第三方脚本加 `async` / 延迟加载
- 把大 JSON 解析挪到 Worker

**场景 B**：Form 提交后刷新页面然后数据没了。

- Network 看提交请求是否成功（POST 状态码？）
- Application → Cookies：会话是否保留
- Redirect 链：是否 302 丢了 referer
- 看后端 API 响应是否 set-cookie
- 排除浏览器拦截第三方 Cookie

---

## 📚 学习建议

1. **每天用命令菜单**（`Ctrl+Shift+P`），两周下来会发现很多隐藏宝藏
2. **刻意练习**：每学一个面板，就用它解决一个真实问题
3. **官方文档**：<https://developer.chrome.com/docs/devtools/> 持续更新
4. **看 Release Notes**：Chrome 每版 DevTools 都有新特性（Twitter `@ChromeDevTools`、devtools tips）
5. **搭配真机 / 跨浏览器**：DevTools 再强也不能代替 Safari Web Inspector、Firefox DevTools
6. **DevTools 不只是前端的**：后端调接口、QA 排 bug、产品做 A/B 都能用

> DevTools 是开发者最近的"显微镜"，练熟它能让日常排障效率翻倍。
