# RESTful API 设计与开发讲义

> 本讲义系统介绍 RESTful API 的设计原则、最佳实践、工程化落地与常见反模式，配合笔试题加深理解。适合后端开发、架构师、API 设计者阅读。
>
> 约定：示例 URL 以 `https://api.example.com` 为基址；示例响应体用 JSON。

## 目录

1. [什么是 REST](#1-什么是-rest)
2. [REST 六大约束与成熟度模型](#2-rest-六大约束与成熟度模型)
3. [资源与 URI 设计](#3-资源与-uri-设计)
4. [HTTP 方法与语义](#4-http-方法与语义)
5. [状态码规范](#5-状态码规范)
6. [请求与响应格式](#6-请求与响应格式)
7. [查询、过滤、排序、分页](#7-查询过滤排序分页)
8. [错误处理](#8-错误处理)
9. [版本化策略](#9-版本化策略)
10. [认证与授权](#10-认证与授权)
11. [幂等性、并发与一致性](#11-幂等性并发与一致性)
12. [缓存与性能](#12-缓存与性能)
13. [安全最佳实践](#13-安全最佳实践)
14. [HATEOAS 与超媒体](#14-hateoas-与超媒体)
15. [API 文档与契约：OpenAPI 与 Swagger UI](#15-api-文档与契约openapi-与-swagger-ui)
16. [测试、Mock 与工程化](#16-测试mock-与工程化)
17. [REST 与 GraphQL / gRPC 对比](#17-rest-与-graphql--grpc-对比)
18. [常见反模式](#18-常见反模式)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 什么是 REST

REST (Representational State Transfer) 是 Roy Fielding 在 2000 年博士论文中提出的一种**分布式系统架构风格**，不是协议也不是标准。它定义了一组约束，满足这些约束的系统被称为 RESTful。

### 1.1 核心思想

- **资源（Resource）** 是 REST 的核心抽象：任何可命名的信息都是资源（文档、图片、服务、集合）
- 每个资源由 **URI** 标识
- 客户端通过**表示**（JSON/XML/HTML 等）操作资源，而不直接操作资源本身
- 使用 HTTP **标准方法**（GET/POST/PUT/PATCH/DELETE）表达操作语义
- 通信**无状态**：每次请求自包含所需信息

### 1.2 RESTful vs REST-like

工业界大量号称"RESTful"的 API 其实只是**基于 HTTP + JSON 的 API**，并未严格遵循 REST 所有约束（尤其是 HATEOAS）。本讲义的目标是给出**工程可落地**的规范，而非学院派的纯粹 REST。

### 📝 笔试题 1-1：REST 的全称？它是协议还是风格？

**答**：Representational State Transfer，"表现层状态转移"。它是一种**架构风格**，不是协议，也不是标准。

---

## 2. REST 六大约束与成熟度模型

### 2.1 六大约束

1. **客户端-服务器（Client-Server）**：关注点分离，UI 与数据独立演进
2. **无状态（Stateless）**：每个请求携带全部上下文，服务端不保存会话
3. **可缓存（Cacheable）**：响应要明确指示是否可缓存及策略
4. **统一接口（Uniform Interface）**：REST 的核心，包含四个子约束：
   - 资源标识（URI）
   - 通过表示操作资源
   - 自描述消息（媒体类型、状态码）
   - HATEOAS（超媒体驱动状态）
5. **分层系统（Layered System）**：客户端不感知中间层（LB、网关、代理、CDN）
6. **按需代码（Code on Demand，可选）**：服务端可下发可执行代码（JS 等）

### 2.2 Richardson 成熟度模型

| 等级 | 描述 | 示例 |
|------|------|------|
| L0 | HTTP 作为传输隧道 | SOAP、RPC-over-HTTP，所有请求都是 POST /endpoint |
| L1 | 引入**资源** | URI 代表资源，但所有操作仍用 POST |
| L2 | 使用 HTTP **动词 + 状态码** | GET/POST/PUT/DELETE，200/201/404 等 |
| L3 | **HATEOAS**（超媒体） | 响应中含可跳转链接，客户端按链接导航 |

大多数"RESTful API" 实际上处于 **L2**。

### 📝 笔试题 2-1：无状态意味着不能有登录态吗？

不。**无状态**指服务端不保存客户端会话，但客户端每次请求可携带凭证（Token、Cookie）。服务端仍可查数据库/缓存拿到用户信息，但**这次请求的状态**必须自包含。典型实现：JWT 将身份信息放入 token 本身。

---

## 3. 资源与 URI 设计

### 3.1 核心原则

- **名词而非动词**：URI 表示"是什么"，HTTP 方法表示"做什么"
- **复数形式统一**：`/users`、`/orders`，而非混用单复数
- **小写 + 短横线**：`/user-profiles`，不用下划线或驼峰
- **层级表达关系**：`/users/{id}/orders`
- **不含文件扩展名**：`.json`/`.xml` 交给 `Accept` 头协商

### 3.2 好与坏的对比

```
✅ GET  /users
✅ POST /users
✅ GET  /users/42
✅ GET  /users/42/orders
✅ PUT  /users/42
✅ DELETE /orders/100

❌ GET  /getUsers
❌ POST /users/create
❌ GET  /user/42      (单数)
❌ POST /user/42/delete
❌ GET  /users.json
```

### 3.3 集合与元素

```
/users           → 集合（collection）
/users/42        → 元素（item）
/users/42/orders → 子集合
/users/42/orders/100 → 子元素
```

### 3.4 非资源型操作

真实业务中有些操作难以建模为 CRUD，两种常见处理：

- **作为动作子资源**：`POST /orders/100/refund`、`POST /users/42/activate`
- **作为状态变更**：`PATCH /orders/100 { "status": "refunded" }`

前者更显式，后者更"纯"。二选一保持一致即可。

### 3.5 查询 vs 路径

- 定位资源 → **路径**：`/users/42`
- 过滤、排序、分页 → **查询参数**：`/users?role=admin&sort=-created_at`

### 3.6 ID 设计

- 避免暴露自增主键（可被枚举探测），考虑：
  - UUID / ULID
  - 短随机字符串（Nano ID）
  - 业务编号（订单号）
- 对于高并发生成：Snowflake、数据库号段

### 📝 笔试题 3-1：下列哪个 URI 最规范？

A. `POST /api/createUser`
B. `POST /api/user/new`
C. `POST /api/users`
D. `GET  /api/users/create`

**答**：C。POST + 复数名词，语义清晰。

### 📝 笔试题 3-2：如何为"用户关注另一个用户"建模？

两种可行设计：

- **资源化**（推荐）：`PUT /users/{id}/following/{targetId}` 关注，`DELETE` 取消
- **集合加成员**：`POST /users/{id}/following { "target_id": ... }`

两者都优于 `POST /follow?from=&to=`。

---

## 4. HTTP 方法与语义

### 4.1 方法语义总表

| 方法 | 语义 | 幂等 | 安全 | 可缓存 | 典型状态码 |
|------|------|------|------|--------|------------|
| GET | 读取 | ✅ | ✅ | ✅ | 200, 304, 404 |
| HEAD | 只要头 | ✅ | ✅ | ✅ | 200, 404 |
| OPTIONS | 方法清单 / CORS 预检 | ✅ | ✅ | ❌ | 200, 204 |
| POST | 创建 / 非幂等动作 | ❌ | ❌ | 条件性 | 201, 202, 204 |
| PUT | 整体替换 / 创建 | ✅ | ❌ | ❌ | 200, 201, 204 |
| PATCH | 部分更新 | 视实现 | ❌ | ❌ | 200, 204 |
| DELETE | 删除 | ✅ | ❌ | ❌ | 200, 202, 204, 404 |

- **安全**：不对服务器状态产生副作用
- **幂等**：多次相同请求，效果与一次相同

### 4.2 PUT vs PATCH

- **PUT**：**整体替换**，请求体是资源的完整新表示。未提供的字段通常被视为置空或保留默认
- **PATCH**：**局部更新**，只包含需要改变的字段

PATCH 的两种风格：

- **JSON Merge Patch**（RFC 7396，简单常用）：
  ```
  PATCH /users/42 HTTP/1.1
  Content-Type: application/merge-patch+json
  { "email": "new@x.com" }
  ```
- **JSON Patch**（RFC 6902，精确）：
  ```
  PATCH /users/42 HTTP/1.1
  Content-Type: application/json-patch+json
  [
    { "op": "replace", "path": "/email", "value": "new@x.com" },
    { "op": "remove",  "path": "/phone" }
  ]
  ```

### 4.3 POST 的双重角色

- **在集合 URL 下**：`POST /users` → 创建资源，返回 `201 Created` + `Location` 头
- **非幂等动作**：`POST /orders/100/refund`

### 4.4 DELETE 的幂等语义

- 首次 `DELETE /users/42` → 200/204
- 再次 `DELETE /users/42` → 应仍返回 204 或 404，**但不应报错**；从客户端视角效果一致
- 业界两派：**返回 404**（更真实）或 **返回 204**（强调幂等）；选择一致即可

### 📝 笔试题 4-1：POST 是否幂等？PUT 是否幂等？举例说明。

- POST：**非幂等**。`POST /orders` 多次会创建多个订单
- PUT：**幂等**。`PUT /users/42 { name: "Bob" }` 多次结果都是 Bob

### 📝 笔试题 4-2：某接口"更新订单状态"，用 PUT 还是 PATCH？

优先 **PATCH**：只修改状态一个字段；用 `PUT` 需要客户端传完整订单表示，容易覆盖掉并发修改的其他字段。另一种显式做法：**PATCH 动作子资源**，如 `POST /orders/100/cancel`。

---

## 5. 状态码规范

### 5.1 状态码分类

| 类 | 范围 | 含义 |
|----|------|------|
| 1xx | 信息 | 100 Continue, 101 Switching |
| 2xx | 成功 | 200, 201, 202, 204, 206 |
| 3xx | 重定向 | 301, 302, 304, 307, 308 |
| 4xx | 客户端错误 | 400, 401, 403, 404, 409, 410, 422, 429 |
| 5xx | 服务端错误 | 500, 502, 503, 504 |

### 5.2 REST 常用状态码

| 代码 | 名称 | 适用场景 |
|------|------|----------|
| **200** | OK | GET/PUT/PATCH 成功 |
| **201** | Created | POST 成功创建，响应带 `Location` |
| **202** | Accepted | 异步处理已受理，未完成 |
| **204** | No Content | DELETE 或 PUT 无响应体 |
| **301** | Moved Permanently | 资源永久迁移 |
| **304** | Not Modified | 协商缓存命中 |
| **400** | Bad Request | 参数错误、JSON 不合法 |
| **401** | Unauthorized | **未认证**（缺/错凭证） |
| **403** | Forbidden | 已认证但**无权限** |
| **404** | Not Found | 资源不存在 |
| **405** | Method Not Allowed | 方法不支持，必须带 `Allow` 头 |
| **406** | Not Acceptable | 内容协商失败 |
| **409** | Conflict | 状态冲突（乐观锁失败、业务冲突） |
| **410** | Gone | 资源曾存在，已永久移除 |
| **412** | Precondition Failed | `If-Match`/`If-Unmodified-Since` 未满足 |
| **413** | Payload Too Large | 请求体过大 |
| **415** | Unsupported Media Type | `Content-Type` 不支持 |
| **422** | Unprocessable Entity | 语法对但**语义错**（参数值非法） |
| **429** | Too Many Requests | 限流，带 `Retry-After` |
| **500** | Internal Server Error | 未分类异常 |
| **502** | Bad Gateway | 网关从上游收到非法响应 |
| **503** | Service Unavailable | 服务不可用（维护/过载），可带 `Retry-After` |
| **504** | Gateway Timeout | 网关等待上游超时 |

### 5.3 常见选择争议

- **400 vs 422**：规范上 `400` 指**语法/协议错**，`422` 指**语义错**（JSON 合法但业务规则失败）。工程上很多团队统一用 400，重要的是一致与清晰的错误体
- **401 vs 403**：`401` 客户端"你没表明身份"；`403` "身份知道但不给"
- **200 vs 204**：DELETE 没啥可返回就用 204；有业务数据就 200
- **404 vs 403**：对敏感资源，为防枚举可统一返回 404（隐藏存在性）

### 📝 笔试题 5-1：客户端用过期 token 访问，应该返回什么？

**401 Unauthorized**。并通过 `WWW-Authenticate` 响应头给出详情（如 `Bearer error="invalid_token"`），客户端据此触发刷新或重登录。

### 📝 笔试题 5-2：`POST /users` 创建成功应返回什么？

```
HTTP/1.1 201 Created
Location: /users/42
Content-Type: application/json

{ "id": 42, "name": "Alice", "created_at": "..." }
```

关键点：**201** + `Location` 头指向新资源 URI + 响应体返回资源表示。

---

## 6. 请求与响应格式

### 6.1 媒体类型与内容协商

客户端声明能接受的类型：

```
Accept: application/json, application/xml;q=0.9
```

服务端选择并返回：

```
Content-Type: application/json; charset=utf-8
```

常用媒体类型：

- `application/json`：主流
- `application/xml`：老系统
- `application/problem+json`（RFC 7807）：统一错误响应
- `application/hal+json`：HATEOAS 超媒体
- `application/vnd.api+json`：JSON:API 规范
- `application/vnd.example.v1+json`：自定义 vendor 类型（用于版本化）

### 6.2 命名风格

- 字段名统一：**snake_case** 或 **camelCase**，择一坚持
- 布尔字段：`is_active`、`has_children`
- 时间字段：ISO 8601 + UTC：`2025-01-15T08:30:00Z`
- 金额：字符串 + 货币码，避免浮点误差：`{ "amount": "100.00", "currency": "USD" }`

### 6.3 响应信封（Envelope）

两派做法：

**A. 无信封**（推荐，简单，HTTP 原生）：

```json
{
  "id": 42,
  "name": "Alice"
}
```

**B. 有信封**（部分团队/旧栈）：

```json
{
  "code": 0,
  "message": "ok",
  "data": { "id": 42, "name": "Alice" }
}
```

**利弊**：信封与 HTTP 状态码容易割裂（HTTP 返 200，`code: -1`），导致缓存、日志、监控失效。推荐用 HTTP 状态码 + RFC 7807 错误体，仅在多通道统一（如 RPC over HTTP）时考虑信封。

### 6.4 请求头常见字段

| 头 | 含义 |
|----|------|
| `Content-Type` | 请求体媒体类型 |
| `Accept` | 期望响应类型 |
| `Accept-Language` | 期望语言（i18n） |
| `Authorization` | 凭证 |
| `If-Match` / `If-None-Match` | 条件请求（ETag） |
| `If-Modified-Since` | 协商缓存 |
| `Idempotency-Key` | 幂等标识（Stripe 风格） |
| `X-Request-Id` | 链路追踪 |

### 📝 笔试题 6-1：为什么金额不建议用浮点？

JSON 的数字按 IEEE 754 双精度表示，`0.1 + 0.2 !== 0.3`。金额计算会产生微小误差，审计困难。方案：**整数最小单位（分/厘）** 或 **字符串**，服务端用 `decimal` 类型处理。

---

## 7. 查询、过滤、排序、分页

### 7.1 过滤

```
GET /users?role=admin&status=active
GET /orders?created_at__gte=2025-01-01&created_at__lt=2025-02-01
GET /products?price__gte=100&price__lte=500
```

复杂查询可借助固定后缀：`__gte / __lte / __in / __like / __not`。

### 7.2 排序

```
GET /users?sort=created_at            # 升序
GET /users?sort=-created_at           # 降序（前缀 -）
GET /users?sort=-created_at,name      # 多字段
```

### 7.3 字段裁剪（Sparse Fieldsets）

```
GET /users?fields=id,name,email
```

减小响应体，提升带宽效率。

### 7.4 分页

三种主流方案：

**A. 偏移分页（Offset）**：

```
GET /users?page=2&page_size=20
GET /users?offset=20&limit=20
```

- 优点：直观，可跳页
- 缺点：翻到深页越来越慢（`OFFSET N` 需跳过 N 条），且**实时数据漂移**

**B. 游标分页（Cursor，推荐大数据量）**：

```
GET /users?cursor=eyJpZCI6MTAwfQ==&limit=20
```

- 返回下一页 `next_cursor`
- 适合无限滚动、一致性强
- 不能直接跳到第 N 页

**C. 键集分页（Keyset / Seek）**：

```
GET /users?after_id=100&limit=20
# SELECT * FROM users WHERE id > 100 ORDER BY id LIMIT 20
```

**返回格式建议**：

```json
{
  "items": [...],
  "pagination": {
    "total": 1234,               // 可选，昂贵时省略
    "page": 2,
    "page_size": 20,
    "next": "/users?page=3&page_size=20"
  }
}
```

或使用 `Link` 头（GitHub 风格）：

```
Link: </users?page=3>; rel="next", </users?page=62>; rel="last"
```

### 7.5 搜索

```
GET /users?q=alice
GET /products?search=phone&category=electronics
```

复杂全文搜索可分离为专用端点 `/search`，底层用 ES/OpenSearch。

### 📝 笔试题 7-1：深度分页为什么慢？如何优化？

`LIMIT 20 OFFSET 1000000` 数据库仍需扫描并跳过前 100 万行。优化：

- 改用**键集分页**：`WHERE id > last_id LIMIT 20`
- 游标编码（如 Base64 的 `{last_id, last_sort_key}`）
- 索引覆盖
- 禁止无限制跳页（UI 只给"下一页"）

---

## 8. 错误处理

### 8.1 原则

- **用 HTTP 状态码表达类别**（4xx 客户端 / 5xx 服务端）
- **响应体给出详情**：可机读的错误码 + 人类可读的 message
- **错误可定位**：携带 trace_id / request_id
- **避免泄漏内部细节**：不回显堆栈、SQL

### 8.2 RFC 7807 Problem Details

业界推荐的统一错误格式，`Content-Type: application/problem+json`：

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "email must be a valid email address",
  "instance": "/users",
  "errors": [
    { "field": "email", "code": "invalid_format", "message": "must be a valid email" },
    { "field": "age",   "code": "out_of_range",   "message": "must be >= 18" }
  ],
  "trace_id": "a1b2c3d4"
}
```

### 8.3 错误码设计

- **业务错误码**：独立于 HTTP 状态码，便于客户端国际化与精确处理
- 建议字符串语义码：`USER_NOT_FOUND`、`BALANCE_INSUFFICIENT`，而非纯数字

### 8.4 字段级错误 vs 请求级错误

```json
{
  "status": 422,
  "title": "Validation failed",
  "errors": [
    { "field": "email", "code": "required" },
    { "field": "password", "code": "too_short" }
  ]
}
```

前端可按 `field` 标红对应输入框。

### 📝 笔试题 8-1：业务异常该 200 还是 4xx？

**4xx**。把业务错误放在 HTTP 200 里（用 `code: -1`）会让中间件、日志、监控、缓存误判。正确做法：HTTP 状态码准确反映结果类别，响应体给出业务细节。

---

## 9. 版本化策略

### 9.1 什么时候需要版本化？

发生**破坏性变更（breaking change）** 时：

- 删除字段/端点
- 改变字段类型或含义
- 必填字段变动
- 默认行为变化

**非破坏性变更**（加字段、加端点、宽松校验）通常不需要升版本。

### 9.2 四种方式

| 方式 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| URI 版本 | `/v1/users` | 直观、路由简单 | URL 与版本耦合，违反 REST 资源唯一性 |
| Header 版本 | `Accept: application/vnd.example.v1+json` | 同一资源 URI 不变 | 调试不便，浏览器访问差 |
| 查询参数 | `/users?version=1` | 简单 | 与过滤参数混淆 |
| Content Type | `Accept: application/json; version=1` | — | 同 Header 版本 |

**业界主流选择**：**URI 版本**（GitHub、Twitter、AWS 一度都用），维护成本低，沟通效率高。

### 9.3 兼容策略

- **只加不减**：新增可选字段，老字段保持有效
- **弃用流程**：标注 `Deprecation` 头 + `Sunset` 头告知下线时间
- **并行维护周期**：老版本至少维护 6-12 个月
- **灰度发布**：按客户端、租户比例切流

```
Deprecation: true
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Link: </v2/users>; rel="successor-version"
```

### 📝 笔试题 9-1：新增一个响应字段，需要升版本吗？

通常**不需要**。客户端应遵循 **Tolerant Reader** 原则——忽略未知字段。但如果字段在现有流程中引入歧义（如同名意义变了），则属于破坏性变更。

---

## 10. 认证与授权

### 10.1 概念辨析

- **Authentication（认证 AuthN）**：你是谁
- **Authorization（授权 AuthZ）**：你能做什么

### 10.2 常见认证方式

#### Basic Auth

```
Authorization: Basic base64(user:pass)
```

仅限 HTTPS；明文+Base64，适合内部脚本，不推荐对外。

#### Bearer Token（含 JWT）

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

- **JWT** 三段：Header.Payload.Signature
- 无状态验证：服务端用密钥/公钥校验签名
- **风险**：无法主动吊销（需维护黑名单）；payload 可见不可伪造
- 建议短有效期 + Refresh Token 机制

#### API Key

```
Authorization: ApiKey <key>
# 或
X-API-Key: <key>
```

用于机器对机器，配合 IP 白名单、限流。

#### OAuth 2.0 / OIDC

- **OAuth 2.0**：授权框架，得到 access_token
- **OIDC**：在 OAuth 2.0 之上加身份层，得到 id_token（JWT）
- 常见 flow：
  - **Authorization Code + PKCE**：Web/Mobile App 标配
  - **Client Credentials**：服务间调用
  - **Device Code**：无浏览器设备
  - 不推荐：Implicit、ROPC（已弃用）

#### 双向 TLS（mTLS）

服务间强身份认证，客户端也需提供证书。

### 10.3 授权模型

- **RBAC**（Role-Based）：按角色授予权限集
- **ABAC**（Attribute-Based）：按属性/上下文决策
- **ReBAC**（Relationship-Based）：基于关系图（如 Google Zanzibar）

### 10.4 权限检查点

- **认证中间件**：解析 token，注入 `current_user`
- **端点/资源层**：按角色/范围判定
- **数据层**：行级过滤，避免"能调接口就看所有数据"

### 10.5 Scope（作用域）

OAuth 常用 scope 限制 token 能力：

```
scope=read:user write:order
```

接口在授权中间件中校验 scope。

### 📝 笔试题 10-1：JWT 如何注销？

JWT 无状态特性意味着签名有效期内天然"不可撤销"。实践方案：

1. **短 access_token + 长 refresh_token**：access 很快过期，登出时使 refresh 失效即可
2. **黑名单**：把已注销 jti 存 Redis，带 TTL
3. **版本号**：在 claims 放 `ver`，数据库维护用户 token_version，变更即全部失效

### 📝 笔试题 10-2：401 和 403 的区别？

- **401**：未认证（Authentication 失败）。客户端该去登录或刷新 token
- **403**：已认证，但权限不足（Authorization 失败）。换账号或申请权限

---

## 11. 幂等性、并发与一致性

### 11.1 幂等性

非幂等操作（POST 创建/转账）在**网络重试**时会重复执行。防御：

#### Idempotency-Key 头（Stripe 风格）

```
POST /payments
Idempotency-Key: 7b2f... (UUID)
Content-Type: application/json
```

服务端：

- 对 `(API key, Idempotency-Key)` 做 upsert 记录
- 首次处理并保存结果；重试则直接返回**之前的结果**
- 存储有效期按业务（通常 24h）

### 11.2 乐观锁：ETag / If-Match

```
# 读取
GET /users/42
200 OK
ETag: "W/\"v7\""

{ "id": 42, "name": "Alice", "email": "a@x.com" }

# 更新
PUT /users/42
If-Match: "W/\"v7\""
{ ... }

# 若版本已变 → 412 Precondition Failed
```

### 11.3 并发写冲突

- **乐观锁**：ETag / version 字段；冲突返回 409 或 412
- **悲观锁**：数据库行锁（`SELECT ... FOR UPDATE`），接口层尽量避免长事务

### 11.4 长任务与异步

```
POST /reports
202 Accepted
Location: /tasks/abc

# 轮询
GET /tasks/abc
200 OK
{ "status": "running", "progress": 60, "result_url": null }

# 完成
{ "status": "succeeded", "result_url": "/reports/abc" }
```

或推送：Webhook、SSE、WebSocket。

### 📝 笔试题 11-1：支付接口如何防重复提交？

- 客户端：按钮禁用 + 生成 `Idempotency-Key` 跟随请求
- 服务端：以 `(user, idempotency_key)` 查表；命中返回既有结果；否则进事务处理并落库
- 另加**业务幂等键**（如 order_id）作为二级防护

---

## 12. 缓存与性能

### 12.1 HTTP 缓存头

- **强缓存**：`Cache-Control`（优先）、`Expires`
  - `Cache-Control: public, max-age=3600`
  - `Cache-Control: private, no-cache` 允许存储但每次协商
  - `Cache-Control: no-store` 完全禁用
- **协商缓存**：`ETag` / `If-None-Match`，`Last-Modified` / `If-Modified-Since`

命中协商缓存返回 `304 Not Modified`，无响应体。

### 12.2 如何选择

- 静态资源 / 列表数据：`Cache-Control: public, max-age=N` + 版本化 URL（`/logo.v3.png`）
- 用户相关数据：`private`，短 TTL
- 敏感数据：`no-store`

### 12.3 API 缓存层次

```
Client → CDN → API Gateway → Service Cache → DB
         ↑
         边缘缓存（全局加速）
```

关键字：**缓存一致性、失效策略、回源风暴**。常用：Cache-Aside、Read-Through、Write-Through、Write-Behind。

### 12.4 批量与聚合

减少请求次数：

- **批量读**：`GET /users?ids=1,2,3`
- **批量写**：`POST /users/batch { "items": [...] }`
- **字段裁剪**：`?fields=id,name`
- **嵌入关联**：`?include=orders`（JSON:API 风格）

### 12.5 压缩与协议

- `Accept-Encoding: gzip, br` → 服务端 `Content-Encoding: br`
- 开启 HTTP/2 多路复用，大幅减少 head-of-line blocking
- 长连接、TLS 会话复用

### 📝 笔试题 12-1：ETag 与 Last-Modified 的区别？

- **Last-Modified**：秒级精度，可能丢失"1 秒内多次修改"
- **ETag**：内容指纹（弱/强），精度与语义更强
- 现代 API 优先 ETag；两者可共存，ETag 优先

---

## 13. 安全最佳实践

### 13.1 传输与存储

- **强制 HTTPS**：HSTS `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- **TLS 1.2+**，禁用 SSLv3/TLS1.0/1.1
- 证书用正规 CA 或企业自建 PKI

### 13.2 输入校验与输出编码

- 白名单校验（类型、长度、枚举、范围、正则）
- 永远把用户输入当不可信
- 防注入：参数化查询、ORM，拒绝字符串拼 SQL
- 防 XSS：输出时按上下文转义；JSON 响应设 `X-Content-Type-Options: nosniff`
- 防路径穿越：校验文件名，存白名单目录

### 13.3 认证与会话

- **密码存储**：bcrypt/argon2 + 每用户 salt；禁止 MD5/SHA1
- **Token 不放 URL**：用 Header，避免进日志/referer
- **CSRF**：对 Cookie-based 会话用 `SameSite=Lax/Strict` + CSRF Token
- **Cookie** 必设 `Secure; HttpOnly; SameSite`

### 13.4 CORS

精准配置 `Access-Control-Allow-Origin`，避免 `*` 配合凭证。

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET,POST,PUT,DELETE
Access-Control-Allow-Headers: Authorization,Content-Type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

### 13.5 防滥用

- **限流**：按 IP / 用户 / API key 做令牌桶；429 + `Retry-After`
- **熔断**：下游失败率超阈值快速失败
- **WAF**：阻断常见注入/扫描
- **审计日志**：关键操作记录 who/what/when

### 13.6 常见响应头

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: no-referrer
Permissions-Policy: geolocation=(), microphone=()
```

### 13.7 数据保护

- 最小化暴露字段（绝不直接回传密码哈希、内部 ID、token）
- 脱敏（手机号 `138****1234`）
- 日志脱敏：卡号、密码、token
- 合规：GDPR、个人信息保护法的访问与删除权

### 📝 笔试题 13-1：如何防止接口被恶意爬取？

多层组合：

1. **强制认证** + 细粒度权限
2. **限流**：IP + 账号维度
3. **验证码/行为校验**：可疑流量降级
4. **签名/时间戳**：请求签名防重放
5. **WAF/风控**：特征识别、异常检测
6. **审计 + 告警**：异常模式实时告警

---

## 14. HATEOAS 与超媒体

### 14.1 什么是 HATEOAS

Hypermedia as the Engine of Application State：**服务端在响应里告诉客户端"下一步可以做什么"**，通过链接驱动状态转移。客户端不需要硬编码 URL 和流程。

```json
{
  "id": 100,
  "status": "paid",
  "_links": {
    "self":   { "href": "/orders/100" },
    "refund": { "href": "/orders/100/refund", "method": "POST" },
    "ship":   { "href": "/orders/100/ship",   "method": "POST" }
  }
}
```

### 14.2 超媒体规范

- **HAL**（`application/hal+json`）：轻量，带 `_links`、`_embedded`
- **JSON:API**（`application/vnd.api+json`）：规定了资源、关系、分页、错误统一格式
- **Siren**：含 actions 描述，更接近"超媒体驱动"

### 14.3 现实评估

HATEOAS 理论上解耦客户端与服务端，实际落地不多：

- 客户端通常针对 API 编码，链接发现带来的复杂性收益小
- **RPC 风格** + OpenAPI 文档已能满足大多数场景

工程折中：在**资源详情**中返回关联资源的 URI，便于客户端组合。

---

## 15. API 文档与契约：OpenAPI 与 Swagger UI

### 15.1 OpenAPI（前 Swagger）

OpenAPI 3.x 是 API 描述的事实标准，YAML/JSON 编写，可驱动：

- 文档生成（Swagger UI、Redoc）
- 客户端/服务端 SDK 生成（OpenAPI Generator）
- Mock Server
- 合同测试

### 15.2 最小示例

```yaml
openapi: 3.0.3
info:
  title: Example API
  version: "1.0.0"
servers:
  - url: https://api.example.com/v1
paths:
  /users/{id}:
    get:
      summary: Get user by id
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: integer }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema: { $ref: "#/components/schemas/User" }
        "404":
          description: Not Found
components:
  schemas:
    User:
      type: object
      required: [id, name]
      properties:
        id:   { type: integer, example: 42 }
        name: { type: string, example: "Alice" }
        email:{ type: string, format: email }
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - bearerAuth: []
```

### 15.3 工程化

- **代码优先**：注解/代码生成 spec（Spring `springdoc`，FastAPI 自带）
- **Spec 优先**：spec 是契约，代码按 spec 实现（推荐跨团队协作）
- **CI 校验**：pr 中比较新旧 spec，防止破坏性变更未升版
- **Lint**：Spectral 等按团队规范自动检查

### 15.4 文档之外

- **Changelog**：版本变更透明
- **示例请求/响应**：含认证、错误
- **Postman / Insomnia collection**：便于消费者试用
- **SDK**：官方/生成 SDK 显著提升采用

### 15.5 Swagger UI 深入

#### 15.5.1 生态关系

"Swagger" 现在指的是一整个工具链，核心组件：

| 组件 | 作用 |
|------|------|
| **OpenAPI Specification (OAS)** | API 描述规范，YAML/JSON；原名 Swagger 2.0，2016 年捐献给 Linux 基金会后更名 OpenAPI 3.x |
| **Swagger Editor** | 浏览器端的 spec 编辑器，带实时校验和预览 |
| **Swagger UI** | 把 OpenAPI spec 渲染成交互式文档网页，**可直接发请求** |
| **Swagger Codegen / OpenAPI Generator** | 从 spec 生成 30+ 语言的 SDK 和服务端骨架 |
| **Swagger Hub** | SmartBear 的托管协作平台 |

**Swagger UI 是"可交互的 HTML 文档"**，不是服务端代码，不负责实现业务，也不负责鉴权——它只是**按 spec 渲染**。

#### 15.5.2 集成方式

**方式 A：各框架内建/社区集成（最常用）**

| 技术栈 | 集成方案 | 默认访问路径 |
|--------|----------|--------------|
| Spring Boot | `springdoc-openapi-starter-webmvc-ui` | `/swagger-ui/index.html` |
| FastAPI (Python) | 内建 | `/docs`（Swagger UI）、`/redoc` |
| Flask | `flasgger` / `flask-smorest` | `/apidocs` |
| Django REST Framework | `drf-spectacular` | `/api/schema/swagger-ui/` |
| Express (Node) | `swagger-ui-express` + `swagger-jsdoc` | 自定义（常用 `/api-docs`） |
| NestJS | `@nestjs/swagger` | `/api` |
| Go (Gin/Echo) | `swaggo/swag` + `gin-swagger` | `/swagger/index.html` |
| .NET Core | `Swashbuckle.AspNetCore` | `/swagger` |

**方式 B：独立部署**

下载 Swagger UI 静态资源（`dist/`），用 Nginx/静态托管，指向 spec URL：

```html
<!-- dist/index.html 的关键初始化 -->
<script>
  window.onload = () => {
    SwaggerUIBundle({
      url: "https://api.example.com/openapi.json",
      dom_id: "#swagger-ui",
      deepLinking: true,
      presets: [SwaggerUIBundle.presets.apis, SwaggerUIStandalonePreset],
      layout: "StandaloneLayout",
      tryItOutEnabled: true,
      persistAuthorization: true
    });
  };
</script>
```

也可用官方 Docker 镜像一行启动：

```bash
docker run -p 8080:8080 \
  -e SWAGGER_JSON_URL=https://api.example.com/openapi.json \
  swaggerapi/swagger-ui
```

**方式 C：Swagger Editor 开发时**

```bash
docker run -p 8081:8080 swaggerapi/swagger-editor
```

写完导出 `openapi.yaml` 给后端和前端双向消费。

#### 15.5.3 Spring Boot 示例

`pom.xml`：

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.3.0</version>
</dependency>
```

`application.yml`：

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    tags-sorter: alpha
    operations-sorter: alpha
    persist-authorization: true
```

Controller 注解：

```java
@Tag(name = "Users", description = "用户管理")
@RestController
@RequestMapping("/users")
public class UserController {

    @Operation(summary = "获取用户", description = "按 ID 查询用户")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "404", description = "不存在")
    })
    @GetMapping("/{id}")
    public UserDto get(@Parameter(description = "用户ID", example = "42")
                       @PathVariable Long id) { ... }
}
```

访问 `http://localhost:8080/swagger-ui.html`，Swagger UI 会自动拉取 `/v3/api-docs` 并渲染。

#### 15.5.4 FastAPI 示例

FastAPI **开箱即用**：声明 pydantic 模型与路径函数，spec 与 UI 自动生成：

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Example API", version="1.0.0",
              docs_url="/docs", redoc_url="/redoc",
              openapi_url="/openapi.json")

class User(BaseModel):
    id: int = Field(..., example=42)
    name: str = Field(..., example="Alice")

@app.get("/users/{id}", response_model=User, tags=["Users"],
         summary="获取用户", responses={404: {"description": "不存在"}})
def get_user(id: int):
    ...
```

访问 `/docs` 即 Swagger UI，`/redoc` 是更阅读友好的 ReDoc 文档。

#### 15.5.5 "Try it out" 交互细节

Swagger UI 最大价值是**在浏览器直接发请求**：

1. 展开端点 → 点击 **Try it out**
2. 填写 path/query/body 参数
3. 点击 **Execute**
4. 看到 **cURL 命令、请求 URL、响应状态、响应体、响应头**

关键机制：

- 请求从**浏览器**直接发往 `servers` 中定义的 URL
- 因此后端必须处理 **CORS**（允许 Swagger UI 所在 origin）
- 若 Swagger UI 与 API 同源部署，可以绕开 CORS

#### 15.5.6 认证接入

在 spec 声明安全方案，UI 会出现 **Authorize** 按钮：

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/authorize
          tokenUrl: https://auth.example.com/token
          scopes:
            read: Read access
            write: Write access

security:
  - bearerAuth: []
```

点击 Authorize 填入 token 后，后续 `Try it out` 会自动带 `Authorization: Bearer xxx`。

`persistAuthorization: true` 可让 token 在刷新后仍保留（开发体验更好；生产要评估是否合规）。

#### 15.5.7 常用配置选项

Swagger UI 构造参数（前端传入 `SwaggerUIBundle({...})` 或框架配置等价项）：

| 参数 | 作用 |
|------|------|
| `url` / `urls` | 指向单个或多个 spec |
| `deepLinking` | 允许 `#/operation` 锚点直接定位端点 |
| `tryItOutEnabled` | 启用交互请求（可全局开关） |
| `filter` | 顶部过滤搜索框 |
| `displayRequestDuration` | 显示请求耗时 |
| `docExpansion` | `none` / `list` / `full`，默认展开级别 |
| `defaultModelsExpandDepth` | 模型区展开深度，`-1` 隐藏 |
| `defaultModelExpandDepth` | 单模型展开深度 |
| `supportedSubmitMethods` | 限制可以 Try 的方法，如 `["get"]` 禁写 |
| `persistAuthorization` | 刷新后保留 token |
| `syntaxHighlight.theme` | 代码高亮主题 |
| `requestInterceptor` / `responseInterceptor` | JS 钩子改写请求/响应 |
| `oauth2RedirectUrl` | OAuth2 回调 |

#### 15.5.8 多 spec / 多版本切换

```javascript
SwaggerUIBundle({
  urls: [
    { url: "/openapi-v1.json", name: "v1" },
    { url: "/openapi-v2.json", name: "v2" }
  ],
  "urls.primaryName": "v2"
});
```

适合网关或多服务聚合。

#### 15.5.9 Swagger UI vs ReDoc vs Stoplight Elements

| 工具 | 强项 | 弱项 |
|------|------|------|
| Swagger UI | 可交互调试（Try it out） | 文档阅读体验一般 |
| ReDoc | 阅读型文档美观，三栏布局 | 默认不能直接发请求 |
| Stoplight Elements | 介于两者之间，可嵌入 | 生态相对小 |

实践中常见组合：`/docs` 给 Swagger UI（调试），`/redoc` 给 ReDoc（对外阅读文档）。

#### 15.5.10 生产环境注意事项

- **是否暴露**：外网 API 可选择开启（提升开发者体验）或只在内网/灰度开；敏感内部系统通常关闭
- **访问控制**：为 `/swagger-ui` 和 `/openapi.json` 加鉴权或 IP 白名单
- **隐藏内部端点**：通过 `@Hidden`（springdoc）/ `include_in_schema=False`（FastAPI）/ tag 分组隐藏管理类接口
- **版本冻结**：构建时把 spec 落为版本化文件，供稳定外部引用
- **CORS 与 CSRF**：Try it out 经过浏览器发送，需服务端允许相应 origin
- **spec 大小**：端点数量大时文档加载慢，按 tag 拆分或启用 lazy 渲染

#### 15.5.11 常见坑

1. **Swagger UI 显示正常但 Try it out 总是 CORS 失败**
   - Swagger UI 页面在 `https://docs.example.com`，API 在 `https://api.example.com`
   - 方案：在 API 网关放行 `docs.example.com` 来源，或 Swagger UI 同源部署
2. **Token 没带上**
   - `securitySchemes` 未在 `security` 或 operation 层声明
   - 点击 Authorize 后没保存（未配 `persistAuthorization`，刷新即失效）
3. **路径不同或 spec 过期**
   - 后端热重载但 Swagger UI 有 HTTP 缓存；开发环境禁用缓存或加随机 query
4. **历史 Swagger 2.0 与 OpenAPI 3.0 混用**
   - 规范不同，工具要匹配：Swagger UI 3.x+ 同时支持两者，但生成器要选对
5. **示例数据泄漏**
   - 真实 token/生产数据被写进 `example`，随 spec 发布到外网
6. **PATCH/PUT 请求体示例为空**
   - schema 中漏写 `example` 或 `default`，Try it out 体验差

#### 15.5.12 自动化与 CI

- **spec 校验**：`swagger-cli validate openapi.yaml` 或 `spectral lint`
- **兼容性检查**：`openapi-diff` / `oasdiff` 比较新旧版本，CI 中拦截破坏性变更
- **SDK 生成**：`openapi-generator-cli generate -i openapi.yaml -g typescript-axios -o client/`
- **契约测试**：`Dredd` / `Schemathesis` 按 spec 生成测试用例

### 📝 笔试题 15-1：Swagger 与 OpenAPI 是什么关系？

Swagger 是一套工具链的**品牌名**（Swagger UI / Editor / Codegen 等）。Swagger 2.0 规范在 2016 年捐献给 Linux 基金会后更名为 **OpenAPI Specification**，最新主版本是 **OpenAPI 3.1**。"Swagger 文档"现在一般指"由 Swagger UI 渲染的 OpenAPI 文档"。

### 📝 笔试题 15-2：在 Swagger UI 点 Try it out 一直 401，可能原因？

- 未点 **Authorize** 填入 token
- `securitySchemes` 只声明未在 operation 或全局 `security` 中应用
- Token 过期；`persistAuthorization` 未开启，刷新页面丢失
- API 网关/反代未透传 `Authorization` 头
- Bearer 格式错（忘记 `Bearer ` 前缀，或放错头字段）

### 📝 笔试题 15-3：生产环境是否应该暴露 `/swagger-ui`？

视团队策略：

- **对外开发者平台**：建议暴露，但配合版本化 spec、访问控制、敏感端点隐藏
- **仅内部服务**：通常**关闭**或只在内网可达，避免泄漏系统拓扑
- **一刀切**：用 profile 控制，`prod` 环境禁用 springdoc / 设置 `include_in_schema=False`

---

## 16. 测试、Mock 与工程化

### 16.1 测试分层

- **单元测试**：Controller、Service、Repository
- **集成测试**：跨层 + DB/缓存（`Testcontainers`）
- **契约测试**：基于 OpenAPI 或 Pact，保证消费者—生产者兼容
- **端到端**：Postman Newman、Cypress
- **性能**：k6、wrk、jmeter

### 16.2 Mock

- **服务端未完成时**：Prism、Mockoon、WireMock 按 OpenAPI 返回示例
- **依赖隔离**：单元测试用内存版 / 假客户端

### 16.3 本地开发体验

- `docker-compose up` 拉起依赖（DB、缓存、MQ）
- 热加载（nodemon、air、reflex）
- 统一 lint/format（Prettier、eslint、golangci-lint、ruff/black）
- **一键 seed** 演示数据

### 16.4 CI/CD

- PR 必跑：lint + 单元 + 集成 + OpenAPI 校验
- 灰度/金丝雀发布
- 可观测：日志、指标、链路（OpenTelemetry）

### 16.5 监控与告警

- **RED**（Rate/Errors/Duration）按端点统计
- 关键 SLO：成功率、P99 延迟
- 结构化日志 + trace_id 串联

---

## 17. REST 与 GraphQL / gRPC 对比

| 维度 | REST | GraphQL | gRPC |
|------|------|---------|------|
| 传输 | HTTP/1.1, HTTP/2 | 通常 HTTP POST | HTTP/2 |
| 序列化 | JSON 为主 | JSON | Protobuf（二进制） |
| 接口定义 | URI + 方法 + OpenAPI | Schema | .proto |
| 取数 | 固定响应，多次请求拼装 | 单次查询按需 | 一元/流 |
| 缓存 | HTTP 缓存天然 | 复杂（需 persisted queries） | 依赖应用实现 |
| 浏览器兼容 | ✅ | ✅ | 需 grpc-web |
| 强类型 SDK | OpenAPI 生成 | Codegen | 原生强 |
| 学习曲线 | 低 | 中 | 中 |
| 典型场景 | 对外开放 API、CRUD | BFF、聚合、移动端省流量 | 内部微服务、流式 |

**建议**：

- 外部/公共 API → **REST + OpenAPI**
- 服务内部 / 性能敏感 → **gRPC**
- 多端聚合 / UI 数据拼装 → **GraphQL（BFF）**
- 混合栈中它们可共存

---

## 18. 常见反模式

❌ **动词入 URI**：`/getUser`、`/deleteUserById`
❌ **用 POST 做一切**：全 200 + `code` 判错
❌ **用 200 返回业务错误**：破坏 HTTP 语义
❌ **无版本化或过度版本化**：要么无法演进，要么版本失控
❌ **把数据库表直接暴露为接口**：耦合存储与 API
❌ **PATCH 传全量覆盖**：违反 PATCH 语义
❌ **分页信息放进 data 数组**：`[{meta}, {item}, ...]`
❌ **时间戳用本地时区**：混乱且难调试
❌ **id 自增暴露**：可被枚举，考虑 UUID
❌ **日志里记 token/密码**：合规与安全风险
❌ **缺乏幂等**：网络抖动导致重复下单/扣款
❌ **错误信息既不能机读也不能人读**：既无 code 也无可读 message

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 下列哪个方法**不是**幂等的？
A. GET  B. PUT  C. DELETE  D. POST

<details><summary>答案</summary>D。</details>

**Q2** 创建用户成功，最合理的响应是？
A. 200 OK
B. 201 Created + Location
C. 204 No Content
D. 202 Accepted

<details><summary>答案</summary>B。</details>

**Q3** 客户端带无效 token 访问受保护资源，应返回？
A. 400  B. 401  C. 403  D. 404

<details><summary>答案</summary>B。</details>

**Q4** 乐观锁冲突时最合适的状态码是？
A. 400  B. 403  C. 409 或 412  D. 500

<details><summary>答案</summary>C。</details>

**Q5** 为避免深度分页性能问题，最佳做法是？
A. 扩大数据库缓存  B. 键集/游标分页  C. 增大 page_size  D. 前端节流

<details><summary>答案</summary>B。</details>

**Q6** 以下哪个不是合理的 URI？
A. `/users/42/orders`
B. `/users/42/orders/100`
C. `/getUserOrders?userId=42`
D. `/orders?user_id=42&status=paid`

<details><summary>答案</summary>C。动词入 URI。</details>

**Q7** PUT 和 PATCH 的根本区别？
A. PUT 更快
B. PATCH 是非幂等的
C. PUT 整体替换，PATCH 局部更新
D. PATCH 必须用 JSON Patch 格式

<details><summary>答案</summary>C。</details>

**Q8** 下列哪个头字段常用于条件更新（乐观锁）？
A. Authorization  B. If-Match  C. Accept-Language  D. Retry-After

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. REST 必须使用 JSON。 ❌（与媒体类型无关）
2. 401 表示已认证但无权限。 ❌（那是 403）
3. PATCH 必定幂等。 ❌（取决于实现）
4. DELETE 第二次调用若资源不存在应当返回错误。 ❌（常见做法仍返回 2xx/404，视策略）
5. 版本加在 URI 中违反了 REST 的纯粹性但工程实用。 ✅
6. 无状态意味着服务器不能有数据库。 ❌（指不保存会话）
7. HATEOAS 是 Richardson L3 的标志。 ✅
8. 429 Too Many Requests 应带 Retry-After 头。 ✅

### 19.3 简答题

**Q1** 设计一个"收藏文章"接口，讨论 URI、方法和幂等性。

- **资源化**：`PUT /users/{uid}/favorites/{articleId}` 收藏，`DELETE` 取消
- 幂等：重复 PUT/DELETE 效果一致，安全
- 返回 `204 No Content` 或 `200 OK + 资源`
- 替代：`POST /users/{uid}/favorites { "article_id": ... }`；用 `Idempotency-Key` 保证重试安全

**Q2** 解释 ETag 如何实现乐观并发控制。

- 服务端在读响应中带 `ETag`（资源的版本指纹）
- 客户端更新时在 `If-Match` 中带回
- 服务端对比当前 ETag：一致则更新并返回新 ETag，不一致则 `412 Precondition Failed`
- 客户端处理冲突：重新拉取 + 合并 + 重试

**Q3** 如何做向后兼容的字段演进？

- 加字段：默认值 + 可选，老客户端忽略
- 改类型：**不要改**，改名并并行存在，逐步迁移
- 删字段：先标 `deprecated`，经过多版本宽限期后移除
- 改语义：必须升版本
- 配合 OpenAPI diff 工具在 CI 中把关

**Q4** 在高并发抢购场景下，如何设计幂等 + 防超卖？

- 客户端按钮防抖 + `Idempotency-Key`
- 服务端：`(user_id, activity_id)` 唯一索引，或 Redis `SETNX` 去重
- 库存扣减：Redis 原子操作（lua / Decr）或数据库乐观锁（`UPDATE ... WHERE stock >= 1`）
- 异步化：创建"预占"订单，`202 Accepted` + 结果查询端点
- 限流 + 熔断 + 排队（令牌桶/漏桶）

### 19.4 设计题

**Q1** 为一个"评论系统"设计 API（支持层级回复、点赞、举报、软删除）。

核心端点：

```
POST   /articles/{aid}/comments                创建
GET    /articles/{aid}/comments?cursor=&limit= 列表（游标分页）
GET    /comments/{cid}                         详情
PATCH  /comments/{cid}                         编辑
DELETE /comments/{cid}                         软删除
POST   /comments/{cid}/replies                 回复
PUT    /comments/{cid}/likes/me                点赞（幂等）
DELETE /comments/{cid}/likes/me                取消赞
POST   /comments/{cid}/reports                 举报
```

要点：

- 树形结构：`parent_id` 或 materialized path
- 分页：按 `created_at DESC` + 游标
- 权限：作者可编辑/删除，管理员可硬删
- 反垃圾：限流、内容审核
- 计数优化：Redis 维护 `like_count`

**Q2** 设计一个"任务异步处理"的 API 流程。

```
POST /exports             202 Accepted + { "task_id": "..." }
GET  /exports/{task_id}   200 { "status": "pending|running|succeeded|failed",
                                "progress": 60, "result_url": "..." }
DELETE /exports/{task_id} 204  (取消)
```

- 任务进度可轮询或 Webhook/SSE 推送
- 结果下载带有有效期的预签名 URL
- 失败提供 `error.code`/`error.message` + trace_id

**Q3** 请为下列需求写出 URI 和方法：
"管理员封禁某用户 24 小时"

两种合理写法：

- **动作子资源**：`POST /users/{id}/bans { "duration": "PT24H", "reason": "..." }`
- **状态字段**：`PATCH /users/{id} { "status": "banned", "banned_until": "..." }`

前者审计日志更清晰，业务更易追踪；后者简单直接。

---

## 📚 学习与落地建议
0.**动手实践**：使用一门编程语言写一个简单的web service，设计并实现GET, POST, PUT, PATCH, DELETE等APIs，并提供相应的swagger ui，能在本机跑起来。
1. **从 L2 起步**：资源建模 + HTTP 方法/状态码，覆盖 80% 需求
2. **一致性优先于完美**：命名、错误体、分页风格在团队内统一
3. **Spec 驱动**：OpenAPI 作为契约，前后端并行开发、自动校验
4. **幂等与可观测是底线**：重试、链路追踪、结构化日志
5. **读权威资料**：
   - RFC 7231/7232/7234（HTTP 语义/条件请求/缓存）
   - RFC 7807（Problem Details）
   - RFC 6749/OIDC 规范
   - JSON:API、HAL、Google API Design Guide、Zalando RESTful API Guidelines
6. **参考业界 API**：GitHub、Stripe、Twilio、AWS API Gateway 都是优秀样板

> 祝你的 API 设计清晰、演进从容。
