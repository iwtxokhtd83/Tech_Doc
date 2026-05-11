# 负载均衡与 Nginx 讲义

> 本讲义分两条主线：**负载均衡**（原理、算法、协议层次、高可用架构）与 **Nginx**（核心概念、配置语法、常用模块、HTTP/反代/TLS/WebSocket/缓存/限流/日志/排障）。每章配"知识点 + 笔试题"。
>
> 约定：Nginx 示例基于 **1.24+ / OSS 版**；系统以 Linux 为主；使用 `nginx -t` 可校验配置。

## 目录

1. [为什么需要负载均衡](#1-为什么需要负载均衡)
2. [负载均衡的协议层次](#2-负载均衡的协议层次)
3. [常见负载均衡算法](#3-常见负载均衡算法)
4. [会话保持与健康检查](#4-会话保持与健康检查)
5. [高可用：主备、Anycast、DNS 轮询](#5-高可用主备anycastdns-轮询)
6. [负载均衡产品选型](#6-负载均衡产品选型)
7. [Nginx 架构与进程模型](#7-nginx-架构与进程模型)
8. [配置文件结构与基础语法](#8-配置文件结构与基础语法)
9. [server 与 location 匹配规则](#9-server-与-location-匹配规则)
10. [反向代理与上游（upstream）](#10-反向代理与上游upstream)
11. [静态资源、gzip 与缓存](#11-静态资源gzip-与缓存)
12. [HTTPS 与 HTTP/2 / HTTP/3 配置](#12-https-与-http2--http3-配置)
13. [WebSocket 配置](#13-websocket-配置)
14. [限流、熔断与防护](#14-限流熔断与防护)
15. [日志、监控与调试](#15-日志监控与调试)
16. [Nginx 作为 TCP/UDP 四层负载](#16-nginx-作为-tcpudp-四层负载)
17. [生产部署实战片段](#17-生产部署实战片段)
18. [常见问题与性能调优](#18-常见问题与性能调优)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 为什么需要负载均衡

### 1.1 核心目标

- **横向扩展**：单机算力/IO 有限，多实例分摊流量
- **高可用**：实例故障自动剔除，用户无感
- **弹性伸缩**：按流量自动扩缩容
- **平滑升级**：灰度、蓝绿、滚动发布
- **安全边界**：集中做 TLS 卸载、WAF、限流、日志审计

### 1.2 请求路径常见层次

```
Client
  │
  ▼
DNS (地理/权重)                 ← 第 0 层
  │
  ▼
Anycast / IP LB (L4)           ← 第 1 层（LVS、云 NLB、硬件）
  │
  ▼
Reverse Proxy (L7)             ← 第 2 层（Nginx、Envoy、ALB）
  │
  ▼
Service Mesh / Sidecar         ← 第 3 层（可选：Istio、Linkerd）
  │
  ▼
Application Instances
```

大型系统通常 **L4 + L7 协同**：L4 承担极高并发转发，L7 做路由与应用层处理。

### 📝 笔试题 1-1：负载均衡解决了单机的哪三类问题？

- 容量不足（扩展性）
- 单点故障（可用性）
- 运维停机（平滑发布）

---

## 2. 负载均衡的协议层次

### 2.1 按 OSI 层次分类

| 层次 | 识别信息 | 典型实现 | 适用场景 |
|------|----------|----------|----------|
| **L2** | MAC 地址 | LVS-DR、交换机聚合 | 同网段透明 |
| **L3** | IP 地址 | ECMP、Anycast | 大规模入口 |
| **L4** | IP + Port（TCP/UDP） | LVS、HAProxy、Nginx stream、云 NLB | 高并发、低延迟、任意协议 |
| **L7** | 应用协议（HTTP、gRPC、DNS...） | Nginx、Envoy、HAProxy、ALB | 路由、改写、鉴权、缓存 |

### 2.2 L4 vs L7 对比

| 维度 | L4 | L7 |
|------|----|----|
| 能看到 | 五元组 | URL/Header/Body（需终结 TLS） |
| 性能 | 极高（每秒百万级连接） | 较低（解析开销） |
| 路由能力 | 按端口 | 按路径、Host、头部、Cookie |
| TLS 终结 | 不处理 | 常在此层终结 |
| 协议支持 | 任意 TCP/UDP | HTTP 族为主 |

### 2.3 直接路由（DR）与隧道

- **NAT 模式**：LB 作为网关，请求和返回都经 LB；实现简单但 LB 是瓶颈
- **DR（Direct Routing）模式**：LB 只改 MAC，响应从 RS 直接回客户端，适合大流量
- **Tunnel（IP-in-IP）**：跨网段 DR 的变种

### 📝 笔试题 2-1：L4 和 L7 选哪个？

看需求：

- 要**基于 URL/Host 路由**或**TLS 终结**或**改写响应** → L7（Nginx/Envoy）
- 要**极致吞吐、支持任意 TCP/UDP**（如 MySQL、Redis、游戏 UDP）→ L4（LVS/NLB）
- 典型生产：**L4 入口 + L7 做应用分发**

---

## 3. 常见负载均衡算法

### 3.1 静态算法

| 算法 | 说明 | 适用 |
|------|------|------|
| **轮询 (Round Robin)** | 依次分配 | 后端同构 |
| **加权轮询 (Weighted RR)** | 按权重分配 | 异构机器 |
| **IP Hash** | 源 IP 取哈希 | 简单会话粘性 |
| **URL Hash / Consistent Hash** | 按 key（URL、userId）哈希 | 缓存命中率高，扩容影响小 |
| **Random** / **Weighted Random** | 随机/加权随机 | 简单，分布均匀 |

### 3.2 动态算法

| 算法 | 说明 |
|------|------|
| **最少连接 (Least Conn)** | 选当前连接最少的实例 |
| **最少响应时间 (Least Time)** | 综合延迟和连接数（Nginx Plus） |
| **EWMA / P2C** | 指数加权平均 + "Power of Two Choices"，Envoy/Linkerd 默认 |
| **资源感知** | CPU/QPS/内存反馈，如 Sidecar 网格 |

### 3.3 一致性哈希

```
  hash(key) 落到环上某点，顺时针找到第一台实例
```

- **扩容只影响局部**：新增实例只接管环上一段
- **虚拟节点**缓解分布不均
- 典型应用：缓存分片、粘性路由

### 📝 笔试题 3-1：加密的 HTTPS 流量能否做 IP Hash 会话保持？

可以。IP Hash 只看源 IP，与是否加密无关；但如果上游是**共享公网 NAT**（大量客户端出口同 IP），会引起热点。此时可改用基于 Cookie 的会话保持（需 L7）。

---

## 4. 会话保持与健康检查

### 4.1 为什么需要会话保持（Session Stickiness）

- 有状态服务（登录态存本地内存）
- WebSocket / SSE 等长连接必须绑定到同一后端
- 上传续传、多步事务

**更好的方案**：**状态外置**（Redis、DB、JWT）。粘性只是妥协，不要长期依赖。

### 4.2 会话保持方式

- **IP Hash**：简单，但 NAT/移动网络 IP 漂移影响大
- **Cookie 插入**：L7 在响应里写 `SERVERID=srv-a`，后续按此路由
- **Cookie 学习**：LB 观察后端 `Set-Cookie`，自动学习映射
- **URL 参数**：把路由信息塞进 URL（不常用）

### 4.3 健康检查

| 类型 | 说明 |
|------|------|
| **被动** | 看转发结果（连接失败/超时/5xx）判断 |
| **主动** | LB 定时探测（TCP 握手 / HTTP GET `/healthz`）|
| **复合** | 被动 + 主动并重 |

健康探测关键参数：

- **间隔（interval）**：多久探一次
- **超时（timeout）**：多长时间没响应判失败
- **阈值（rise/fall）**：连续 N 次健康/N 次失败才切换状态，防抖
- **探测路径**：应用层应提供**轻量**、**不含外部依赖**的 `/healthz`

### 📝 笔试题 4-1：`/healthz` 里应该检查数据库连接吗？

**谨慎**。深度健康检查（含 DB/Redis）会让**下游抖动**导致**所有实例同时被摘**，形成级联雪崩。推荐：

- `/healthz`（liveness）：仅判断进程本身
- `/readyz`（readiness）：判断是否可承接流量，可含轻量依赖
- 深度依赖检查仅用于报警监控，不参与 LB 判定

---

## 5. 高可用：主备、Anycast、DNS 轮询

### 5.1 LB 自身的高可用

LB 自己不能是单点。

- **VRRP / Keepalived**：主备通过虚拟 IP (VIP) 漂移
- **Active-Active**：多个 LB 同时工作，上游靠 ECMP/Anycast 分发
- **云厂商负载均衡**：天生多可用区、自动故障切换

### 5.2 Anycast

多个地理/数据中心宣告**同一个 IP**，BGP 选路自动把用户路由到最近节点。典型：DNS 根服务器、Cloudflare、大型 CDN 入口。

### 5.3 DNS 层面负载均衡

- **多 A 记录轮询**：客户端解析到不同 IP
- **GSLB**（Global Server LB）：基于地理、健康度返回最近最健康的 IP
- **权重 DNS**：Route 53、阿里云解析支持
- **缺陷**：DNS 缓存 TTL 不可控，故障切换慢

### 📝 笔试题 5-1：LB 要做到高可用，通常怎么部署？

- 云上：直接用厂商 LB（多 AZ 冗余）
- 自建 L4：LVS/HAProxy 双机 + Keepalived（VIP 主备）
- 多机房：DNS GSLB / Anycast + BGP
- 监控：LB 本身指标（活跃连接、后端健康、丢包）

---

## 6. 负载均衡产品选型

| 产品 | 层次 | 特点 |
|------|------|------|
| **LVS (IPVS)** | L4 | Linux 内核态，极高性能，靠 keepalived 做 HA |
| **HAProxy** | L4/L7 | 配置简单、稳定，TCP/HTTP 通用 |
| **Nginx (OSS / Plus)** | L4/L7 | 反代 + Web Server 一体，模块丰富 |
| **Envoy** | L7 | 云原生，xDS 动态配置，Istio/Linkerd 核心 |
| **Traefik** | L7 | 对 Docker/K8s 友好，自动服务发现 |
| **Apache APISIX / Kong** | L7 | 网关，插件生态强（鉴权、限流、可观测） |
| **F5 Big-IP / A10** | 硬件 | 传统企业、金融 |
| **云厂商**：AWS ALB/NLB、阿里 SLB、GCP LB | 托管 | 按需付费，免运维 |

**选型直觉**：

- 面向互联网入口：Nginx / ALB
- 微服务内网东西向：Envoy + Service Mesh
- 极致四层：LVS / NLB
- API 网关需求：APISIX / Kong / Spring Cloud Gateway

---

## 7. Nginx 架构与进程模型

### 7.1 Master-Worker 模型

```
┌─────────┐        manage        ┌──────────┐
│ master  │ ───────────────────▶ │ worker 1 │
│         │ ───────────────────▶ │ worker 2 │
│         │ ───────────────────▶ │ worker N │
└─────────┘                      └──────────┘
      │
      │ bind: 80/443
      ▼
   listen sockets (shared)
```

- **master**：读取配置、管理 worker、热重载、信号处理（**不处理请求**）
- **worker**：单进程多连接，基于 **事件驱动 + 非阻塞 IO**（epoll/kqueue）处理请求
- 一个 worker 可同时处理数千到数万连接

### 7.2 为什么快

- 事件驱动 + 异步 IO（epoll）
- 单线程内**无锁**处理，避免上下文切换
- 共享监听 socket，多 worker 负载均衡
- 模块化：只加载需要的
- 优化的内存池、零拷贝（`sendfile`）、静态文件直传

### 7.3 信号管理

```bash
nginx -t                 # 语法检查
nginx -s reload          # 热重载配置（master 启新 worker，老 worker 优雅退出）
nginx -s reopen          # 重新打开日志文件（配合 logrotate）
nginx -s stop            # 立即停止
nginx -s quit            # 优雅停止（处理完现有请求）
```

### 📝 笔试题 7-1：`nginx -s reload` 做了什么？

- master 重新读取配置并校验
- 启动**新 worker**使用新配置
- 老 worker 不再接新连接，**处理完存量后优雅退出**
- 过程中对外**无服务中断**

---

## 8. 配置文件结构与基础语法

### 8.1 文件结构

```
main                           ← 全局
  events                       ← 连接处理
  http                         ← HTTP 模块
    upstream
    server
      location
  stream                       ← TCP/UDP 四层
  mail                         ← 邮件代理（少用）
```

### 8.2 基本配置（骨架）

```nginx
# main 块
user  nginx;                              # 运行用户
worker_processes auto;                    # 通常等于 CPU 核数
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections 10240;             # 每 worker 最大连接数
    use epoll;                            # Linux 推荐
    multi_accept on;                      # 一次尽量多 accept
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    keepalive_timeout  65;
    types_hash_max_size 2048;
    server_tokens      off;               # 隐藏版本号

    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    access_log /var/log/nginx/access.log main;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    include /etc/nginx/conf.d/*.conf;      # 拆分子配置
}
```

### 8.3 指令与上下文

- 每条指令以 `;` 结尾
- 块用 `{}` 包裹，可嵌套
- 大多数指令**有明确作用域**（main / events / http / server / location），写错位置会报错
- 变量以 `$` 开头，如 `$host`、`$remote_addr`

### 8.4 常用内置变量

| 变量 | 含义 |
|------|------|
| `$host` | 请求 Host 头（去端口） |
| `$http_host` | 原始 Host 头 |
| `$remote_addr` | 客户端 IP（直连者） |
| `$http_x_forwarded_for` | `X-Forwarded-For` 头 |
| `$request_uri` | 完整 URI（含 query） |
| `$uri` | 规范化的 URI（不含 query） |
| `$args` / `$arg_name` | query string / 某参数 |
| `$scheme` | `http` / `https` |
| `$request_method` | GET/POST/... |
| `$status` | 响应状态 |
| `$body_bytes_sent` | 响应体大小 |
| `$request_time` | 请求全程耗时 |
| `$upstream_*` | 上游相关指标 |
| `$server_name` / `$server_port` | 匹配到的 server 与端口 |

### 📝 笔试题 8-1：`$host` 和 `$http_host` 区别？

- `$http_host`：完全原样的 Host 请求头（如 `example.com:8080`）
- `$host`：按优先级取：请求行里的 host → Host 头 → 匹配的 `server_name`，**总为小写且不含端口**

反代给后端更建议 `$host`，避免端口混乱。

---

## 9. server 与 location 匹配规则

### 9.1 监听与虚拟主机

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    # ...
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    # ...
}
```

匹配顺序：

1. **精确** `example.com`
2. **前置通配** `*.example.com`
3. **后置通配** `example.*`
4. **正则**（按出现顺序）
5. 都未匹配 → `default_server`（同 listen 的第一个或显式标记）

```nginx
server {
    listen 80 default_server;
    return 444;          # 直接关闭连接（防止被扫描）
}
```

### 9.2 location 匹配

```nginx
location [ = | ~ | ~* | ^~ ] /uri/ { ... }
```

**优先级**（从高到低）：

1. `=` 精确匹配：`location = /exact { ... }`
2. `^~` 前缀匹配（命中后不再尝试正则）：`location ^~ /static/ { ... }`
3. `~` 区分大小写的正则：`location ~ \.php$ { ... }`
4. `~*` 不区分大小写的正则：`location ~* \.(jpg|png)$ { ... }`
5. 普通前缀匹配（选**最长前缀**）：`location /api/ { ... }`

算法概述：

- 先找精确 `=` → 命中即结束
- 找最长前缀匹配（记住）；若该前缀带 `^~` → 直接用，跳过正则
- 按出现顺序匹配正则；第一个命中的胜
- 若无正则命中，用之前记录的最长前缀

### 9.3 常见示例

```nginx
# 静态资源（长命中率）
location ^~ /static/ {
    root /var/www;
    expires 30d;
    add_header Cache-Control "public, immutable";
}

# API
location /api/ {
    proxy_pass http://backend/;     # 结尾斜杠含义见下章
}

# 图片
location ~* \.(jpg|jpeg|png|gif|webp)$ {
    expires 7d;
    access_log off;
}

# 精确健康检查端点
location = /healthz {
    access_log off;
    return 200 "ok";
}
```

### 📝 笔试题 9-1：匹配顺序为什么把 `=` 设为最高优先级？

- 精确匹配语义最强、范围最小，优先给 **确定路径**（如 `/favicon.ico`、`/healthz`）命中
- 避免被正则"顺带"抢走，带来不必要的计算
- 对高频固定 URL 明显提升性能

---

## 10. 反向代理与上游（upstream）

### 10.1 最小反代

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

### 10.2 `proxy_pass` 末尾斜杠的坑

```nginx
location /api/ {
    proxy_pass http://backend;         # 请求 /api/foo → http://backend/api/foo
}

location /api/ {
    proxy_pass http://backend/;        # 请求 /api/foo → http://backend/foo
}
```

**规则**：`proxy_pass` URL **带路径（含根 `/`）** 时，Nginx 会用 `proxy_pass` 的 URL **替换** `location` 匹配到的前缀；不带路径时，原 URI 整体追加。

### 10.3 upstream 块

```nginx
upstream backend {
    # 算法
    least_conn;                        # 最少连接；默认是轮询
    # ip_hash;                         # 会话粘性
    # hash $request_uri consistent;    # 一致性哈希

    server 10.0.0.1:8080 weight=3 max_fails=3 fail_timeout=10s;
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;       # 备用（主全挂后启用）
    server 10.0.0.4:8080 down;         # 暂时停用

    keepalive 32;                      # 到上游的空闲长连接池
    keepalive_timeout 60s;
    keepalive_requests 1000;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";       # 启用 keepalive 的必要条件
    }
}
```

### 10.4 被动健康检查

OSS Nginx 依赖 **被动探测**：

- `max_fails=N`：在 `fail_timeout` 窗口内错误次数达到 N 即标记不可用
- `fail_timeout=T`：既是错误计数窗口，也是不可用持续时间

想要**主动健康检查**（路径 / 响应体匹配）需要 **Nginx Plus** 或 **第三方模块**（如 `nginx_upstream_check_module` / OpenResty 实现 / Tengine）。

### 10.5 故障转移与重试

```nginx
proxy_next_upstream error timeout http_502 http_503 http_504;
proxy_next_upstream_tries 3;
proxy_next_upstream_timeout 10s;
```

### 10.6 缓冲区相关

```nginx
proxy_buffering on;                 # 默认开启；大响应体先缓冲再给客户端
proxy_buffers 8 16k;                # 8 个 16KB 缓冲
proxy_buffer_size 16k;              # 响应头专用
proxy_busy_buffers_size 32k;
client_max_body_size 50m;           # 允许的请求体大小（上传限制）
client_body_buffer_size 128k;
```

**流式接口**（SSE、实时日志）要关闭缓冲：

```nginx
location /stream {
    proxy_pass http://backend;
    proxy_buffering off;
    proxy_cache off;
    proxy_read_timeout 3600s;
}
```

### 📝 笔试题 10-1：为什么启用上游 keepalive 需要 `proxy_http_version 1.1` 和 `Connection ""`？

HTTP/1.0 不支持 keep-alive 默认行为；HTTP/1.1 默认保持连接。客户端→Nginx 的 `Connection: close` 头若原样转发给上游，上游也会关闭连接，keepalive 池形同虚设。显式清空该头可确保复用上游连接。

---

## 11. 静态资源、gzip 与缓存

### 11.1 静态资源服务

```nginx
location /static/ {
    root /var/www;              # 实际路径：/var/www/static/...
    # 或：
    # alias /var/www/assets/;   # 实际路径：/var/www/assets/<去掉 /static/ 后的部分>
    try_files $uri $uri/ =404;
    expires 30d;
    add_header Cache-Control "public, immutable";
    access_log off;
    log_not_found off;
}
```

`root` 把 location 前缀**拼到**路径后；`alias` **替换** location 前缀。URL 末尾带 `/` 时 `alias` 必须也以 `/` 结尾。

### 11.2 gzip 压缩

```nginx
gzip on;
gzip_comp_level 5;               # 1~9；5 为平衡点
gzip_min_length 1k;
gzip_types text/plain text/css application/json application/javascript
           application/xml application/xml+rss image/svg+xml;
gzip_vary on;                    # 加 Vary: Accept-Encoding
gzip_proxied any;                # 对代理来的请求也压缩
```

可选更优 **Brotli**（需 `ngx_brotli` 模块）：对文本类资源压缩率优于 gzip 15%-25%。

### 11.3 浏览器缓存头

```nginx
# HTML 不缓存，保证快速上线
location / {
    add_header Cache-Control "no-cache";
}

# 构建产物带 hash，可长缓存
location ~* \.(?:js|css)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# 图片中长缓存
location ~* \.(?:jpg|png|webp|gif|ico)$ {
    expires 30d;
    add_header Cache-Control "public";
}
```

### 11.4 Nginx 自身缓存（proxy_cache）

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:100m
                 inactive=1h max_size=10g use_temp_path=off;

server {
    location /api/products/ {
        proxy_pass http://backend;
        proxy_cache api_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_key $scheme$host$request_uri;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;                          # 防"惊群"回源
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

`$upstream_cache_status`：`HIT / MISS / EXPIRED / STALE / UPDATING / REVALIDATED / BYPASS`。

### 📝 笔试题 11-1：`root` 和 `alias` 的关键区别？

```nginx
location /static/ { root  /var/www; }      # /static/a.js → /var/www/static/a.js
location /static/ { alias /var/www/assets/; } # /static/a.js → /var/www/assets/a.js
```

`root` **拼接**；`alias` **替换**。规模大时 `alias` 更常见；注意 `alias` 尾部 `/` 与 location 末尾 `/` 要保持一致。

---

## 12. HTTPS 与 HTTP/2 / HTTP/3 配置

### 12.1 最小 HTTPS

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;       # TLS 1.3 下已无意义
    ssl_session_cache   shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    ssl_stapling        on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / { ... }
}

# HTTP → HTTPS 重定向
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### 12.2 多域名与 SNI

同一 IP 上承载多域名证书依赖 TLS SNI：

```nginx
server {
    listen 443 ssl http2;
    server_name a.com;
    ssl_certificate ...;
    ssl_certificate_key ...;
}
server {
    listen 443 ssl http2;
    server_name b.com;
    ssl_certificate ...;
    ssl_certificate_key ...;
}
```

### 12.3 HTTP/3（QUIC）

Nginx 1.25+ 支持：

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport;        # UDP 443
    http2  on;
    http3  on;
    add_header Alt-Svc 'h3=":443"; ma=86400';

    ssl_certificate ...;
    ssl_certificate_key ...;
    ssl_protocols TLSv1.3;            # QUIC 仅用 TLS 1.3
}
```

防火墙需放行 **UDP 443**。

### 12.4 证书自动化（Let's Encrypt）

```bash
# 使用 certbot
certbot --nginx -d example.com -d www.example.com
# 定时续期（cron 或 systemd timer）
certbot renew --quiet --deploy-hook "nginx -s reload"
```

### 📝 笔试题 12-1：启用 HSTS 的注意事项？

- 仅在**确认全站 HTTPS**后开启，否则用户无法再访问 HTTP 回退
- 先用短 max-age 灰度（如 300s），再提高到 1 年
- `includeSubDomains` 会影响所有子域
- `preload` 需提交到浏览器列表，**一旦加入几乎不可逆**，谨慎

---

## 13. WebSocket 配置

### 13.1 核心要点

WebSocket 握手是 HTTP/1.1 Upgrade 请求，Nginx **默认会剥离** `Upgrade`/`Connection` 头，必须**显式转发**。

### 13.2 标准配置

```nginx
# 把 Upgrade 头映射到 Connection 值
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream ws_backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    # WebSocket 是长连接，粘性由握手那一刻决定，后续不再切换
    # 如业务需要每连接固定 → ip_hash 或 hash $http_x_session_id
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # ...TLS 配置略...

    location /ws {
        proxy_pass http://ws_backend;

        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 关键：空闲读超时必须 > 心跳间隔（通常 25-30 秒）
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
        proxy_connect_timeout 10s;

        # 禁用缓冲（实时性需求）
        proxy_buffering off;
    }
}
```

### 13.3 关键超时

| 参数 | 作用 | 建议 |
|------|------|------|
| `proxy_connect_timeout` | 与上游建连超时 | 5-10s |
| `proxy_send_timeout` | 写入上游两次操作间的最大间隔 | 1h+ |
| `proxy_read_timeout` | **从上游读取**两次操作间的最大间隔 | 1h+ |
| `keepalive_timeout` | 与客户端的空闲保持 | 65s（需 > 心跳） |
| `client_body_timeout` / `client_header_timeout` | 客户端读超时 | 默认即可 |

**常见事故**：ALB 默认 60s 空闲超时，或 Nginx `proxy_read_timeout` 60s，而应用心跳 90s → 连接频繁被中断。修复方式：**超时 > 心跳 × 2**。

### 13.4 wss:// 终结

TLS 在 Nginx 侧终结后，内网走明文 `ws://` 到上游即可。若上游也要 TLS，配 `proxy_pass https://...` + 上游证书校验。

### 13.5 负载均衡策略

- 默认轮询即可，长连接**一旦建立不再切换**
- 若需把特定用户路由到固定节点：用 `hash $arg_uid consistent;` 或 `hash $cookie_uid consistent;`
- Socket.IO 长轮询降级模式**必须**粘性（`ip_hash` 或 cookie），否则握手与传输被分发到不同实例会乱

### 📝 笔试题 13-1：Nginx 反代 WebSocket 时频繁断线，为什么？

最常见原因：**空闲超时**。WebSocket 长连接在无数据流动时被 Nginx / 上游 LB 视为空闲关闭。

修复：

1. 调大 `proxy_read_timeout` / `proxy_send_timeout`（>心跳 2 倍）
2. 应用层 20-30s 发心跳
3. 确认上层 LB（ALB / CDN）空闲超时也对应调整
4. 出现 `1006` / `upstream prematurely closed` 日志即此类问题

---

## 14. 限流、熔断与防护

### 14.1 请求速率限制（limit_req）

基于**令牌桶**：

```nginx
# http 块
limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
limit_req_zone $server_name        zone=perserver:10m rate=1000r/s;

server {
    location /api/ {
        limit_req zone=perip burst=20 nodelay;
        limit_req zone=perserver burst=5000;
        limit_req_status 429;
        # ...
    }
}
```

- `rate=10r/s`：平均每秒 10 个
- `burst=20`：允许突发 20 个排队
- `nodelay`：突发不排队（超过立即 429）；不加 nodelay 则会延迟放行

### 14.2 并发连接限制（limit_conn）

```nginx
limit_conn_zone $binary_remote_addr zone=perip_conn:10m;

server {
    location /download/ {
        limit_conn perip_conn 5;
        limit_rate 1m;          # 每连接限速 1MB/s
    }
}
```

### 14.3 基于 IP 的访问控制

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    deny all;
}

# 或使用 geo / map 精细判断
geo $is_blocked {
    default         0;
    1.2.3.0/24      1;
}
```

### 14.4 防盗链

```nginx
location ~* \.(jpg|png|webp)$ {
    valid_referers none blocked *.example.com;
    if ($invalid_referer) { return 403; }
}
```

### 14.5 WAF

- **ModSecurity** + **OWASP CRS**：成熟 WAF，Nginx/Apache 均可集成
- 云厂商 WAF：直接在入口开启
- 简单规则可用 `map` + `return 403` 拦截

### 14.6 客户端大小 / 慢连接防护

```nginx
client_max_body_size  10m;
client_body_timeout   10s;
client_header_timeout 10s;
send_timeout          10s;
large_client_header_buffers 4 16k;
reset_timedout_connection on;
```

### 📝 笔试题 14-1：`limit_req` 的 `burst` 与 `nodelay` 区别？

- 只设 `burst=20`：允许 20 个突发请求**排队**，以 `rate` 速率被放行
- 加 `nodelay`：突发 20 个**立即**放行，但桶占用不减，其后新请求需等桶恢复；超过 burst 仍立即 429

**对用户体验更好**：`burst + nodelay`，避免突发被延迟。

---

## 15. 日志、监控与调试

### 15.1 日志格式

```nginx
log_format json escape=json
    '{"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":$status,'
    '"bytes":$body_bytes_sent,'
    '"rt":$request_time,'
    '"upstream":"$upstream_addr",'
    '"ucs":"$upstream_cache_status",'
    '"uct":"$upstream_connect_time",'
    '"urt":"$upstream_response_time",'
    '"ua":"$http_user_agent",'
    '"xff":"$http_x_forwarded_for"}';

access_log /var/log/nginx/access.json json;
error_log  /var/log/nginx/error.log warn;
```

JSON 格式便于直接送入 ELK/Loki。

### 15.2 条件日志

```nginx
map $status $loggable {
    ~^[23]  0;             # 2xx/3xx 不记日志（看业务）
    default 1;
}
access_log /var/log/nginx/access.log json if=$loggable;
```

### 15.3 日志轮转

用系统 `logrotate`：

```
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

`USR1` 让 nginx **重新打开日志文件**（无需重启）。

### 15.4 状态监控

```nginx
# stub_status（OSS）
location = /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

输出：

```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6  Writing: 179  Waiting: 106
```

- 生产级：**Nginx VTS 模块 / Prometheus `nginx-prometheus-exporter`** 暴露更丰富指标
- `OpenResty / APISIX` 本身内置更完整监控

### 15.5 调试技巧

- `nginx -T`：输出**合并后的完整配置**，排查 include 是否生效
- `nginx -t`：语法检查
- `strace -f -p <worker pid>`：系统调用跟踪
- 临时加 `error_log /var/log/nginx/debug.log debug;`（需编译时带 `--with-debug`）
- 请求链路打点：自定义 header `X-Request-Id` 贯穿 LB → Nginx → App → Log

### 📝 笔试题 15-1：线上 502 飙升如何排查？

1. 看 `error.log`，常见原因：
   - `connect() failed (111: Connection refused)`：上游未启动
   - `upstream prematurely closed connection`：上游崩溃或超时
   - `no live upstreams`：所有上游被摘，可能是健康探测失败
2. `stub_status` / 监控：active 连接、4xx/5xx 比率、upstream 响应时间
3. 上游日志：应用是否 OOM / GC / 异常
4. 临时降级：`proxy_next_upstream` 重试、备机切换、限流
5. 根因：资源、依赖（DB）、代码 bug

---

## 16. Nginx 作为 TCP/UDP 四层负载

`stream` 模块处理非 HTTP 流量。

### 16.1 TCP 反代

```nginx
stream {
    upstream mysql_backend {
        server 10.0.0.11:3306 max_fails=2 fail_timeout=10s;
        server 10.0.0.12:3306 backup;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
        proxy_connect_timeout 3s;
        proxy_timeout         1h;      # 数据空闲超时
        proxy_next_upstream   on;
    }
}
```

### 16.2 UDP 反代（如 DNS / QUIC）

```nginx
stream {
    upstream dns_pool {
        server 10.0.0.1:53;
        server 10.0.0.2:53;
    }
    server {
        listen 53 udp reuseport;
        proxy_pass dns_pool;
        proxy_responses 1;              # 期望的响应包数
        proxy_timeout   5s;
    }
}
```

### 16.3 TLS Passthrough / Termination

`stream` 支持 TLS 终结（`ssl_preread` 做 SNI 路由）：

```nginx
stream {
    map $ssl_preread_server_name $backend {
        a.example.com  a_pool;
        b.example.com  b_pool;
    }
    upstream a_pool { server 10.0.0.1:443; }
    upstream b_pool { server 10.0.0.2:443; }

    server {
        listen 443;
        proxy_pass $backend;
        ssl_preread on;            # 只读 SNI 不解密
    }
}
```

适合把 HTTPS 透传到不同上游实例（例如多租户入口）。

### 📝 笔试题 16-1：什么情况下用 `stream` 而不是 `http`？

- 非 HTTP 协议：数据库、Redis、MQ、游戏自定义 TCP、UDP DNS
- 需要 **TLS 透传**（不在 Nginx 解密）
- 纯四层转发，追求极致吞吐

---

## 17. 生产部署实战片段

### 17.1 API 网关

```nginx
upstream api_v1 {
    server 10.0.1.1:8080 max_fails=3 fail_timeout=10s;
    server 10.0.1.2:8080;
    keepalive 64;
}
upstream api_v2 {
    server 10.0.2.1:8080;
    server 10.0.2.2:8080;
    keepalive 64;
}

map $http_x_api_version $api_pool {
    default  api_v1;
    "2"      api_v2;
    "2.0"    api_v2;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    # ...TLS...

    location / {
        proxy_pass http://$api_pool;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Request-Id $request_id;    # 1.11.0+
    }
}
```

### 17.2 前后端分离 SPA

```nginx
server {
    listen 80;
    server_name app.example.com;
    root /var/www/spa;

    # 静态资源
    location /assets/ {
        try_files $uri =404;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API 反代
    location /api/ {
        proxy_pass http://api_backend/;
    }

    # SPA 路由回退
    location / {
        try_files $uri /index.html;
    }
}
```

### 17.3 灰度/金丝雀发布

```nginx
upstream canary {
    server 10.0.0.10:8080;
}
upstream stable {
    server 10.0.0.11:8080;
    server 10.0.0.12:8080;
}

split_clients "${remote_addr}${http_user_agent}${date_gmt}" $pool {
    5%     canary;
    *      stable;
}

server {
    location / {
        proxy_pass http://$pool;
    }
}
```

`split_clients` 按哈希稳定分流；也可结合 cookie 做定向灰度：

```nginx
map $cookie_canary $pool {
    default stable;
    "1"     canary;
}
```

### 17.4 Maintenance 模式

```nginx
# 根据文件存在与否判断
server {
    set $maintenance 0;
    if (-f /etc/nginx/maintenance.on) { set $maintenance 1; }

    location / {
        if ($maintenance = 1) {
            return 503;
        }
        proxy_pass http://backend;
    }

    error_page 503 /maintenance.html;
    location = /maintenance.html { root /var/www/err; internal; }
}
```

### 17.5 集成 Basic Auth（临时防护）

```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

```nginx
location /admin/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://backend/admin/;
}
```

---

## 18. 常见问题与性能调优

### 18.1 常见 502 / 504 / 499

| 状态 | 含义 | 常见原因 |
|------|------|----------|
| **502 Bad Gateway** | 上游返回非法响应或连接失败 | 上游未启动 / 崩溃 / `proxy_read_timeout` |
| **504 Gateway Timeout** | 等待上游超时 | 上游慢、超时太小 |
| **499**（Nginx 自定义） | **客户端主动断开** | 客户端超时、用户离开页面、LB/浏览器 abort |
| **413 Request Entity Too Large** | 请求体超 `client_max_body_size` | 上传文件调大该项 |
| **upstream prematurely closed connection** | 上游主动断 | keepalive_requests/timeout 不匹配、崩溃 |

### 18.2 性能调优清单

**系统层**：

```bash
ulimit -n                             # 单进程 fd 上限
# /etc/security/limits.conf
nginx soft nofile 1048576
nginx hard nofile 1048576

# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.core.netdev_max_backlog = 65535
```

**Nginx 配置**：

```nginx
worker_processes auto;
worker_rlimit_nofile 1048576;

events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65s;
    keepalive_requests 10000;
    open_file_cache          max=10000 inactive=60s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;
    reset_timedout_connection on;
}
```

**上游 keepalive**（常被遗漏但极关键）：

```nginx
upstream app {
    server ...;
    keepalive 64;
    keepalive_timeout 60s;
    keepalive_requests 1000;
}
server {
    location / {
        proxy_pass http://app;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

### 18.3 部分 vs 完整热重载

- **`nginx -s reload`**：秒级完成，对大多数变更足够
- 热重载**无法改**：master 进程启动时读取的若干参数（如 `worker_processes` 需重启）
- 模块热升级：`kill -USR2 <master>` + `kill -QUIT <old>`，零停机升级二进制

### 18.4 故障快速恢复

- 备好**回滚脚本**：保留上一份 conf，快速 diff
- `nginx -T` 定期 dump 到配置仓库（可入 Git）
- 配置变更 **CI 校验** `nginx -t`
- 只读日志分区防写满
- 关键路径（`/healthz`）绕过所有 `if/rewrite/auth`

### 📝 笔试题 18-1：499 状态码代表什么？出现大量 499 怎么办？

499 是 **Nginx 自定义**状态码，表示**客户端在服务端返回响应前已断开连接**。大量 499 的常见原因：

1. 上游响应太慢（客户端超时先到）
2. 客户端网络差 / 用户关掉页面
3. 某些 LB / CDN 对客户端 timeout 比 Nginx 短

排查：看 `request_time` 与 `upstream_response_time` 对比，优化上游；客户端侧检查重试与超时策略。

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 下列哪项**不是** L7 负载均衡的能力？
A. 按 URL 路由
B. 基于 Cookie 粘性
C. 透传任意 TCP 协议
D. TLS 终结

<details><summary>答案</summary>C。</details>

**Q2** Nginx 中优先级最高的 location 匹配方式是？
A. 普通前缀  B. `^~` 前缀  C. `=` 精确  D. `~` 正则

<details><summary>答案</summary>C。</details>

**Q3** `proxy_pass http://backend/` 与 `proxy_pass http://backend`（末尾无斜杠）的区别？
A. 无区别
B. 前者用上游 URL 替换 location 前缀，后者把原 URI 原样拼接
C. 前者走 HTTP/1.0
D. 前者支持 keepalive

<details><summary>答案</summary>B。</details>

**Q4** WebSocket 反代必需的两个头是？
A. `Host` 和 `Referer`
B. `Upgrade` 和 `Connection`
C. `X-Forwarded-For` 和 `X-Real-IP`
D. `Accept` 和 `Content-Type`

<details><summary>答案</summary>B。</details>

**Q5** 下列关于 `nginx -s reload` 描述**错误**的是？
A. 不中断现有连接
B. 可以修改 `worker_processes` 立即生效
C. 先校验配置
D. 老 worker 处理完存量请求再退出

<details><summary>答案</summary>B（需重启）。</details>

**Q6** 下列哪项会**关闭**对上游的连接复用（keepalive）？
A. 指定 `keepalive 64;`
B. 设置 `proxy_http_version 1.1;`
C. 不清空 `Connection` 头
D. 在 location 中写 `proxy_pass http://app;`

<details><summary>答案</summary>C。</details>

**Q7** Nginx 反代 WebSocket 频繁断开，最可能的原因是？
A. SSL 证书过期
B. `proxy_read_timeout` 小于心跳间隔
C. 使用 HTTP/2
D. gzip 开启

<details><summary>答案</summary>B。</details>

**Q8** 关于 `stream` 模块的陈述，正确的是？
A. 只能做 HTTP 反代
B. 可以做 TCP/UDP 四层反代
C. 不支持 SSL
D. 与 http 块可以混写

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. Nginx 是多线程架构。 ❌（多进程 + 事件驱动）
2. `$host` 包含端口号。 ❌
3. `alias` 与 location 前缀会被**替换**，`root` 会被**拼接**。 ✅
4. IP Hash 可以保证一个用户始终打到同一台后端。 ✅（公网 NAT 例外时不完全可靠）
5. `nginx -s reload` 不会有服务中断。 ✅
6. 启用 `proxy_buffering` 对流式接口有益。 ❌
7. 健康检查里加深度依赖检查更好。 ❌（容易级联雪崩）
8. `limit_req` 用的是令牌桶算法。 ✅

### 19.3 简答题

**Q1** Nginx 为什么性能高？

- 多进程 Master-Worker，每 worker 单线程事件驱动（epoll）
- 非阻塞 IO + 零拷贝（`sendfile`）
- 内存池、高效缓冲
- 模块化，按需加载

**Q2** 如何区分"Nginx 到上游的 keepalive"和"客户端到 Nginx 的 keepalive"？

- **客户端→Nginx**：`keepalive_timeout`（http/server 作用域），默认开启
- **Nginx→上游**：`upstream` 块内 `keepalive 64;` + location 里 `proxy_http_version 1.1;` + `proxy_set_header Connection "";`

两者是独立配置，经常被混淆。

**Q3** 对于一个上传接口，报 `413 Request Entity Too Large`，怎么处理？

```nginx
http {
    client_max_body_size 100m;
}
# 或 server/location 级别覆盖
```

同时评估：

- 上游是否也有大小限制（Spring `server.servlet.multipart.max-file-size` 等）
- 大文件走分片上传 / 预签名直传对象存储，不经过 Nginx

**Q4** 反代 WebSocket 需要哪些关键配置？

```nginx
# 1. map
map $http_upgrade $connection_upgrade { default upgrade; '' close; }

# 2. location
location /ws {
    proxy_pass http://ws_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    proxy_buffering off;
}
```

### 19.4 实操题

**Q1** 将 `http://example.com` 和 `http://www.example.com` 永久跳转到 `https://www.example.com`。

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://www.example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;
    # ssl_cert ...
    return 301 https://www.example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.example.com;
    # ssl_cert ...
    # ... 正常业务 ...
}
```

**Q2** 给 `/api/` 接口按客户端 IP 限流 20r/s，突发 50 不延迟。

```nginx
limit_req_zone $binary_remote_addr zone=apilim:10m rate=20r/s;

server {
    location /api/ {
        limit_req zone=apilim burst=50 nodelay;
        limit_req_status 429;
        proxy_pass http://api_backend;
    }
}
```

**Q3** 在 Nginx 上把 `/v1/*` 转发到 `svc-a`，`/v2/*` 转发到 `svc-b`，保留原始 URI。

```nginx
upstream svc_a { server 10.0.0.1:8080; }
upstream svc_b { server 10.0.0.2:8080; }

server {
    listen 80;
    server_name api.example.com;

    location /v1/ { proxy_pass http://svc_a; }      # 不加斜杠 → /v1/foo 保持
    location /v2/ { proxy_pass http://svc_b; }

    # 公共头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

**Q4** 同时提供 WSS 和 HTTPS，根据 URL 前缀区分：`/ws` 是 WebSocket，其他是 REST。

```nginx
map $http_upgrade $connection_upgrade { default upgrade; '' close; }

upstream api   { server 10.0.0.1:8080; keepalive 32; }
upstream wsapp { server 10.0.0.1:8090; }

server {
    listen 443 ssl http2;
    server_name api.example.com;
    # ssl ...

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_http_version 1.1;

    location /ws {
        proxy_pass http://wsapp;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
    }

    location / {
        proxy_pass http://api;
        proxy_set_header Connection "";
    }
}
```

---

## 📚 学习建议

1. **动手改小 conf**：本机起一个 Nginx，反代一个 echo 服务，跟着本讲义逐项跑一遍
2. **看官方 docs**：`nginx.org/en/docs/` 模块文档最权威；`ngx_http_core_module`、`ngx_http_proxy_module` 必读
3. **读开源 conf**：GitHub 上有大量生产 nginx.conf 模板（`h5bp/server-configs-nginx` 等），学习最佳实践
4. **做压测**：`wrk` / `ab` / `k6` 压出短板，才能真正体会各参数作用
5. **思考边界**：单 Nginx 极限是多少？何时上 LVS/Envoy/ALB？何时拆 Gateway？架构没有银弹，权衡带宽、延迟、弹性、成本
6. **监控 + 日志先行**：新上线业务先备齐 `stub_status`/Prometheus/JSON 日志，再谈性能

> 祝你的流量稳稳落到每一台后端。
