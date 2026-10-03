# kind 部署指南

> 本文档记录使用 **kind（Kubernetes IN Docker）** 在本地快速搭建 Kubernetes 集群的完整方法，覆盖单节点、多节点、多控制面（HA）、Ingress 端口映射、本地镜像仓库、持久化存储、CI 集成与故障排查。
>
> 文档基准版本：**kind `v0.33.0`**（2026-08 发布），默认节点镜像 `kindest/node:v1.37.0`（Kubernetes `v1.37.0`）。
> 版本会持续更新，安装前建议查看 <https://github.com/kubernetes-sigs/kind/releases> 确认最新版本与节点镜像 digest。

## 目录

- [1. kind 简介与选型](#1-kind-简介与选型)
- [2. 环境准备](#2-环境准备)
- [3. 快速开始](#3-快速开始)
- [4. 集群配置文件详解](#4-集群配置文件详解)
- [5. 多节点与 HA 集群](#5-多节点与-ha-集群)
- [6. 端口映射与 Ingress](#6-端口映射与-ingress)
- [7. 本地镜像与私有仓库](#7-本地镜像与私有仓库)
- [8. 持久化存储](#8-持久化存储)
- [9. 部署本仓库的练习清单](#9-部署本仓库的练习清单)
- [10. 常用操作](#10-常用操作)
- [11. CI 集成](#11-ci-集成)
- [12. 与 K3s / minikube 的选择](#12-与-k3s--minikube-的选择)
- [13. 故障排查](#13-故障排查)
- [14. 附录](#14-附录)

---

## 1. kind 简介与选型

### 1.1 什么是 kind

kind 是 Kubernetes SIG 官方维护的**用 Docker 容器充当 K8s 节点**的工具：每个「节点」就是一个运行 systemd + kubelet + containerd 的 Docker 容器，集群则通过 kubeadm 在容器内引导生成。

```
┌─────────────────────── 宿主机（Windows / macOS / Linux）───────────────────────┐
│  Docker Engine                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ kind-control │  │ kind-worker  │  │ kind-worker2 │  │ external-load-    │  │
│  │  -plane      │  │              │  │              │  │ balancer (haproxy)│  │
│  │ systemd      │  │ systemd      │  │ systemd      │  │  多控制面时自动创建 │  │
│  │ kubelet      │  │ kubelet      │  │ kubelet      │  └───────────────────┘  │
│  │ containerd   │  │ containerd   │  │ containerd   │                         │
│  │ etcd/apiserver│ │ kube-proxy   │  │ kube-proxy   │                         │
│  └──────────────┘  └──────────────┘  └──────────────┘                         │
└───────────────────────────────────────────────────────────────────────────────┘
        ↑ kubectl 通过 127.0.0.1:<随机端口> 访问（由 kind 映射到容器）
```

### 1.2 核心特点

| 特性 | 说明 |
|---|---|
| 官方血统 | Kubernetes SIG 项目，**K8s 自身 CI 就用 kind 测试**，行为与真实集群高度一致 |
| 节点即容器 | 无需虚拟机，秒级创建；删除集群 = 删除容器，环境绝对干净 |
| 多节点 / 多控制面 | 支持任意节点拓扑，多控制面时自动创建 haproxy 外部负载均衡容器 |
| 配置能力强 | 完整支持 `kubeadmConfigPatches`、`containerdConfigPatches`，可模拟大量真实场景 |
| 启动极快 | 单节点约 20~60 秒（镜像已缓存时更快） |
| 镜像可预加载 | `kind load docker-image` 免去在节点内拉取镜像 |
| 天生适合 CI | 官方提供 GitHub Actions，用完即焚 |

### 1.3 局限性（务必了解）

| 限制 | 说明 |
|---|---|
| **节点数据非持久** | 删除集群即丢失全部数据；需持久化的内容必须用 `extraMounts` 挂载到宿主机 |
| 不模拟真实网络 | 节点间为 Docker 桥接网络，没有真实物理网络延迟/丢包，**不能用于性能压测** |
| 负载均衡是容器 | HA 场景的 LB 是 haproxy 容器，无法验证真实 LB（F5/LVS）行为 |
| 不支持跨主机 | 所有节点必须在同一台宿主机上 |
| 依赖 Docker | 需 Docker Engine / Docker Desktop（也支持 rootful Podman） |
| Windows/macOS 网络受限 | Docker Desktop 下容器 IP 不可从宿主机直接访问，NodePort 需借助端口映射或 `port-forward` |
| 生产不可用 | 定位是开发与测试，**不要用于生产环境** |

### 1.4 适用场景

- 本地验证 YAML 清单、Operator、Helm Chart 是否可用
- CI/CD 流水线中的集成测试（GitHub Actions / GitLab CI）
- 学习 K8s 对象与多节点调度、污点、亲和性等概念
- 复现上游 issue、验证版本升级行为

> 💡 本仓库的分工：**kind = 免服务器、快速验证**（本目录）；**K3s = 真实可用的常驻集群**（见 [K3s 部署指南](../K3s/README.md)）。两者共用同一套 YAML 清单。

---

## 2. 环境准备

### 2.1 依赖与资源

| 依赖 | 要求 |
|---|---|
| Docker | Docker Engine 20.10+ / Docker Desktop（Windows、macOS）；也支持 rootful Podman |
| kubectl | 建议与节点镜像的 K8s 版本相差不超过 ±1 个次版本 |
| 内存 | 单节点 ≥ 2 GB 可用；**1 控制面 + 3 worker 建议 ≥ 8 GB** |
| 磁盘 | 每个节点镜像约 1~2 GB，多集群会累积占用 |
| CPU | 多节点建议 ≥ 4 核 |

> ⚠️ **Docker Desktop 用户**：默认内存配额通常只有 2 GB，创建多节点集群会卡在 `Starting control-plane`。请先在 **Settings → Resources** 中把内存调到 8 GB 以上。

### 2.2 安装 kind

**Windows**

```powershell
# 方式一：winget（推荐）
winget install Kubernetes.kind

# 方式二：Chocolatey
choco install kind

# 方式三：Scoop
scoop install kind

# 方式四：手动下载
# 从 https://github.com/kubernetes-sigs/kind/releases 下载 kind-windows-amd64
# 重命名为 kind.exe 并放入 PATH 中的目录

# 安装 kubectl
winget install Kubernetes.kubectl
```

**macOS / Linux**

```bash
# macOS
brew install kind kubectl

# Linux（amd64，注意锁定版本与校验）
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# 校验 digest（官方发布页提供 .sha256sum 文件）
curl -Lo kind.sha256sum https://github.com/kubernetes-sigs/kind/releases/download/v0.33.0/kind-linux-amd64.sha256sum
sha256sum -c kind.sha256sum --ignore-missing

# 通过 Go 安装
go install sigs.k8s.io/kind@v0.33.0
```

### 2.3 验证环境

```bash
kind version           # 期望：kind v0.33.0 go1.xx windows/amd64
docker version         # 确认 Docker 正常运行
docker info | grep -i -E 'cgroup|memory'   # 查看 cgroup 版本与内存配额
kubectl version --client
```

**Docker 资源检查（关键）：**

```bash
# Docker Desktop：确认可用内存（多节点集群至少 8 GB）
docker info --format '{{.MemTotal}}'
```

**cgroup 说明：** 现代发行版与 Docker Desktop 默认使用 cgroup v2，兼容性最好。若宿主机为 cgroup v1（较老的 CentOS 7 等），部分较新的节点镜像可能启动失败，此时可改用较早的节点镜像或升级内核/发行版。

---

## 3. 快速开始

### 3.1 创建默认集群（单节点）

```bash
# 最小命令：创建名为 kind 的单节点集群（控制面 + 可调度）
kind create cluster

# 指定名称与版本
kind create cluster --name dev --image kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5

# 等待就绪并设置超时
kind create cluster --name dev --wait 5m
```

> 📌 官方要求**使用 `@sha256` digest 固定节点镜像**，避免同名 tag 被覆盖导致行为不一致（见 [发布说明](https://github.com/kubernetes-sigs/kind/releases/tag/v0.33.0)）。

kind 会自动完成：

1. 拉取节点镜像并启动节点容器（容器名 `<集群名>-control-plane`、`<集群名>-worker` …）
2. 在容器内用 kubeadm 引导集群（启动 etcd、apiserver、controller-manager、scheduler、CoreDNS、kube-proxy）
3. 生成 kubeconfig 并写入 `~/.kube/config`，上下文名为 `kind-<集群名>`
4. 将 API Server 端口映射到宿主机 `127.0.0.1:<随机端口>`

### 3.2 验证集群

```bash
kubectl cluster-info --context kind-kind
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass          # kind 默认提供 standard（local-path-provisioner）
```

预期输出：

```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   45s   v1.37.0
```

### 3.3 删除集群

```bash
kind delete cluster --name dev        # 删除指定集群
kind delete clusters --all            # 删除全部集群
```

### 3.4 部署一个应用验证

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx

# Windows / macOS（Docker Desktop）下容器 IP 不可直连，用 port-forward 验证
kubectl port-forward svc/nginx 8080:80
# 浏览器访问 http://localhost:8080

# Linux 下可直接用节点容器 IP 访问 NodePort
NODE_IP=$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' kind-control-plane)
NODEPORT=$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODEPORT
```

---

## 4. 集群配置文件详解

kind 配置为声明式 YAML，`apiVersion` 固定为 `kind.x-k8s.io/v1alpha4`。

### 4.1 顶层字段

```yaml
kind: Cluster                        # 固定值
apiVersion: kind.x-k8s.io/v1alpha4   # 固定值
name: my-cluster                     # 集群名（也可由 --name 覆盖）
nodes: []                            # 节点定义（见 4.2）
networking: {}                       # 网络配置（见 4.3）
kubeadmConfigPatches: []             # kubeadm 配置补丁（见 4.4）
containerdConfigPatches: []          # containerd 配置补丁（见 7.3）
featureGates: {}                     # 特性门控，如 {"ExpandCSIVolumes": true}
runtimeConfig: {}                    # API Server 运行时配置，如 {"api/alpha": "true"}
```

### 4.2 nodes 字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `role` | string | **必填**，`control-plane` 或 `worker` |
| `image` | string | 节点镜像，**建议带 `@sha256` digest 固定** |
| `labels` | map | 节点标签，如 `tier: backend` |
| `extraPortMappings` | list | 宿主机 ↔ 节点容器端口映射（见 [第 6 章](#6-端口映射与-ingress)） |
| `extraMounts` | list | 宿主机 ↔ 节点容器目录挂载（见 [第 8 章](#8-持久化存储)） |
| `kubeadmConfigPatches` | list | 该节点专属的 kubeadm 补丁（优先级高于全局） |

```yaml
nodes:
  - role: control-plane
    image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
    labels:
      node-role: control
  - role: worker
    labels:
      disktype: ssd
  - role: worker
```

> ⚠️ 顺序即创建顺序；节点容器名依次为 `<集群名>-control-plane`、`<集群名>-worker`、`<集群名>-worker2`…（第 2 个及之后的 worker 带数字后缀）。

### 4.3 networking 字段

| 字段 | 默认值 | 说明 |
|---|---|---|
| `ipFamily` | `ipv4` | 可选 `ipv4` / `ipv6` / `dual` |
| `apiServerAddress` | `127.0.0.1` | API 监听地址；改为 `0.0.0.0` 可让局域网其他机器访问（开发便利，注意安全） |
| `apiServerPort` | 随机 | 固定 API 端口（便于脚本与工具配置固定 kubeconfig） |
| `podSubnet` | `10.244.0.0/16` | Pod 网段 |
| `serviceSubnet` | `10.96.0.0/12` | Service 网段 |
| `disableDefaultCNI` | `false` | 设为 `true` 时不装默认 CNI（用于自装 Calico/Cilium） |
| `kubeProxyMode` | `iptables` | 可改 `ipvs` / `nftables` |
| `dnsSearch` | — | 为节点注入自定义 DNS 搜索域 |

```yaml
networking:
  ipFamily: ipv4
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443                 # 固定端口，多集群时务必错开
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  disableDefaultCNI: false
  kubeProxyMode: "ipvs"
```

> ⚠️ **同一宿主机运行多个集群时，`apiServerPort` 必须不同**，否则第二个集群创建失败。多集群建议不固定该字段，让 kind 自动分配。

### 4.4 kubeadmConfigPatches

用于修改 kubeadm 引导配置，是 kind 最强大的定制手段。补丁通过 `kind:` 字段区分作用对象：

```yaml
kubeadmConfigPatches:
  # ① 控制面初始化配置（InitConfiguration）
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  # ② 集群级配置（ClusterConfiguration）：指定控制面端点、启用审计等
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        audit-log-path: /var/log/kubernetes/audit.log
  # ③ 加入配置（JoinConfiguration）：影响 worker 节点
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        max-pods: "110"
```

常用场景：给节点打 `ingress-ready` 标签、设置 `controlPlaneEndpoint`、开启审计日志、调整 kubelet 参数。

---

## 5. 多节点与 HA 集群

### 5.1 本仓库现成的集群配置

| 文件 | 拓扑 | 特征 |
|---|---|---|
| [`../kind-4nodes.yaml`](../kind-4nodes.yaml) | 1 控制面 + 3 worker | 最简多节点练习配置 |
| [`kind-ha.yaml`](kind-ha.yaml) | 2 控制面 + 3 worker | 多控制面，且每个节点配置了 80/443/30080/30443 端口映射（宿主机端口各不相同） |
| [`kind-test.yaml`](kind-test.yaml) | 1 控制面 + 2 worker | 自定义网段、`disableDefaultCNI: true`、`kubeProxyMode: ipvs`、固定 API 端口 6443 |
| [`kind-config-single.yaml`](kind-config-single.yaml) | 1 控制面（可调度） | 最小可用配置 + 80/443 映射，用于 Ingress 练习 |
| [`kind-config-registry.yaml`](kind-config-registry.yaml) | 1 控制面 + 2 worker | 带本地镜像仓库（`localhost:5001`）+ Ingress 端口映射 |

**1 控制面 + 3 worker（最常用）：**

```bash
kind create cluster --config ../kind-4nodes.yaml
kubectl get nodes
```

**已有集群的端口映射含义**（[`kind-ha.yaml`](kind-ha.yaml)）：

| 节点 | 容器端口 80 → 宿主机 | 443 → 宿主机 | 30080 → 宿主机 | 30443 → 宿主机 |
|---|---|---|---|---|
| control-plane | 180 | 1443 | 10080 | 10443 |
| control-plane2 | 280 | 2443 | 20080 | 20443 |
| worker | 380 | 3443 | 30080 | 30443 |
| worker2 | 480 | 4443 | 40080 | 40443 |
| worker3 | 580 | 5443 | 55080 | 55443 |

> 💡 该配置的设计意图是：**每个节点的同一容器端口映射到不同的宿主机端口**，从而可以在宿主机上分别访问各个节点，避免端口冲突。这是同一宿主机跑多节点时的常见做法。

### 5.2 多控制面（HA）的行为

当配置中出现 **≥ 2 个 `control-plane` 节点**时，kind 会自动：

1. 创建额外容器 `<集群名>-external-load-balancer`（haproxy）作为 API Server 统一入口
2. 集群的 `controlPlaneEndpoint` 指向该负载均衡器
3. kubeconfig 中的 `server` 指向负载均衡器映射到宿主机的端口

```bash
# 查看负载均衡容器
docker ps --filter "name=external-load-balancer"

# 查看控制面端点
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'

# 查看 etcd 成员（多控制面 = 多 etcd 成员）
docker exec kind-ha-control-plane etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

自定义控制面端点（如需固定地址）：

```yaml
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    controlPlaneEndpoint: "192.168.31.100:6443"
```

> ⚠️ kind 的 HA **仅用于验证多控制面拓扑能否正常引导**，haproxy 容器不等同于真实负载均衡设备，无法用于验证真实故障切换与性能。

### 5.3 节点规划建议

| 场景 | 推荐拓扑 | 大概内存 |
|---|---|---|
| 快速验证单个对象 | 1 控制面（默认单节点） | 2 GB |
| 练习调度 / 亲和性 / DaemonSet | 1 控制面 + 2~3 worker | 4~8 GB |
| 验证多控制面 / etcd | 3 控制面 + 2 worker | 8~12 GB |
| 验证高可用 + Ingress | 3 控制面 + 3 worker | 12 GB+ |

---

## 6. 端口映射与 Ingress

### 6.1 为什么必须做端口映射

kind 的「节点」是 Docker 容器，其内部监听的端口**不会自动暴露到宿主机**。要让宿主机（及浏览器）访问集群内的 80/443，必须在集群配置里声明 `extraPortMappings`：

```yaml
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80        # 节点容器内监听的端口
        hostPort: 8080           # 宿主机端口（必须未被占用）
        listenAddress: "0.0.0.0" # 监听地址，0.0.0.0 允许局域网访问
        protocol: TCP            # TCP / UDP / SCTP
```

**`extraPortMappings` 字段表：**

| 字段 | 必填 | 说明 |
|---|---|---|
| `containerPort` | ✅ | 节点容器内的端口 |
| `hostPort` | ✅ | 宿主机端口，**同一宿主机上不可重复** |
| `listenAddress` | ❌ | 默认 `0.0.0.0`；设为 `127.0.0.1` 则仅本机可访问 |
| `protocol` | ❌ | 默认 `TCP` |

### 6.2 部署 ingress-nginx（kind 官方方式）

```bash
# 1) 创建带 80/443 映射的集群（使用本目录模板）
kind create cluster --config kind-config-single.yaml

# 2) 确认节点已带 ingress-ready 标签
kubectl get nodes --show-labels | grep ingress-ready
# 若没有该标签，手动补上（kind 的 ingress 清单依赖它做 nodeSelector）
kubectl label node kind-control-plane ingress-ready=true

# 3) 部署 kubeadm 兼容的 ingress-nginx（kind 专用清单：hostPort + 容忍控制面污点）
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# 4) 等待就绪
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

> 📌 必须使用 **kind 专用清单**（`provider/kind/deploy.yaml`），它使用 `hostPort: 80/443` 而非 LoadBalancer/NodePort，并容忍控制面污点。用通用 NodePort 清单会因 kind 没有 LB 实现而停在 `EXTERNAL-IP: <pending>`。
>
> 💡 kind 默认**不给控制面节点加 `NoSchedule` 污点**（与 kubeadm 默认行为不同），因此单节点集群可直接跑业务 Pod；确认方式：
> ```bash
> kubectl describe node kind-single-control-plane | grep Taints
> ```

### 6.3 验证 Ingress

```bash
# 部署测试应用
kubectl create deployment web --image=nginx:latest
kubectl expose deployment web --port=80

# 创建 Ingress
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF

# 访问（宿主机 80 端口已映射到节点容器 80）
curl http://localhost
```

**请求链路：** 宿主机 `:80` → 节点容器 `:80`（`extraPortMappings`）→ ingress-nginx（hostPort 80）→ Service → Pod。

### 6.4 NodePort 的访问差异（重要）

| 宿主机 | 访问方式 |
|---|---|
| **Linux** | 可直接访问节点容器 IP：`curl http://$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' kind-control-plane):<NodePort>` |
| **Windows / macOS（Docker Desktop）** | 容器 IP 从宿主机**不可路由**，两种方案：① `kubectl port-forward svc/<svc> 8080:<port>`；② 在集群配置中把该 NodePort 加进 `extraPortMappings`（需重建集群） |

> ⚠️ 很多「kind 里 NodePort 访问不了」的问题都源于此 —— 不是集群故障，而是 Docker Desktop 的网络隔离。

---

## 7. 本地镜像与私有仓库

### 7.1 为什么需要预加载镜像

节点容器内的 containerd 会独立拉取镜像。若镜像只在宿主机 Docker 中、或需要离线/加速，可用 `kind load` 直接把镜像导入节点，**无需重启集群**：

```bash
# 从宿主机 Docker 导入（镜像必须已存在于宿主机）
docker pull nginx:1.27
kind load docker-image nginx:1.27 --name dev

# 批量导入
kind load docker-image nginx:1.27 redis:7 alpine:3.20 --name dev

# 从 tar 归档导入（适合离线分发）
docker save myapp:v1 -o myapp.tar
kind load image-archive myapp.tar --name dev

# 验证：进入节点查看镜像
docker exec -it dev-control-plane crictl images | grep nginx
```

### 7.2 使用本地镜像仓库（推荐用于多节点）

多节点集群中逐节点 `load` 效率低，搭建一个本地 registry 更优雅：

```bash
# ── ① 启动 registry 容器 ──────────────────────────────
reg_name='kind-registry'
reg_port='5001'
docker run -d --restart=always \
  -p "127.0.0.1:${reg_port}:5000" \
  --network bridge \
  --name "${reg_name}" \
  registry:2

# ── ② 用带 containerdConfigPatches 的配置创建集群 ──────
kind create cluster --config kind-config-registry.yaml

# ── ③ 把 registry 接入 kind 网络（容器可解析名字的关键）──
docker network connect kind "${reg_name}"

# ── ④ 告诉集群本地仓库地址（供工具链发现，可选但推荐）──
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${reg_port}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF
```

**使用方式：** 镜像名带 `localhost:5001` 前缀即可自动路由到该仓库。

```bash
docker tag myapp:v1 localhost:5001/myapp:v1
docker push localhost:5001/myapp:v1          # 写入本地 registry

kubectl create deployment myapp --image=localhost:5001/myapp:v1
```

配置模板见 [`kind-config-registry.yaml`](kind-config-registry.yaml)，其核心是：

```yaml
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5001"]
      endpoint = ["http://kind-registry:5000"]
```

> 📌 说明：`localhost:5001` 是**集群内使用的镜像名前缀**（宿主机上暴露的端口），`http://kind-registry:5000` 是**节点容器在 Docker 网络内实际访问的地址**，二者端口不同是刻意设计。
>
> ⚠️ 较新的节点镜像使用 containerd 2.x，`registry.mirrors` 配置项已被标记为废弃但仍可用；如遇不生效，可改用 `extraMounts` 把 `hosts.toml` 挂载到节点内的 `/etc/containerd/certs.d/<registry>/`。

### 7.3 镜像加速（国内环境）

节点内拉取 Docker Hub 镜像较慢时，可通过 `containerdConfigPatches` 配置加速地址，本仓库 [`kind-test.yaml`](kind-test.yaml) 就是这种写法：

```yaml
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://<你的加速地址>"]
```

> ⚠️ **注意**：`kind-test.yaml` 中的 `https://0ebrf618.mirror.aliyuncs.com` 是**账号专属的阿里云加速地址**（形如 `<随机串>.mirror.aliyuncs.com`），通常已失效，且第三方镜像站可用性变化频繁。建议：
> - 优先使用 7.2 的本地 registry 方案（把镜像 `docker pull` + `push` 到本地仓库）
> - 或改用 `kind load docker-image` 直接导入
> - 若坚持用加速地址，请登录阿里云容器镜像服务控制台获取当前有效的专属地址

---

## 8. 持久化存储

### 8.1 默认存储与其生命周期

kind 默认已提供 `standard` StorageClass（基于 local-path-provisioner，与 K3s 的 `local-path` 同类）：

```bash
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY
# standard (default)   rancher.io/local-path   Delete
```

```yaml
# 直接使用默认存储类
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
```

> ⚠️ **关键认知**：数据实际存放在**节点容器内的 `/var/local-path-provisioner`**。`kind delete cluster` 后容器被删除，**数据一并丢失**。需要跨集群重建保留数据时，必须使用 `extraMounts`。

### 8.2 extraMounts：把宿主机目录挂进节点

```yaml
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /data/kind/storage        # 宿主机路径（Windows 用 /d/kind/storage 或 C:\... 形式见下）
        containerPath: /var/local-path-provisioner
        readOnly: false
        selinuxRelabel: false
        propagation: Bidirectional          # 让挂载在容器内外双向可见（挂载传播）
```

**Windows（Docker Desktop）路径写法：**

```yaml
    extraMounts:
      - hostPath: /c/Users/HL/kind-data     # Docker Desktop 下用 /c/... 形式
        containerPath: /var/local-path-provisioner
        propagation: Bidirectional
```

然后用 `hostPath` 直接挂载验证：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: busybox:latest
      command: ["sh", "-c", "echo hello > /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      hostPath:
        path: /var/local-path-provisioner
        type: DirectoryOrCreate
```

### 8.3 使用 NFS（复用本仓库的 PV 配置）

若在 kind 节点内使用本仓库的 [`../nfs-pv.yaml`](../nfs-pv.yaml)（NFS 服务器 `192.168.31.217:/nfs`），需满足：

1. **节点容器能访问 NFS 服务器**——节点属于 Docker 桥接网络，通常可经宿主机的默认路由访问局域网，先在节点内验证：
   ```bash
   docker exec -it kind-control-plane sh -c 'apt-get update -qq && apt-get install -y -qq nfs-common && mount -t nfs 192.168.31.217:/nfs /mnt && ls /mnt && umount /mnt'
   ```
   > kind 节点镜像基于 Debian，**默认未安装 `nfs-common`**，否则 kubelet 挂载 NFS 会报 `mount: wrong fs type` 或 `bad option`。
2. **每个节点都要装** `nfs-common`（worker 上运行 Pod 时会实际执行挂载）：
   ```bash
   for n in $(kind get nodes); do
     docker exec $n sh -c 'apt-get update -qq && apt-get install -y -qq nfs-common'
   done
   ```
3. 之后即可正常 `kubectl apply -f ../nfs-pv.yaml ../nfs-pvc1.yaml`。

> 💡 更省事的替代方案：用 **`extraMounts`** 把宿主机上已挂载好的 NFS 目录（或任意目录）挂进节点，再用 `hostPath` 提供存储 —— 这样完全绕开节点内安装 NFS 客户端的问题，是本仓库场景下最实用的做法。

### 8.4 存储方案对比

| 方案 | 生命周期 | 多节点共享 | 适用 |
|---|---|---|---|
| 默认 `standard`（local-path） | 随集群删除而丢失 | ❌ | 临时测试、无状态验证 |
| `extraMounts` + hostPath | **独立于集群** | 各节点各自挂载 | 需要保留数据的持久化验证 |
| NFS PV/PVC | 独立于集群 | ✅ RWX | 复用本仓库既有 NFS 配置 |
| 本地 registry + 外部存储 | 独立 | — | 组合方案 |

---

## 9. 部署本仓库的练习清单

kind 集群建好后，可直接验证仓库内所有清单。

### 9.1 基础对象

```bash
# 一次性应用全部基础对象（Pod / Deployment / StatefulSet / DaemonSet / CronJob）
kubectl apply -f ../yaml/

kubectl get all
kubectl get pods -o wide                 # 观察 Pod 在多节点上的分布
kubectl get pods --field-selector status.phase!=Running -A    # 找出异常 Pod
```

### 9.2 nginx 部署 + NodePort

```bash
kubectl apply -f ../nginx-deploy.yaml    # 4 副本，挂载 nfs-pvc1
kubectl apply -f ../nginx-service.yaml   # NodePort 30080

kubectl get deploy,pod,svc -o wide
kubectl rollout status deploy/nginx-deploy
```

> ⚠️ `nginx-deploy.yaml` 引用了 `claimName: nfs-pvc1`，需先按 [8.3 节](#83-使用-nfs复用本仓库的-pv-配置) 让 PVC 处于 `Bound`，否则 Pod 会卡在 `Pending`：
> ```bash
> kubectl get pvc
> kubectl describe pod <pod> | tail -20
> ```
>
> 若只想验证 Deployment 本身，可临时把卷改为 `emptyDir`，或先创建 PVC 对应的 PV。

### 9.3 Kubernetes Dashboard

```bash
kubectl apply -f ../k8sdashboard.yaml
kubectl -n kubernetes-dashboard get pods,svc
kubectl -n kubernetes-dashboard create token admin-user

# kind 下访问（NodePort 或 port-forward 二选一）
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard 8443:443
# 浏览器访问 https://localhost:8443
```

### 9.4 Helm 部署

```bash
helm install jenkins oci://registry-1.docker.io/bitnamicharts/jenkins \
  --namespace jenkins --create-namespace \
  -f ../helm/bitnami-jenkins.yaml \
  --set jenkinsPassword='<你的强密码>'

kubectl -n jenkins get pods,svc,pvc
```

> 💡 kind 中 Jenkins 这类较重的应用容易因**内存不足**而 OOM，建议给 Docker Desktop 分配 ≥ 8 GB，或调小 `resources` 后再部署。

---

## 10. 常用操作

### 10.1 集群与上下文管理

```bash
kind get clusters                        # 列出所有集群
kind get nodes --name dev                # 列出某集群的节点
kind get kubeconfig --name dev           # 输出 kubeconfig（不修改 ~/.kube/config）
kind get kubeconfig --name dev > dev.conf
export KUBECONFIG=dev.conf

kubectl config get-contexts              # 查看上下文（kind-dev / kind-prod …）
kubectl config use-context kind-dev      # 切换
kubectl --context kind-dev get nodes     # 临时指定上下文
```

### 10.2 进入节点排查

```bash
# 进入节点容器（节点内有完整的 systemd / kubelet / containerd）
docker exec -it dev-control-plane bash

# 节点内常用命令
crictl ps -a                 # 容器列表
crictl images                # 镜像列表
crictl logs <container-id>   # 容器日志
journalctl -u kubelet -f     # kubelet 日志
systemctl status kubelet
ls /etc/kubernetes/manifests # 静态 Pod 清单

# 从宿主机直接看节点内进程
docker top dev-control-plane
```

### 10.3 日志导出（排查失败创建的利器）

```bash
kind export logs ./kind-logs --name dev
# 目录内含每个节点的 kubelet / containerd / 容器日志，适合打包给他人分析

# 创建失败时保留节点容器，便于人工检查
kind create cluster --name probe --retain
```

### 10.4 节点重启与集群暂停

```bash
docker restart dev-control-plane         # 重启单个节点（模拟节点故障）
docker stop dev-worker                   # 停掉一个 worker，观察 Pod 驱逐
docker start dev-worker
```

### 10.5 构建自定义节点镜像

需要预装软件（如 `nfs-common`、调试工具）或使用自定义 K8s 版本时：

```bash
# 基于源码构建（需要 Go 环境）
kind build node-image --image my-node:v1 .

# 用自定义镜像创建集群
kind create cluster --image my-node:v1
```

---

## 11. CI 集成

kind 是 CI 中测试 K8s 清单的主流方案，官方提供 GitHub Actions。

### 11.1 GitHub Actions 示例

```yaml
name: k8s-manifest-test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 创建 kind 集群（可指定版本与配置）
      - uses: engineerd/setup-kind@v0.6.2
        with:
          version: v0.33.0
          config: K8s/kind/kind-config-single.yaml

      - name: 等待集群就绪
        run: |
          kubectl wait --for=condition=Ready nodes --all --timeout=300s
          kubectl cluster-info
          kubectl get nodes

      - name: 应用本仓库清单
        run: |
          kubectl apply -f K8s/yaml/
          kubectl rollout status deploy/nginx-deploy --timeout=180s || true

      - name: 断言资源状态
        run: |
          kubectl get pods -A
          # 若存在非 Running 的 Pod 则失败
          ! kubectl get pods -A --no-headers | grep -vE 'Running|Completed'

      - name: 导出日志（失败时）
        if: failure()
        run: kind export logs ./kind-logs

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: kind-logs
          path: ./kind-logs
```

### 11.2 CI 实践建议

| 建议 | 原因 |
|---|---|
| 固定 kind 版本与节点镜像 digest | 避免上游更新导致流水线行为漂移 |
| 使用单节点集群 | CI 中多节点收益低、启动慢、资源消耗大 |
| 预加载镜像（`kind load docker-image`） | 避免每个节点重复拉取，显著缩短时间 |
| 失败时 `kind export logs` + 上传产物 | 无图形界面时这是唯一的排查手段 |
| 用完即删（runner 天然销毁） | 无需清理逻辑 |
| 加 `--wait` 与显式断言 | 避免「集群没起来但用例通过」的假绿 |

---

## 12. 与 K3s / minikube 的选择

| 维度 | kind | K3s | minikube |
|---|---|---|---|
| 节点形态 | Docker 容器 | 物理机 / 虚拟机 | 虚拟机或容器 |
| 启动速度 | 最快（20~60 秒） | 快（1 分钟） | 较慢（3~5 分钟） |
| 资源占用 | 中（每节点 1~2 GB） | 低（Server 2 GB 起） | 高 |
| 多节点 | ✅ 支持 | ✅ 原生 | ⚠️ 实验性 |
| 多控制面 HA | ✅（haproxy 容器） | ✅（嵌入式 etcd） | ❌ |
| 数据持久性 | ❌ 随集群删除丢失 | ✅ 落在宿主机磁盘 | ✅ |
| 网络真实性 | 低（Docker 桥接） | 高（真实网络） | 中 |
| 常驻可用性 | ❌ 不适合 | ✅ 适合 | ⚠️ 一般 |
| 主要用途 | 本地开发、**CI 测试** | 边缘 / 家庭 / 小规模生产 | 单机学习 |

**决策建议：**

- 只想快速验证一份 YAML 或跑 CI → **kind**
- 想让服务 7×24 跑起来（NAS、家庭服务器）→ **K3s**
- 想体验最标准的发行版安装流程 → kubeadm（见 K3s 文档第 1 章对比表）

---

## 13. 故障排查

### 13.1 常见问题速查

| 现象 | 可能原因 | 处理方式 |
|---|---|---|
| 卡在 `Starting control-plane` 很久后失败 | Docker 内存不足 | Docker Desktop → Resources 调至 ≥ 8 GB；`docker info --format '{{.MemTotal}}'` 确认 |
| `ERROR: failed to create cluster: node(s) already exist` | 同名集群残留 | `kind delete cluster --name <名>` 后重建；或 `docker ps -a` 清理残留容器 |
| `ERROR: failed to create cluster: port is already allocated` | `hostPort` / `apiServerPort` 被占用 | `netstat -ano \| findstr :80`（Windows）或 `ss -lntp`（Linux）定位占用进程；更换端口 |
| 节点 `NotReady` | 节点内 kubelet/CNI 未就绪 | `docker exec <node> journalctl -u kubelet -n 100`；核对 cgroup 版本 |
| 镜像拉取极慢或超时 | 无镜像加速 / 网络受限 | 用 `kind load docker-image` 预加载，或配置本地 registry（第 7 章） |
| `kubectl` 连不上 API | kubeconfig 上下文不对 / 端口未映射 | `kubectl config current-context`；`docker ps` 看节点容器是否运行 |
| NodePort 在 Windows/macOS 访问不了 | 容器 IP 不可从宿主机路由 | 用 `kubectl port-forward`，或把端口加入 `extraPortMappings` 后重建集群 |
| Ingress `EXTERNAL-IP: <pending>` | 用了通用 LoadBalancer 清单 | 改用 [kind 专用 ingress 清单](#62-部署-ingress-nginxkind-官方方式)（hostPort 方式） |
| Ingress Pod 起不来（`0/1 Pending`） | 节点缺 `ingress-ready` 标签 | `kubectl label node <node> ingress-ready=true` |
| PVC 一直 `Pending` | StorageClass 缺失 / PV 不匹配 | `kubectl get storageclass`；检查 PVC 与 PV 的 `storageClassName`、容量、accessModes |
| Pod 卡 `ContainerCreating` | NFS 客户端缺失 / 挂载失败 | 节点内 `apt-get install nfs-common`；`kubectl describe pod` 看 Events |
| 应用 OOMKilled | 节点内存配额不足 | 提高 Docker 内存；调小应用 `resources.requests/limits` |
| WSL2 下创建失败 | WSL2 资源限制 / systemd 未开 | 在 `%UserProfile%\.wslconfig` 提高 `memory`；确认 Docker Desktop 使用 WSL2 后端 |
| `kind load` 报镜像不存在 | 镜像不在宿主机 Docker 中 | 先 `docker pull`，或用 `kind load image-archive` 从 tar 导入 |
| 删除集群后数据全丢 | local-path 数据在节点容器内 | 必须用 `extraMounts` 持久化（第 8 章） |

### 13.2 排查命令序列

```bash
# 第一层：kind 与 Docker 层面
kind get clusters
docker ps -a --filter "name=kind"            # 节点容器是否都在运行
docker logs <cluster>-control-plane 2>&1 | tail -50
docker info | grep -i -E 'memory|cgroup'

# 第二层：集群层面
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# 第三层：节点内部
docker exec -it <cluster>-control-plane bash
  # 容器内：
  #   systemctl status kubelet
  #   journalctl -u kubelet -n 100 --no-pager
  #   crictl ps -a
  #   crictl logs <container-id>

# 第四层：日志打包
kind export logs ./kind-logs --name <cluster>
```

### 13.3 彻底重置环境

```bash
# 删除所有 kind 集群与其容器、网络
kind delete clusters --all

# 清理残留容器与网络（kind delete 未清干净时）
docker ps -a --filter "label=io.x-k8s.kind.cluster" -q | xargs -r docker rm -f
docker network ls | grep kind

# 清理无用的节点镜像（每个约 1~2 GB）
docker images 'kindest/node' --format '{{.Repository}}:{{.Tag}} {{.Size}}'
docker image prune -a
```

---

## 14. 附录

### 14.1 命令速查

```bash
# ── 集群生命周期 ──────────────────────────────────────
kind create cluster                              # 默认单节点，名称 kind
kind create cluster --name dev --wait 5m         # 指定名称与超时
kind create cluster --config <file>              # 使用配置文件
kind create cluster --image <image>              # 指定节点镜像
kind create cluster --retain                     # 失败时保留容器
kind delete cluster --name dev
kind delete clusters --all

# ── 查询 ──────────────────────────────────────────────
kind get clusters
kind get nodes --name dev
kind get kubeconfig --name dev
kind export logs ./logs --name dev
kind version

# ── 镜像 ──────────────────────────────────────────────
kind load docker-image <img> --name dev
kind load image-archive <file.tar> --name dev

# ── 高级 ──────────────────────────────────────────────
kind build node-image --image my-node:v1 .
```

### 14.2 配置文件字段速查

| 层级 | 字段 | 关键取值 |
|---|---|---|
| 顶层 | `kind` / `apiVersion` | `Cluster` / `kind.x-k8s.io/v1alpha4` |
| 顶层 | `name` | 集群名 |
| 顶层 | `nodes[]` | `role`（control-plane/worker）、`image`、`labels`、`extraPortMappings`、`extraMounts`、`kubeadmConfigPatches` |
| 顶层 | `networking` | `ipFamily`、`apiServerAddress`、`apiServerPort`、`podSubnet`、`serviceSubnet`、`disableDefaultCNI`、`kubeProxyMode`、`dnsSearch` |
| 顶层 | `kubeadmConfigPatches[]` | 按 `kind: InitConfiguration / ClusterConfiguration / JoinConfiguration` 区分 |
| 顶层 | `containerdConfigPatches[]` | 镜像加速、registry mirror |
| 顶层 | `featureGates` / `runtimeConfig` | 特性门控 / API 运行时开关 |

### 14.3 节点镜像对照（kind v0.33.0 预构建）

| Kubernetes | 镜像 |
|---|---|
| v1.37.0 | `kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5` |
| v1.36.4 | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` |
| v1.35.8 | `kindest/node:v1.35.8@sha256:07b2536e30b803ed61d1677a79df6115f798ce64c80f9e22f6ed45afd09323c0` |
| v1.34.11 | `kindest/node:v1.34.11@sha256:44e222ee2132dab25ff87301682f89eb82c7880ea3a1bf543bfe9708fd08d67d` |

> ⚠️ 节点镜像必须与宿主机架构一致（amd64 / arm64），且必须使用 `@sha256` digest 才能保证拿到对应版本的镜像。
> 本仓库的 [`kind-test.yaml`](kind-test.yaml) 中固定的是较老的 `v1.27.3`，如需新版本请替换该行。

### 14.4 路径与文件速查

| 位置 | 说明 |
|---|---|
| `~/.kube/config` | kind 自动写入的 kubeconfig（上下文 `kind-<集群名>`） |
| `/<集群名>-control-plane` | 节点容器名（`docker exec` 用） |
| `/etc/kubernetes/manifests/` | 节点内静态 Pod 清单（etcd/apiserver 等） |
| `/etc/kubernetes/pki/` | 节点内证书目录 |
| `/var/local-path-provisioner` | 默认 StorageClass 的数据目录（节点容器内） |
| `/etc/containerd/certs.d/` | 节点内 containerd 仓库配置（新版本） |
| `kind-logs/` | `kind export logs` 的输出目录 |

### 14.5 本目录文件说明

| 文件 | 说明 |
|---|---|
| [`README.md`](README.md) | 本文档，kind 部署全流程指南 |
| [`kind-ha.yaml`](kind-ha.yaml) | 2 控制面 + 3 worker，含各节点独立端口映射（HA 练习） |
| [`kind-test.yaml`](kind-test.yaml) | 1 控制面 + 2 worker，自定义网段、ipvs 模式、固定 API 端口 |
| [`kind-config-single.yaml`](kind-config-single.yaml) | 单节点 + 80/443 映射，用于 Ingress 最小练习 |
| [`kind-config-registry.yaml`](kind-config-registry.yaml) | 单控制面 + 2 worker + 本地镜像仓库 + Ingress 端口映射 |

> 📌 另有两个根目录配置文件：[`../kind-4nodes.yaml`](../kind-4nodes.yaml)（1 控制面 + 3 worker，最简多节点）与 [`../kind-test.yaml`](../kind-test.yaml)（1 控制面 + 3 worker + 镜像加速）。后续可考虑统一收拢到本目录，避免同名文件混淆。

### 14.6 参考链接

- kind 官方文档：<https://kind.sigs.k8s.io/>
- kind 快速开始：<https://kind.sigs.k8s.io/docs/user/quick-start/>
- kind 配置参考：<https://kind.sigs.k8s.io/docs/user/configuration/>
- kind 本地镜像仓库：<https://kind.sigs.k8s.io/docs/user/local-registry/>
- kind Ingress 指南：<https://kind.sigs.k8s.io/docs/user/ingress/>
- kind 版本发布：<https://github.com/kubernetes-sigs/kind/releases>
- ingress-nginx kind 部署清单：<https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml>
- 本仓库 K3s 部署指南：[`../K3s/README.md`](../K3s/README.md)

---

> 📝 本文档基于 kind `v0.33.0`（默认节点镜像 `kindest/node:v1.37.0`）编写，示例端口与路径均与仓库内既有配置文件保持一致；实际使用时请按环境调整端口、路径与镜像地址。
