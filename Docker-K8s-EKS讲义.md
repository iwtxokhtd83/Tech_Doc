# Docker、Kubernetes 与 EKS 讲义

> 本讲义按三段递进组织：**Docker（容器基础）→ Kubernetes（编排核心）→ Amazon EKS（AWS 托管 K8s）**。每章配"知识点 + 笔试题"。
>
> 约定：示例基于 Docker 24+、Kubernetes 1.28+、EKS 1.28+；命令行默认 Linux/macOS，Windows 请用 PowerShell 等价写法。

## 目录

1. [容器化基础与核心概念](#1-容器化基础与核心概念)
2. [Docker 架构与对象](#2-docker-架构与对象)
3. [镜像与 Dockerfile 最佳实践](#3-镜像与-dockerfile-最佳实践)
4. [容器运行与资源管理](#4-容器运行与资源管理)
5. [存储与网络](#5-存储与网络)
6. [Docker Compose 与多容器编排](#6-docker-compose-与多容器编排)
7. [Kubernetes 架构概览](#7-kubernetes-架构概览)
8. [核心工作负载对象](#8-核心工作负载对象)
9. [Service、Ingress 与网络](#9-serviceingress-与网络)
10. [存储：Volume / PVC / StorageClass](#10-存储volume--pvc--storageclass)
11. [配置与密钥：ConfigMap / Secret](#11-配置与密钥configmap--secret)
12. [调度与资源管理](#12-调度与资源管理)
13. [自动扩缩容与自愈](#13-自动扩缩容与自愈)
14. [RBAC、命名空间与多租户](#14-rbac命名空间与多租户)
15. [Helm 与 GitOps](#15-helm-与-gitops)
16. [可观测性与运维](#16-可观测性与运维)
17. [Amazon EKS 入门](#17-amazon-eks-入门)
18. [EKS 核心集成与最佳实践](#18-eks-核心集成与最佳实践)
19. [综合笔试练习](#19-综合笔试练习)

---

## 1. 容器化基础与核心概念

### 1.1 容器 vs 虚拟机

```
传统虚拟机                         容器
┌────────────┐  ┌────────────┐    ┌────────────┐  ┌────────────┐
│   App      │  │   App      │    │   App      │  │   App      │
├────────────┤  ├────────────┤    ├────────────┤  ├────────────┤
│  Bins/Libs │  │ Bins/Libs  │    │ Bins/Libs  │  │ Bins/Libs  │
├────────────┤  ├────────────┤    └────────────┘  └────────────┘
│   Guest OS │  │  Guest OS  │    ┌──────────────────────────────┐
├────────────┴──┴────────────┤    │  Container Runtime (containerd)│
│       Hypervisor           │    ├──────────────────────────────┤
├────────────────────────────┤    │       Host OS (Linux)        │
│       Host OS              │    ├──────────────────────────────┤
└────────────────────────────┘    │       Hardware               │
```

| 维度 | VM | 容器 |
|------|----|----|
| 隔离 | 硬件级 | 进程级 |
| 启动 | 分钟 | 毫秒-秒 |
| 体积 | GB | MB |
| 性能开销 | 高 | 极低 |
| 内核 | 独立 | 共享宿主 |

### 1.2 容器背后的 Linux 技术

- **Namespace**：隔离进程视图（PID / Mount / Network / IPC / UTS / User / Cgroup）
- **Cgroups (v2)**：限制 CPU / 内存 / IO / 网络等资源
- **Union FS**：分层文件系统（overlay2 为主）
- **Capabilities**：细粒度 root 权限拆分
- **Seccomp / AppArmor / SELinux**：系统调用与安全策略

### 1.3 OCI 规范

- **Image Spec**：镜像格式
- **Runtime Spec**：运行时接口（`runc`、`crun`）
- **Distribution Spec**：镜像分发（registry）

结果：**Docker、containerd、Podman、CRI-O 都可互通**，镜像一次构建到处运行。

### 📝 笔试题 1-1：容器和虚拟机的本质区别？

虚拟机虚拟化**硬件**，运行完整 OS；容器只是**被隔离与限制的进程**，与宿主共享内核。因此容器更轻量、启动更快、密度更高，但隔离性弱于 VM（内核级攻击可能逃逸）。

---

## 2. Docker 架构与对象

### 2.1 组件

```
┌────────────┐      ┌────────────────────┐
│ docker CLI │──────│   dockerd (Daemon) │
└────────────┘      │    ┌─────────────┐ │
                    │    │ containerd  │ │
                    │    │   + runc    │ │
                    │    └─────────────┘ │
                    └────────────────────┘
                           │
                           ▼
                   Namespaces / Cgroups
```

- **Docker CLI** → **Docker Daemon (`dockerd`)** → **containerd** → **runc**
- 现代 Kubernetes 已抛弃 dockershim，直接用 **containerd / CRI-O**
- Docker Desktop 在 Mac/Win 里跑一个 Linux VM 来承载容器

### 2.2 核心对象

- **Image（镜像）**：只读模板
- **Container（容器）**：运行中的镜像实例
- **Volume（卷）**：持久化数据
- **Network**：容器网络
- **Registry**：镜像仓库（Docker Hub / ECR / Harbor）

### 2.3 常用命令速查

```bash
# 镜像
docker pull nginx:1.25
docker images
docker rmi nginx:1.25
docker build -t myapp:1.0 .
docker tag myapp:1.0 ghcr.io/me/myapp:1.0
docker push ghcr.io/me/myapp:1.0
docker image prune -a          # 清理悬空镜像

# 容器
docker run -d --name web -p 8080:80 nginx
docker ps / docker ps -a
docker stop web / docker start web / docker restart web
docker rm -f web
docker exec -it web sh
docker logs -f --tail 100 web
docker stats                   # 实时资源
docker inspect web             # 详细元数据

# Volume / Network
docker volume ls / create / rm
docker network ls / inspect

# 构建上下文
docker build --target prod -t app:prod .
docker buildx build --platform linux/amd64,linux/arm64 --push -t repo/app:1.0 .
```

### 2.4 `run` 的常用参数

```bash
docker run \
  --name myapp \
  -d                              # 后台
  --restart unless-stopped        # 重启策略
  -p 8080:80                      # 端口映射 host:container
  -v /host/path:/data             # 挂载目录
  -v myvol:/data                  # 具名卷
  -e ENV=prod                     # 环境变量
  --env-file .env
  --memory=512m --cpus=1          # 资源限制
  --read-only                     # 根文件系统只读
  --user 1000:1000                # 非 root 运行
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE
  --network mynet
  myapp:1.0 [args...]
```

### 📝 笔试题 2-1：`docker exec` 和 `docker attach` 的区别？

- `exec`：在已运行容器里**启动一个新进程**（常用 `sh`/`bash`）；退出不影响容器
- `attach`：把当前终端"接回"容器的**主进程 stdin/stdout/stderr**；`Ctrl+C` 会杀掉主进程

生产调试一律用 `exec`，更安全。

---

## 3. 镜像与 Dockerfile 最佳实践

### 3.1 分层模型

```
Image
 ├── Layer 1 (base)     只读
 ├── Layer 2
 ├── Layer 3
 └── Container Writable Layer  (运行时)
```

- 每条 Dockerfile 指令一层（部分会合并）
- 层间**内容寻址**去重，拉镜像只下缺失层
- 容器运行产生的写入落在**可写层**（容器删除即丢）

### 3.2 基础 Dockerfile 模板（Node.js 示例）

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/healthz || exit 1
CMD ["node", "dist/server.js"]
```

### 3.3 最佳实践清单

- **多阶段构建（multi-stage）**：编译环境与运行环境分离，镜像更小
- **选对 base 镜像**：
  - 极小：`scratch` / `distroless` / `alpine`
  - 折中：`-slim` 变体
  - 全功能：`ubuntu` / `debian`
- **固定 tag**：用 `node:20.11-alpine` 或 **摘要** `node@sha256:...`，避免"今天构建明天不一样"
- **合理分层**：
  - 先复制依赖清单再装依赖 → 代码变动不重装
  - `npm ci` / `pip install --no-cache-dir`
- **`.dockerignore`**：剔除 `node_modules`、`.git`、`dist`、`.env`
- **最小权限**：`USER`、`--read-only`、`--cap-drop`
- **非 root 运行**
- **签名与扫描**：`cosign`、`trivy`、`grype`
- **HEALTHCHECK**：便于编排系统感知
- **日志到 stdout/stderr**，不写文件
- **不要在镜像里放密钥**

### 3.4 常见指令对比

- `CMD` vs `ENTRYPOINT`：`ENTRYPOINT` 是命令主体，`CMD` 是默认参数
- `ADD` vs `COPY`：能用 `COPY` 就用 `COPY`；`ADD` 支持 URL/解压，副作用大
- `RUN` 使用 `&&` 合并减少层数 → Buildx 下更推荐 `RUN --mount` 缓存依赖

### 3.5 镜像瘦身技巧

- 多阶段 + distroless / scratch
- `apt-get ... --no-install-recommends && rm -rf /var/lib/apt/lists/*`
- 删除中间产物和缓存
- 用 `dive` 工具分析各层大小
- Java：用 **jlink** 裁剪 JRE；Go：静态编译 + `FROM scratch`

### 📝 笔试题 3-1：`COPY` 一个大目录为什么要先 COPY `package.json` 再 `npm ci`？

Docker 构建**按层缓存**：源码变动不会使"安装依赖"那一层失效。若 `COPY . .` 放在 `npm ci` 之前，每次代码改动都要重装所有依赖，构建时间剧增。

---

## 4. 容器运行与资源管理

### 4.1 重启策略

```bash
--restart no              # 默认，不自动重启
--restart on-failure:3    # 失败重启，最多 3 次
--restart always          # 总是
--restart unless-stopped  # 除非手动停止
```

### 4.2 资源限制（Cgroups）

```bash
--cpus=2                  # 2 个 CPU
--cpu-shares=512          # 相对权重（默认 1024）
--memory=512m
--memory-swap=1g
--pids-limit=200
--blkio-weight=500
```

Linux 内核 OOM 杀容器时，**未限制内存的容器风险更高**；生产务必设限。

### 4.3 运行时安全

```bash
--read-only                                 # 根文件系统只读
--tmpfs /tmp                                # 需要写时用 tmpfs
--security-opt no-new-privileges:true
--cap-drop ALL --cap-add NET_BIND_SERVICE
--user 1000:1000                            # 指定 UID/GID
--pids-limit 100
```

### 4.4 日志与监控

```bash
# 日志驱动（默认 json-file）
--log-driver=json-file --log-opt max-size=50m --log-opt max-file=5
--log-driver=fluentd
--log-driver=awslogs --log-opt awslogs-group=/app/prod

# 监控
docker stats              # 实时
docker events             # 生命周期事件
```

### 📝 笔试题 4-1：容器崩溃后日志去哪了？

默认 `json-file` 驱动下，日志存于 `/var/lib/docker/containers/<id>/*-json.log`。容器被 **删除**（`docker rm`）后日志一并删除。生产应：

- 设置 `max-size` / `max-file` 防磁盘爆
- 用集中日志（Fluentd/Filebeat → ES/Loki/CloudWatch）
- 重要审计日志落到持久卷或外部

---

## 5. 存储与网络

### 5.1 存储三种方式

| 方式 | 位置 | 管理 | 适合 |
|------|------|------|------|
| **Volume** (推荐) | `/var/lib/docker/volumes/` | Docker 管理 | 持久化数据、跨容器共享 |
| **Bind Mount** | 任意宿主路径 | 用户管理 | 开发挂代码、配置文件 |
| **tmpfs** | 宿主内存 | 临时 | 敏感数据、无需持久 |

```bash
# Volume
docker volume create mydata
docker run -v mydata:/data ...

# Bind mount
docker run -v $(pwd)/src:/app/src ...

# tmpfs
docker run --tmpfs /tmp ...
```

### 5.2 网络驱动

- **bridge**（默认）：同机容器互通，外部靠端口映射
- **host**：容器直接使用宿主网络，没有隔离，性能高
- **none**：无网络
- **overlay**：跨主机（Swarm）
- **macvlan**：容器有独立 MAC，接入物理网

### 5.3 自定义网络实战

```bash
docker network create mynet
docker run -d --name db --network mynet postgres:16
docker run -d --name app --network mynet -e DB_HOST=db myapp
```

- 同一自定义网络内**自动 DNS 解析**容器名 → IP
- 默认 `bridge` 网络**没有**名字解析，推荐总是创建自定义网络

### 📝 笔试题 5-1：`-v` 挂载时，主机路径不存在会怎样？

- 对 **bind mount**：Docker 会自动创建宿主路径（为目录）；容器内覆盖掉镜像对应目录
- 对 **volume**：首次 mount 时会把镜像里相应目录**内容拷贝到卷**（仅首次且卷为空）

---

## 6. Docker Compose 与多容器编排

### 6.1 docker-compose.yaml 示例

```yaml
services:
  app:
    build:
      context: .
      target: runtime
    image: myapp:1.0
    restart: unless-stopped
    env_file: .env
    environment:
      DB_HOST: db
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8080:8080"
    networks: [backend]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/healthz"]
      interval: 30s
      timeout: 3s
      retries: 3

  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
    networks: [backend]

volumes:
  pgdata:

networks:
  backend:
```

### 6.2 常用命令

```bash
docker compose up -d                # 启动
docker compose ps
docker compose logs -f app
docker compose exec app sh
docker compose build --no-cache app
docker compose pull                 # 拉最新镜像
docker compose down                 # 停止+删容器+删默认网络
docker compose down -v              # 连 volume 也删（危险）
```

### 6.3 使用场景

- **本地开发**：一条命令拉起所有依赖（DB/Redis/MQ）
- **CI**：起完整栈跑集成测试
- **简单单机生产**：小项目可用（多副本/自愈弱于 K8s）

对生产多节点仍推荐 **Kubernetes**。

### 📝 笔试题 6-1：Compose 多次启动导致数据丢失怎么办？

最常见原因：

- 使用了 **匿名卷** 或 `down -v`
- 命名卷名变了（项目名变化导致前缀变动，`docker compose -p` 固定项目名）
- 改了 image/volumes 导致重建

方案：始终显式命名 volume，避免 `-v` 误删；重要数据定期 `docker run --rm -v pgdata:/src busybox tar cf - /src | ...` 备份。

---

## 7. Kubernetes 架构概览

### 7.1 从 Docker 到 K8s

单机容器解决不了：

- **多机编排**：大量实例跨节点调度
- **自愈**：节点/容器挂掉自动恢复
- **扩缩容**：按负载自动伸缩
- **服务发现**：动态地址
- **滚动更新 / 回滚**

**Kubernetes** 是事实标准的容器编排系统。

### 7.2 集群架构

```
┌────────────── Control Plane ───────────────┐
│  ┌──────────┐  ┌──────────────┐            │
│  │ API Srv  │  │  etcd        │            │
│  └──────────┘  └──────────────┘            │
│  ┌──────────┐  ┌──────────────────┐        │
│  │ Scheduler│  │ Controller Mgr   │        │
│  └──────────┘  └──────────────────┘        │
└────────────────────────────────────────────┘
                 │
┌────────── Worker Nodes ─────────────┐
│  ┌──────────┐  ┌──────────────┐     │
│  │ kubelet  │  │ kube-proxy   │     │
│  └──────────┘  └──────────────┘     │
│  ┌──────────────────────────────┐   │
│  │  Container Runtime (containerd)│ │
│  └──────────────────────────────┘   │
│   Pods / Pods / Pods ...            │
└─────────────────────────────────────┘
```

- **API Server**：所有操作的唯一入口（REST/gRPC）
- **etcd**：强一致 KV，存所有集群状态
- **Scheduler**：为 Pod 选择合适节点
- **Controller Manager**：各种控制循环（Replica、Node、Job...）
- **kubelet**：节点代理，管理 Pod 生命周期
- **kube-proxy**：节点网络转发（iptables/ipvs/eBPF）
- **Container Runtime**：`containerd` / `CRI-O`

### 7.3 声明式 API + 控制循环

```
User → 期望状态 (YAML)
        │
        ▼
    API Server → etcd
        │
        ▼
   Controllers 不断对比 "期望" vs "实际"，驱使实际向期望收敛
```

**核心思想**：你说你要什么，K8s 自己想办法做到。

### 7.4 kubectl 入门

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deploy,svc -n default
kubectl describe pod mypod
kubectl logs -f mypod -c container-name
kubectl exec -it mypod -- sh
kubectl apply -f deploy.yaml
kubectl delete -f deploy.yaml
kubectl top pod
kubectl port-forward svc/web 8080:80
kubectl rollout status deploy/web
kubectl rollout undo deploy/web
kubectl explain pod.spec.containers
kubectl config get-contexts / use-context
```

### 📝 笔试题 7-1：API Server 挂了集群还能跑吗？

**能继续运行现有工作负载**：kubelet 会按最后一次从 API Server 拉到的 Pod 规格维持现状，服务继续对外。但**所有管控操作**（`kubectl apply`、调度新 Pod、扩缩容、滚动升级、自愈）都会停摆，直到 API Server / etcd 恢复。

---

## 8. 核心工作负载对象

### 8.1 Pod：最小调度单元

- **一个或多个容器共享** Network / Volumes / IPC
- 典型"多容器"场景：sidecar（日志 agent、代理、数据同步）
- **Pod 是短暂的**，重建 IP 会变——正式流量走 Service 不走 Pod IP

```yaml
apiVersion: v1
kind: Pod
metadata: { name: web }
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports: [{ containerPort: 80 }]
      resources:
        requests: { cpu: 100m, memory: 128Mi }
        limits:   { cpu: 500m, memory: 256Mi }
```

### 8.2 ReplicaSet

维持**指定副本数**的 Pod。通常不直接用它，用上层 Deployment 间接管理。

### 8.3 Deployment：无状态服务

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 25%, maxUnavailable: 25% }
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: web
          image: myapp:1.2.0
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /livez, port: 8080 }
            initialDelaySeconds: 30
```

操作：

```bash
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=3
kubectl scale deploy/web --replicas=10
```

### 8.4 StatefulSet：有状态服务

- Pod 名稳定（`web-0`、`web-1`、`web-2`）
- 有序创建 / 删除
- 每 Pod 绑定**独占** PVC
- 适合 MySQL、Kafka、ZK、Redis 主从

### 8.5 DaemonSet：每节点一个

- 日志 agent（Fluent Bit）、监控（node-exporter）、网络（CNI）等
- 新节点加入自动部署

### 8.6 Job / CronJob

- **Job**：一次性任务，成功即结束
- **CronJob**：按 cron 表达式定时跑

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: cleanup }
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - { name: cleanup, image: myapp:1.0, command: ["node","cleanup.js"] }
```

### 8.7 探针（Probes）

- **livenessProbe**：活不活，挂则重启容器
- **readinessProbe**：准备好否，未就绪则**从 Service 摘除**
- **startupProbe**：启动慢的应用，期间忽略 liveness

设置要点：`readinessProbe` 永远要有，`livenessProbe` 不要和 readiness 一样（避免刚慢一下就被重启造成雪崩）。

### 📝 笔试题 8-1：Deployment 和 StatefulSet 选型？

- **无状态、可随机替换** → Deployment（Web、API、Worker）
- **需要稳定身份、稳定存储、顺序性** → StatefulSet（DB、MQ、缓存的数据节点）
- 生产上，**能用 Deployment 就不用 StatefulSet**；有状态服务越来越多用**Operator + StatefulSet**

---

## 9. Service、Ingress 与网络

### 9.1 Pod 网络模型

- 每个 Pod 一个 IP（集群内可达，集群外不可达）
- **Pod IP 不稳定**，Pod 重建即变

### 9.2 Service：稳定的访问入口

```yaml
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }
  ports:
    - { port: 80, targetPort: 8080 }
  type: ClusterIP              # 默认
```

类型：

- **ClusterIP**（默认）：集群内部 VIP
- **NodePort**：每节点暴露一个端口（30000-32767）
- **LoadBalancer**：请云厂商创建一个外部 LB（EKS 用 AWS ELB）
- **ExternalName**：DNS CNAME

Service 背后由 **kube-proxy** 用 iptables/ipvs/eBPF 规则转发到 Pod。

### 9.3 Headless Service

```yaml
spec:
  clusterIP: None
```

不分配 VIP，直接把 DNS 解析到所有 Endpoint Pod IP，**StatefulSet 配套使用**。

### 9.4 Ingress：七层 HTTP/HTTPS 入口

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: [example.com]
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

- **Ingress Controller**：实际干活的组件（Nginx、Traefik、AWS ALB、HAProxy、Envoy）
- 负责路由、TLS 终结、限流等

### 9.5 Gateway API（未来方向）

Ingress 的继任者，更强的表达能力（HTTPRoute / TCPRoute / GRPCRoute），多角色分工（GatewayClass / Gateway / Route）。云原生项目逐步迁移。

### 9.6 NetworkPolicy：网络隔离

默认 Pod 间全通。用 NetworkPolicy 限制：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-only-from-app }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api } }
      ports: [{ protocol: TCP, port: 5432 }]
```

需要 CNI 支持（Calico、Cilium、AWS VPC CNI + Calico）。

### 📝 笔试题 9-1：Service 和 Ingress 的关系？

- **Service** 是**四层**（TCP/UDP）的内部虚拟 IP 或节点端口
- **Ingress** 是**七层**（HTTP/HTTPS）反向代理，支持基于域名/路径的路由，底层仍依赖 Service 转到 Pod

生产暴露 HTTP：**Ingress → Service → Pod**。

---

## 10. 存储：Volume / PVC / StorageClass

### 10.1 核心概念

- **Volume**：Pod 生命周期内的存储（`emptyDir`、`hostPath`、`configMap`、`secret`、`persistentVolumeClaim` 等）
- **PV (PersistentVolume)**：集群级别的真实存储（NFS、EBS、Ceph、NAS）
- **PVC (PersistentVolumeClaim)**：用户对存储的"申请"
- **StorageClass**：描述如何动态创建 PV（provisioner + 参数）

### 10.2 动态供给流程

```
用户写 PVC → 匹配 StorageClass → 云 Provider 动态创建 PV → 绑定 PVC → 挂载到 Pod
```

### 10.3 示例

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests: { storage: 20Gi }

---
apiVersion: apps/v1
kind: StatefulSet
...
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: gp3
        resources:
          requests: { storage: 20Gi }
```

### 10.4 访问模式

- **RWO**（ReadWriteOnce）：单节点读写（大多数块存储如 EBS）
- **ROX**（ReadOnlyMany）：多节点只读
- **RWX**（ReadWriteMany）：多节点读写（NFS、EFS、FSx）
- **RWOP**（ReadWriteOncePod）：仅单 Pod 读写

### 10.5 回收策略

- `Retain`：手动回收，数据保留
- `Delete`：随 PVC 删除一起删（云盘默认）
- `Recycle`：已废弃

生产**关键数据用 `Retain`** 防误删。

### 📝 笔试题 10-1：两个 Pod 同时挂同一个 EBS 卷，能读写吗？

不行。EBS 只支持 **RWO**，同一时刻一个节点挂载。需要共享写就用 **EFS / FSx（RWX）**，或者改为"每 Pod 独立卷"（StatefulSet）。

---

## 11. 配置与密钥：ConfigMap / Secret

### 11.1 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-cfg }
data:
  LOG_LEVEL: "info"
  app.yaml: |
    server:
      port: 8080
```

两种消费方式：

```yaml
# 环境变量
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef: { name: app-cfg, key: LOG_LEVEL }

# 文件挂载
volumes:
  - name: cfg
    configMap: { name: app-cfg }
volumeMounts:
  - name: cfg
    mountPath: /etc/app
```

### 11.2 Secret

```bash
kubectl create secret generic db-cred \
  --from-literal=username=app \
  --from-literal=password='S3cret!'
```

- 默认 **Base64 编码（非加密）**；开启 **EncryptionConfiguration** 才在 etcd 里加密
- 生产建议：
  - **External Secrets Operator** 从 Vault / AWS Secrets Manager / Parameter Store 同步
  - **Sealed Secrets**（Bitnami）把加密后的 secret 放 Git
  - EKS 上 **IRSA + Secrets Manager** 组合

### 11.3 热更新

- 以 **Volume 形式挂载** 的 ConfigMap / Secret **自动同步**（有秒级延迟）
- 以 **env 形式**的需要重建 Pod 才生效

### 📝 笔试题 11-1：Secret 里 Base64 编码不是加密吗？

不是。Base64 是编码，**任何人都能解码**。Kubernetes 默认仅把 Secret 的 data 字段做 Base64 便于传输。**真正安全**需要：

- etcd 静态加密（`EncryptionConfiguration`）
- 严格 RBAC（限定读 Secret 的主体）
- 外部 Secret 管理系统（Vault / AWS SM）

---

## 12. 调度与资源管理

### 12.1 资源请求与限制

```yaml
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```

- **requests**：调度器按此预留；也影响 QoS
- **limits**：运行时硬上限；CPU 超会被限流，内存超会被 **OOMKilled**
- **CPU 单位**：`1` = 1 核，`100m` = 0.1 核
- **内存单位**：`Mi`（二进制）/ `M`（十进制）

### 12.2 QoS 等级

- **Guaranteed**：每容器 requests == limits（最高等级，最后被驱逐）
- **Burstable**：至少有一个 requests/limits 不相等
- **BestEffort**：都没设（最低，最先被驱逐）

### 12.3 调度机制

- **nodeSelector**：简单标签匹配
- **Affinity / AntiAffinity**：灵活的软/硬亲和
- **Taints & Tolerations**：节点"排斥"，Pod"容忍"
- **Topology Spread Constraints**：按可用区/节点打散

示例：反亲和把同应用副本分散到不同节点：

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels: { app: web }
```

### 12.4 Taints & Tolerations

```bash
kubectl taint nodes gpu-node gpu=true:NoSchedule
```

```yaml
tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

专门的 GPU/大内存节点只让匹配的 Pod 上去。

### 12.5 PriorityClass 与抢占

高优先级 Pod 可抢占低优先级；集群资源紧张时保护关键服务。

### 📝 笔试题 12-1：Pod 被 OOMKilled 了怎么排查？

1. `kubectl describe pod <pod>` 看 `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`
2. 比对 `limits.memory` 与应用实际占用
3. 调查代码内存泄漏（heap dump / pprof）
4. 是否 JVM、Node 等 runtime 需要显式设置**容器感知的内存参数**（老 JDK 默认看宿主而非 cgroup 限制）
5. 必要时调大 limits 或优化内存使用

---

## 13. 自动扩缩容与自愈

### 13.1 HPA（Horizontal Pod Autoscaler）

根据 CPU / Memory / 自定义指标扩缩**副本数**：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

- 依赖 **metrics-server**（资源指标）或 **Prometheus Adapter**（自定义）
- **KEDA** 可按事件驱动（Kafka Lag、SQS、Redis、HTTP QPS）

### 13.2 VPA（Vertical Pod Autoscaler）

根据实际用量调整 requests/limits（一般用在离线任务，在线服务慎用，会引起重启）。

### 13.3 Cluster Autoscaler

节点不够时**自动加机器**：

- 观察 Pending Pod，按需扩节点组
- 空闲阈值触发缩容
- 与云厂商集成（AWS ASG、GCE MIG）

### 13.4 Karpenter（推荐，尤其在 EKS）

AWS 开源的现代调度器：

- **无需预定义节点组**：按 Pod 需求挑合适 EC2 类型
- 调度更快（秒级）
- 成本优化（自动选 spot、捆绑到合适规格）
- 成为 EKS 社区主流选择

### 13.5 自愈机制

- **Pod 崩溃**：controller 自动重建
- **节点失联**：Pod 被驱逐到其他节点
- **liveness 失败**：重启容器
- **readiness 失败**：摘出 Service

**PodDisruptionBudget** 保护业务不被运维动作 OT 击穿：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web-pdb }
spec:
  minAvailable: 2            # 同时最多让 1 个不可用（副本 3 时）
  selector: { matchLabels: { app: web } }
```

### 📝 笔试题 13-1：节点被排空（drain）时 Pod 何去何从？

- 被驱逐的 Pod 根据所属控制器（Deployment/StatefulSet）重新调度到其他节点
- DaemonSet 的 Pod 不受 drain 直接影响（除非 `--ignore-daemonsets=false`）
- **PDB** 确保关键服务不会因为 drain 造成副本数低于阈值

---

## 14. RBAC、命名空间与多租户

### 14.1 Namespace

逻辑隔离单元。默认常见：`default`、`kube-system`、`kube-public`、`kube-node-lease`。

生产实践：按业务/团队划分 namespace，资源配额与 NetworkPolicy 分别绑定。

### 14.2 RBAC 核心对象

- **Role** / **ClusterRole**：权限集（可做什么操作）
- **RoleBinding** / **ClusterRoleBinding**：把角色绑定给用户 / 组 / ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: prod, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: prod, name: read-pods }
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 14.3 ServiceAccount

Pod 默认挂载 `default` SA，通过 token 与 API Server 交互。应用需要调用 K8s API（如 Operator）或**云资源**（EKS IRSA）时，创建专用 SA 并绑定精确权限。

### 14.4 资源配额

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { namespace: team-a, name: default }
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

配合 **LimitRange** 设置默认 requests/limits，避免用户漏填。

### 14.5 多租户选项

| 方案 | 隔离度 | 成本 |
|------|--------|------|
| **命名空间** | 弱 | 低 |
| **虚拟集群 (vCluster)** | 中 | 中 |
| **独立集群 / 独立 VPC** | 强 | 高 |

强安全要求优先**独立集群**。

### 📝 笔试题 14-1：Pod 里想调用 K8s API，怎么做？

1. 创建 ServiceAccount 并通过 RoleBinding 授予精确权限
2. Pod `spec.serviceAccountName` 指定该 SA
3. Pod 内 `/var/run/secrets/kubernetes.io/serviceaccount/token` 自动挂载 token
4. 应用用该 token 作为 Bearer 调用 `https://kubernetes.default.svc`

用官方 client-go / Python kubernetes / Java Fabric8 client 自动拾取即可。

---

## 15. Helm 与 GitOps

### 15.1 Helm

K8s 的包管理器。核心概念：

- **Chart**：应用模板包
- **Values**：参数文件（按环境覆盖）
- **Release**：一次安装的实例

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install myapp bitnami/redis --values my-values.yaml
helm upgrade --install myapp ./mychart -f values-prod.yaml
helm rollback myapp 2
helm list
```

Chart 目录：

```
mychart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    _helpers.tpl
```

### 15.2 Kustomize

无模板变量，纯**声明式 + 补丁**：

```
base/
  deployment.yaml
  kustomization.yaml
overlays/
  dev/kustomization.yaml
  prod/kustomization.yaml
```

```bash
kubectl apply -k overlays/prod/
```

对比：**Helm** 灵活但模板复杂；**Kustomize** 简单直观、差异化管理清晰。很多团队混用：Helm 装第三方，Kustomize 管自有服务。

### 15.3 GitOps

- Git 是**唯一真相源**
- 所有集群状态由声明式仓库描述
- **Pull 模式**：Agent 在集群内拉取并 reconcile
- 工具：**ArgoCD**（推荐）、**Flux**
- 优势：可审计、可回滚、PR 评审即发布

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: web-prod }
spec:
  destination:
    namespace: prod
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/org/manifests
    path: overlays/prod
    targetRevision: main
  project: default
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

### 📝 笔试题 15-1：Helm 和 Kustomize 怎么选？

- **Helm**：安装第三方、参数复杂、需要共享复用 → 首选
- **Kustomize**：自有服务、强调"基础 + 环境差异" → 首选
- **混合**：用 `helmfile` / ArgoCD 同时管理两者

---

## 16. 可观测性与运维

### 16.1 日志

- **标准输出/错误** 由 kubelet 采集
- 节点 DaemonSet 运行 **Fluent Bit / Fluentd**，推送到 **Loki / Elasticsearch / CloudWatch / Datadog**
- 日志结构化 + trace_id

### 16.2 指标

- **Prometheus**：拉模式，scrape 各服务 `/metrics`
- **kube-state-metrics**：集群对象状态
- **node-exporter**：节点硬件
- **Grafana**：可视化
- **Alertmanager**：告警

托管替代：**Managed Prometheus (AMP)**、Datadog、New Relic。

### 16.3 链路追踪

- **OpenTelemetry** → Jaeger / Tempo / X-Ray
- 网关/Sidecar 注入 `traceparent`；应用透传

### 16.4 集群健康排障速查

```bash
kubectl get events --sort-by=.lastTimestamp -A | tail -50
kubectl top pod -A | sort -k3 -h
kubectl top node
kubectl describe pod <pod>
kubectl logs <pod> --previous          # 上一次容器日志（崩溃后）
kubectl get pods -A -o wide --field-selector status.phase!=Running
kubectl get pv,pvc -A
kubectl get node -o wide
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl cordon <node> / uncordon <node>
```

### 📝 笔试题 16-1：Pod 一直 `Pending` 可能原因？

常见：

1. 节点资源不足（请求 CPU/Mem 过大）
2. 匹配不到节点（nodeSelector / Affinity / Taint）
3. PVC 未绑定（StorageClass 问题、容量不够）
4. 调度器问题（Scheduler 挂了、污点）
5. 超过命名空间 ResourceQuota
6. 镜像拉不下来（`ImagePullBackOff`，严格说不是 Pending 而是 CreatingContainer 阶段，但常并发）

`kubectl describe pod` 看 Events 基本能一眼定位。

---

## 17. Amazon EKS 入门

### 17.1 EKS 是什么

AWS 托管的 Kubernetes 服务：

- **控制平面全托管**（多 AZ，自动升级由 AWS 管）
- **数据平面**由用户管理（EC2 节点、Fargate、Karpenter）
- 深度集成 AWS 生态：**VPC CNI / IAM / ALB / EBS / EFS / CloudWatch**

### 17.2 架构

```
┌── Control Plane (AWS 托管) ──┐
│  API Server / etcd / ...     │
│  跨 3 个 AZ                  │
└──────────────────────────────┘
      │
┌──────────────── 数据面 ──────────────┐
│  Managed Node Groups (EC2 ASG)      │
│  Self-Managed Nodes                 │
│  Fargate Profiles (无服务器容器)    │
│  Karpenter (按需扩容)               │
└─────────────────────────────────────┘
```

### 17.3 创建集群（eksctl）

```bash
eksctl create cluster \
  --name prod \
  --region us-east-1 \
  --version 1.28 \
  --nodegroup-name ng-1 \
  --node-type m6i.large \
  --nodes 3 --nodes-min 2 --nodes-max 10 \
  --with-oidc \
  --managed

aws eks update-kubeconfig --name prod --region us-east-1
kubectl get nodes
```

或用 **Terraform / CloudFormation / CDK**（生产推荐）。

### 17.4 节点方案对比

| 方案 | 特点 | 适合 |
|------|------|------|
| **Managed Node Groups** | AWS 托管 ASG，自动升级和 bootstrap | 日常稳定工作负载 |
| **Self-Managed Nodes** | 完全自己 ASG/启动模板 | 深度定制 |
| **Fargate** | 无服务器容器，每 Pod 单机 | 隔离性要求高、突发 |
| **Karpenter** | 按需选择 EC2 规格，秒级扩容 | 现代首选，省成本 |

### 17.5 版本管理

- EKS 支持的 K8s 版本窗口通常 4 个以内
- **强制升级**政策：旧版本 EOL 后 AWS 会强制升到下一版
- 升级顺序：**控制平面 → Add-on → 节点组**
- 升级前：备份 etcd、评估 API 弃用、测试环境演练

### 📝 笔试题 17-1：EKS 和自建 K8s 的区别？

- EKS：**控制面免运维**（AWS 管 API Server / etcd / 升级），深度集成 AWS
- 自建：控制面自管，灵活但责任大

金钱 vs 时间的权衡：**大多数生产团队选 EKS/GKE/AKS 托管**，除非有非常特定的需求（合规、特殊网络、成本极限控制）。

---

## 18. EKS 核心集成与最佳实践

### 18.1 IRSA（IAM Roles for Service Accounts）

**EKS 最核心的身份能力**：让 Pod 用 AWS IAM 角色访问 AWS 资源，无需把 AccessKey 放 Secret。

```bash
# 1. 集群开启 OIDC Provider
eksctl utils associate-iam-oidc-provider --cluster prod --approve

# 2. 为 SA 绑定 IAM 角色
eksctl create iamserviceaccount \
  --cluster prod \
  --namespace app \
  --name s3-uploader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

Pod spec 使用：

```yaml
spec:
  serviceAccountName: s3-uploader
```

**EKS Pod Identity**（2023+）是更新的替代，简化了 OIDC 步骤。

### 18.2 VPC CNI

- Pod **直接拿 VPC IP**，与 EC2 同网段
- 无 overlay 开销，性能好
- **代价**：每实例 IP 数受 ENI/subnet 容量限制 → 规划好 CIDR（/16 起步），考虑 **prefix delegation** 提升密度

### 18.3 AWS Load Balancer Controller

让 K8s Ingress / Service 自动创建 **ALB / NLB**：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...:certificate/xxx
spec:
  rules: ...
```

推荐 **target-type=ip**（直接指向 Pod IP），延迟低、健康检查准确。

### 18.4 存储集成

- **EBS CSI Driver**：RWO 块存储（`gp3` 推荐）
- **EFS CSI Driver**：RWX NFS，适合多 Pod 共享
- **FSx for Lustre / OpenZFS**：高性能

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: gp3 }
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
```

### 18.5 自动扩容

**Karpenter（推荐）**：

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata: { name: default }
spec:
  template:
    spec:
      requirements:
        - { key: kubernetes.io/arch, operator: In, values: [amd64] }
        - { key: karpenter.k8s.aws/instance-category, operator: In, values: [m,c,r] }
        - { key: karpenter.k8s.aws/instance-generation, operator: Gt, values: ["4"] }
        - { key: karpenter.sh/capacity-type, operator: In, values: [spot, on-demand] }
      nodeClassRef:
        name: default
  limits: { cpu: "1000" }
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

- 按 Pod 需求秒级启 EC2
- 智能组合 spot + 按需实例
- 自动合并低利用节点

### 18.6 日志与监控

- **CloudWatch Container Insights**：开箱即用（Fluent Bit + CW agent）
- **Amazon Managed Prometheus (AMP)** + **Managed Grafana (AMG)**
- 第三方：Datadog、New Relic、Dynatrace 均有 EKS 集成

### 18.7 安全最佳实践

- **IRSA / Pod Identity** 代替静态凭证
- **最小 IAM 权限**
- 私有 API Endpoint（`endpointPrivateAccess: true`）
- 节点在私有子网，通过 NAT 出网
- **Secrets** 存 **AWS Secrets Manager / Parameter Store**，用 External Secrets Operator 同步
- **镜像扫描**：ECR 启用增强扫描
- **网络策略**：Calico / Cilium 加强
- **GuardDuty EKS Protection**：威胁检测
- **审计日志**：开启 EKS audit 日志投到 CloudWatch Logs

### 18.8 成本优化

- **Karpenter + Spot**：可节省 50%+
- **合理 requests/limits**：避免 over-request
- **使用 Graviton（ARM）**：性能/美元更好
- **Savings Plans / Reserved Instance** 打底
- **关闭非工作时长的 dev 集群**
- **定期 `kubectl top` 审视 Pod 资源配置**

### 18.9 高可用

- **至少 3 个 AZ**：控制面 AWS 已多 AZ；节点组跨 AZ
- **PDB + 反亲和** 保证业务 Pod 跨 AZ 分散
- **多集群 / 多区域**：关键业务 DR
- **定期演练**：kill Pod / drain 节点 / Chaos Mesh

### 📝 笔试题 18-1：Pod 需要访问 S3 和 DynamoDB，如何授权？

使用 **IRSA**：

1. 为 IAM 角色附加 S3 / DynamoDB 的权限策略
2. 创建 ServiceAccount 并把 IAM Role 绑定（注解 `eks.amazonaws.com/role-arn`）
3. Pod spec 指定 `serviceAccountName`
4. AWS SDK 自动通过 OIDC 拿到临时凭证

避免用户把静态 AccessKey 放在 Secret——那是反模式。

### 📝 笔试题 18-2：EKS 节点 IP 不够用了怎么办？

- 检查 **VPC / 子网 CIDR**，评估是否扩大或加副子网
- 启用 **VPC CNI prefix delegation**：一个 ENI 能分配 `/28` 的 16 个 IP，而非每 IP 一个 ENI
- 换更大机型（ENI 和 IP 容量更高）
- 引入 **Cilium / Calico VXLAN** 避免直占 VPC IP
- 使用 **IPv6 集群** 根治（较新，需规划）

---

## 19. 综合笔试练习

### 19.1 选择题

**Q1** 下列哪项**不是**容器实现隔离的基础？
A. Namespace  B. Cgroups  C. Union FS  D. Hypervisor

<details><summary>答案</summary>D。</details>

**Q2** Dockerfile 中的 `CMD` 与 `ENTRYPOINT` 区别？
A. 完全等价
B. ENTRYPOINT 是命令主体，CMD 是默认参数
C. CMD 不能被覆盖
D. ENTRYPOINT 只能在 FROM 之前

<details><summary>答案</summary>B。</details>

**Q3** K8s 中每个 Pod 拥有一个 IP，这个 IP 是？
A. 节点 IP
B. 集群外部 LB IP
C. Pod 网络内的 IP（可变）
D. 容器名字解析结果

<details><summary>答案</summary>C。</details>

**Q4** 下列哪个对象适合部署**有状态 DB**？
A. Deployment  B. DaemonSet  C. StatefulSet  D. Job

<details><summary>答案</summary>C。</details>

**Q5** `livenessProbe` 失败会？
A. 从 Service 摘除
B. 重启容器
C. 增加副本
D. 告警

<details><summary>答案</summary>B。摘除是 `readinessProbe`。</details>

**Q6** EKS 中让 Pod 访问 S3 的**推荐**方式？
A. 把 AccessKey 放 Secret
B. 节点 IAM Role 下放
C. IRSA（IAM Roles for Service Accounts）
D. hardcode 在代码里

<details><summary>答案</summary>C。</details>

**Q7** Ingress 的主要作用？
A. 四层 TCP 分发
B. 七层 HTTP/HTTPS 路由
C. 替代 Service
D. 存储持久化

<details><summary>答案</summary>B。</details>

**Q8** 下列关于 PV 回收策略，错误的是？
A. Retain 保留数据
B. Delete 随 PVC 一起删
C. 生产数据建议用 Delete
D. 默认策略取决于 StorageClass

<details><summary>答案</summary>C（建议 Retain）。</details>

### 19.2 判断题

1. Docker 容器必须以 root 运行。 ❌（`USER` 可指定非 root）
2. 镜像分层可以大幅减少拉取流量。 ✅
3. `docker rm` 会删除容器使用的命名卷。 ❌（需 `-v`）
4. Kubernetes 的 Service 就是云上的 LB。 ❌（ClusterIP 仅内部）
5. 默认 Pod 之间是网络隔离的。 ❌（默认全通，需 NetworkPolicy）
6. Secret 默认是加密存储的。 ❌（仅 Base64 编码）
7. EKS 控制面由 AWS 全托管。 ✅
8. Karpenter 需要预先定义节点组。 ❌

### 19.3 简答题

**Q1** 简述 Docker 构建缓存命中规则。

按**指令序列**从前往后检查：

- 基础镜像相同 → 共用
- 指令文本相同且**涉及文件内容哈希相同** → 命中缓存
- 一旦某层 miss，之后的层**全部重新构建**

所以改动概率小的指令放前面（`FROM`、`COPY package*.json`、`RUN npm ci`），变化频繁的（源码）放后面。

**Q2** K8s 集群一个 Pod 处于 `CrashLoopBackOff`，排查步骤？

1. `kubectl describe pod` 看 Events 和 Last State
2. `kubectl logs <pod> --previous` 看上次退出的日志
3. `kubectl exec` 或临时改 `command` 为 `sleep infinity` 进容器调试
4. 检查 probe：是否 liveness 探测过严、应用启动慢但没 startupProbe
5. 检查资源限制：是否 OOMKilled
6. 检查配置/Secret 挂载正确性
7. 检查依赖服务是否可达（DB/Redis）

**Q3** EKS 上部署一个 HTTP 服务全流程大纲？

1. Dockerfile + 多阶段构建
2. 推送镜像到 **ECR**（或 Docker Hub）
3. K8s manifests：Deployment + Service + Ingress（ALB 注解）
4. ConfigMap / Secret；敏感走 Secrets Manager + ESO
5. IRSA 授予 Pod 所需 AWS 权限
6. HPA / Karpenter 扩缩容
7. Helm / ArgoCD 部署
8. Route53 指向 ALB；ACM 证书
9. 观测：CloudWatch / AMP-AMG / Datadog
10. 告警、PDB、反亲和、健康检查

**Q4** 如何在 K8s 上进行蓝绿 / 金丝雀发布？

- **蓝绿**：同时跑两版 Deployment，切 Service selector 即切流量；或由 Ingress/Gateway 路由切换
- **金丝雀**：
  - 手工：两个 Deployment（v1、v2），副本数按比例，一个 Service 聚合
  - Ingress 权重：Nginx Ingress、ALB 的 `WeightedTargetGroups`
  - 工具：**Argo Rollouts** / **Flagger**，自动根据指标推进或回滚

### 19.4 实操/设计题

**Q1** 写一个最小 Dockerfile：Python + Flask，非 root，多阶段。

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir -w /wheels -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
RUN useradd -u 1000 app
COPY --from=build /wheels /wheels
RUN pip install --no-cache-dir /wheels/* && rm -rf /wheels
COPY . .
USER app
EXPOSE 8000
HEALTHCHECK CMD wget -qO- http://localhost:8000/healthz || exit 1
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

**Q2** 写一个 Deployment + Service + Ingress（ALB）YAML，副本 3，滚动更新。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web, namespace: app }
spec:
  replicas: 3
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      serviceAccountName: web
      containers:
        - name: web
          image: 1234567890.dkr.ecr.us-east-1.amazonaws.com/web:1.0
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          readinessProbe: { httpGet: { path: /readyz, port: 8080 }, periodSeconds: 5 }
          livenessProbe:  { httpGet: { path: /livez,  port: 8080 }, periodSeconds: 10, initialDelaySeconds: 30 }
---
apiVersion: v1
kind: Service
metadata: { name: web, namespace: app }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:...:certificate/xxx
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

**Q3** 设计一个 EKS 生产集群的最佳实践清单（≥ 10 项）。

1. 至少 **3 AZ** 节点组
2. 私有 API endpoint，节点在私有子网
3. **IRSA / Pod Identity** 授权，不放 AccessKey
4. **Karpenter** + Spot 混合
5. 镜像存 **ECR**，开启扫描
6. **External Secrets Operator + Secrets Manager**
7. **PDB + 反亲和 + TopologySpreadConstraints**
8. **HPA** + metrics-server 或 KEDA
9. **AWS Load Balancer Controller** 暴露服务
10. 日志送 **CloudWatch / Loki**，指标 **AMP**，追踪 **X-Ray / Tempo**
11. **NetworkPolicy**（Calico / Cilium）
12. **GuardDuty EKS Protection** + Audit 日志
13. **升级演练**：控制面 → Add-on → 节点组
14. **Chaos Mesh / AWS FIS** 混沌测试
15. 成本：Savings Plan + Graviton + 规律审视 Pod requests

**Q4** 生产发生"某 Service 的新版本发布后错误率飙升"，回滚思路？

1. **立刻回滚**：`kubectl rollout undo deploy/web` 或 ArgoCD 上 revert Git commit
2. 观察监控：错误率、延迟是否恢复
3. 保留现场：拿到异常 Pod 日志、事件、metrics，dump heap（Java）
4. 关闭相关 feature flag（如果有）
5. 复盘：CI 测试盲点 / 缺少契约测试 / canary 阶段指标未监控
6. 加强：PR 必须经金丝雀 + 观测门禁再推进到全量

---

## 📚 学习建议

1. **先动手跑起来**：单机 Docker → minikube / kind → 云上 EKS，每步都实操
2. **读 kubectl --help 与 `kubectl explain`**：官方文档最权威
3. **理解声明式思想**：YAML 只是形式，"期望 vs 实际 + 控制循环"是本质
4. **掌握两个 API**：Kubernetes API（Pod/Service/Deployment 等）与 AWS API（IAM/VPC/ELB）
5. **生产意识**：从第一天就想 **探针 / 资源 / 安全 / 观测 / 备份 / 成本**
6. **关注生态**：Helm / ArgoCD / Karpenter / OpenTelemetry / Gateway API 是当下必备

> 祝你的容器轻量高效，集群稳稳运行。
