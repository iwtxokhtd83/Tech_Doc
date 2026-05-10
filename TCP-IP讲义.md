# TCP/IP 协议栈讲义

> 本讲义系统讲解 TCP/IP 协议栈，覆盖从物理到应用的各层关键协议、数据包格式、核心机制与常见笔试题。适合网络工程师、后端开发、面试复习。
>
> 阅读约定：RFC 为主流规范来源；`MTU` 默认 1500，`MSS` 默认 1460；`n` 表示字节数或序号。

## 目录

1. [网络分层模型](#1-网络分层模型)
2. [物理层与数据链路层](#2-物理层与数据链路层)
3. [网络层：IP 协议](#3-网络层ip-协议)
4. [ARP 与 ICMP](#4-arp-与-icmp)
5. [IP 路由与子网划分](#5-ip-路由与子网划分)
6. [传输层：UDP](#6-传输层udp)
7. [传输层：TCP 基础](#7-传输层tcp-基础)
8. [TCP 三次握手与四次挥手](#8-tcp-三次握手与四次挥手)
9. [TCP 可靠传输机制](#9-tcp-可靠传输机制)
10. [TCP 流量控制与拥塞控制](#10-tcp-流量控制与拥塞控制)
11. [TCP 状态机与常见状态](#11-tcp-状态机与常见状态)
12. [DNS](#12-dns)
13. [HTTP 与 HTTPS](#13-http-与-https)
14. [其他常用应用层协议](#14-其他常用应用层协议)
15. [NAT 与防火墙](#15-nat-与防火墙)
16. [Socket 编程基础](#16-socket-编程基础)
17. [抓包与排障工具](#17-抓包与排障工具)
18. [常见 Socket 错误码深入](#18-常见-socket-错误码深入)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 网络分层模型

### 1.1 OSI 七层 vs TCP/IP 四层 / 五层

| OSI 七层 | TCP/IP 四层 | TCP/IP 五层（教学） | 典型协议 / 设备 |
|----------|-------------|---------------------|------------------|
| 应用层 | 应用层 | 应用层 | HTTP, DNS, FTP, SMTP, SSH |
| 表示层 | 应用层 | 应用层 | TLS, JPEG, ASCII |
| 会话层 | 应用层 | 应用层 | RPC, NetBIOS |
| 传输层 | 传输层 | 传输层 | TCP, UDP, QUIC |
| 网络层 | 网际层 | 网络层 | IP, ICMP, ARP, 路由器 |
| 数据链路层 | 网络接入层 | 数据链路层 | 以太网, PPP, 交换机 |
| 物理层 | 网络接入层 | 物理层 | 网线, 光纤, 集线器 |

### 1.2 数据封装与解封装

从上到下：每层给数据加**首部**（部分加尾部）。

```
Application:    [应用数据]
Transport:      [TCP头|   应用数据   ]   → 段 Segment
Network:        [IP头 |TCP头| 数据  ]   → 包 Packet / 数据报
Link:           [帧头 |IP头|TCP头|数据|帧尾] → 帧 Frame
Physical:       比特流
```

接收端逐层剥壳，数据流向上送到对应进程。

### 1.3 各层寻址

| 层 | 寻址方式 | 范围 |
|----|----------|------|
| 数据链路层 | MAC 地址（48 位） | 局域网内 |
| 网络层 | IP 地址 | 跨网络 |
| 传输层 | 端口号 | 主机内进程 |
| 应用层 | URL / 域名 | 可读资源标识 |

### 📝 笔试题 1-1

描述浏览器访问 `https://www.example.com` 的完整过程（面试高频）。

**关键步骤**：

1. **DNS 解析**：递归/迭代查询，获取 IP（命中本地/浏览器/系统/路由器缓存则跳过）
2. **TCP 连接**：三次握手到 443 端口
3. **TLS 握手**：协商版本/密码套件，交换证书与密钥
4. **HTTP 请求**：发送 `GET / HTTP/1.1`（或 HTTP/2 帧）
5. **服务器响应**：返回 HTML
6. **渲染**：解析 HTML → 构建 DOM → 请求 CSS/JS/图片 → 合成渲染
7. **TCP 断开**：HTTP keep-alive 或四次挥手

---

## 2. 物理层与数据链路层

### 2.1 物理层

负责将**比特流**转换成物理信号。关注带宽、编码、传输介质。不做展开。

### 2.2 以太网帧格式（Ethernet II）

```
| 前导 8B | 目的MAC 6B | 源MAC 6B | 类型 2B | 数据 46-1500B | FCS 4B |
```

- **MAC 地址**：48 位，前 24 位 OUI 厂商码，后 24 位序号；全 1 为广播 `FF:FF:FF:FF:FF:FF`
- **类型字段**：`0x0800` IPv4，`0x86DD` IPv6，`0x0806` ARP
- **FCS**：帧校验序列（CRC-32）
- **MTU**：最大传输单元，以太网默认 1500 字节（数据部分）

### 2.3 交换机工作原理

- 基于 **MAC 地址表**转发
- **自学习**：收到帧时记录源 MAC 与端口映射
- **泛洪**：未知目的 MAC 时，除入端口外所有端口转发
- **广播**：收到广播帧向所有端口转发（不含入口）
- 工作在**二层**；冲突域按端口隔离，广播域按 VLAN 隔离

### 2.4 VLAN（虚拟局域网）

- IEEE 802.1Q 标准，帧中插入 **4 字节 VLAN Tag**（含 12 位 VID）
- 最多 4094 个 VLAN（0 和 4095 保留）
- 作用：隔离广播域、安全分区、跨交换机逻辑分组

### 📝 笔试题 2-1：交换机与集线器的区别？

- 集线器（Hub）：物理层设备，**所有端口共享带宽**，半双工，一个冲突域
- 交换机（Switch）：数据链路层设备，**每端口独立带宽**，全双工，按端口隔离冲突域

---

## 3. 网络层：IP 协议

### 3.1 IPv4 报文格式（20 字节固定头）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |    DSCP   |ECN|         Total Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Identification         |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     TTL       |   Protocol    |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Source IP                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Destination IP                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Options (variable)                        |
```

**关键字段**：

- **Version**：4 或 6
- **IHL**：首部长度，以 4 字节为单位，最小 5（20 字节）
- **Total Length**：总长（含首部），最大 65535
- **TTL**：生存时间，每经过一个路由器减 1，减为 0 丢弃（防环）
- **Protocol**：上层协议，`6=TCP`，`17=UDP`，`1=ICMP`
- **Checksum**：只校验首部
- **Identification / Flags / Offset**：**分片重组**用

### 3.2 IP 分片

- 当数据包大于链路 MTU 时，路由器分片
- 所有分片 `Identification` 相同；`MF`(More Fragments) 标志置位（最后一片为 0）；`Offset` 以 8 字节为单位
- 接收端 IP 层重组；**任一分片丢失整个包丢弃**
- 现代网络倾向**避免分片**：TCP 协商 MSS；IPv6 禁止路由器分片（只能源端分），需 Path MTU Discovery

### 3.3 IPv4 地址

- 32 位，点分十进制：`192.168.1.1`
- **分类**（历史）：
  - A：`0.0.0.0 - 127.255.255.255`，默认掩码 `/8`
  - B：`128.0.0.0 - 191.255.255.255`，`/16`
  - C：`192.0.0.0 - 223.255.255.255`，`/24`
  - D：`224.0.0.0 - 239.255.255.255`，多播
  - E：`240.0.0.0 - 255.255.255.255`，保留
- **私有地址**（RFC 1918）：
  - `10.0.0.0/8`
  - `172.16.0.0/12`
  - `192.168.0.0/16`
- **特殊地址**：
  - `0.0.0.0`：本机 / 未指定
  - `127.0.0.0/8`：回环
  - `169.254.0.0/16`：链路本地（APIPA）
  - `255.255.255.255`：受限广播

### 3.4 IPv6 简介

- 128 位地址，写作 8 组 16 位十六进制：`2001:0db8::1`
- 去掉分类和广播，使用**多播 + 任播**
- 头部简化到 40 字节固定长度（取消校验和、分片字段）
- 内置 IPSec 支持；**路由器不分片**
- 过渡技术：双栈、隧道（6to4, 6in4）、NAT64/DNS64

### 📝 笔试题 3-1：TTL 归零后如何处理？

路由器丢弃该包并**发送 ICMP Time Exceeded** 给源 IP。这是 `traceroute` 的工作原理：逐步增大 TTL（1, 2, 3...）观察每跳返回的 ICMP。

### 📝 笔试题 3-2：为什么 IPv4 首部校验和只校验首部？

- 上层（TCP/UDP）已有覆盖数据的校验和，避免重复
- 每经过一跳 TTL 改变就要**重新计算**，只算首部能减小路由器开销

---

## 4. ARP 与 ICMP

### 4.1 ARP（地址解析协议）

作用：**IP → MAC 地址**映射（同一广播域内）。

**流程**：

1. 发送方查 ARP 缓存；未命中则广播 ARP Request
2. 目标主机收到后**单播**回复 ARP Reply
3. 发送方更新缓存（通常超时 1-20 分钟）

**ARP 报文结构**（28 字节）：操作码 `1` 请求 / `2` 应答。

**ARP 欺骗**：攻击者伪造 ARP 应答，劫持局域网流量。防御：静态 ARP、DAI（Dynamic ARP Inspection）。

### 4.2 免费 ARP（Gratuitous ARP）

主机发送目的 IP 为自身 IP 的 ARP Request：

- 检测 IP 冲突
- 主动通知网内其他主机更新缓存（VRRP 主备切换用）

### 4.3 ICMP（网际控制报文协议）

IP 的**错误和控制**伴生协议，封装在 IP 包中（Protocol = 1）。

**常用类型**：

| Type | Code | 含义 |
|------|------|------|
| 0 | 0 | Echo Reply（ping 响应） |
| 3 | 0-15 | Destination Unreachable |
| 3 | 4 | Fragmentation Needed（PMTUD 用） |
| 5 | 0 | Redirect（路由重定向） |
| 8 | 0 | Echo Request（ping 请求） |
| 11 | 0 | Time Exceeded（TTL 超时） |
| 12 | 0 | Parameter Problem |

### 📝 笔试题 4-1：ping 和 traceroute 的区别？

- **ping**：发 ICMP Echo Request，收 Echo Reply，测连通性与 RTT
- **traceroute**：
  - Unix 传统做法：发 UDP 到高端口，依次设置 TTL=1,2,3...，根据 ICMP Time Exceeded 获取每跳
  - Windows `tracert`：直接用 ICMP Echo
  - 终点：收到目的不可达（UDP 版本）或 Echo Reply（ICMP 版本）

---

## 5. IP 路由与子网划分

### 5.1 子网掩码与 CIDR

- 掩码：网络位为 1，主机位为 0
- **CIDR**：`IP/前缀长度`，如 `192.168.1.0/24`
- 网络地址：IP **按位与**掩码；广播地址：主机位全 1

**示例**：`192.168.10.130/26`

- 掩码：`255.255.255.192`
- 每子网主机数：`2^6 - 2 = 62`
- 子网范围：`192.168.10.128 - 192.168.10.191`
- 网络地址 `.128`；广播 `.191`；可用 `.129 - .190`

### 5.2 子网划分技巧

给定需求主机数 `n`，求所需主机位：`ceil(log2(n + 2))`（扣掉网络和广播）。

给定子网数 `m`，从原网络借 `ceil(log2(m))` 位作子网号。

### 5.3 路由表与转发

路由器维护**路由表**，每条含：目的网络、掩码、下一跳、出接口、度量值。

**最长前缀匹配**：选择匹配位数最多的路由条目。

```
目的 IP: 192.168.1.100
候选:
  0.0.0.0/0      -> 10.0.0.1       (默认)
  192.168.0.0/16 -> 10.0.0.2
  192.168.1.0/24 -> 10.0.0.3       ← 最长匹配
```

### 5.4 路由协议分类

- **内部网关协议 IGP**（自治系统内）
  - **RIP**：距离矢量，按跳数，最大 15，收敛慢（已淘汰）
  - **OSPF**：链路状态，按带宽算开销，Dijkstra 算最短路径，收敛快
  - **IS-IS**：类似 OSPF，多用于运营商
- **外部网关协议 EGP**（自治系统间）
  - **BGP**：路径矢量，基于策略，互联网骨干协议

### 📝 笔试题 5-1：给出 `172.16.5.0/22` 的网络地址与广播地址。

- `/22` 掩码 `255.255.252.0`
- 第三段网络位 6 位：`00000100`(4) 掩码 `11111100`，结果 `00000100`(4)
- 网络地址：`172.16.4.0`
- 广播地址：`172.16.7.255`
- 可用主机：`172.16.4.1 - 172.16.7.254`（1022 个）

---

## 6. 传输层：UDP

### 6.1 UDP 报文格式（8 字节头）

```
|  源端口 2B  |  目的端口 2B  |
|  长度 2B   |  校验和 2B   |
|          数据            |
```

### 6.2 特性

- **无连接**：不握手，直接发
- **不可靠**：不保证到达、不保证有序、不重传
- **面向报文**：一个 `sendto` 对应一个 UDP 数据报，**不拆不合**
- **首部仅 8 字节**，开销小
- **支持一对多**：单播、广播、多播均可

### 6.3 典型应用

- DNS 查询（小查询快速响应）
- DHCP
- NTP
- 视频/语音流（容忍少量丢包，延迟敏感）
- QUIC（在 UDP 上实现可靠传输，HTTP/3 基础）
- 游戏、SNMP

### 📝 笔试题 6-1：UDP 校验和的特殊性？

UDP 校验和覆盖：UDP 首部 + 数据 + **伪首部**（源 IP、目的 IP、协议号、UDP 长度）。这让校验具备**端到端**语义，也是 NAT 会破坏校验和的原因。IPv4 下 UDP 校验和可选；IPv6 下强制。

---

## 7. 传输层：TCP 基础

### 7.1 TCP 报文格式（最少 20 字节头）

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Acknowledgment Number                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Data  |  Rsv  |C|E|U|A|P|R|S|F|          Window Size          |
| Offset|       |W|C|R|C|S|S|Y|I|                               |
|       |       |R|E|G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
```

### 7.2 关键字段

- **序列号（SEQ）**：本报文段**第一个字节**的编号
- **确认号（ACK）**：**期望收到**对方下一个字节的编号（累积确认）
- **Data Offset**：首部长度（4 字节为单位）
- **窗口大小**：本端接收窗口剩余字节数（滑动窗口）
- **控制位**：
  - **SYN**：建立连接
  - **ACK**：确认有效
  - **FIN**：释放连接
  - **RST**：强制复位
  - **PSH**：尽快递交
  - **URG**：紧急指针有效
- **Options**：MSS、窗口缩放、SACK、时间戳等

### 7.3 特性

- **面向连接**：握手建立，挥手释放
- **可靠传输**：重传、排序、去重、校验
- **面向字节流**：无报文边界；应用层自行处理粘包
- **全双工**：双向独立数据流
- **一对一**：不支持多播

### 📝 笔试题 7-1：TCP 与 UDP 的区别？

| 维度 | TCP | UDP |
|------|-----|-----|
| 连接 | 面向连接 | 无连接 |
| 可靠 | 可靠 | 不可靠 |
| 顺序 | 保证有序 | 不保证 |
| 流量/拥塞控制 | 有 | 无 |
| 头部 | 20-60 字节 | 8 字节 |
| 语义 | 字节流 | 数据报 |
| 应用 | HTTP, SSH, FTP | DNS, 视频, 游戏, QUIC |

---

## 8. TCP 三次握手与四次挥手

### 8.1 三次握手

```
Client                           Server
  |                                |
  |---- SYN(seq=x) --------------->| [LISTEN]
  |                                |
  |<--- SYN+ACK(seq=y, ack=x+1) ---|
  |                                |
  |---- ACK(seq=x+1, ack=y+1) ---->| [ESTABLISHED]
  |                                |
```

**状态迁移**：

- Client：`CLOSED → SYN_SENT → ESTABLISHED`
- Server：`LISTEN → SYN_RCVD → ESTABLISHED`

**为什么需要三次？**

- 两次不够：若首个 SYN 在网络中滞留后突然到达，服务端以为要建立新连接，造成**资源浪费**
- 三次握手让双方都确认：**自己的发送和接收能力 + 对方的发送和接收能力**

### 8.2 四次挥手

```
Client                          Server
  |                                |
  |---- FIN(seq=u) --------------->|
  | [FIN_WAIT_1]                   | [CLOSE_WAIT]
  |<--- ACK(ack=u+1) --------------|
  | [FIN_WAIT_2]                   |
  |                                | (处理剩余数据)
  |<--- FIN(seq=v) ----------------|
  |                                | [LAST_ACK]
  |---- ACK(ack=v+1) ------------->|
  | [TIME_WAIT]                    | [CLOSED]
  | 等 2*MSL                       |
  | [CLOSED]                       |
```

**为什么四次？**

TCP 全双工，每个方向独立关闭。服务端收到 FIN 后，可能还有数据未发完，所以不会立即回复 FIN，把 ACK 和 FIN 拆成两次。

### 8.3 TIME_WAIT 与 2×MSL

- **MSL (Maximum Segment Lifetime)**：报文最大生存时间，Linux 默认 30s（即 `TIME_WAIT` 持续 60s）
- **作用**：
  1. 确保最后一个 ACK 到达对端；若对端未收到会重传 FIN，本端才能回应
  2. 让旧连接的残留报文在网络中消失，避免干扰同一四元组的新连接

**大量 TIME_WAIT 的应对**：

- `SO_REUSEADDR` / `SO_REUSEPORT`
- 调整 `net.ipv4.tcp_tw_reuse`（Linux）
- 复用长连接（连接池、HTTP keep-alive）
- **慎用** `tcp_tw_recycle`（Linux 4.12+ 已移除，易与 NAT 冲突）

### 8.4 半连接队列与全连接队列

服务端维护两个队列：

- **SYN 队列（半连接）**：收到 SYN 未完成三握的连接
- **Accept 队列（全连接）**：完成三握等待 `accept()` 取走的连接

队列满会导致连接丢失或客户端超时。

- 半连接队列满 + `tcp_syncookies=1` 时启用 **SYN Cookies** 防 SYN Flood
- 全连接队列满时行为由 `tcp_abort_on_overflow` 控制（0：丢弃 ACK，1：回 RST）

### 📝 笔试题 8-1：客户端发 SYN 后挂了，服务端怎么办？

服务端处于 `SYN_RCVD`，会重传 SYN+ACK，默认 5 次，间隔按指数退避，最终约 63s 放弃，连接释放。`tcp_synack_retries` 可调。

### 📝 笔试题 8-2：为什么 TIME_WAIT 是 2×MSL 而不是 1×MSL？

确保**一次往返的最坏情况**：最后的 ACK 丢失 + 对端重传 FIN，各需一个 MSL。

### 📝 笔试题 8-3：能不能三次握手携带数据？

- 第一次 SYN 和第二次 SYN+ACK **不能带数据**（开销大易被 DoS）
- 第三次 ACK **可以带数据**，连接已建立

**TCP Fast Open (TFO)**：允许第一次 SYN 带数据，通过 Cookie 鉴权，减少 RTT。

---

## 9. TCP 可靠传输机制

### 9.1 累积确认 + 选择性确认

- 默认**累积确认**：`ack=n` 表示 `n-1` 及之前全部收到
- **SACK (Selective ACK)**：在选项中反馈**非连续**的已收段，减少不必要重传

### 9.2 超时重传（RTO）

- RTO = SRTT + 4 × RTTVAR（Jacobson/Karn 算法）
- 每次重传 RTO 翻倍（**指数退避**）
- 重传后不用该次 RTT 更新 SRTT（**Karn 算法**）

### 9.3 快速重传 / 快速恢复

**快速重传**：收到 3 个重复 ACK 即刻重传对应段，不必等 RTO。

**快速恢复**：配合快速重传，**不回到慢启动**，而是 `cwnd = ssthresh`，进入拥塞避免。

### 9.4 序号回绕（PAWS）

序号 32 位，在高速网络可能**绕回**。TCP 时间戳选项 + PAWS（Protect Against Wrapped Sequences）可判断新旧包。

### 📝 笔试题 9-1：收到乱序包 TCP 会发什么？

立即回复**重复 ACK**，ACK 号仍是期望的下一字节，不会跳过。用于触发发送方快速重传。

---

## 10. TCP 流量控制与拥塞控制

### 10.1 流量控制（接收方驱动）

- **滑动窗口**：接收方在 ACK 中通告 `Window Size`，告诉发送方还能接收多少字节
- **零窗口**：接收方缓冲满时通告 `Window=0`，发送方暂停
- **零窗口探测**：发送方定时发 1 字节探测，防止 ACK 丢失死锁
- **糊涂窗口综合症 (SWS)**：
  - 接收方对策：窗口至少增长到 MSS 或缓冲一半才通告
  - 发送方对策：**Nagle 算法**，小包攒到 MSS 或无未确认包才发

### 10.2 拥塞控制（网络驱动）

四大算法：**慢启动、拥塞避免、快速重传、快速恢复**。

#### 拥塞窗口 cwnd 与阈值 ssthresh

- 发送窗口 = `min(cwnd, rwnd)`
- `cwnd` 受拥塞控制调整，`rwnd` 受接收方通告

#### 慢启动

- 初始 `cwnd = 1 MSS`（新 RFC 允许 10）
- **每收到一个 ACK，cwnd +1**（即每 RTT 翻倍，指数增长）
- `cwnd ≥ ssthresh` 后进入拥塞避免

#### 拥塞避免

- 每 RTT 增加 1 MSS（线性增长）

#### 丢包处理

- **超时重传**：
  - `ssthresh = cwnd / 2`
  - `cwnd = 1`
  - 回到慢启动
- **三个重复 ACK（快速重传）**：
  - `ssthresh = cwnd / 2`
  - `cwnd = ssthresh`
  - 进入**快速恢复**（跳过慢启动）

### 10.3 常见拥塞控制算法

| 算法 | 特点 |
|------|------|
| Reno / NewReno | 经典，丢包驱动 |
| CUBIC | Linux 默认，立方函数增长，适合高 BDP 网络 |
| BBR | Google 提出，基于**带宽 × 延迟**建模，不依赖丢包 |
| Vegas | 基于 RTT 变化，早期拥塞感知 |

### 📝 笔试题 10-1：cwnd 和 rwnd 的区别？

- **rwnd**：接收方通告，反映**接收方缓冲**剩余
- **cwnd**：发送方本地维护，反映**网络拥塞**状况
- 实际发送窗口取二者最小值

### 📝 笔试题 10-2：为什么要区分超时重传和快速重传的处理？

三重复 ACK 表明**后续包已到达**，网络并非完全拥塞，只是丢了少量；保留较大 cwnd 能快速恢复吞吐。超时则说明网络严重拥塞，需**彻底回退**。

---

## 11. TCP 状态机与常见状态

### 11.1 状态列表

| 状态 | 说明 |
|------|------|
| CLOSED | 初始或已完全关闭 |
| LISTEN | 服务端监听 |
| SYN_SENT | 客户端发 SYN 等 SYN+ACK |
| SYN_RCVD | 服务端收 SYN 发 SYN+ACK 等 ACK |
| ESTABLISHED | 数据传输中 |
| FIN_WAIT_1 | 主动方发 FIN 等 ACK |
| FIN_WAIT_2 | 收到对方 ACK 等对方 FIN |
| CLOSE_WAIT | 被动方收到 FIN，等应用调用 close |
| LAST_ACK | 被动方发 FIN 等 ACK |
| TIME_WAIT | 主动方等 2×MSL |
| CLOSING | 双方同时关，少见 |

### 11.2 排障关键点

- 大量 `TIME_WAIT`（主动关闭端）：考虑长连接、`SO_REUSEADDR`、`tcp_tw_reuse`
- 大量 `CLOSE_WAIT`（被动关闭端）：应用**未调用 close**，代码 bug
- `SYN_RCVD` 堆积：可能遭受 **SYN Flood**，开 SYN Cookies
- 大量 `ESTABLISHED` 但无数据：连接泄漏或未启用 keepalive

### 11.3 TCP Keepalive

- 用于检测**长时间空闲**的连接是否存活
- Linux 参数：
  - `tcp_keepalive_time`：首次探测前空闲时间（默认 7200s）
  - `tcp_keepalive_intvl`：两次探测间隔（默认 75s）
  - `tcp_keepalive_probes`：失败阈值（默认 9）
- 应用层通常另实现心跳，粒度更可控

### 📝 笔试题 11-1：CLOSE_WAIT 很多说明什么？

**应用代码没有正确 `close()`**。对端已经关闭了写端（发 FIN），本端 TCP 回了 ACK 进入 `CLOSE_WAIT`，但应用没有感知或忘记关闭文件描述符。排查方向：`close` 遗漏、异常分支未关闭、读循环未处理 EOF。

---

## 12. DNS

### 12.1 DNS 层次结构

```
.                              ← 根
├── com.                       ← 顶级域 TLD
│   └── example.com.           ← 二级域
│       └── www.example.com.   ← 主机
├── org.
├── cn.
└── ...
```

### 12.2 查询流程

1. 浏览器缓存 → 操作系统缓存（hosts、stub resolver）
2. 本地 DNS（递归解析器，如运营商或 `8.8.8.8`）
3. 递归解析器**迭代查询**：根 → TLD → 权威
4. 权威返回记录，递归解析器返回给客户端

```
Client --------递归--------> Local DNS
                              |
                              |--迭代--> 根 (. 返回 com NS)
                              |--迭代--> com TLD (返回 example.com NS)
                              |--迭代--> example.com 权威 (返回 A)
                              |
Client <------ Answer --------+
```

### 12.3 常见记录类型

| 类型 | 说明 |
|------|------|
| A | 域名 → IPv4 |
| AAAA | 域名 → IPv6 |
| CNAME | 域名别名 |
| MX | 邮件交换，带优先级 |
| NS | 权威域名服务器 |
| TXT | 任意文本，用于 SPF/DKIM/域名验证 |
| PTR | IP → 域名（反向解析） |
| SOA | 区的起始授权 |
| SRV | 服务定位 |
| CAA | 指定可颁发证书的 CA |

### 12.4 DNS 传输

- 查询/响应默认使用 **UDP 53**
- 响应超过 512 字节或区传输 (AXFR)：改走 **TCP 53**
- 现代扩展：EDNS0 提升 UDP 载荷上限；**DoT** (DNS over TLS, 853)、**DoH** (DNS over HTTPS, 443) 提供加密

### 12.5 TTL 与缓存

每条记录带 TTL，决定解析器缓存时长。缩短 TTL 有利于快速切换，但增加查询负载。

### 📝 笔试题 12-1：`dig www.example.com` 返回 `CNAME example.cdn.com A ...` 说明什么？

域名配置了 CDN。原域名通过 CNAME 指向 CDN 厂商的别名，CDN 侧再做智能解析，按用户就近返回 A 记录。

---

## 13. HTTP 与 HTTPS

### 13.1 HTTP 版本演进

| 版本 | 关键特性 |
|------|----------|
| HTTP/0.9 | 仅 GET，纯文本 |
| HTTP/1.0 | 头部、状态码、MIME；默认短连接 |
| HTTP/1.1 | **keep-alive**、管线化、分块编码、Host 头、缓存控制 |
| HTTP/2 | 二进制分帧、**多路复用**、头部压缩 HPACK、服务端推送 |
| HTTP/3 | 基于 **QUIC/UDP**，0-RTT 握手，解决队头阻塞 |

### 13.2 请求 / 响应格式

```
请求:
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: text/html

响应:
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Cache-Control: max-age=3600

<html>...</html>
```

### 13.3 常用方法

| 方法 | 语义 | 幂等 | 安全 |
|------|------|------|------|
| GET | 读取 | ✅ | ✅ |
| POST | 创建/提交 | ❌ | ❌ |
| PUT | 整体替换 | ✅ | ❌ |
| PATCH | 局部更新 | ❌（规范上可幂等） | ❌ |
| DELETE | 删除 | ✅ | ❌ |
| HEAD | 仅头 | ✅ | ✅ |
| OPTIONS | 预检 | ✅ | ✅ |

**安全**：不修改资源；**幂等**：多次调用结果相同。

### 13.4 状态码

| 类 | 范围 | 含义 |
|----|------|------|
| 1xx | 信息 | 100 Continue, 101 Switching Protocols |
| 2xx | 成功 | 200 OK, 201 Created, 204 No Content, 206 Partial Content |
| 3xx | 重定向 | 301 永久, 302 临时, 304 Not Modified, 307/308 保留方法 |
| 4xx | 客户端错 | 400, 401, 403, 404, 409, 413, 429 |
| 5xx | 服务端错 | 500, 502, 503, 504 |

**301 vs 302 vs 307**：

- 301 永久，浏览器会缓存；302 临时
- 早期 302 在客户端可能改 POST 为 GET，307/308 保留原方法

### 13.5 缓存

**强缓存**：

- `Expires`（HTTP/1.0，绝对时间）
- `Cache-Control: max-age=3600, public/private, no-cache, no-store, must-revalidate`（HTTP/1.1，优先）

**协商缓存**：

- `Last-Modified` / `If-Modified-Since`
- `ETag` / `If-None-Match`（更精准，避免秒级精度和非修改性变化问题）

命中协商缓存返回 `304 Not Modified`，无响应体。

### 13.6 Cookie 与 Session

- **Cookie**：服务端 `Set-Cookie` 下发，客户端自动携带
- 属性：`Domain`, `Path`, `Expires/Max-Age`, `Secure`, `HttpOnly`, `SameSite`
- **Session**：服务端维护状态，SessionID 存于 Cookie
- **JWT**：自包含 token，三段（Header.Payload.Signature），无需服务端存储

### 13.7 HTTPS = HTTP + TLS

- 默认端口 **443**
- 保证：**机密性（加密）、完整性（MAC）、身份认证（证书）**

**TLS 1.2 握手（简化）**：

1. ClientHello（版本、随机数、密码套件列表）
2. ServerHello（选择套件、随机数）+ Certificate + ServerKeyExchange + ServerHelloDone
3. Client 验证证书，生成 PreMaster，用服务器公钥加密发送（RSA）或 ECDHE 交换
4. 双方用 Client Random + Server Random + PreMaster 算出对称密钥
5. ChangeCipherSpec + Finished（双向）
6. 应用数据用对称密钥加密传输

**TLS 1.3**：

- 移除不安全套件，仅保留 AEAD（如 AES-GCM、ChaCha20-Poly1305）
- 默认 ECDHE（前向保密）
- 握手 **1-RTT**，重连支持 **0-RTT**

### 13.8 HTTP/2 关键点

- 单 TCP 连接上多**流（Stream）**并发，二进制帧
- 解决 HTTP/1.1 队头阻塞（应用层），但**底层 TCP 仍有队头阻塞**
- **HPACK** 头部压缩，静态表 + 动态表 + 哈夫曼
- 服务端推送（实际使用率低，已被弃用）

### 13.9 HTTP/3 关键点

- 基于 **QUIC over UDP**
- 连接迁移：网络切换（WiFi → 4G）不断连，连接 ID 标识
- **彻底消除队头阻塞**：丢一个流的包不影响其他流
- 内置 TLS 1.3

### 📝 笔试题 13-1：GET 和 POST 的区别？

- 语义：GET 读，POST 写
- 幂等性：GET 幂等，POST 非幂等
- 参数位置：GET 在 URL / query，POST 在 body（也可放 URL）
- 长度：GET 受 URL 长度限制（服务端实现）；POST 几乎无限
- 缓存：GET 默认可缓存；POST 默认不可
- 编码：POST 支持 `multipart/form-data` 上传文件

> "GET 一次请求，POST 两次请求" 的说法仅针对某些实现（预检、100-continue），不通用。

### 📝 笔试题 13-2：输入 URL 到看到页面发生了什么？（详细版）

1. URL 解析：协议、主机、端口、路径、query
2. 检查强缓存，命中直接读本地
3. DNS 解析（层级缓存 → 递归查询）
4. 建立 TCP 连接（三握），HTTPS 则继续 TLS 握手
5. 发送 HTTP 请求
6. 服务器处理，返回响应（命中协商缓存返回 304）
7. 浏览器解析 HTML → 构建 DOM 和 CSSOM → 合成 Render Tree
8. 解析过程中遇到外链资源（CSS/JS/图片/字体）发起子请求（可能新连接）
9. JS 执行可能操作 DOM 触发重排/重绘
10. 首屏渲染；后续懒加载
11. 连接复用或关闭

---

## 14. 其他常用应用层协议

### 14.1 SSH（22）

- 安全远程登录
- 三阶段：版本协商 → 密钥交换（DH/ECDH）→ 用户认证（密码/公钥/Kerberos）

### 14.2 FTP（20/21）

- 双通道：控制连接（21）+ 数据连接（20 或临时端口）
- **主动模式**：服务端主动连客户端（客户端防火墙常阻挡）
- **被动模式**：客户端连服务端提供的临时端口（更友好）
- FTP 明文；生产推荐 **SFTP（走 SSH）** 或 **FTPS（FTP over TLS）**

### 14.3 SMTP / POP3 / IMAP

- **SMTP**（25/465/587）：发送邮件
- **POP3**（110/995）：下载并通常删除
- **IMAP**（143/993）：在服务器保留，多设备同步

### 14.4 WebSocket

- 握手走 HTTP `Upgrade: websocket`，成功后走独立的二进制帧协议
- **全双工、长连接**，用于聊天、实时推送、在线协作

### 14.5 gRPC

- 基于 HTTP/2 + Protocol Buffers
- 支持一元/服务端流/客户端流/双向流
- 性能好，但对 HTTP/2 代理要求高

---

## 15. NAT 与防火墙

### 15.1 NAT（网络地址转换）

解决 IPv4 地址不足。核心类型：

- **静态 NAT**：一对一
- **动态 NAT**：池内选择
- **NAPT / PAT**：多私网 IP 共用一公网 IP，靠端口区分（家用常见）

**NAT 行为类型**（影响 P2P）：

- Full Cone：同一内网 (IP:Port) 映射固定，任何外网都可发回
- Restricted Cone：只接收曾通信过的外网 IP
- Port Restricted Cone：要求外网 IP 和端口都匹配
- Symmetric：针对不同外网地址用**不同**映射（难以穿透）

**NAT 穿透**：STUN、TURN、ICE。

### 15.2 防火墙

- **包过滤防火墙**：按五元组（源/目 IP、源/目 Port、协议）过滤
- **状态检测**：跟踪 TCP 状态，只放行合法上下文
- **应用层防火墙 / WAF**：解析 HTTP，防 SQL 注入、XSS
- **主机防火墙**：iptables / nftables / Windows Firewall

### 📝 笔试题 15-1：NAT 打洞原理？

双方内网客户端通过 STUN 服务器告知各自的**公网映射地址**，再同时向对方公网地址发包，利用 NAT 映射的**已有条目**允许反向流量回传。Symmetric NAT 因映射不固定难以打洞，需 TURN 中继。

---

## 16. Socket 编程基础

### 16.1 套接字五元组

`(协议, 源 IP, 源端口, 目的 IP, 目的端口)` 唯一标识一条 TCP/UDP 流。

### 16.2 TCP 服务端骨架（C 风格）

```c
int sfd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
bind(sfd, (struct sockaddr*)&addr, sizeof(addr));
listen(sfd, backlog);                  // backlog: accept 队列长度
while (1) {
    int cfd = accept(sfd, NULL, NULL); // 阻塞等待
    // 处理 cfd: read/write
    close(cfd);
}
close(sfd);
```

### 16.3 客户端骨架

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, (struct sockaddr*)&server, sizeof(server));
send(fd, buf, len, 0);
recv(fd, buf, sizeof(buf), 0);
close(fd);
```

### 16.4 I/O 模型

| 模型 | 描述 | 代表 |
|------|------|------|
| 阻塞 I/O | read 阻塞到数据到 | `read` |
| 非阻塞 I/O | 立即返回，需轮询 | `O_NONBLOCK` + 轮询 |
| I/O 多路复用 | 单线程监听多 fd | `select / poll / epoll / kqueue` |
| 信号驱动 | fd 可读时发 SIGIO | 很少用 |
| 异步 I/O | 内核完成后通知 | Linux `io_uring`, Windows IOCP |

**epoll 优势**（相比 select/poll）：

- 事件通知而非轮询全部 fd
- 支持大量连接
- 两种模式：**LT**（水平触发，默认）、**ET**（边沿触发，性能更高但需一次读干净）

### 16.5 粘包 / 拆包

TCP 是字节流，**不保证消息边界**。应用层必须自定义消息分界：

- **定长**：每条消息固定长度
- **分隔符**：如 HTTP 的 `\r\n\r\n`
- **长度前缀**：头部带 body 长度（最常用）

### 16.6 常用 Socket 选项

| 选项 | 作用 |
|------|------|
| `SO_REUSEADDR` | 允许 TIME_WAIT 的端口被重新绑定 |
| `SO_REUSEPORT` | 多进程/线程同端口并行 accept |
| `SO_KEEPALIVE` | 启用 TCP keepalive |
| `TCP_NODELAY` | 关 Nagle，小包立即发（低延迟） |
| `TCP_CORK` | 攒满再发（批量） |
| `SO_LINGER` | close 时如何处理未发送数据 |
| `SO_RCVBUF / SO_SNDBUF` | 收/发缓冲大小 |

### 📝 笔试题 16-1：`TCP_NODELAY` 与 Nagle 冲突时怎么办？

Nagle 算法会攒小包，延迟敏感场景（如游戏、交易）需 `TCP_NODELAY=1` 关闭。同时接收端 **延迟 ACK** 也可能引入额外延迟，两者相遇会导致明显卡顿，建议**都关闭**或使用请求/响应一次性写完。

---

## 17. 抓包与排障工具

### 17.1 命令速查

| 工具 | 场景 | 示例 |
|------|------|------|
| `ping` | 连通性、RTT | `ping -c 4 baidu.com` |
| `traceroute / tracert` | 路径跟踪 | `traceroute baidu.com` |
| `nslookup / dig` | DNS 查询 | `dig @8.8.8.8 example.com` |
| `netstat / ss` | 查连接和端口 | `ss -tnp` |
| `lsof` | 查文件/socket | `lsof -iTCP:80` |
| `tcpdump` | 命令行抓包 | `tcpdump -i eth0 tcp port 443 -w a.pcap` |
| `wireshark` | 图形抓包分析 | 过滤 `tcp.port==443 && tcp.flags.syn==1` |
| `curl` | HTTP 调试 | `curl -v https://example.com` |
| `iperf3` | 带宽测试 | `iperf3 -s` / `iperf3 -c host` |
| `mtr` | ping+traceroute | `mtr baidu.com` |
| `ip / ifconfig` | 接口与路由 | `ip addr`, `ip route` |
| `nc` | 端口测试 | `nc -zv host 80` |

### 17.2 Wireshark 过滤表达式

```
ip.addr == 10.0.0.1
tcp.port == 443
tcp.flags.syn == 1 and tcp.flags.ack == 0   # SYN 包
http.request.method == "POST"
tcp.analysis.retransmission                  # 重传
tcp.stream eq 3                              # 某条流
```

### 17.3 常见问题排查思路

- **连不上**：
  1. `ping` 基础连通
  2. `telnet / nc` 端口是否开
  3. 路由和 DNS
  4. 防火墙、安全组、iptables
  5. 服务是否 `LISTEN`（`ss -lntp`）
- **慢**：
  1. `mtr` 定位丢包段
  2. `tcpdump` 抓包看重传、窗口、RTT
  3. 服务端负载、GC、DB 等
- **偶断**：
  1. keepalive 是否启用
  2. 中间设备（NAT、LB）空闲超时
  3. 应用异常退出

---

## 18. 常见 Socket 错误码深入

网络编程中最高频的三个"连接类"错误是 **ECONNRESET**、**ETIMEDOUT**、**ESOCKETTIMEDOUT**。理解它们的触发条件、区别以及可能的根因，是定位生产问题的关键能力。

### 18.1 错误码速查

| 错误码 | errno | 触发层 | 语义 | 典型表现 |
|--------|-------|--------|------|----------|
| `ECONNREFUSED` | 111 | 内核（TCP） | 目的端口无人监听（收到 RST） | `connect` 立即失败 |
| `ECONNRESET` | 104 | 内核（TCP） | 对端发来 RST，连接被强制复位 | `read/write` 中途失败 |
| `ECONNABORTED` | 103 | 内核（TCP） | 本端在三握完成前中止，或 accept 前被对端 RST | `accept` 返回错误 |
| `ETIMEDOUT` | 110 | 内核（TCP） | 内核层面连接或重传超时（如 SYN 未得到响应、数据重传达到上限） | `connect` 或长连接空闲后失败 |
| `ESOCKETTIMEDOUT` | — | **应用层/库** | 应用设置的 socket 读写超时（`SO_RCVTIMEO` / 库级 timeout）触发 | `read/recv/query` 超过设定时间 |
| `EPIPE` | 32 | 内核（TCP） | 向已关闭（收到 RST 或 FIN 后）的连接写数据 | `write` 失败 + SIGPIPE |
| `EHOSTUNREACH` | 113 | 内核（IP/ICMP） | 目的主机不可达 | 路由不通 |
| `ENETUNREACH` | 101 | 内核（IP） | 目的网络不可达 | 无路由 |

> 说明：
> - `ECONNRESET`、`ETIMEDOUT` 是 POSIX `errno`，Linux/macOS/Windows（对应 `WSAECONNRESET`、`WSAETIMEDOUT`）均有对应。
> - **`ESOCKETTIMEDOUT` 不是 POSIX 标准 errno**，而是 **Node.js / 部分 HTTP 客户端库**（如 `request`、`axios`、`got`、Python `requests` 等）在应用层设置的**读写超时**触发的错误名。Python `requests` 对应 `ReadTimeout`；Java 对应 `SocketTimeoutException`。

### 18.2 ECONNRESET 深入

#### 18.2.1 本质

对端发送了 **RST 报文**，内核将本端连接立即置为关闭态，再 `read/write` 返回 `ECONNRESET`（或读到 0 后再写触发 `EPIPE`）。

RST 并非正常关闭（FIN），跳过四次挥手，**不保证已发送数据被处理**。

#### 18.2.2 常见触发场景

1. **对端进程崩溃 / 异常退出**
   - 操作系统在进程退出时关闭 socket；若内核缓冲还有未读数据或处于半开状态，会发 RST

2. **对端主动 `close()` 时缓冲区仍有未读数据**
   - 标准关闭是发 FIN；若 `close` 时接收队列仍有数据，Linux 会发 **RST** 表示异常关闭
   - 规避：服务端应先 `shutdown(WR)` 再读干净 FIN，再 `close`

3. **`SO_LINGER` 设为 `{l_onoff=1, l_linger=0}`**
   - `close()` 时立即发 RST 而不是 FIN，丢弃发送缓冲
   - 某些高并发场景为释放资源会这样配置，代价是丢数据

4. **写入已被对端关闭的连接**
   - 对端已发 FIN，本端继续写数据 → 对端回 RST → 本端后续 `write` 触发 `EPIPE` 或 `ECONNRESET`

5. **中间设备（LB / NAT / 防火墙）强制断链**
   - 空闲超时（如 AWS ELB 默认 60s、NAT 表老化）踢掉连接时通常发 RST
   - 有状态防火墙认为会话非法会发 RST

6. **连接到"半打开"状态的连接**
   - 一端异常断电 / 网线断开，另一端感知不到；恢复后收到"陌生"报文会回 RST

7. **端口被回收**
   - 客户端复用 `(ip:port)` 对应到服务端早已释放的连接，四元组冲突

8. **应用主动发 RST**
   - 如 Nginx `reset_timedout_connection on;` 清理异常连接
   - Node.js `socket.destroy()` 也可能触发 RST

#### 18.2.3 排查思路

- `tcpdump -i any host X.X.X.X and port Y -w out.pcap`，Wireshark 过滤 `tcp.flags.reset == 1`，看 RST **是谁发的**
- 对比双方日志时间线，定位是**主动 reset** 还是**被动异常**
- 检查中间设备（LB/NAT/防火墙）空闲超时设置
- 检查应用是否设置 `SO_LINGER=0`、是否在 `close` 前读干净
- 检查长连接是否启用了应用层心跳

### 18.3 ETIMEDOUT 深入

#### 18.3.1 本质

**内核 TCP 栈**触发的超时。常见两类：

- **connect 超时**：发 SYN 后未在内核允许的重试次数内收到 SYN+ACK
  - Linux `tcp_syn_retries` 默认 6，约 **127 秒**
- **数据重传超时**：`write` 的数据超过 `tcp_retries2`（默认 15）次重传仍未得到 ACK，内核放弃连接
  - 通常对应几分钟到十几分钟

#### 18.3.2 常见触发场景

1. **目标 IP 可路由但目的主机"黑洞"**
   - 防火墙 DROP（不是 REJECT）SYN，不回任何包；客户端一直重试直到超时
   - 典型：公司安全组未放行端口、IP 不存在但路由器默认转发

2. **中间链路拥塞 / 丢包严重**
   - 往返包大量丢失，TCP 重传次数用完

3. **对端系统 hung 住**
   - OOM、IO hang、内核卡住，TCP 栈不响应 SYN 或 ACK

4. **长连接在 NAT/LB 下静默失效**
   - 双方都认为连接存在但中间已断，`keepalive` 未启用或间隔过长
   - 业务包发出后无 ACK，重传用尽报 `ETIMEDOUT`

5. **物理链路故障**
   - 网线松动、网卡丢包、交换机故障

6. **路由不对称**
   - 包能发出去但回程路由不通，TCP 认为丢包

#### 18.3.3 与 ECONNREFUSED 的区别

- `ECONNREFUSED`：有包回来（RST），说明**主机可达但端口没人监听**
- `ETIMEDOUT`：**没有任何回包**，可能是防火墙 DROP、主机宕、网络不通

### 18.4 ESOCKETTIMEDOUT（应用层读写超时）

#### 18.4.1 本质

**应用/库层定义**的超时：TCP 连接本身还活着，但在约定时间内没收到数据，库主动取消并抛错。不同语言/库有不同命名：

| 语言 / 库 | 对应错误 |
|-----------|----------|
| Node.js（默认 socket 超时） | `ESOCKETTIMEDOUT` / `ETIMEDOUT` |
| Node.js `http.request` | `ECONNRESET`（`timeout` 事件后 destroy） |
| Python `requests` | `requests.exceptions.ReadTimeout` |
| Python `socket.settimeout` | `socket.timeout`（继承自 `OSError`） |
| Java | `java.net.SocketTimeoutException` |
| Go | `net.Error` 且 `err.Timeout() == true` |

底层机制多为 `SO_RCVTIMEO` / `SO_SNDTIMEO`，或用 `setTimeout` + `destroy`。

#### 18.4.2 常见触发场景

1. **服务端处理慢**
   - DB 慢查询、下游 RPC 慢、GC 停顿、线程池满
2. **数据量大而 timeout 过小**
   - 大响应体、文件上传下载，未按大小/吞吐设置超时
3. **长轮询 / SSE 未配置合适 timeout**
4. **网络抖动但未严重到内核 ETIMEDOUT**
5. **应用忘记调用 `end()` / 对端 hang**

#### 18.4.3 调优建议

- **区分 connect timeout 与 read/write timeout**：连接建立应快（秒级），读取应按业务调整
- **分级超时**：客户端 timeout < 网关 timeout < 服务端 timeout，避免级联重试放大
- **幂等重试**：超时触发后仅对**幂等请求**重试，非幂等请求谨慎
- **断路器**：连续超时应熔断，避免打爆下游

### 18.5 ECONNRESET vs ETIMEDOUT vs ESOCKETTIMEDOUT 对比

| 维度 | ECONNRESET | ETIMEDOUT | ESOCKETTIMEDOUT |
|------|------------|-----------|-----------------|
| 触发层 | 内核（收到 RST） | 内核（TCP 超时） | 应用/库（用户设定 timeout） |
| 连接状态 | 已被对端/中间设备强关 | 发包无响应或重传耗尽 | TCP 连接通常还活着 |
| 发包方向 | 对端→本端有 RST | 双方无包或丢包 | 本端发了对端无响应 |
| 恢复方式 | 重新建连 | 重新建连 | 可继续用原连接或新建 |
| 粗略时间 | 立即 | 秒~数分钟 | 秒级（用户配置） |

### 18.6 典型"连接被复位"案例剖析

**现象**：Node.js HTTP 客户端偶发 `ECONNRESET`，并发越高越多。

**可能原因层层排查**：

1. **Keep-Alive 空闲超时竞态**
   - 客户端复用连接的瞬间，服务端刚好达到空闲超时发 FIN/RST
   - 解决：客户端空闲时间略短于服务端，或收到 `retry-able` 错误自动重试

2. **Nginx worker_connections / keepalive_requests 达到上限**
   - 连接被主动关闭
   - 解决：调大 `keepalive_requests`，或减小连接复用次数

3. **LB 空闲超时低于应用 keep-alive**
   - AWS ELB 默认 60s，若应用 keep-alive 设 120s 则会踩雷
   - 解决：应用侧 keep-alive < LB 空闲超时

4. **客户端连接池有效性检测缺失**
   - 复用连接时未校验 `socket.destroyed`
   - 解决：开启 pre-write 校验或 pool 的 `validateOnCheckout`

### 18.7 写出健壮网络代码的 Checklist

- ✅ 同时设置 **connect timeout** 与 **read/write timeout**
- ✅ 使用**连接池**并配置 **健康检查**、**最大空闲时间**
- ✅ **应用层心跳**，不要完全依赖 TCP keepalive
- ✅ 捕获 `ECONNRESET/ETIMEDOUT/EPIPE` 并按**幂等性**决定重试策略
- ✅ 对 `SIGPIPE` 做 `ignore` 或用 `MSG_NOSIGNAL` 发送（Linux）
- ✅ 大响应分块读、带进度超时，而非一次性 read
- ✅ 关键链路启用 **全链路超时预算**（deadline/context）
- ✅ 生产环境预埋 **tcpdump / eBPF** 采样能力，便于偶发问题取证

### 📝 笔试题 18-1：客户端连接 3306 端口报 `ECONNREFUSED` 和 `ETIMEDOUT` 分别可能是什么原因？

- `ECONNREFUSED`：**主机可达，3306 端口无监听**——MySQL 未启动，或只 `bind 127.0.0.1` 导致外部连接被拒
- `ETIMEDOUT`：**没收到任何回包**——防火墙/安全组 DROP、IP 或主机不可达、中间网络故障

### 📝 笔试题 18-2：HTTP 客户端报 `ESOCKETTIMEDOUT`，服务端日志却显示请求已成功处理，为什么？

应用层读超时触发时，TCP 数据可能**正在传输**：服务端完成处理并开始写回响应，但客户端设置的读超时更短或网络慢，客户端先抛错并 `destroy` 了 socket。此时**请求不幂等**的话不能无脑重试，否则会重复处理。

### 📝 笔试题 18-3：`SO_LINGER` 设为 `{1, 0}` 有什么效果和风险？

- 效果：`close()` 不走 FIN 挥手，直接发 **RST**，并丢弃发送缓冲
- 用途：快速释放端口，避免大量 `TIME_WAIT`
- 风险：对端读端可能读不到残留数据 / 收到 `ECONNRESET`；对有事务保证的协议（HTTP 管线化、DB 连接）可能导致数据丢失

### 📝 笔试题 18-4：长连接空闲几分钟后第一次发包就报 `ETIMEDOUT`，为什么？

中间 NAT / LB / 防火墙的**会话表老化**把连接条目移除了。两端的 TCP 状态仍是 `ESTABLISHED`，但中间设备丢弃后续包且不发 RST，本端重传用尽报 `ETIMEDOUT`。解决：**缩短应用层心跳间隔**至小于中间设备空闲超时，或启用 TCP keepalive 并调小 `tcp_keepalive_time`。

### 📝 笔试题 18-5：向一个收到了 FIN 的 socket 继续写会怎样？

- 第一次 `write`：可能成功（TCP 允许单向写），但对端会回 **RST**
- 之后的 `write`：返回 `EPIPE`，并默认触发 `SIGPIPE` 导致进程退出
- 规避：忽略 SIGPIPE 或用 `send(MSG_NOSIGNAL)`；收到 EOF 后立即结束写端

---

## 19. 综合笔试练习

### 18-1 选择题

**Q1** TCP 建立连接需要几次握手？
A. 1  B. 2  C. 3  D. 4

<details><summary>答案</summary>C。</details>

**Q2** 下列哪个协议工作在应用层？
A. TCP  B. IP  C. ARP  D. DNS

<details><summary>答案</summary>D。</details>

**Q3** HTTPS 默认端口是？
A. 80  B. 443  C. 8080  D. 22

<details><summary>答案</summary>B。</details>

**Q4** 当 TCP 报文段超时未确认时，正确的做法是？
A. 保持 cwnd 不变
B. cwnd 减半
C. cwnd = 1，重新慢启动
D. cwnd 翻倍

<details><summary>答案</summary>C。</details>

**Q5** 下列哪种状态不属于主动关闭方？
A. FIN_WAIT_1  B. FIN_WAIT_2  C. TIME_WAIT  D. CLOSE_WAIT

<details><summary>答案</summary>D。CLOSE_WAIT 属于被动关闭方。</details>

**Q6** 下列关于 UDP 的说法**错误**的是？
A. 不保证到达
B. 不保证顺序
C. 支持多播和广播
D. 拥塞发生时会降速

<details><summary>答案</summary>D。UDP 无拥塞控制。</details>

**Q7** IP 地址 `192.168.10.128/27` 的可用主机数？
A. 30  B. 32  C. 62  D. 14

<details><summary>答案</summary>A。`2^5 - 2 = 30`。</details>

**Q8** 下列哪个字段每经过一个路由器会变化？
A. 源 IP  B. 目的 IP  C. TTL  D. 协议号

<details><summary>答案</summary>C。</details>

### 18-2 判断题

1. TCP 是面向字节流的，UDP 是面向报文的。 ✅
2. MAC 地址全球唯一，IP 地址可以动态分配。 ✅
3. TIME_WAIT 出现在被动关闭方。 ❌（主动关闭方）
4. ARP 用于 IP → MAC 的解析。 ✅
5. DNS 只能用 UDP。 ❌（响应过大或区传输用 TCP）
6. HTTPS 能防止中间人攻击的前提是客户端正确校验证书。 ✅
7. 三次握手中第三次的 ACK 可以携带数据。 ✅
8. HTTP/2 解决了所有队头阻塞。 ❌（TCP 层面仍存在，HTTP/3 才彻底解决）

### 18-3 简答题

**Q1** 解释"TCP 粘包"并给出解决方案。

TCP 是字节流协议，发送方连续 `send` 的多个"消息"可能被合并或拆分发送到接收方。解决思路：应用层定义消息边界，常用**长度前缀**、**分隔符**、**定长消息**。

**Q2** HTTPS 为什么使用非对称 + 对称混合加密？

- 非对称加密：安全地交换密钥，但计算开销大
- 对称加密：速度快，适合大量数据
- 握手用非对称协商对称密钥，之后用对称密钥加密业务数据，兼顾安全与性能

**Q3** 为什么 UDP 比 TCP 快？举一个"UDP 更合适"的场景。

UDP 无连接、无握手、无重传、无拥塞控制，开销小。**实时语音/视频、DNS 查询、游戏**等容忍丢包、追求低延迟的场景更适合 UDP。

**Q4** TIME_WAIT 过多的影响与优化？

占用端口资源，新连接可能因端口不足而失败。优化：长连接复用、`SO_REUSEADDR`、服务端减少主动关闭（让客户端先关）、Linux 调 `tcp_tw_reuse`。

**Q5** DNS 查询中 CNAME 和 A 记录的关系？

CNAME 是别名，指向另一个域名；最终必须解析到 A/AAAA 才能拿到 IP。查询过程中递归解析器会继续对 CNAME 目标递归查询，直到得到 A 记录。

**Q6** 描述 TCP 拥塞控制中 cwnd 的一个完整周期。

```
| cwnd
|         ／|＼    ／|＼          
|        ／ | ＼  ／ | ＼         ← 丢包触发
|       ／  |  ＼／  |  ＼        
|      ／   慢   拥避   丢 → 减半
|     ／    启   (线增)            
|____／______________________  t
```

从慢启动指数增长，到达 ssthresh 后进入线性的拥塞避免，遇 3 重复 ACK 则 cwnd 减半进入快速恢复；若超时则 cwnd=1 回到慢启动。

### 18-4 编程/设计题

**Q1** 设计一个基于 TCP 的简易 RPC 协议。

- 帧格式：`Magic(4B) | Version(1B) | Type(1B) | RequestID(4B) | Length(4B) | Payload`
- Payload 用 JSON 或 Protobuf 序列化 `{method, params}`
- 接收端用长度前缀解包，按 RequestID 路由回调
- 心跳：定期发 `Type=Ping`，超时无响应即重连

**Q2** 写一段 Python 代码发起 HTTPS GET 并打印响应头。

```python
import socket, ssl

ctx = ssl.create_default_context()
with socket.create_connection(("example.com", 443), timeout=5) as sock:
    with ctx.wrap_socket(sock, server_hostname="example.com") as ssock:
        ssock.sendall(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
        data = b""
        while chunk := ssock.recv(4096):
            data += chunk
print(data.split(b"\r\n\r\n", 1)[0].decode())
```

**Q3** 用 `epoll` 画出一个回显服务的事件循环伪代码。

```
epfd = epoll_create();
epoll_ctl(ADD, listen_fd, EPOLLIN);
while (true) {
    n = epoll_wait(events, timeout);
    for e in events:
        if e.fd == listen_fd:
            client = accept();
            set_nonblock(client);
            epoll_ctl(ADD, client, EPOLLIN | EPOLLET);
        elif e.events & EPOLLIN:
            while (read(e.fd, buf) > 0):
                write(e.fd, buf);    // 回显
            if EAGAIN: continue;
            if 0 or error: close(e.fd);
}
```

**Q4** 某服务在高并发下出现大量 `CLOSE_WAIT`，排查思路？

1. `ss -tanp | grep CLOSE_WAIT` 确认进程
2. 检查代码：连接处理完是否 `close()`；异常路径是否遗漏
3. 查是否有连接池泄漏：每次新建未归还
4. 观察对端是否主动断开频繁（如客户端短超时）
5. 临时降级：减小空闲连接 timeout 或重启

---

## 📚 复习建议

1. **分层记忆**：每层的功能 / 代表协议 / 典型设备各画一张表
2. **画时序图**：三握四挥、TLS 握手、DNS 查询都要会徒手画
3. **动手抓包**：Wireshark 抓一次 `curl https://...`，逐帧对照理论
4. **看 RFC**：重点 RFC 793（TCP）、RFC 791（IP）、RFC 7540（HTTP/2）、RFC 9000（QUIC）
5. **实战排障**：熟悉 `ss`、`tcpdump`、`dig` 的常用参数，面试时很加分

> 祝笔试顺利！
