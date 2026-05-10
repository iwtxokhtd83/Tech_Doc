# WebSocket 讲义

> 本讲义系统介绍 WebSocket 协议原理、握手机制、帧格式、使用场景、常见坑点，并给出 **Node.js / Python / Java / Go** 服务端和 **浏览器 / Node.js / Python** 客户端的完整可运行代码示例。每章配"知识点 + 笔试题"。
>
> 约定：协议规范 RFC 6455；URL 协议 `ws://`（明文）/ `wss://`（TLS）；默认端口 80 / 443。

## 目录

1. [为什么需要 WebSocket](#1-为什么需要-websocket)
2. [握手过程](#2-握手过程)
3. [帧格式与数据传输](#3-帧格式与数据传输)
4. [连接生命周期与关闭](#4-连接生命周期与关闭)
5. [子协议、扩展与压缩](#5-子协议扩展与压缩)
6. [心跳与保活](#6-心跳与保活)
7. [浏览器 WebSocket API](#7-浏览器-websocket-api)
8. [Node.js 服务端与客户端](#8-nodejs-服务端与客户端)
9. [Python 服务端与客户端](#9-python-服务端与客户端)
10. [Java 服务端与客户端](#10-java-服务端与客户端)
11. [Go 服务端与客户端](#11-go-服务端与客户端)
12. [聊天室实战：广播与房间](#12-聊天室实战广播与房间)
13. [鉴权、CORS 与安全](#13-鉴权cors-与安全)
14. [反向代理与负载均衡](#14-反向代理与负载均衡)
15. [扩展：集群、消息广播与消息队列](#15-扩展集群消息广播与消息队列)
16. [测试与排障](#16-测试与排障)
17. [WebSocket vs SSE vs 长轮询 vs WebTransport](#17-websocket-vs-sse-vs-长轮询-vs-webtransport)
18. [综合笔试练习](#18-综合笔试练习)

---

## 1. 为什么需要 WebSocket

### 1.1 HTTP 的局限

- HTTP 是**请求-响应**模型，服务器无法主动推送
- 实时场景的传统方案：
  - **短轮询**：定时请求，延迟高、资源浪费
  - **长轮询**：服务端 hang 住请求直到有数据或超时
  - **SSE**：服务端单向推送，基于 HTTP
  - **Comet**：长连接上多次响应流，复杂

### 1.2 WebSocket 的优势

- **全双工**：客户端和服务端同时收发
- **低延迟**：建立后是原生 TCP 上的帧协议，无 HTTP 头开销
- **一次握手，长期连接**：省去反复建连成本
- **与 HTTP 复用端口**：通过 Upgrade 机制升级，经 80/443 可穿越大多数代理

### 1.3 典型应用

- 实时聊天、IM
- 在线协作（文档编辑、白板）
- 实时行情、订单推送（金融）
- 游戏状态同步
- 实时仪表盘、日志追踪
- IoT 设备双向控制
- WebRTC 信令通道

### 📝 笔试题 1-1：WebSocket 和 HTTP 是什么关系？

WebSocket 不是独立新协议，而是**在 HTTP 之上升级**：通过一次 HTTP `Upgrade` 握手完成协议切换，之后同一 TCP 连接上运行 WebSocket 帧协议。因此可以复用 HTTP 的端口、代理、TLS 基础设施，但正式运行时已**脱离 HTTP 语义**。

---

## 2. 握手过程

### 2.1 握手请求（Client → Server）

```
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://app.example.com
Sec-WebSocket-Protocol: chat, v10.stomp
Sec-WebSocket-Extensions: permessage-deflate
```

### 2.2 握手响应（Server → Client）

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

- **101** 状态码表示协议切换成功
- 其他返回（200/400/401/403/426 等）代表握手失败

### 2.3 Sec-WebSocket-Accept 算法

服务端以 **固定 GUID** 拼接客户端 key，SHA-1 后 Base64：

```
accept = base64( sha1( client_key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
```

这是**防止意外升级**（如缓存代理误以为是普通 HTTP）的弱校验，不是安全机制。

### 2.4 关键字段

| 字段 | 作用 |
|------|------|
| `Upgrade: websocket` / `Connection: Upgrade` | 协议升级标识 |
| `Sec-WebSocket-Version: 13` | RFC 6455 版本号 |
| `Sec-WebSocket-Key` | 随机 16 字节 base64，握手校验用 |
| `Sec-WebSocket-Protocol` | 子协议选择（类似 Accept） |
| `Sec-WebSocket-Extensions` | 扩展协商（如压缩） |
| `Origin` | 浏览器发起时自动带，服务端应校验防跨域滥用 |

### 📝 笔试题 2-1：握手成功的 HTTP 状态码是？

**101 Switching Protocols**。其他响应（200、403 等）均表示握手失败，客户端收到后不会进入 WebSocket 模式。

---

## 3. 帧格式与数据传输

### 3.1 WebSocket 帧结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key (only from client) |
+-------------------------------+-------------------------------+
|                     Payload Data (masked if from client)      |
```

### 3.2 字段详解

- **FIN**：是否最后一帧（分片消息用）
- **RSV1-3**：扩展保留位（如 `permessage-deflate` 用 RSV1 表示压缩）
- **opcode**：帧类型
  - `0x0` 延续帧
  - `0x1` 文本帧（UTF-8）
  - `0x2` 二进制帧
  - `0x8` 关闭帧
  - `0x9` Ping
  - `0xA` Pong
- **MASK**：客户端发给服务端**必须**掩码，服务端发给客户端**不能**掩码
- **Payload len**：
  - 0-125：本身就是长度
  - 126：后面 2 字节才是真正长度（≤ 65535）
  - 127：后面 8 字节（≤ 2⁶³）
- **Masking-key**：4 字节，客户端每帧随机生成
- **Payload Data**：实际数据，客户端发送时需要 XOR 掩码

### 3.3 掩码的作用

防止被中间代理（如 HTTP 代理）误解析为缓存 poisoning 攻击载荷。解码：

```
decoded[i] = encoded[i] ^ mask[i % 4]
```

### 3.4 分片消息

大消息可分成多帧发送：

- 首帧：`FIN=0`, `opcode=1/2`
- 中间帧：`FIN=0`, `opcode=0`
- 末帧：`FIN=1`, `opcode=0`

控制帧（Ping/Pong/Close）**不能分片**，载荷 ≤ 125 字节。

### 📝 笔试题 3-1：为什么只有客户端发送的帧需要掩码？

为了阻止**跨协议攻击**：如果一个普通浏览器 bug 让攻击者能诱导浏览器发 WebSocket 帧，中间的 HTTP 代理可能把攻击载荷当成受害网站的合法请求做缓存 poisoning。掩码让攻击者无法预测最终落在代理缓存里的字节。服务端→客户端方向没有这类威胁，因此不加掩码。

---

## 4. 连接生命周期与关闭

### 4.1 状态机

```
CONNECTING → OPEN → CLOSING → CLOSED
```

浏览器 `WebSocket.readyState` 常量：`0 / 1 / 2 / 3`。

### 4.2 关闭帧

任一方发送 **opcode=0x8** 的 Close 帧，载荷含：

- **2 字节状态码**（Close Code）
- 可选 UTF-8 理由（≤ 123 字节）

收到后应回发 Close 帧（最好带相同状态码），随后关闭底层 TCP。

### 4.3 常见 Close Code

| 码 | 名称 | 说明 |
|----|------|------|
| 1000 | Normal Closure | 正常关闭 |
| 1001 | Going Away | 端点下线（刷新/关闭页面） |
| 1002 | Protocol Error | 协议错 |
| 1003 | Unsupported Data | 收到不支持的帧类型 |
| 1005 | No Status（保留） | **不能发**，仅库内部用 |
| 1006 | Abnormal Closure（保留） | 未正常关闭就断开 |
| 1007 | Invalid Payload | 不合法 UTF-8 |
| 1008 | Policy Violation | 策略违规（鉴权失败） |
| 1009 | Message Too Big | 消息过大 |
| 1011 | Internal Error | 服务端异常 |
| 1012 | Service Restart | 重启 |
| 1013 | Try Again Later | 过载 |
| 4000-4999 | 应用自定义 | 业务自由使用 |

`1006` 通常表示连接**未经正常握手关闭**（断网、崩溃），这是**客户端重连的关键判据**。

### 4.4 半关闭

TCP 层面可半关闭，但 WebSocket 规范要求发送 Close 后不应再发数据帧；实现上多数库把双向收发一起关闭。

### 📝 笔试题 4-1：客户端收到 `1006` 该怎么处理？

`1006` 说明连接异常断开（无正常关闭帧）。客户端应：

1. 清理本地连接状态
2. **指数退避重连**：初始 1s，逐次乘以 2 并加随机抖动，最多到 30s
3. 重连成功后重新订阅/鉴权/同步增量状态
4. 给用户"离线/重连中"UI 反馈

---

## 5. 子协议、扩展与压缩

### 5.1 子协议（Sub-Protocol）

业务层约定的**消息语义协议**，如 `chat.v1`、`graphql-ws`、`mqtt`、`stomp`。

握手协商：

```
Client:  Sec-WebSocket-Protocol: graphql-ws, graphql-transport-ws
Server:  Sec-WebSocket-Protocol: graphql-transport-ws       ← 选其一
```

客户端收到服务端选定的子协议后按约定格式收发消息。

### 5.2 扩展（Extension）

**`permessage-deflate`**（RFC 7692）是唯一被广泛采用的扩展，对整个消息做 DEFLATE 压缩。

```
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

**代价**：CPU 开销 + 每连接的压缩上下文内存。高并发服务端（尤其 Node.js）要评估，必要时关闭。

### 5.3 常见业务子协议

- **STOMP over WebSocket**：消息订阅模型
- **GraphQL over WebSocket**：订阅推送（`graphql-ws` 新标准）
- **SignalR**（.NET）、**Socket.IO**（Node 生态，自己的协议，非纯 WS）
- **MQTT over WebSocket**：IoT 常见

### 📝 笔试题 5-1：Socket.IO 是 WebSocket 吗？

不完全是。Socket.IO 是一个**上层协议 + 传输层抽象**，优先使用 WebSocket 传输，**降级**到 HTTP 长轮询。它在 WebSocket 之上又封了自己的帧格式、房间、命名空间、ACK、自动重连等，因此**普通 WebSocket 客户端无法直接连接 Socket.IO 服务**，反之亦然。

---

## 6. 心跳与保活

### 6.1 为什么要心跳

- 中间设备（LB、NAT、代理）通常有**空闲连接超时**（如 60s）
- 长时间无数据连接会被悄悄断开
- 客户端/服务端进程崩溃时 TCP 不一定能及时感知

### 6.2 协议层 Ping/Pong

- opcode `0x9` Ping / `0xA` Pong
- 对端收到 Ping 必须尽快回 Pong（载荷原样回显）
- 浏览器 API **不允许**用户代码主动发 Ping，由浏览器自动处理（或不支持）

### 6.3 应用层心跳（推荐）

大多数生产系统在应用层约定心跳消息：

```json
{ "type": "ping", "ts": 1700000000 }
{ "type": "pong", "ts": 1700000000 }
```

- 客户端每 20-30s 发一次
- N 次未收到回应即判断死连，主动断开并重连
- 便于跨 API 层（如 Socket.IO、STOMP）统一处理

### 6.4 参数参考

- 心跳间隔：20-30 秒（小于中间设备超时）
- 超时阈值：2-3 次未响应即判定断开
- 服务端也应主动检测客户端是否响应

### 📝 笔试题 6-1：为什么不直接依赖 TCP keepalive？

- TCP keepalive 默认 2 小时才触发，**粒度太粗**
- 操作系统级别，应用层不易控制
- 无法感知**应用卡死**（TCP 仍在跑，但进程挂了）
- 中间 NAT 可能已经丢弃状态，keepalive 空跑

应用层心跳能精确可控，并可携带业务语义（时间戳、版本号）。

---

## 7. 浏览器 WebSocket API

### 7.1 基础用法

```javascript
const ws = new WebSocket("wss://echo.example.com/ws", ["chat.v1"]);

// 连接建立
ws.addEventListener("open", () => {
  console.log("open");
  ws.send(JSON.stringify({ type: "hello", name: "Alice" }));
});

// 收消息
ws.addEventListener("message", (e) => {
  // e.data 可能是 string / Blob / ArrayBuffer
  console.log("recv", e.data);
});

// 出错
ws.addEventListener("error", (e) => {
  console.error("error", e);
});

// 关闭
ws.addEventListener("close", (e) => {
  console.log("close", e.code, e.reason, "wasClean:", e.wasClean);
});
```

### 7.2 发送二进制

```javascript
// 让 message 事件直接给 ArrayBuffer
ws.binaryType = "arraybuffer";

// 发送 ArrayBuffer
const buf = new Uint8Array([1, 2, 3, 4]);
ws.send(buf.buffer);

// 发送 Blob（如文件）
const blob = new Blob(["hello"], { type: "text/plain" });
ws.send(blob);
```

### 7.3 可靠重连封装

```javascript
class ReliableWS {
  constructor(url, protocols = [], opts = {}) {
    this.url = url;
    this.protocols = protocols;
    this.opts = { maxDelay: 30000, heartbeatMs: 25000, ...opts };
    this.listeners = {};
    this.retry = 0;
    this.alive = false;
    this.connect();
  }
  connect() {
    this.ws = new WebSocket(this.url, this.protocols);
    this.ws.addEventListener("open", () => {
      this.retry = 0;
      this.alive = true;
      this.startHeartbeat();
      this.emit("open");
    });
    this.ws.addEventListener("message", (e) => {
      this.alive = true;
      if (e.data === "pong") return;
      this.emit("message", e.data);
    });
    this.ws.addEventListener("close", (e) => {
      this.stopHeartbeat();
      this.emit("close", e);
      if (e.code !== 1000) this.scheduleReconnect();
    });
    this.ws.addEventListener("error", (e) => this.emit("error", e));
  }
  scheduleReconnect() {
    this.retry++;
    const delay = Math.min(1000 * 2 ** this.retry, this.opts.maxDelay);
    const jitter = Math.random() * 500;
    setTimeout(() => this.connect(), delay + jitter);
  }
  startHeartbeat() {
    this.hbTimer = setInterval(() => {
      if (!this.alive) { this.ws.close(); return; }
      this.alive = false;
      this.safeSend("ping");
    }, this.opts.heartbeatMs);
  }
  stopHeartbeat() { clearInterval(this.hbTimer); }
  send(data) { this.safeSend(data); }
  safeSend(data) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) this.ws.send(data);
  }
  on(evt, cb) { (this.listeners[evt] ||= []).push(cb); }
  emit(evt, ...args) { (this.listeners[evt] || []).forEach(cb => cb(...args)); }
  close() { this.ws && this.ws.close(1000, "bye"); }
}

const rws = new ReliableWS("wss://example.com/ws");
rws.on("message", console.log);
```

### 📝 笔试题 7-1：`ws.send(obj)` 不行，为什么？

`WebSocket.send` 只接受 `string`/`ArrayBuffer`/`Blob`/`ArrayBufferView`。对象必须先序列化：

```javascript
ws.send(JSON.stringify(obj));
```

---

## 8. Node.js 服务端与客户端

Node 的事实标准是 [`ws`](https://github.com/websockets/ws)。另有基于其上的 `Socket.IO`（增加了房间、命名空间、自动降级等）。

### 8.1 安装

```bash
npm init -y
npm install ws
```

### 8.2 服务端

```javascript
// server.js
import { WebSocketServer } from "ws";
import http from "http";

const server = http.createServer();
const wss = new WebSocketServer({ server, path: "/ws" });

const clients = new Set();

wss.on("connection", (ws, req) => {
  console.log("connected from", req.socket.remoteAddress);
  clients.add(ws);

  ws.isAlive = true;
  ws.on("pong", () => { ws.isAlive = true; });

  ws.on("message", (data, isBinary) => {
    // 广播给所有其他客户端
    const msg = isBinary ? data : data.toString();
    for (const c of clients) {
      if (c !== ws && c.readyState === ws.OPEN) {
        c.send(msg, { binary: isBinary });
      }
    }
  });

  ws.on("close", (code, reason) => {
    clients.delete(ws);
    console.log("closed", code, reason.toString());
  });

  ws.on("error", (err) => console.error("ws err", err));

  ws.send(JSON.stringify({ type: "welcome", time: Date.now() }));
});

// 心跳检测：30s 未响应即断开
setInterval(() => {
  for (const ws of clients) {
    if (!ws.isAlive) {
      ws.terminate();
      clients.delete(ws);
      continue;
    }
    ws.isAlive = false;
    ws.ping();              // 协议级 Ping
  }
}, 30000);

server.listen(8080, () => console.log("ws://localhost:8080/ws"));
```

### 8.3 客户端（Node）

```javascript
// client.js
import WebSocket from "ws";

const ws = new WebSocket("ws://localhost:8080/ws");

ws.on("open", () => {
  console.log("open");
  ws.send("hello from node client");
});

ws.on("message", (data) => console.log("recv:", data.toString()));

ws.on("close", (code, reason) =>
  console.log("close", code, reason.toString()));

ws.on("error", console.error);
```

### 8.4 集成到 Express / Fastify

```javascript
import express from "express";
import { WebSocketServer } from "ws";

const app = express();
app.get("/", (_, res) => res.send("ok"));

const server = app.listen(8080);
const wss = new WebSocketServer({ noServer: true });

server.on("upgrade", (req, socket, head) => {
  // 这里可做鉴权：校验 req.headers.cookie 或 ?token=...
  if (req.url === "/ws") {
    wss.handleUpgrade(req, socket, head, (ws) => {
      wss.emit("connection", ws, req);
    });
  } else {
    socket.destroy();
  }
});
```

### 📝 笔试题 8-1：生产中为什么推荐 `noServer: true` + `server.upgrade` 写法？

- 与现有 HTTP 框架（Express/Koa/Fastify）**共享同一端口**
- 可以在升级前做**鉴权 / 路径路由 / 限流**，拒绝非法升级
- 对不同 path 注册不同 `WebSocketServer`，实现多端点

---

## 9. Python 服务端与客户端

Python 主流库是 [`websockets`](https://websockets.readthedocs.io)（async，纯 WebSocket）和 [`aiohttp`](https://docs.aiohttp.org)（包括 HTTP + WebSocket）。旧项目也常见 `websocket-client`（同步客户端）。

### 9.1 安装

```bash
pip install websockets
```

### 9.2 服务端（asyncio）

```python
# server.py
import asyncio
import json
import logging
from websockets.asyncio.server import serve

logging.basicConfig(level=logging.INFO)
clients = set()

async def handler(ws):
    clients.add(ws)
    try:
        await ws.send(json.dumps({"type": "welcome"}))
        async for msg in ws:
            # msg: str 或 bytes
            # 广播给其他客户端
            await asyncio.gather(
                *[c.send(msg) for c in clients if c is not ws],
                return_exceptions=True,
            )
    finally:
        clients.discard(ws)

async def main():
    async with serve(
        handler, "0.0.0.0", 8080,
        ping_interval=20, ping_timeout=20,
        max_size=1 * 1024 * 1024,     # 最大消息 1MB
    ):
        print("listening ws://0.0.0.0:8080")
        await asyncio.Future()

if __name__ == "__main__":
    asyncio.run(main())
```

`ping_interval` / `ping_timeout` 由库自动做协议级心跳，**无需手动 ping**。

### 9.3 客户端（asyncio）

```python
# client.py
import asyncio
import json
from websockets.asyncio.client import connect

async def main():
    async with connect("ws://localhost:8080") as ws:
        await ws.send(json.dumps({"type": "hello", "user": "alice"}))
        async for msg in ws:
            print("recv:", msg)

asyncio.run(main())
```

### 9.4 同步客户端（旧 API，兼容性好）

```python
# sync_client.py
from websockets.sync.client import connect

with connect("ws://localhost:8080") as ws:
    ws.send("hi")
    print(ws.recv())
```

### 9.5 FastAPI / Django Channels

**FastAPI**：

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            data = await ws.receive_text()
            await ws.send_text(f"echo: {data}")
    except WebSocketDisconnect:
        print("client disconnected")
```

**Django**：用 **Channels** + ASGI。

### 📝 笔试题 9-1：`websockets` 库的 `ping_interval` 与 `ping_timeout` 分别表示？

- `ping_interval`：空闲多少秒自动发 Ping
- `ping_timeout`：发送 Ping 后多少秒没收到 Pong 就视为超时并关闭

常用组合 `ping_interval=20, ping_timeout=20`，兼顾存活探测与快速失败。

---

## 10. Java 服务端与客户端

Java 有 **JSR-356**（`javax/jakarta.websocket`）和 **Spring WebSocket**，客户端常用 OkHttp、Tyrus。

### 10.1 Spring Boot 服务端

`pom.xml`：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

`WebSocketConfig.java`：

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new EchoHandler(), "/ws")
                .setAllowedOrigins("*");
    }
}

class EchoHandler extends TextWebSocketHandler {
    private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        sessions.add(session);
        session.sendMessage(new TextMessage("{\"type\":\"welcome\"}"));
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        String payload = message.getPayload();
        TextMessage out = new TextMessage(payload);
        for (WebSocketSession s : sessions) {
            if (s.isOpen() && !s.getId().equals(session.getId())) {
                s.sendMessage(out);
            }
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session);
    }
}
```

### 10.2 STOMP + SockJS

对复杂订阅/广播可以使用 STOMP：

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry r) {
        r.addEndpoint("/stomp").setAllowedOriginPatterns("*").withSockJS();
    }
    @Override
    public void configureMessageBroker(MessageBrokerRegistry r) {
        r.enableSimpleBroker("/topic", "/queue");
        r.setApplicationDestinationPrefixes("/app");
    }
}

@Controller
class ChatController {
    @MessageMapping("/chat.send")
    @SendTo("/topic/chat")
    public Map<String, Object> send(Map<String, Object> msg) {
        return msg;
    }
}
```

### 10.3 Java 客户端（OkHttp）

```java
// gradle: implementation "com.squareup.okhttp3:okhttp:4.12.0"
OkHttpClient client = new OkHttpClient.Builder()
    .pingInterval(Duration.ofSeconds(20))
    .build();

Request req = new Request.Builder().url("ws://localhost:8080/ws").build();

client.newWebSocket(req, new WebSocketListener() {
    @Override public void onOpen(WebSocket ws, Response r) {
        ws.send("hello from java");
    }
    @Override public void onMessage(WebSocket ws, String text) {
        System.out.println("recv: " + text);
    }
    @Override public void onClosing(WebSocket ws, int code, String reason) {
        ws.close(1000, null);
    }
    @Override public void onFailure(WebSocket ws, Throwable t, Response r) {
        t.printStackTrace();
    }
});
```

### 10.4 JSR-356 客户端（Tyrus）

```java
ClientManager client = ClientManager.createClient();
Session session = client.connectToServer(new Endpoint() {
    @Override public void onOpen(Session s, EndpointConfig cfg) {
        s.addMessageHandler(String.class, msg -> System.out.println("recv: " + msg));
        try { s.getBasicRemote().sendText("hi"); } catch (Exception ignored) {}
    }
}, URI.create("ws://localhost:8080/ws"));
```

### 📝 笔试题 10-1：Spring WebSocket 的 STOMP 与原生 WebSocket 有什么关系？

STOMP 是**运行在 WebSocket 之上的帧协议**，定义了 `CONNECT`/`SEND`/`SUBSCRIBE`/`MESSAGE` 等命令和订阅 / 目的地语义。Spring 对其做了封装，自动帮你做消息分发、路由到 `@MessageMapping`。客户端需要使用 STOMP 库（如 `stomp.js`），不能直接用原生 `new WebSocket` 发原始 JSON。

---

## 11. Go 服务端与客户端

Go 生态常用 [`gorilla/websocket`](https://github.com/gorilla/websocket)（成熟稳定）与 `nhooyr/websocket`（更现代）。

### 11.1 服务端（gorilla）

```go
package main

import (
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  4096,
    WriteBufferSize: 4096,
    CheckOrigin:     func(r *http.Request) bool { return true }, // 生产要校验
}

type Hub struct {
    mu      sync.Mutex
    clients map[*websocket.Conn]bool
}

var hub = &Hub{clients: map[*websocket.Conn]bool{}}

func handler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil { log.Println(err); return }

    hub.mu.Lock(); hub.clients[conn] = true; hub.mu.Unlock()
    defer func() {
        hub.mu.Lock(); delete(hub.clients, conn); hub.mu.Unlock()
        conn.Close()
    }()

    conn.SetReadLimit(1 << 20)
    conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    conn.SetPongHandler(func(string) error {
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    // 定期 ping
    go func() {
        t := time.NewTicker(25 * time.Second)
        defer t.Stop()
        for range t.C {
            if err := conn.WriteControl(websocket.PingMessage, nil,
                time.Now().Add(5*time.Second)); err != nil { return }
        }
    }()

    for {
        mt, data, err := conn.ReadMessage()
        if err != nil { return }

        hub.mu.Lock()
        for c := range hub.clients {
            if c == conn { continue }
            c.WriteMessage(mt, data)
        }
        hub.mu.Unlock()
    }
}

func main() {
    http.HandleFunc("/ws", handler)
    log.Println("listening :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 11.2 客户端（gorilla）

```go
package main

import (
    "log"
    "time"

    "github.com/gorilla/websocket"
)

func main() {
    c, _, err := websocket.DefaultDialer.Dial("ws://localhost:8080/ws", nil)
    if err != nil { log.Fatal(err) }
    defer c.Close()

    c.WriteMessage(websocket.TextMessage, []byte("hello from go"))

    done := make(chan struct{})
    go func() {
        defer close(done)
        for {
            _, msg, err := c.ReadMessage()
            if err != nil { log.Println("read:", err); return }
            log.Printf("recv: %s", msg)
        }
    }()

    select {
    case <-done:
    case <-time.After(30 * time.Second):
        c.WriteMessage(websocket.CloseMessage,
            websocket.FormatCloseMessage(websocket.CloseNormalClosure, "bye"))
    }
}
```

---

## 12. 聊天室实战：广播与房间

### 12.1 业务模型

- **连接（Connection）**：一个客户端 = 一个 WebSocket 会话
- **用户（User）**：登录鉴权后绑定到连接
- **房间（Room）**：一组订阅了同一话题的连接
- **消息（Message）**：带 type、sender、ts、payload

### 12.2 Node.js 房间实现

```javascript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

// room -> Set<ws>
const rooms = new Map();
// ws -> { user, rooms: Set<string> }
const metas = new WeakMap();

function join(ws, room) {
  if (!rooms.has(room)) rooms.set(room, new Set());
  rooms.get(room).add(ws);
  metas.get(ws).rooms.add(room);
}

function leave(ws, room) {
  rooms.get(room)?.delete(ws);
  metas.get(ws)?.rooms.delete(room);
}

function broadcast(room, msg, except) {
  const set = rooms.get(room);
  if (!set) return;
  const str = JSON.stringify(msg);
  for (const c of set) {
    if (c !== except && c.readyState === c.OPEN) c.send(str);
  }
}

wss.on("connection", (ws, req) => {
  metas.set(ws, { user: null, rooms: new Set() });

  ws.on("message", (raw) => {
    let msg;
    try { msg = JSON.parse(raw.toString()); } catch { return ws.close(1007); }
    const meta = metas.get(ws);

    switch (msg.type) {
      case "auth":
        // TODO: 验证 token
        meta.user = { id: msg.uid, name: msg.name };
        ws.send(JSON.stringify({ type: "auth.ok" }));
        break;
      case "join":
        join(ws, msg.room);
        broadcast(msg.room,
          { type: "system", text: `${meta.user.name} joined` }, ws);
        break;
      case "leave":
        broadcast(msg.room,
          { type: "system", text: `${meta.user.name} left` }, ws);
        leave(ws, msg.room);
        break;
      case "chat":
        if (!meta.user) return;
        broadcast(msg.room, {
          type: "chat", from: meta.user.name,
          text: msg.text, ts: Date.now()
        });
        break;
      case "ping":
        ws.send(JSON.stringify({ type: "pong", ts: Date.now() }));
        break;
    }
  });

  ws.on("close", () => {
    const meta = metas.get(ws);
    if (!meta) return;
    for (const r of meta.rooms) {
      rooms.get(r)?.delete(ws);
      broadcast(r, { type: "system", text: `${meta.user?.name ?? "?"} left` });
    }
  });
});
```

### 12.3 协议建议

统一消息信封，留出扩展：

```json
{
  "id": "uuid",              // 客户端消息 id，便于 ACK
  "type": "chat|auth|join|...",
  "room": "general",
  "payload": { "...": "..." },
  "ts": 1700000000000
}
```

### 📝 笔试题 12-1：用 `Map<room, Set<ws>>` 存房间有什么问题？

- 连接断开时**必须清理**该连接在所有房间中的引用，否则内存泄漏
- 高并发下 `Set.iterator` 广播需注意并发修改（Node 单线程无并发；Go/Java 要加锁或用并发集合）
- 对大房间广播是 O(N) 发送，需评估单实例上限；超过则要分片或分房间

---

## 13. 鉴权、CORS 与安全

### 13.1 握手阶段鉴权

WebSocket 握手是一次 HTTP 请求，可以：

- **Cookie**：浏览器自动带，服务端在 `upgrade` 事件里校验
- **Query token**：`wss://api/ws?token=xxx`（**token 可能进日志**，有风险）
- **Sub-protocol**：`Sec-WebSocket-Protocol: bearer, <jwt>`，服务端解析
- **自定义头**：仅非浏览器客户端可用（浏览器 `new WebSocket` 不能加自定义头）

**推荐**：短期一次性 ticket，建连时交换为 session，避免 token 泄漏。

### 13.2 浏览器安全模型

- 浏览器不受同源策略**拒绝 WebSocket 跨源**，但握手带 `Origin` 头
- 服务端**必须校验 Origin**：

```javascript
// Node ws
const wss = new WebSocketServer({
  verifyClient: (info) => {
    return info.origin === "https://app.example.com";
  }
});
```

### 13.3 TLS（wss://）

生产必须 `wss://`：

- 防止中间人篡改
- 穿越企业代理更顺畅（许多代理对明文 WS 不友好）
- HSTS 场景下浏览器会自动把 `ws://` 升级为 `wss://`

### 13.4 输入校验与限流

- **消息大小限制**（`maxPayload`），防内存攻击
- **速率限制**：每连接 QPS、消息数
- **JSON 深度限制**：避免恶意嵌套
- **脏数据过滤**：拒绝非法 UTF-8、protocol error 即断开

### 13.5 CSRF 思考

WebSocket **无 CORS 预检**，但 Cookie 仍会自动带。所以：

- 始终校验 `Origin`
- Cookie 必须 `SameSite=Lax` / `Strict`
- 或使用 token 而不依赖 Cookie

### 📝 笔试题 13-1：浏览器 `new WebSocket` 能加 `Authorization` 头吗？

**不能**。浏览器 API 不支持自定义头。解决：

1. 登录时签发短期一次性 ticket → 作为 query 传递（`?t=xxx`），握手成功立即失效
2. 或用 HttpOnly Cookie（服务端在 upgrade 时读取）
3. 或在 `Sec-WebSocket-Protocol` 里夹带 token（hack，需服务端配合）

---

## 14. 反向代理与负载均衡

### 14.1 Nginx

```nginx
# 关键：Upgrade 头在 proxy_pass 时会被过滤，需显式转发
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream ws_backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    # WebSocket 默认轮询即可；粘性视业务
    # ip_hash;            # 基于客户端 IP 会话粘性
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    location /ws {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 长连接相关超时：必须大于心跳间隔
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### 14.2 其他

- **HAProxy**：成熟，配 `option http-keep-alive` + ACL 区分 WS
- **AWS ALB**：原生支持，默认空闲超时 60s，**必须调大**或应用心跳 < 60s
- **Envoy / Traefik**：现代微服务网关，原生支持 WebSocket
- **CDN**：Cloudflare、Fastly 支持 WebSocket，但需注意 CDN 端空闲超时

### 14.3 会话粘性（Sticky Session）

WebSocket 是**长连接**，一旦建立就绑在某个后端实例上，LB **不再为每帧路由**。所以：

- 粘性在 WS **握手**时决定，不是每次消息决定
- 状态（如房间映射）只能在该实例上。**扩展集群**需要消息队列（见下章）

### 📝 笔试题 14-1：Nginx 反代 WebSocket 时配置的关键点？

1. `proxy_http_version 1.1`（HTTP/1.1 才有 Upgrade 语义）
2. `proxy_set_header Upgrade $http_upgrade`
3. `proxy_set_header Connection $connection_upgrade`（需通过 map 设置）
4. 调大 `proxy_read_timeout` / `proxy_send_timeout`
5. 生产 `wss://`（SSL 配置好）

---

## 15. 扩展：集群、消息广播与消息队列

### 15.1 单机瓶颈

单个 WebSocket 实例能承载数万到十几万并发连接，受：

- 文件描述符（`ulimit -n`）
- 内存（每连接几十 KB~数百 KB）
- CPU（序列化、压缩、广播）

### 15.2 多实例广播

实例 A 的用户发的消息要送到实例 B 上的订阅者，需要**跨实例通信**。常用：

- **Redis Pub/Sub**：简单轻量，适合消息量不大的聊天室
- **Kafka**：高吞吐，可持久化，多分区
- **NATS**：低延迟、主题订阅，WS 生态友好
- **RabbitMQ**：企业场景
- **云厂商服务**：AWS API Gateway WebSocket、Ably、Pusher、PubNub

### 15.3 Node.js + Redis 广播示例

```javascript
import { WebSocketServer } from "ws";
import Redis from "ioredis";

const pub = new Redis();
const sub = new Redis();
const wss = new WebSocketServer({ port: 8080 });

const rooms = new Map();            // room -> Set<ws>

sub.psubscribe("room:*");
sub.on("pmessage", (_, channel, raw) => {
  const room = channel.slice(5);    // strip "room:"
  const set = rooms.get(room);
  if (!set) return;
  for (const c of set) if (c.readyState === c.OPEN) c.send(raw);
});

wss.on("connection", (ws) => {
  ws.on("message", (raw) => {
    const msg = JSON.parse(raw.toString());
    if (msg.type === "join") {
      if (!rooms.has(msg.room)) rooms.set(msg.room, new Set());
      rooms.get(msg.room).add(ws);
    } else if (msg.type === "chat") {
      pub.publish(`room:${msg.room}`, JSON.stringify({
        from: msg.user, text: msg.text, ts: Date.now(),
      }));
    }
  });
});
```

每个实例都是**订阅者 + 发布者**，实例间通过 Redis 频道共享消息。

### 15.4 横向扩展注意事项

- **用户路由**：按用户 ID 分片到固定实例（hash），或每实例保存 `userId → ws` 表，跨实例通过 MQ 定向
- **状态一致性**：用户在哪个实例？用 Redis 存 `user:online` 标记
- **消息顺序**：单房间尽量走同一 Kafka 分区
- **离线消息**：服务端存未读，重连后拉取

---

## 16. 测试与排障

### 16.1 命令行测试

- **`wscat`**（Node）：
  ```bash
  npm i -g wscat
  wscat -c wss://example.com/ws
  ```
- **`websocat`**（Rust，功能更强）：
  ```bash
  websocat wss://example.com/ws
  websocat -t - ws://localhost:8080   # 带交互
  ```
- **`curl`** 可测**握手**（不支持数据帧）：
  ```bash
  curl -i -N \
    -H "Connection: Upgrade" -H "Upgrade: websocket" \
    -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
    -H "Sec-WebSocket-Version: 13" \
    -H "Origin: http://example.com" \
    http://localhost:8080/ws
  ```

### 16.2 浏览器 DevTools

Chrome DevTools **Network → 过滤 WS** 标签可查看：

- 握手请求/响应
- 帧列表（方向、长度、opcode、内容预览）
- 连接状态、持续时间

### 16.3 抓包

Wireshark 过滤：

```
websocket
tcp.port == 8080
```

可看到完整帧；TLS 需导入 `SSLKEYLOGFILE` 才能解密。

### 16.4 常见问题

| 现象 | 可能原因 | 排查 |
|------|----------|------|
| 握手 403/401 | Origin / token 校验失败 | 查服务端日志和请求头 |
| 握手 400 | 缺少 Upgrade 头；中间代理剥离 | 确认代理 `proxy_set_header Upgrade` |
| 连接空闲一会儿就断 | LB 空闲超时 < 心跳间隔 | 缩短心跳或调大 LB 超时 |
| 大消息断开 | 超 `maxPayload` | 服务端日志看帧大小；分片发送 |
| 跨域失败 | `Origin` 未放行 | 服务端放行；或同域部署 |
| `1006` 频发 | 移动网络切换、代理 RST | 重连 + 指数退避 |
| CPU 高 | 过多小消息 + `permessage-deflate` 压缩 | 评估关闭压缩或聚合小包 |
| 内存涨 | 连接泄漏，未清理房间引用 | 统计活跃连接数 vs 房间引用数 |

### 📝 笔试题 16-1：`curl` 能完整测试 WebSocket 吗？

只能测**握手**（101 响应）。因为 WebSocket 握手成功后是二进制帧协议，`curl` 没实现帧解析，无法发送/接收 WebSocket 数据帧。完整测试需要 `wscat` / `websocat` / 浏览器 / 任意 WebSocket 库。

---

## 17. WebSocket vs SSE vs 长轮询 vs WebTransport

| 维度 | 长轮询 | SSE | WebSocket | WebTransport |
|------|--------|-----|-----------|---------------|
| 方向 | 客户端发，服务端回 | 服务端 → 客户端 | 全双工 | 全双工 |
| 协议 | HTTP | HTTP/1.1+ | 升级后自有协议 | 基于 HTTP/3 (QUIC) |
| 文本/二进制 | 任意 | 仅文本 | 都支持 | 都支持 |
| 浏览器支持 | 所有 | 较广 | 广 | 新兴，Chrome/Edge |
| 自动重连 | 需自实现 | 原生（`EventSource`） | 需自实现 | 视库 |
| 多路复用 | 无 | 无 | 无 | ✅（流） |
| 中间件友好 | 最好 | 好 | 视代理 | 需 HTTP/3 支持 |

**选型建议**：

- 单向推送（仪表盘、通知、日志流）→ **SSE**（实现简单、自带重连）
- 双向实时（聊天、协作、游戏）→ **WebSocket**
- 高性能、多路复用、未来向 → **WebTransport**（HTTP/3 普及中）
- 极老浏览器 / 极简需求 → **长轮询**

### 📝 笔试题 17-1：SSE 和 WebSocket 的主要区别？

- SSE 是**单向**（服务端→客户端），WebSocket 是**全双工**
- SSE 基于 HTTP 文本流（`text/event-stream`），WebSocket 是独立帧协议
- SSE 内置自动重连、事件 ID；WebSocket 需要自己实现
- SSE 只能发文本，WebSocket 支持二进制
- 浏览器兼容：SSE 老一些但简单；WebSocket 更通用

---

## 18. 综合笔试练习

### 18.1 选择题

**Q1** WebSocket 握手成功的状态码是？
A. 200  B. 101  C. 200 + Upgrade 头  D. 204

<details><summary>答案</summary>B。</details>

**Q2** 下列关于帧掩码描述正确的是？
A. 服务端必须加掩码
B. 客户端发给服务端的数据帧必须加掩码
C. 掩码是为了加密
D. 掩码长度是 8 字节

<details><summary>答案</summary>B。掩码不是加密，4 字节，目的是防代理缓存污染攻击。</details>

**Q3** 下列哪个 Close Code 表示"连接异常断开"？
A. 1000  B. 1001  C. 1006  D. 1011

<details><summary>答案</summary>C。</details>

**Q4** 在 Nginx 反向代理 WebSocket 时，哪项**不是**必需配置？
A. `proxy_http_version 1.1`
B. `proxy_set_header Upgrade $http_upgrade`
C. `proxy_set_header Connection $connection_upgrade`
D. `proxy_cache on`

<details><summary>答案</summary>D。</details>

**Q5** 浏览器端 `WebSocket` 构造函数中的第二个参数 `protocols` 指的是？
A. TLS 版本
B. 子协议（Sec-WebSocket-Protocol）
C. 超时时间
D. HTTP 方法

<details><summary>答案</summary>B。</details>

**Q6** 关于 Socket.IO，错误的是？
A. 可降级为长轮询
B. 自带房间和命名空间
C. 普通 WebSocket 客户端可直接连接 Socket.IO 服务
D. 协议在 WebSocket 之上

<details><summary>答案</summary>C。</details>

**Q7** 对于需要跨多个 WebSocket 实例广播的场景，通常用？
A. 直接 HTTP 调用
B. 共享 Redis Pub/Sub / Kafka / NATS
C. 共享数据库
D. 定时轮询

<details><summary>答案</summary>B。</details>

**Q8** SSE 与 WebSocket 相比的核心差异？
A. SSE 不能发送二进制
B. SSE 是双向
C. SSE 无法自动重连
D. SSE 基于 UDP

<details><summary>答案</summary>A。</details>

### 18.2 判断题

1. WebSocket 必须使用 TLS。 ❌（生产推荐，规范不强制）
2. 控制帧（Ping/Pong/Close）可以分片。 ❌
3. 客户端和服务端都可以先发 Close 帧。 ✅
4. WebSocket 消息保留边界（不会像 TCP 字节流那样被拆）。 ✅（库按帧重组为消息）
5. `permessage-deflate` 压缩总能提升性能。 ❌（CPU 开销与内存可能得不偿失）
6. 浏览器 WebSocket 可以设置自定义 Header。 ❌
7. 单 WebSocket 连接上同一时刻只能发一个消息。 ✅（消息层面串行；底层可能分片）
8. `wss://` 默认端口 443。 ✅

### 18.3 简答题

**Q1** WebSocket 断线重连的完整策略？

1. 监听 `close`，判断 code：`1000` 正常不重连，其他尝试重连
2. **指数退避 + 抖动**：1s → 2s → 4s ... 封顶 30s，每次 ±随机 ms
3. 重连后重新做：认证 → 恢复订阅/房间 → 同步增量状态
4. 应用层心跳超时主动断开触发重连
5. 限制最大重试次数或时长，超过后降级 UI
6. 监控连接数、重连率，告警异常

**Q2** 设计一个支持 100 万连接的聊天系统的大致架构？

- **接入层**：多实例 WS Gateway（Node/Go），ALB/Nginx 分发，单实例 5-10 万连接
- **用户路由**：连接时记录 `user_id → gateway_instance`（Redis）
- **消息投递**：
  - 点对点：查路由，通过内部 RPC 或 MQ 定向
  - 房间广播：pub/sub（Redis/NATS），所有 gateway 订阅
- **持久化**：消息写 Kafka → 写 DB/Elasticsearch；离线消息写 Redis/DB
- **监控**：连接数、消息 QPS、延迟、错误码分布
- **灰度发布**：连接优雅迁移（通知客户端重连到新实例）
- **限流降级**：每连接 QPS 限制；高峰期关闭非核心推送

**Q3** 为什么浏览器 WebSocket 不支持自定义 Header？

为遵循浏览器安全模型：如果允许任意 Header，`Origin` / `Cookie` / `Authorization` 语义被用户代码篡改，服务端原本基于头的安全决策（同源、鉴权）会被绕过。浏览器明确：**握手的 HTTP 头由浏览器自动构造**，开发者仅能指定子协议（通过构造函数参数）。

### 18.4 实操题

**Q1** 用 Node 写一个 echo 服务端，对每条消息前加 `echo: ` 返回，支持心跳。

```javascript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  ws.isAlive = true;
  ws.on("pong", () => ws.isAlive = true);

  ws.on("message", (data, isBinary) => {
    if (isBinary) { ws.send(data); return; }
    ws.send(`echo: ${data.toString()}`);
  });
});

setInterval(() => {
  for (const ws of wss.clients) {
    if (!ws.isAlive) { ws.terminate(); continue; }
    ws.isAlive = false;
    ws.ping();
  }
}, 30000);
```

**Q2** 写一个浏览器端最小 WebSocket 聊天示例（HTML 内嵌）。

```html
<!DOCTYPE html>
<html>
<body>
  <input id="in" autofocus /><button id="send">Send</button>
  <pre id="log"></pre>
  <script>
    const log = (t) => document.getElementById("log").textContent += t + "\n";
    const ws = new WebSocket("ws://localhost:8080");
    ws.addEventListener("open", () => log("[open]"));
    ws.addEventListener("message", (e) => log("< " + e.data));
    ws.addEventListener("close", (e) => log(`[close ${e.code}]`));
    document.getElementById("send").onclick = () => {
      const v = document.getElementById("in").value;
      ws.send(v); log("> " + v);
    };
  </script>
</body>
</html>
```

**Q3** 在现有 Express 应用中增加 `/ws` 端点，仅允许携带 `?token=xxx` 的客户端连接。

```javascript
import express from "express";
import { WebSocketServer } from "ws";
import url from "url";
import jwt from "jsonwebtoken";

const app = express();
app.get("/", (_, res) => res.send("ok"));
const server = app.listen(8080);

const wss = new WebSocketServer({ noServer: true });

server.on("upgrade", (req, socket, head) => {
  const { pathname, query } = url.parse(req.url, true);
  if (pathname !== "/ws") { socket.destroy(); return; }

  try {
    req.user = jwt.verify(query.token, process.env.JWT_SECRET);
  } catch {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n"); socket.destroy(); return;
  }

  wss.handleUpgrade(req, socket, head, (ws) => {
    wss.emit("connection", ws, req);
  });
});

wss.on("connection", (ws, req) => {
  ws.send(`welcome ${req.user.name}`);
});
```

**Q4** 某场景高并发短消息，CPU 打满，你怎么优化？

- **关闭或限制 `permessage-deflate`**：压缩小消息往往得不偿失
- **聚合小消息**：应用层每 N ms flush 一次批量
- **复用缓冲区**：避免频繁 `JSON.stringify` 的分配
- **二进制协议**：Protobuf/MsgPack 比 JSON 更快更小
- **水平扩展 + MQ**：降低单实例负载
- **合理广播粒度**：粗房间拆细，避免每条消息广播给无关连接
- **分析火焰图**：定位热点（序列化？压缩？遍历？）

---

## 📚 学习建议

1. **读 RFC 6455**：篇幅不长，权威且清晰
2. **抓一次真实握手 + 帧**：Wireshark / DevTools 加深理解
3. **手写一次最小协议栈**：做一个能和浏览器对话的 WS 服务，对帧格式会了如指掌
4. **先稳定再优化**：心跳、重连、关闭清理这三件套稳了再谈性能
5. **不要轻易上 Socket.IO 之类**：除非确实需要其高级特性；原生 WebSocket + 约定协议更透明可控
6. **压测**：用 `artillery` / `k6` / 自写脚本打到瓶颈，提前了解容量

> 祝你的连接常开、消息不丢。
