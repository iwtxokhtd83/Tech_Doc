# 监控与 Observability 讲义

> 本讲义系统讲解可观测性（Observability）的概念模型与工程实践：**Metrics（Prometheus/Grafana）、Logs（ELK/OpenSearch/Loki）、Traces（OpenTelemetry/Jaeger）**，以及业务服务如何正确集成到这套体系。每章配"知识点 + 笔试题"。
>
> 约定：Kubernetes + 云环境为主要上下文，示例技术栈以 Prometheus 2.x、Grafana 10+、OpenSearch 2.x、OpenTelemetry 1.x 为主。

## 目录

1. [监控 vs 可观测性](#1-监控-vs-可观测性)
2. [Observability 三大支柱](#2-observability-三大支柱)
3. [指标（Metrics）与 Prometheus](#3-指标metrics与-prometheus)
4. [PromQL 入门与告警](#4-promql-入门与告警)
5. [Grafana 仪表盘与面板](#5-grafana-仪表盘与面板)
6. [日志（Logs）与 ELK / OpenSearch](#6-日志logs与-elk--opensearch)
7. [日志采集：Filebeat / Fluent Bit / Vector](#7-日志采集filebeat--fluent-bit--vector)
8. [链路追踪（Traces）与 OpenTelemetry](#8-链路追踪traces与-opentelemetry)
9. [三支柱关联：日志-指标-链路打通](#9-三支柱关联日志-指标-链路打通)
10. [SLI / SLO / SLA / 错误预算](#10-sli--slo--sla--错误预算)
11. [告警策略与值班](#11-告警策略与值班)
12. [Kubernetes 可观测性栈](#12-kubernetes-可观测性栈)
13. [服务集成：Java 示例](#13-服务集成java-示例)
14. [服务集成：Node.js 示例](#14-服务集成nodejs-示例)
15. [服务集成：Python / Go 示例](#15-服务集成python--go-示例)
16. [前端与移动端可观测性](#16-前端与移动端可观测性)
17. [成本、采样与数据治理](#17-成本采样与数据治理)
18. [排障实战剧本](#18-排障实战剧本)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 监控 vs 可观测性

### 1.1 定义

- **监控（Monitoring）**：你**预先知道**要关心的指标和阈值，出异常时被通知
- **可观测性（Observability）**：系统**对外暴露足够信息**，让你在事后能回答**没预想过**的问题

> Monitoring tells you **when** something is wrong; Observability tells you **why**.

### 1.2 为什么单靠监控不够

- 分布式系统故障模式**组合爆炸**，无法穷举阈值
- 新版本上线、灰度、跨区域问题常超出历史经验
- 面向用户的 SLO（延迟、错误率）比单机指标更重要
- 需要**任意维度下钻**：某个 API、某个用户、某个版本、某个地区

### 1.3 可观测性三要素

```
Logs       — "发生了什么"（事件详情）
Metrics    — "有多少、多快"（数值趋势）
Traces     — "哪里、怎么发生"（调用链）
```

三者互补，缺一不可：

- 看 Metrics 发现"订单服务错误率飙升"
- 看 Traces 定位到哪个下游 span 慢/错
- 看 Logs 读到具体异常堆栈与上下文

### 📝 笔试题 1-1：监控和可观测性的区别？

监控是**主动推送"已知"问题**：按预设指标和阈值产生告警。
可观测性是**系统本身具备被外部追问的能力**：通过丰富的 Metrics/Logs/Traces，事后能灵活排查**未预见**的故障模式。
好的监控是好的可观测性的**使用者之一**，而不是全部。

---

## 2. Observability 三大支柱

### 2.1 对比一览

| 维度 | Metrics | Logs | Traces |
|------|---------|------|--------|
| 数据形态 | 时序数值 | 结构化事件 | 跨服务调用树 |
| 存储量 | 小（聚合） | 大 | 中（可采样） |
| 查询速度 | 极快 | 中 | 中 |
| 维度 | 低基数 tag | 任意字段 | trace/span |
| 典型产品 | Prometheus/VictoriaMetrics/Mimir | ELK/OpenSearch/Loki | Jaeger/Tempo/X-Ray |
| 告警友好 | ✅ | 中 | 弱 |
| 根因分析 | 弱 | 强 | 极强 |

### 2.2 黄金信号（Google SRE 四大指标）

- **Latency**：请求延迟
- **Traffic**：请求量
- **Errors**：错误率
- **Saturation**：资源饱和度（"系统还剩多少"）

### 2.3 RED（面向请求/服务）

- **Rate**：每秒请求数
- **Errors**：错误数/率
- **Duration**：耗时（P50/P95/P99）

### 2.4 USE（面向资源）

- **Utilization**：使用率
- **Saturation**：排队长度 / 等待
- **Errors**：错误数

工程上：面向业务用 **RED**，面向机器用 **USE**；结合黄金信号覆盖全局。

### 📝 笔试题 2-1：你只有时间关注几个指标，怎么选？

对每个对外服务至少监控 **RED**：请求速率、错误率、P95/P99 延迟；再加**资源饱和度**（CPU/内存/连接/队列深度）。对依赖（DB、缓存、MQ）用 **USE**。这就是 SRE 推崇的**黄金信号**。

---

## 3. 指标（Metrics）与 Prometheus

### 3.1 Prometheus 简介

- **开源**、**Pull 模式**、**多维时序数据库**
- 通过 HTTP 端点抓取目标的 `/metrics`
- **PromQL** 强大查询
- CNCF 毕业项目，事实标准

### 3.2 架构

```
┌─────────────┐   scrape /metrics   ┌──────────┐
│ Targets     │◀────────────────────│Prometheus│
│ (exporters, │                     │  server  │
│  apps, ...) │                     └────┬─────┘
└─────────────┘                          │ TSDB
                                         ▼
                                   PromQL API
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                       Grafana     Alertmanager      Federation
```

### 3.3 指标类型

- **Counter**：单调递增（请求数、错误数）
- **Gauge**：可增可减（当前连接数、内存使用）
- **Histogram**：分桶计数（延迟分布）；客户端累计，服务端可算 `rate + histogram_quantile`
- **Summary**：客户端直接算分位数；服务端难聚合，**分布式下不推荐**

### 3.4 数据模型

```
http_requests_total{method="GET", path="/api/users", status="200"}  12345
```

- **名称**：`metric_name`
- **Labels**：键值对维度
- **值**：数值
- **时间戳**：自动带

**命名规范（推荐）**：

- `xxx_total`：累计 counter
- `xxx_bytes` / `xxx_seconds`：单位后缀
- `xxx_duration_seconds_bucket`：Histogram

### 3.5 Exporter 家族

自己写不了的系统，由 exporter 代理暴露指标：

- **node_exporter**：主机 CPU/内存/磁盘/网络
- **kube-state-metrics**：K8s 对象状态
- **cAdvisor / kubelet**：容器级
- **blackbox_exporter**：黑盒探测（HTTP/TCP/ICMP）
- **mysqld_exporter / redis_exporter / kafka_exporter / postgres_exporter / nginx_exporter**
- **JMX exporter**：JVM
- **pushgateway**：短任务"push"代理（谨慎使用）

### 3.6 服务发现

Prometheus 支持多种 SD 自动找到目标：

- `kubernetes_sd_configs`：K8s 原生
- `consul_sd_configs` / `nomad_sd_configs`
- `ec2_sd_configs`
- `file_sd_configs`：静态文件
- `dns_sd_configs`

### 3.7 Prometheus 配置示例

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  external_labels:
    cluster: prod-us-east-1

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port,
                        __address__]
        action: replace
        regex: ([^:]+)(?::\d+)?;(.+)
        replacement: $2:$1
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/*.yml
```

Pod 只需打注解即可被发现：

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

### 3.8 长期存储方案

Prometheus 本地 TSDB 通常仅保留 15-30 天。长期存储选项：

- **Thanos**：对象存储（S3）冷热分层 + 全局查询
- **VictoriaMetrics**：高性能、低成本 TSDB
- **Grafana Mimir**：水平扩展、多租户
- **Amazon Managed Prometheus (AMP)** / **Google Managed Service for Prometheus**

### 📝 笔试题 3-1：Pull vs Push 有什么优缺点？

| 维度 | Pull（Prometheus） | Push（StatsD、Datadog Agent） |
|------|--------------------|-----------------------------|
| 目标发现 | 需服务发现 | 客户端知道 server |
| 防火墙 | Prom 需能访问目标 | 目标能出网即可 |
| 目标存活 | 自动检测 `up{}` | 无法区分"没推送"与"死了" |
| 临时任务 | 困难（需 Pushgateway） | 天然支持 |
| 多消费方 | 多个 Prom 可同时 pull | 需多 sink |

选 Pull 的核心理由：**便于检测目标存活**与**统一控制抓取频率**。


---

## 4. PromQL 入门与告警

### 4.1 基础选择器

```promql
http_requests_total                                   # 所有系列
http_requests_total{status="500"}                     # 标签过滤
http_requests_total{status=~"5.."}                    # 正则
http_requests_total{status!~"2.."}                    # 排除
http_requests_total[5m]                               # 范围向量（最近 5 分钟）
```

### 4.2 常用函数

```promql
rate(http_requests_total[5m])                         # 每秒增量（仅 counter）
increase(http_requests_total[1h])                     # 1 小时总增
sum(rate(http_requests_total[5m])) by (service)       # 按服务聚合 QPS
sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)                                                      # P95 延迟
avg_over_time(cpu_usage[1h])
max_over_time(queue_length[10m])
```

### 4.3 典型查询范例

```promql
# 每服务 QPS（RED - Rate）
sum by (service) (rate(http_requests_total[5m]))

# 错误率（RED - Errors）
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
 / sum by (service) (rate(http_requests_total[5m]))

# P99 延迟（RED - Duration）
histogram_quantile(0.99,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)

# 可用性（1 - 错误率）
1 - sum(rate(http_requests_total{status=~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))

# CPU 使用率 (node)
100 - avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# 磁盘使用率
100 * (node_filesystem_size_bytes - node_filesystem_free_bytes)
     / node_filesystem_size_bytes

# Pod 重启 5 分钟内次数
sum by (pod, namespace) (increase(kube_pod_container_status_restarts_total[5m]))
```

### 4.4 告警规则

```yaml
groups:
  - name: service-sli
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (service) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "{{ $labels.service }} 5xx error rate > 5%"
          runbook: "https://wiki/runbooks/5xx"
          description: |
            current value = {{ $value | printf \"%.2f\" }}

      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.service }} P99 > 1s"
```

### 4.5 Alertmanager

```yaml
route:
  receiver: default
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h
  routes:
    - match: { severity: critical }
      receiver: pagerduty
    - match: { team: platform }
      receiver: slack-platform

receivers:
  - name: default
    email_configs:
      - to: oncall@example.com
  - name: pagerduty
    pagerduty_configs:
      - service_key: xxx
  - name: slack-platform
    slack_configs:
      - api_url: https://hooks.slack.com/...
        channel: '#alerts-platform'

inhibit_rules:
  - source_match: { severity: 'critical' }
    target_match: { severity: 'warning' }
    equal: ['alertname', 'service']
```

要点：

- **分组（grouping）**：同类告警合并成一条通知
- **抑制（inhibit）**：上游告警压制下游重复告警
- **静默（silence）**：计划变更期间暂停
- **升级（escalation）**：未处理告警自动升级值班

### 📝 笔试题 4-1：为什么告警表达式通常配 `for: 10m`？

避免瞬时抖动触发误报。条件必须**持续满足 `for` 时长**才真正发告警，同时也防止频繁震荡（flapping）。选择 `for` 太短噪音多、太长响应慢，一般 5-15 分钟为宜。

---

## 5. Grafana 仪表盘与面板

### 5.1 核心概念

- **Data Source**：Prometheus、OpenSearch、Loki、CloudWatch、MySQL...
- **Dashboard**：多 Panel 组合
- **Panel**：单个图表（time series / stat / gauge / table / bar / heatmap / logs / traces...）
- **Variable**：下拉筛选（`$service`、`$namespace`）
- **Annotation**：在图上标注事件（发布、故障）
- **Alert**：Grafana 侧告警（与 Prometheus 互补）

### 5.2 一个典型仪表盘布局（以服务为例）

```
[SLO 摘要] [可用性 30d] [错误预算剩余]
[请求速率 RED]  [错误率]  [P50/P95/P99]
[上游依赖延迟]  [DB QPS]   [Redis Hit Rate]
[Pod 数量]     [CPU]      [内存]
[近 100 条错误日志（Loki）] [近 10 条慢调用（Trace）]
```

### 5.3 变量与模板

```
Variable: service
Query: label_values(http_requests_total, service)
Multi-value, Include All option
```

Panel 里用：

```
sum by (pod) (rate(http_requests_total{service="$service"}[5m]))
```

### 5.4 仪表盘即代码（Dashboard as Code）

- **手编 JSON**：可读性差
- **Grafonnet**（Jsonnet）
- **grafana-operator**：K8s CR
- **Terraform provider**
- **Perses**（新一代基于文本）

生产强烈建议**版本化**仪表盘到 Git，避免线上手改丢失。

### 5.5 告警（Grafana Alerting）

Grafana 9+ 的统一告警：

- 支持多数据源（不止 Prom）
- 统一规则编辑
- 通知模板与多通道
- 与 Alertmanager 可集成/替代

### 5.6 常见面板小贴士

- 统一单位（seconds、bytes、percent）
- 分位数用 **heatmap**，比多条百分位线更直观
- 给关键阈值画水平线
- 每个 panel 点进去要能**下钻**（跳 Logs / Traces）
- 合理 `refresh` 间隔，不要全 5 秒，省 Prom 压力

### 📝 笔试题 5-1：如何在一个延迟面板里同时展示多个分位数？

```promql
histogram_quantile(0.50, sum by (le) (rate(req_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le) (rate(req_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(req_duration_seconds_bucket[5m])))
```

在同一面板加 3 个 query，Legend 设为 `P50/P95/P99`。若维度多，建议用 heatmap 面板一图看分布趋势。

---

## 6. 日志（Logs）与 ELK / OpenSearch

### 6.1 ELK / EFK / OpenSearch 生态

- **ELK**：Elasticsearch + Logstash + Kibana
- **EFK**：Elasticsearch + Fluentd/Fluent Bit + Kibana（K8s 更常见）
- **OpenSearch**：AWS 主导的 Elasticsearch fork，**Apache 2.0 协议**，兼容 ES 7.10 API；在 AWS 生态是 ES 的替代
- **Opensearch Dashboards**：对应 Kibana

### 6.2 架构

```
App ── stdout ── Fluent Bit (节点 DaemonSet)
                    │
                    ▼
             (可选: Kafka 缓冲)
                    │
                    ▼
             Elasticsearch / OpenSearch 集群
                    │
                    ▼
             Kibana / OSD (查询、可视化)
```

**生产最佳做法**：应用**只写 stdout/stderr**（结构化 JSON），由采集侧集中处理。

### 6.3 结构化日志

```json
{
  "ts": "2025-01-15T10:00:00.123Z",
  "level": "ERROR",
  "service": "order",
  "env": "prod",
  "pod": "order-7d8c-abc",
  "trace_id": "a1b2c3",
  "span_id": "def456",
  "user_id": 42,
  "order_id": "o-100",
  "event": "payment.failed",
  "reason": "insufficient_balance",
  "msg": "Failed to charge user"
}
```

**必带字段**：`ts`、`level`、`service`、`trace_id`、`event`/`msg`。便于后续按任意维度查询。

### 6.4 索引与生命周期

- **按时间索引**：`logs-orders-2025.01.15`（每天一个）
- **Index Template**：统一 mapping、分片、副本
- **ILM (Index Lifecycle Management)**：Hot / Warm / Cold / Delete 自动迁移
  - Hot：SSD，7 天
  - Warm：HDD，30 天
  - Cold：对象存储 / Snapshot，90+ 天
  - Delete：过期清理

### 6.5 基础查询（Kibana / Dashboards Query Language）

```
service: "order" AND level: "ERROR"
service: "order" AND status >= 500
event: "payment.failed" AND amount > 100
message: "timeout" AND NOT service: "canary"
trace_id: "abc123"          # 按 trace 聚合
```

### 6.6 权衡与替代

| 方案 | 优势 | 劣势 |
|------|------|------|
| **Elasticsearch / OpenSearch** | 强大全文检索、聚合 | 资源消耗大、索引成本高 |
| **Loki**（Grafana） | 按 label + 压缩块，便宜 | 不适合任意字段聚合，检索弱 |
| **CloudWatch Logs** | 免运维，AWS 集成 | 查询性能一般，贵 |
| **Datadog / New Relic** | 一站式 | 商业成本 |
| **Splunk** | 企业级强大 | 最贵 |

**经验**：流量大、日志体量爆炸时，考虑**指标替代日志**（把计数做成 metric）、**采样**、**冷热分离**。

### 📝 笔试题 6-1：应用日志应直接写 Elasticsearch 吗？

**不应该**。原因：

- 同步写远端 ES 会被网络故障拖住业务
- 对 ES 集群形成不可控的写入压力
- 无法集中过滤/脱敏/路由

**标准做法**：应用写 stdout（结构化 JSON）→ 节点 Agent（Fluent Bit/Filebeat）→ 可选缓冲（Kafka）→ ES/OpenSearch。应用与存储**完全解耦**。

---

## 7. 日志采集：Filebeat / Fluent Bit / Vector

### 7.1 采集器选型

| 工具 | 开发方 | 特点 |
|------|--------|------|
| **Filebeat** | Elastic | 轻量、与 ES 原生 |
| **Fluentd** | CNCF | 插件丰富，Ruby 实现（较重） |
| **Fluent Bit** | CNCF | C 写，极轻量，K8s 主流 |
| **Vector** | Datadog | Rust 写，性能好，配置友好 |
| **Logstash** | Elastic | 功能全，**较重**，少在节点 |
| **Promtail** | Grafana | 配套 Loki |

### 7.2 Fluent Bit 典型配置（K8s DaemonSet）

```ini
[SERVICE]
    Flush        5
    Log_Level    info
    Parsers_File parsers.conf

[INPUT]
    Name             tail
    Tag              kube.*
    Path             /var/log/containers/*.log
    Parser           cri
    DB               /var/log/flb_kube.db
    Mem_Buf_Limit    10MB
    Skip_Long_Lines  On
    Refresh_Interval 5

[FILTER]
    Name             kubernetes
    Match            kube.*
    Kube_URL         https://kubernetes.default.svc:443
    Merge_Log        On
    Keep_Log         Off
    K8S-Logging.Parser  On
    Annotations      Off

[FILTER]
    Name    record_modifier
    Match   *
    Record  env prod
    Record  cluster eks-prod

[OUTPUT]
    Name            es
    Match           *
    Host            opensearch.example.com
    Port            443
    TLS             On
    Suppress_Type_Name On
    Logstash_Format On
    Logstash_Prefix logs
    Retry_Limit     5
```

### 7.3 采集端常见职责

- **解析**：多行合并（Java 堆栈）、正则解析、JSON 自动识别
- **丰富**：补上 `pod`、`namespace`、`env`、`cluster`、`region`
- **脱敏**：正则替换 token/密码/身份证
- **路由**：按日志类型送不同后端（ES 存业务、CloudWatch 存审计）
- **缓冲**：磁盘队列应对下游短暂不可用

### 7.4 日志采样与过滤

高日志量场景，源头就要控制：

- **分级采样**：DEBUG 100%、INFO 10%、ERROR 100%
- **路径过滤**：健康检查、静态资源不记录
- **大体积字段截断**：长 body/栈截断

### 7.5 弹性与背压

- Agent 必须有**本地缓冲 + 磁盘队列**
- 下游（Kafka/ES）挂时丢弃策略要明确
- **日志不能阻塞业务**：应用写 stdout 是关键（OS 管道缓冲足够）

### 📝 笔试题 7-1：K8s 中日志采集为什么用 DaemonSet？

- 每节点跑一个 Agent，挂载 `/var/log/containers` 读取所有 Pod 日志
- 资源占用可预期（每节点一份），不随 Pod 扩缩容
- 减少配置复杂度（应用无感，写 stdout 即可）
- 失败范围小（单节点挂了不影响别的节点）

替代模式是 Sidecar（每 Pod 一个 agent），隔离更好但成本高，一般只在特殊场景用。

---

## 8. 链路追踪（Traces）与 OpenTelemetry

### 8.1 基础概念

- **Trace**：一次用户请求的完整路径
- **Span**：一次内部/RPC 调用，带起止时间、属性、事件
- **Trace Context**：在服务间传递，通常以 `traceparent` HTTP header（W3C 标准）

### 8.2 为什么需要

微服务一个请求涉及多个服务，纯日志无法还原完整顺序与耗时分布。Trace 让你看到：

```
[API Gateway]  ─────────────────── 200ms
  └─ [Order Svc]              ── 180ms
      ├─ [User Svc]      ── 30ms
      ├─ [Inventory Svc] ── 50ms
      └─ [Payment Svc]   ── 90ms
          └─ [DB]        ── 80ms    ← 瓶颈
```

### 8.3 OpenTelemetry（事实标准）

OpenTelemetry（OTel）统一了 Metrics + Logs + Traces 的**数据模型、SDK、Collector**。

```
App SDK ──OTLP──▶ OTel Collector ──▶ Jaeger / Tempo / X-Ray
                              ──▶ Prometheus / Cortex
                              ──▶ Loki / OpenSearch
```

特点：

- **厂商中立**：一次埋点，换后端无痛
- **自动注入**（auto-instrumentation）：很多语言只需加 agent
- **统一上下文**：trace_id 贯穿三支柱

### 8.4 采样

- **Head-based**：入口决定（比如 10% 采样），简单但可能漏关键错
- **Tail-based**：收集所有 span 再决定（出错的一定保留），更贵更全，由 Collector 实现
- **Adaptive**：按服务吞吐动态调整

### 8.5 Collector（OTel Collector）

中心化处理器，可部署成 Agent（边车/节点）或 Gateway（集群内独立）：

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
      http: { endpoint: 0.0.0.0:4318 }

processors:
  batch: {}
  memory_limiter: { check_interval: 5s, limit_percentage: 75 }
  resourcedetection: { detectors: [env, eks, ec2] }
  tail_sampling:
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,   type: latency,     latency: { threshold_ms: 500 } }
      - { name: random, type: probabilistic, probabilistic: { sampling_percentage: 10 } }

exporters:
  otlp/jaeger: { endpoint: jaeger:4317, tls: { insecure: true } }
  prometheus:  { endpoint: 0.0.0.0:9464 }
  loki:        { endpoint: http://loki:3100/loki/api/v1/push }

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters:  [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters:  [prometheus]
```

### 8.6 后端产品

- **Jaeger**：CNCF，成熟
- **Grafana Tempo**：对象存储便宜、Grafana 集成好
- **SkyWalking**：国产活跃、含 APM
- **AWS X-Ray** / **GCP Cloud Trace** / **Azure Monitor**
- **Zipkin**：老牌

### 📝 笔试题 8-1：Trace context 如何在微服务间传递？

通过标准 HTTP header（或 gRPC metadata）：

```
traceparent: 00-<trace_id>-<parent_span_id>-01
tracestate:  vendor-specific
baggage:     user_id=42,tenant=acme
```

OpenTelemetry SDK 会在发出请求时自动注入（instrumented HTTP client）、收到请求时自动提取，从而自然串起整条链路。消息队列场景则注入到消息 header。

---

## 9. 三支柱关联：日志-指标-链路打通

### 9.1 关联的核心：统一标识

每条数据都应包含：

- `service`：哪个服务
- `env`：环境
- `trace_id` / `span_id`：哪次请求 / 哪个步骤
- `version`：哪个版本
- 业务键（可选）：`user_id` / `tenant` / `request_id`

### 9.2 典型关联路径（Grafana 内部联动）

```
Grafana 指标面板异常
   │
   ├─▶ 跳转对应时间段的 Logs（Loki/OpenSearch）
   │      └─ 过滤出含 trace_id 的错误日志
   │
   └─▶ 点日志行里的 trace_id，跳转 Trace 视图
          └─ 看到具体 span，再跳回对应 span 的日志
```

Grafana 的 **"exemplars"**、**"data links"**、**"correlations"** 特性就是为此而生。

### 9.3 Exemplars

Prometheus 2.x 支持 exemplar：在 histogram bucket 附带一个代表性样本（含 trace_id）。Grafana 会在图上显示小点，点击即跳对应 Trace。OTel Collector 可把采样中的 trace 与 metric 关联。

### 9.4 集成示例

```
┌─── App (埋点) ───┐
│  OTel SDK:        │
│    metrics → Prom │
│    logs   → stdout│
│    traces → OTLP  │
└─────────┬────────┘
          │
          ▼
  OTel Collector
    │   │   │
    ▼   ▼   ▼
  Prom  Loki Tempo/Jaeger
    └────┼────┘
         ▼
      Grafana (统一展示 + 跳转)
```

### 📝 笔试题 9-1：trace_id 一定要写进日志吗？

**是**。否则你虽然有三支柱，却无法将它们关联起来。做法：

- 日志框架里读取 MDC / context，把 `trace_id`、`span_id` 作为结构化字段输出
- 出站请求透传（HTTP header / gRPC metadata）
- OTel 的 logs SDK 会自动注入

---

## 10. SLI / SLO / SLA / 错误预算

### 10.1 定义

- **SLI (Indicator)**：指标本身（成功率、P99 延迟）
- **SLO (Objective)**：内部目标（99.9% 月成功率）
- **SLA (Agreement)**：对外承诺（违约赔偿）
- **Error Budget**：`1 - SLO`，允许的"失败预算"

### 10.2 为什么需要 SLO

- 让**技术与业务对齐**：发布/稳定怎么权衡有定量依据
- 告警从"单机 CPU 90%"升级为"用户成功率低于 SLO"
- 形成**错误预算驱动**的工程文化

### 10.3 典型 SLI 模型

- **Availability**：`好请求 / 总请求`
- **Latency**：`P99 延迟 < 300ms 的比例`
- **Quality**：业务级别（订单成功率）

### 10.4 错误预算窗口

```
月度 SLO 99.9% = 43.2 分钟 downtime / 月
月度 SLO 99.99% = 4.32 分钟 / 月
```

- **燃烧率（Burn Rate）**：当前错误消耗预算的速度
- 快速燃烧 → 紧急告警；慢速燃烧 → 工单级别
- 预算耗尽 → **冻结新功能发布，专注稳定性**

### 10.5 多窗口多燃烧率告警（Google SRE 推荐）

```yaml
- alert: ErrorBudgetBurnFast
  expr: |
    error_budget_burn_rate_1h > 14.4
    and
    error_budget_burn_rate_5m > 14.4
  for: 2m
  labels: { severity: critical }

- alert: ErrorBudgetBurnSlow
  expr: |
    error_budget_burn_rate_6h > 6
    and
    error_budget_burn_rate_30m > 6
  for: 15m
  labels: { severity: warning }
```

数值来源：14.4 意味着 2% 预算 / 1 小时烧 × 14.4 ≈ 30 天预算 / 2 天。

### 📝 笔试题 10-1：为什么比起告警"错误率 > 5%"，SLO 燃烧率告警更好？

- 把**用户体验**和**预算消耗速度**对齐
- 避免"偶发尖峰触发告警 → 值班辛苦又白忙"
- 多窗口（长+短）既能快速响应重大故障，又能避免短抖动误报
- 结合错误预算，可用数据驱动"何时该冻结发布"

---

## 11. 告警策略与值班

### 11.1 告警设计原则

- **基于症状**：用户感知的问题（错误率、延迟）而非单一组件
- **可操作**：每条告警都要能对应明确行动，否则就是噪音
- **分级**：P0 即时电话 / P1 小时级 / P2 次日工时
- **带 runbook**：故障书（检查点、常见原因、操作步骤）
- **定期审计**：长期不触发 → 是否还需要；频繁触发 → 阈值/逻辑有问题

### 11.2 降噪手段

- Alertmanager **grouping + inhibit + silence**
- **多窗口多燃烧率**避免短抖动
- **依赖抑制**：DB 挂了抑制所有上游的"调 DB 失败"告警
- 维护期主动 **silence**
- 告警疲劳是严重问题，**警报越少越严肃越被尊重**

### 11.3 值班（On-Call）文化

- **轮值制**：每周/每两周轮换
- **工具**：PagerDuty / Opsgenie / VictorOps / Grafana OnCall
- **响应时间 SLA**：P0 15 分钟内响应
- **事后复盘（Postmortem）**：无责文化（blameless），总结改进项
- **值班手册**：架构图、登录方式、常用命令、升级联系人

### 11.4 演练与混沌

- 主动制造故障：杀 Pod、断网、填盘
- 工具：**Chaos Mesh**、**LitmusChaos**、**AWS FIS**
- 游戏日（Game Day）：团队一起模拟事故，验证 SLO、runbook、告警

### 📝 笔试题 11-1：团队告警已经非常多，凌晨频繁打扰，怎么办？

1. 审计所有告警：**过去 30 天未 actionable 的告警直接关闭**
2. 改 SLO 燃烧率 + 多窗口告警，减少瞬时尖峰告警
3. 按严重度分路由：非 P0 不电话，只记 ticket
4. 抑制下游连带告警
5. 所有告警补齐 runbook 与负责人
6. 事后复盘所有夜间告警 → 根因消除

---

## 12. Kubernetes 可观测性栈

### 12.1 常见技术栈

- **Metrics**：Prometheus + kube-state-metrics + node-exporter + cAdvisor
  + 长期：Thanos / Mimir / VictoriaMetrics / AMP
- **Logs**：Fluent Bit DaemonSet → ES/OpenSearch/Loki/CloudWatch
- **Traces**：OTel Collector → Jaeger/Tempo/X-Ray
- **Dashboard**：Grafana
- **Alert**：Alertmanager / Grafana Alerting / PagerDuty

### 12.2 kube-prometheus-stack（推荐）

Helm 一键装齐：Prometheus Operator、Alertmanager、Grafana、常用 dashboard 与告警规则。

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f values.yaml
```

### 12.3 ServiceMonitor / PodMonitor

Prometheus Operator 的声明式 CR，让业务服务一键被抓取：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-svc
  labels: { release: monitoring }       # 匹配 Prometheus CR 的 selector
spec:
  selector:
    matchLabels: { app: order }
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

搭配 Deployment + Service 暴露 `/metrics` 即可。

### 12.4 K8s 必看指标

- **节点**：CPU、内存、磁盘、网络、文件句柄
- **Pod**：重启次数、Pending 数、资源使用 vs requests/limits
- **Deployment**：副本差值、rollout 状态
- **网络**：Service 5xx、Ingress 延迟
- **调度**：调度延迟、pending reason
- **API Server**：延迟、错误率、QPS
- **etcd**：延迟、commit、member 健康

### 12.5 必做的几件事

- 每个应用都暴露 `/metrics`，用 ServiceMonitor 接入
- 每个 Pod 打结构化日志到 stdout
- Trace 从网关到服务端端贯穿，注入 trace_id 到日志
- 告警走 SLO 燃烧率，不滥发
- 仪表盘有 RED + USE + K8s 健康

### 📝 笔试题 12-1：`kube-state-metrics` 和 `metrics-server` 区别？

- **kube-state-metrics**：把 K8s **对象的状态**（Deployment 副本、Pod 状态等）暴露成 Prometheus 指标，用于报表和告警
- **metrics-server**：轻量，只提供 **CPU/内存实时用量**（给 `kubectl top`、HPA 使用），不存历史

两者并存，职责不同。

---

## 13. 服务集成：Java 示例

Spring Boot + Micrometer + OpenTelemetry 是 Java 生态主流。

### 13.1 依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry.instrumentation</groupId>
  <artifactId>opentelemetry-spring-boot-starter</artifactId>
  <version>2.x</version>
</dependency>
```

### 13.2 配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    prometheus:
      enabled: true
    health:
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      env: ${ENV:prod}
  tracing:
    sampling:
      probability: 0.1

otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317
  resource:
    attributes:
      service.name: ${spring.application.name}
      service.version: ${BUILD_VERSION:unknown}
```

Actuator 提供：

- `/actuator/prometheus` → 指标
- `/actuator/health` → 探针（K8s `readinessProbe`/`livenessProbe` 指向 `/actuator/health/readiness` / `/liveness`）

### 13.3 自定义指标

```java
@RestController
class OrderController {
    private final Counter orderCreated;
    private final Timer   orderLatency;

    OrderController(MeterRegistry reg) {
        this.orderCreated = Counter.builder("orders_created_total")
            .description("created orders")
            .tag("service", "order")
            .register(reg);
        this.orderLatency = Timer.builder("order_create_seconds")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(reg);
    }

    @PostMapping("/orders")
    public Order create(@RequestBody Req req) {
        return orderLatency.record(() -> {
            Order o = service.create(req);
            orderCreated.increment();
            return o;
        });
    }
}
```

### 13.4 结构化日志（Logback JSON）

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>trace_id</includeMdcKeyName>
      <includeMdcKeyName>span_id</includeMdcKeyName>
      <customFields>{"service":"order","env":"${ENV:-dev}"}</customFields>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

OpenTelemetry 的 logback instrumentation 会自动把 `trace_id/span_id` 塞进 MDC。

### 13.5 Trace（自动 + 手动）

**自动**：`opentelemetry-spring-boot-starter` 已为 HTTP、JDBC、Redis、Kafka、gRPC 等注入 span。

**手动加业务 span**：

```java
@Autowired Tracer tracer;

public void process(Order o) {
    Span span = tracer.spanBuilder("order.process").startSpan();
    try (var s = span.makeCurrent()) {
        span.setAttribute("order.id", o.getId());
        span.setAttribute("order.amount", o.getAmount());
        // do work
    } catch (Exception ex) {
        span.recordException(ex);
        span.setStatus(StatusCode.ERROR);
        throw ex;
    } finally {
        span.end();
    }
}
```

### 13.6 Dockerfile 片段

```dockerfile
FROM eclipse-temurin:21-jre-alpine
ADD target/app.jar /app.jar
# 可选：用 Java Agent 自动注入（无需改代码）
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.0.0/opentelemetry-javaagent.jar /otel.jar
ENV JAVA_TOOL_OPTIONS="-javaagent:/otel.jar \
  -Dotel.service.name=order \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317"
ENTRYPOINT ["java","-jar","/app.jar"]
```

### 📝 笔试题 13-1：Spring Boot 里 `Histogram` 的分位数建议用哪种方式？

推荐 **服务端聚合**：Micrometer 暴露 `publishPercentileHistogram()` 出 bucket，再用 PromQL `histogram_quantile` 跨实例聚合。**不要**使用 `publishPercentiles(...)` + 仅客户端分位数，因为单实例分位数无法跨实例求和。

---

## 14. 服务集成：Node.js 示例

### 14.1 依赖

```bash
npm i prom-client
npm i @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
      @opentelemetry/exporter-trace-otlp-grpc @opentelemetry/exporter-metrics-otlp-grpc
npm i pino pino-pretty
```

### 14.2 启动前加载 OTel

`tracing.js`：

```javascript
// 必须在业务代码之前 require
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'order',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.VERSION || 'dev',
    'deployment.environment': process.env.ENV || 'prod',
  }),
  traceExporter: new OTLPTraceExporter({ url: process.env.OTLP_ENDPOINT || 'http://otel-collector:4317' }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

启动：`node -r ./tracing.js server.js`。

### 14.3 Metrics（prom-client）

```javascript
const express = require('express');
const client = require('prom-client');

client.collectDefaultMetrics({ prefix: 'node_' });

const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP latency',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.005, 0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

const app = express();

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => {
    end({ method: req.method, route: req.route?.path || req.path, status: res.statusCode });
  });
  next();
});

app.get('/metrics', async (_, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.get('/healthz', (_, res) => res.send('ok'));

app.listen(8080);
```

### 14.4 结构化日志（Pino）

```javascript
const pino = require('pino');
const { context, trace } = require('@opentelemetry/api');

const log = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: { service: 'order', env: process.env.ENV },
  timestamp: pino.stdTimeFunctions.isoTime,
  mixin() {
    const span = trace.getSpan(context.active());
    if (!span) return {};
    const { traceId, spanId } = span.spanContext();
    return { trace_id: traceId, span_id: spanId };
  },
});

log.info({ user_id: 42, event: 'order.created' }, 'order created');
```

---

## 15. 服务集成：Python / Go 示例

### 15.1 Python（FastAPI + OTel）

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp \
            prometheus-client structlog
opentelemetry-bootstrap -a install
```

```python
# app.py
from fastapi import FastAPI
from prometheus_client import Counter, Histogram, make_asgi_app
import structlog, time

app = FastAPI()
app.mount("/metrics", make_asgi_app())

REQS = Counter("http_requests_total", "Total requests", ["method", "path", "status"])
LAT  = Histogram("http_request_duration_seconds", "Latency",
                 ["method", "path"],
                 buckets=(.005, .01, .05, .1, .5, 1, 2, 5))

log = structlog.get_logger()

@app.middleware("http")
async def metrics_middleware(req, call_next):
    t0 = time.perf_counter()
    resp = await call_next(req)
    LAT.labels(req.method, req.url.path).observe(time.perf_counter() - t0)
    REQS.labels(req.method, req.url.path, resp.status_code).inc()
    return resp

@app.get("/healthz")
def healthz(): return {"ok": True}

@app.get("/orders/{oid}")
def get_order(oid: str):
    log.info("order.fetch", order_id=oid)
    return {"id": oid}
```

启动（自动埋点）：

```bash
opentelemetry-instrument \
  --traces_exporter otlp \
  --metrics_exporter otlp \
  --logs_exporter otlp \
  --service_name order \
  uvicorn app:app --host 0.0.0.0 --port 8080
```

### 15.2 Go（net/http + OTel + prometheus）

```go
package main

import (
    "context"
    "log"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

var (
    reqTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total", Help: "Total HTTP requests",
    }, []string{"method", "path", "status"})
    reqDur = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name: "http_request_duration_seconds", Help: "Latency",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "path"})
)

func initTracer(ctx context.Context) (*sdktrace.TracerProvider, error) {
    exp, err := otlptracegrpc.New(ctx, otlptracegrpc.WithInsecure(),
        otlptracegrpc.WithEndpoint("otel-collector:4317"))
    if err != nil { return nil, err }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp),
        sdktrace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("order"),
            semconv.ServiceVersionKey.String("1.0"),
        )),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

func instrument(h http.HandlerFunc, path string) http.Handler {
    return otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        t0 := time.Now()
        rw := &statusRW{ResponseWriter: w, status: 200}
        h(rw, r)
        reqDur.WithLabelValues(r.Method, path).Observe(time.Since(t0).Seconds())
        reqTotal.WithLabelValues(r.Method, path, itoa(rw.status)).Inc()
    }), path)
}

func main() {
    ctx := context.Background()
    tp, _ := initTracer(ctx)
    defer tp.Shutdown(ctx)

    http.Handle("/metrics", promhttp.Handler())
    http.Handle("/healthz", http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
        w.Write([]byte("ok"))
    }))
    http.Handle("/orders", instrument(ordersHandler, "/orders"))

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

（`statusRW` 与 `ordersHandler` 省略。）

---

## 16. 前端与移动端可观测性

### 16.1 为什么关心

后端 SLO 绿≠用户体验绿。前端要采集：

- **Core Web Vitals**：LCP、CLS、INP
- **JS 异常**、**资源加载失败**
- **API 调用延迟/错误**
- **用户交互路径**（Session Replay）

### 16.2 技术方案

- **Sentry**：错误 + 性能 + session replay
- **Datadog RUM**：一体化
- **Google Web Vitals**：轻量、开源
- **OpenTelemetry JS**（Browser SDK）：把前端 trace 发到 Collector，与后端 trace 打通

### 16.3 与后端打通

- 前端发请求前生成 `traceparent` header
- 后端 OTel 自动接续，同一 trace_id 跨端可视化
- 问题从"前端看到的慢 API"一路追到后端具体 span

### 📝 笔试题 16-1：只有后端监控为什么不够？

用户感受到的"慢"可能源于：

- CDN / DNS
- 前端 JS 阻塞（大 bundle、长任务）
- 渲染路径卡顿（CLS / Layout Thrash）
- 网络差
- 客户端 API 重试风暴

这些后端看不到；没有**前端可观测**就无法准确衡量 **用户级 SLO**。

---

## 17. 成本、采样与数据治理

### 17.1 可观测性的成本陷阱

- 日志：按 GB 存储，量大时月账单惊人
- 指标：**高基数 label**（如 `user_id` 作 label）会让 Prometheus 爆炸
- Traces：100% 全采样成本不可承受

### 17.2 控制手段

**Metrics**：

- **避免高基数 label**：`user_id`、`request_id`、`path` 带参数 → 用日志/Trace 记录，metric 只打低基数维度
- **合理的直方图 buckets**：bucket 太多导致 series 爆炸
- **recording rules**：预聚合减少查询时压力
- **删除无人看的指标**

**Logs**：

- **分级采样**：DEBUG 1%、INFO 50%、WARN/ERROR 100%
- **关键路径全量**：支付、登录
- 非关键路径按流量采样
- **索引 vs 存储分离**：冷数据下 S3，查询时按需
- **字段白名单**：避免全文索引所有字段

**Traces**：

- **Head-based 10-20%** + **Tail-based** 保留错误/慢请求
- 保留多少取决于预算，但**错误率 100%** 是底线

### 17.3 数据治理

- 统一命名（`service`、`env`、`version`）
- 统一时间戳 / 时区（UTC）
- 敏感字段脱敏（密码、token、身份证、卡号、手机）
- 生命周期管理（ILM / 对象存储冷归档）
- 多租户权限（Grafana 组织、OpenSearch roles）

### 📝 笔试题 17-1：同事把 `user_id` 打进 Prometheus label，引发线上问题，为什么？

**基数爆炸**：每个独立 label 组合 = 一个时间序列。几百万用户 × 其他维度 → 千万级序列，Prometheus 内存、磁盘、查询全部崩溃。

正确做法：`user_id` 放在 **日志**或 **trace attribute** 里；指标 label 只保留低基数维度（`service`、`method`、`status_code`、`region` 等）。

---

## 18. 排障实战剧本

### 18.1 通用排障流程

```
收到告警 → 确认范围 → 看仪表盘（RED + USE + 依赖）
            │
            ▼
        定位异常服务 → 看对应 Traces（选慢/错的 trace）
            │
            ▼
        定位异常 span → 过 trace_id 看 Logs
            │
            ▼
        定位代码/配置/依赖 → 回滚 / 修复 / 降级
            │
            ▼
        事后复盘（Postmortem）
```

### 18.2 常见场景剧本

**① 错误率突然上升**

1. Grafana 看按服务/endpoint 的 error rate，定位受害点
2. 相邻时间有发布？`kubectl rollout undo`
3. 查最近 5xx 日志，看异常堆栈 / 具体原因
4. 上游依赖（DB / Redis / 下游服务）正常？
5. 降级：开启 feature flag 关闭问题功能

**② 延迟升高但错误率正常**

1. 看 P95/P99 图，比较基线
2. 看 Traces，定位慢 span
3. 常见：DB 慢查询、Redis 热 key、GC 停顿、下游超时
4. 数据库：慢查询日志 + EXPLAIN
5. 缓存：命中率、热 key
6. GC：heap 占用、STW 时长
7. 临时：扩容 Pod / 开熔断降级

**③ Pod 频繁重启 CrashLoopBackOff**

- `kubectl describe pod` + `kubectl logs --previous`
- 看 Events：OOMKilled？探针失败？镜像拉不下？
- 看应用启动日志

**④ 日志查不到 / 指标断了**

- Agent 存活？`kubectl get pods -n logging`
- 下游（ES/Loki/Prom）是否健康
- 磁盘满？`df -h`
- 配置变更？diff 最近 commit

**⑤ 数据库相关告警**

- 连接池用满？慢查询堆积？
- `pt-query-digest` / 慢日志分析
- 读写分离/缓存是否命中
- Kill 最慢几条 query 临时缓解

### 18.3 事后复盘模板（Postmortem）

1. **摘要**：什么时候 / 影响范围 / 持续时长
2. **时间线**：从第一条信号到恢复的关键动作
3. **根因分析**：直接原因 + 深层原因（5 Whys）
4. **对用户影响**：错误请求数、失败金额、SLO 消耗
5. **做对了什么**（值得称赞）
6. **做错了什么**（不责备个人）
7. **改进项**：带负责人与期限
8. **经验沉淀**：新增 runbook / 告警 / 演练

### 📝 笔试题 18-1：某服务发布后 5 分钟内 P99 从 200ms 涨到 2s，怎么查？

1. 立即看 Trace，挑一条慢请求：哪个 span 慢了？
2. 对比发布前后 Commit diff
3. 看数据库查询指标：是否新增 N+1 查询 / 缺索引
4. 看依赖服务：是否因新版调用模式导致下游慢
5. 看资源：GC、线程池、连接池饱和度
6. 决策：若确认新版问题 → 立即 `rollout undo`；拿数据复盘

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** Observability 三支柱是？
A. Alerts / Dashboards / Logs
B. Metrics / Logs / Traces
C. CPU / Memory / Disk
D. RED / USE / Golden Signals

<details><summary>答案</summary>B。</details>

**Q2** Prometheus 主要采用哪种数据获取模式？
A. Push  B. Pull  C. 两者皆可但默认 Push  D. 广播

<details><summary>答案</summary>B。</details>

**Q3** 下列哪个 PromQL **不正确**？
A. `rate(http_requests_total[5m])`
B. `histogram_quantile(0.99, sum by(le)(rate(x_bucket[5m])))`
C. `sum(http_requests_total{})`
D. `rate(cpu_usage)`（直接 rate gauge）

<details><summary>答案</summary>D。</details>

**Q4** 关于 `user_id` 放 Prometheus label，正确的是？
A. 推荐，方便过滤
B. 禁止，会导致基数爆炸
C. 无影响
D. 必须转成哈希

<details><summary>答案</summary>B。</details>

**Q5** 把 trace_id 写入日志的作用？
A. 加密安全
B. 关联同一次请求的日志与 trace
C. 做压缩
D. 提升 ES 查询速度

<details><summary>答案</summary>B。</details>

**Q6** 下列关于 SLO 描述错误的是？
A. SLO 是内部目标
B. SLA 通常松于 SLO
C. Error Budget = 1 - SLO
D. 发布策略可绑定错误预算

<details><summary>答案</summary>B。SLA 通常严于 / 等于（对外承诺），应 "松于" 写成 "严于"；常见表述是 SLO 严于 SLA。</details>

**Q7** K8s 里收集容器日志通常采用？
A. 应用直接写 ES
B. DaemonSet 运行 Fluent Bit
C. Pod 内 sidecar 与业务共享磁盘
D. 手动 `kubectl logs`

<details><summary>答案</summary>B（B 最常见；C 也可）。</details>

**Q8** 下列哪个场景最适合 Histogram 而不是 Summary？
A. 单实例简单应用
B. 分布式多实例需要跨实例聚合分位数
C. 只关心总计数
D. Gauge 性质的内存使用

<details><summary>答案</summary>B。</details>

### 19.2 判断题

1. 监控必然包含可观测性，两者等价。 ❌
2. Prometheus 适合长期存储无限历史数据。 ❌（需 Thanos/Mimir/VM）
3. OpenSearch 与 Elasticsearch 完全不兼容。 ❌（兼容到 ES 7.10 API）
4. 应用直接把日志写到 ES 是推荐做法。 ❌
5. Grafana 可作为多种数据源的统一前端。 ✅
6. 尾部采样（tail-based）比头部采样更省成本。 ❌（通常更贵，但信息更全）
7. 错误预算可用于决定发布节奏。 ✅
8. OpenTelemetry 统一了 Metrics/Logs/Traces 的采集与导出模型。 ✅

### 19.3 简答题

**Q1** 从指标异常到具体代码行，典型的三支柱联动路径是？

1. Grafana 指标面板发现异常（如 P99 飙升）
2. 点击对应时间段 → 跳转到该时段的 Traces，找出慢/错的 trace
3. 点 trace 里某个错的 span → 跳转到该 span 关联的 Logs
4. 按 trace_id 过滤日志，看到堆栈/SQL/参数
5. 定位代码 / SQL / 配置 / 下游服务

关键前提：所有 data 都带 `trace_id` + `service` + `env`，Grafana 配置了 data links / correlations。

**Q2** 微服务应用标准埋点三件事？

1. **Metrics**：暴露 `/metrics`（RED + 自定义业务指标）
2. **Logs**：结构化 JSON 到 stdout，含 `trace_id`
3. **Traces**：OpenTelemetry 自动 + 关键业务手动 span

配合 K8s 探针 `/healthz` `/readyz` 完成上线。

**Q3** 日志字段如何设计便于排障？

统一字段集合：

- `ts`（ISO8601 UTC）
- `level`
- `service`、`env`、`version`、`region`
- `trace_id`、`span_id`
- `event`：事件语义码（`order.created`、`payment.failed`）
- 业务键：`user_id`、`order_id`、`request_id`
- `msg`：给人看的描述
- 错误相关：`error.type`、`error.stack`

避免把 JSON blob 混合进 message 字段而是作为子对象，便于 ES 建索引。

**Q4** 如何降低可观测性成本又不失关键信息？

- Metrics：去高基数 label、加 recording rules 预聚合
- Logs：分级采样 + 冷热分离；高价值事件全量
- Traces：head 10-20% + tail 保错误/慢请求
- 存储：ILM 自动归档；长保留走对象存储
- 持续审计：哪些 dashboard/alert/log 实际没人看？清理

### 19.4 设计题

**Q1** 为一个 100 服务、500 Pod 的 EKS 集群设计可观测性方案。

**指标**：

- `kube-prometheus-stack` + **Thanos**（对象存储 S3 长期）
- 每服务 Helm chart 含 ServiceMonitor
- 自定义业务指标走 Micrometer / prom-client / OpenTelemetry metrics

**日志**：

- Fluent Bit DaemonSet → **OpenSearch** 集群（3 master + 6 data，ILM：7d hot / 30d warm / 90d cold → snapshot）
- 关键服务日志同时写 S3 做审计归档
- 统一 index template：`logs-<service>-<yyyy.MM.dd>`

**Trace**：

- OpenTelemetry SDK 集成应用；OTel Collector DaemonSet 做 agent + Gateway 做中心化处理
- Head 10% 采样；Tail 规则：错误 100%、延迟 > 500ms 100%
- 后端 **Grafana Tempo** (S3) 存 30 天

**可视化**：

- Grafana 统一平台，dashboard 代码化（Git + Jsonnet）
- 关联 Prom + Loki/OpenSearch + Tempo，点击跳转

**告警**：

- Alertmanager → PagerDuty + Slack
- SLO 燃烧率多窗口告警（1h/5m + 6h/30m）
- 抑制规则 + runbook 链接

**文化**：

- 所有服务统一埋点模板（脚手架）
- 新服务上线 Checklist：metrics 暴露、log 结构化、trace 接入、dashboard、告警、runbook
- 季度审计告警噪音与 dashboard 有效性

**Q2** 你刚接手一个没有任何监控的老系统，从零搭建可观测性的优先级？

1. **第 1 周**：把 **stdout 结构化 JSON 日志**跑起来，至少带 service/level/ts；接一份节点日志采集到现成平台（CloudWatch 最快）
2. **第 2 周**：节点 node-exporter + K8s kube-state-metrics（或等价系统） + Prometheus，**仪表盘先看 RED**
3. **第 3 周**：业务关键路径手动埋 3-5 个自定义指标（下单数、支付成功率、关键延迟）
4. **第 4 周**：挑核心接口接 OpenTelemetry trace，优先在网关和最重要的服务上
5. **第 5 周**：SLO 定义（至少核心业务 1-2 条） + 基础告警（5 条以内，够用）
6. **持续**：每次故障复盘补一个可观测性改进项

核心原则：**先用起来，再做全面**；不要一开始就追求"完美栈"。

---

## 📚 学习建议

1. **先跑一遍全栈**：minikube / kind 本地跑 kube-prometheus-stack + Loki + Tempo + Grafana，手动点一遍 UI
2. **读 Google SRE 书**：《Site Reliability Engineering》《The SRE Workbook》，SLO 章节必读
3. **读 Prometheus 官方文档**：`prometheus.io/docs/` 权威完整
4. **OpenTelemetry 官方示例**：各语言 `getting-started`，动手改
5. **关注成本**：高基数、长保留、100% 采样——三大烧钱源头
6. **文化先行**：再好的工具也得团队愿意用；从 runbook、postmortem 开始建文化

> 祝你的系统看得见、摸得着、讲得清。
