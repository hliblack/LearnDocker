# K3s 集群部署指南

> 本文档记录使用 **K3s** 从零部署 Kubernetes 集群的完整流程，覆盖单节点、多节点、高可用（HA）三种拓扑，以及组件定制、存储、备份恢复、升级与故障排查。
>
> 文档基准版本：**K3s `v1.37.1+k3s1`**（2026-10 发布），内置 Kubernetes `v1.37.1`、containerd `v2.3.4-k3s1`、etcd `v3.7.1-k3s3`、Traefik `v3.7.13`、CoreDNS `v1.14.7`、metrics-server `v0.9.0`、local-path-provisioner `v0.0.37`、Flannel `v0.28.4`。
> 版本会持续更新，安装前建议查看 <https://github.com/k3s-io/k3s/releases> 确认最新版本号。

## 目录

- [1. K3s 简介与选型](#1-k3s-简介与选型)
- [2. 环境准备](#2-环境准备)
- [3. 集群拓扑规划](#3-集群拓扑规划)
- [4. 单节点部署（All-in-One）](#4-单节点部署all-in-one)
- [5. 多节点集群部署](#5-多节点集群部署)
- [6. 高可用（HA）集群部署](#6-高可用ha集群部署)
- [7. 配置文件驱动部署](#7-配置文件驱动部署)
- [8. 镜像仓库与离线安装](#8-镜像仓库与离线安装)
- [9. 组件定制](#9-组件定制)
- [10. 存储方案](#10-存储方案)
- [11. 应用部署验证](#11-应用部署验证)
- [12. 日常运维](#12-日常运维)
- [13. 备份与恢复](#13-备份与恢复)
- [14. 升级与卸载](#14-升级与卸载)
- [15. 故障排查](#15-故障排查)
- [16. 附录](#16-附录)

---

## 1. K3s 简介与选型

### 1.1 什么是 K3s

K3s 是 Rancher（SUSE）推出的**轻量级、CNCF 认证的 Kubernetes 发行版**，整个控制面打包为**单个二进制文件**（约 80 MB），通过 systemd 以单进程方式运行，默认使用 SQLite（单节点）或嵌入式 etcd（HA）作为数据存储。

核心特点：

| 特性 | 说明 |
|---|---|
| 单二进制 | 控制面组件（apiserver / controller-manager / scheduler / kubelet / kube-proxy）合并为一个进程，无额外依赖 |
| 默认精简 | 内置 Flannel CNI、CoreDNS、Traefik Ingress、ServiceLB、local-path 存储、metrics-server |
| 存储轻量 | 单节点默认 SQLite（经 Kine 适配），HA 使用嵌入式 etcd，无需独立 etcd 集群 |
| 资源占用低 | Server 最低 2 核 2 GB，Agent 最低 1 核 512 MB |
| 运维简单 | 证书自动轮换、etcd 快照自动化、支持 system-upgrade-controller 灰度升级 |
| 完全兼容 | 通过 CNCF 一致性认证，`kubectl` / Helm / 标准 YAML 全部通用 |

### 1.2 与其他方案的对比

| 对比项 | K3s | kind | minikube | 标准 kubeadm |
|---|---|---|---|---|
| 定位 | 生产 / 边缘 / 家庭实验室 | 本地测试 CI | 本地开发 | 标准生产集群 |
| 节点形态 | 物理机 / 虚拟机 / 树莓派 | Docker 容器 | 虚拟机或容器 | 物理机 / 虚拟机 |
| 资源占用 | 低 | 中 | 高 | 高 |
| 多节点 | ✅ 原生支持 | ✅ | ❌（实验性） | ✅ |
| 高可用 | ✅ 嵌入式 etcd | ❌ | ❌ | ✅ 需外部 etcd |
| 数据存储 | SQLite / etcd / 外部 DB | etcd（容器内） | etcd | etcd |
| 默认 Ingress | Traefik | 无 | 无 | 无 |
| 安装耗时 | 1 分钟 | 1 分钟 | 3~5 分钟 | 30 分钟以上 |
| 适合场景 | 家庭 NAS、边缘、小规模生产 | 本仓库的 [kind](../kind) 练习 | 单机学习 | 大规模生产 |

> 💡 本仓库的分工建议：**kind** 用于快速验证 YAML 与 CI（`Kind` 无需真实机器）；**K3s** 用于把练习真正跑成可用集群（如部署在 NAS 虚拟机或家庭服务器上）。

### 1.3 适用场景

- 家庭实验室 / NAS 上的自托管应用编排（与本仓库 `Docker/docker-compose` 中的服务对应）
- 边缘计算、IoT 网关、树莓派集群
- 资源受限环境（2 核 2 GB 起）
- 需要轻量 HA 的小规模生产环境

---

## 2. 环境准备

### 2.1 系统要求

| 角色 | CPU | 内存 | 磁盘 | 说明 |
|---|---|---|---|---|
| Server（控制面 + 工作负载） | ≥ 2 核 | ≥ 2 GB | ≥ 20 GB | 单节点模式同时承担 Worker 职责 |
| Server（纯控制面，HA） | ≥ 2 核 | ≥ 4 GB | ≥ 40 GB | 3 台起步，etcd 数量必须为奇数 |
| Agent（工作节点） | ≥ 1 核 | ≥ 512 MB | ≥ 20 GB | 按业务负载调整 |

支持的系统：Ubuntu 20.04+ / Debian 11+ / CentOS 8+ / Rocky / Alma / RHEL / openSUSE / Alpine / Raspberry Pi OS。

> ⚠️ K3s **仅支持 Linux**。Windows 上体验请用 WSL2（需开启 systemd）或 `k3d`（Docker 内运行 K3s，用法与 kind 类似），参见 [6.4 节](#64-windows-环境下的替代方案)。

### 2.2 端口要求

| 协议 | 端口 | 用途 | 开放范围 |
|---|---|---|---|
| TCP | 6443 | Kubernetes API Server | 所有节点 → Server |
| TCP | 2379-2380 | 嵌入式 etcd 客户端与对等通信 | 仅 Server 之间 |
| TCP | 10250 | kubelet API | Server ↔ Agent |
| TCP | 8472 | Flannel VXLAN（如启用） | 所有节点互访 |
| UDP | 8472 | Flannel VXLAN | 所有节点互访 |
| UDP | 51820 / 51821 | Flannel WireGuard（如启用） | 所有节点互访 |
| TCP | 30000-32767 | NodePort 服务 | 客户端 → 所有节点 |

> 家庭内网环境通常无需额外配置；跨网段或云主机需在安全组放行上述端口。

### 2.3 系统初始化（所有节点执行）

```bash
# 1) 关闭 swap（kubelet 要求）
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab      # 永久生效

# 2) 加载内核模块
sudo tee /etc/modules-load.d/k3s.conf <<'EOF'
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 3) 内核参数
sudo tee /etc/sysctl.d/99-k3s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 4) 时间同步（节点时间偏差过大会导致证书与 Token 校验失败）
sudo timedatectl set-ntp true
timedatectl status | grep -E 'System clock|NTP'

# 5) 验证 cgroup 版本（v1/v2 均可，v2 为推荐）
stat -fc %T /sys/fs/cgroup/
```

**防火墙配置：**

```bash
# firewalld（CentOS / Rocky / RHEL）
sudo firewall-cmd --permanent --add-port={6443,2379,2380,10250,8472}/tcp
sudo firewall-cmd --permanent --add-port={8472,51820,51821}/udp
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload

# ufw（Ubuntu / Debian）
sudo ufw allow 6443/tcp && sudo ufw allow 10250/tcp
sudo ufw allow 8472/udp && sudo ufw allow 8472/tcp
```

**SELinux（RHEL / CentOS / Rocky 8+）：** 需额外安装 SELinux 策略包，并在 K3s 中开启 `selinux: true`（见 [7.3 参数映射表](#73-常用参数映射表)）：

```bash
# 1) 添加 Rancher 仓库并安装策略包
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.rancher.io/k3s/stable/common/centos/8/noarch/
sudo yum install -y k3s-selinux

# 2) 确认 SELinux 处于 enforcing 状态
getenforce

# 3) 之后再安装 K3s，并在 config.yaml 中开启 selinux: true
```

> 💡 若不使用 `selinux: true`，在 enforcing 模式下 K3s 可能无法正常启动 Pod（权限被拒绝）。详见官方文档 <https://docs.k3s.io/advanced#selinux-support>。

**必装工具：**

```bash
sudo apt install -y curl tar iptables        # Debian/Ubuntu
sudo yum install -y curl tar iptables        # RHEL 系
```

### 2.4 主机规划示例

沿用本仓库现有网段 `192.168.31.0/24`（NFS 服务器为 `192.168.31.217`，见 [`nfs-pv.yaml`](../nfs-pv.yaml)）：

| 主机名 | IP | 角色 | 配置 | 说明 |
|---|---|---|---|---|
| k3s-vip | 192.168.31.100 | VIP（虚拟 IP） | — | HA 集群统一入口，由 kube-vip / Keepalived 提供 |
| k3s-server-1 | 192.168.31.101 | Server（etcd 成员） | 4C / 8G | 首个 Server，`--cluster-init` |
| k3s-server-2 | 192.168.31.102 | Server（etcd 成员） | 4C / 8G | 加入集群 |
| k3s-server-3 | 192.168.31.103 | Server（etcd 成员） | 4C / 8G | 加入集群 |
| k3s-agent-1 | 192.168.31.111 | Agent | 2C / 4G | 工作节点 |
| k3s-agent-2 | 192.168.31.112 | Agent | 2C / 4G | 工作节点 |
| nas | 192.168.31.217 | NFS 服务端 | — | 提供 `/nfs` 共享存储 |

各节点需配置主机名与 hosts 解析：

```bash
# 每个节点执行（示例为 server-1）
sudo hostnamectl set-hostname k3s-server-1
sudo tee -a /etc/hosts <<'EOF'
192.168.31.100  k3s-vip
192.168.31.101  k3s-server-1
192.168.31.102  k3s-server-2
192.168.31.103  k3s-server-3
192.168.31.111  k3s-agent-1
192.168.31.112  k3s-agent-2
EOF
```

---

## 3. 集群拓扑规划

### 3.1 三种拓扑

| 方案 | 节点数 | 数据存储 | 适用场景 | 故障容忍 |
|---|---|---|---|---|
| **单节点** | 1 | SQLite | 学习、家庭服务、边缘设备 | 无（单点） |
| **单 Server + 多 Agent** | 1 + N | SQLite | 小规模，控制面无冗余要求 | 控制面单点 |
| **多 Server + 多 Agent（HA）** | 3 + N | 嵌入式 etcd | 生产 / 家庭实验室高可用 | 容忍 1 台 Server 故障 |

### 3.2 安装方式选择

| 方式 | 命令特征 | 适用场景 |
|---|---|---|
| 官方脚本在线安装 | `curl -sfL https://get.k3s.io \| sh -` | 有公网，最省事 |
| 国内镜像脚本 | `curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh \| INSTALL_K3S_MIRROR=cn sh -` | 国内网络下载慢 |
| 离线（air-gap）安装 | `INSTALL_K3S_SKIP_DOWNLOAD=true` | 无外网生产环境 |
| 配置文件驱动 | 预置 `/etc/rancher/k3s/config.yaml` | 生产环境推荐（可版本化） |
| k3d（容器内） | `k3d cluster create` | Windows / macOS 本地体验 |

---

## 4. 单节点部署（All-in-One）

最简单的起步方式，1 台机器同时承担控制面与工作负载。

### 4.1 一键安装

```bash
# 官方脚本（默认安装最新稳定版）
curl -sfL https://get.k3s.io | sh -

# 国内网络加速（使用 Rancher 中国镜像站）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -

# 指定版本（生产环境务必锁定版本）
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 sh -

# 安装时直接传参：允许非 root 读取 kubeconfig，并把本机 IP 加入证书 SAN
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --write-kubeconfig-mode 0644 --tls-san 192.168.31.101" sh -
```

安装脚本做了什么：

1. 下载 `k3s` 二进制到 `/usr/local/bin/k3s`，并创建 `kubectl`/`crictl`/`ctr` 软链接
2. 写入 systemd 服务 `k3s.service` 并启动
3. 生成 kubeconfig `/etc/rancher/k3s/k3s.yaml` 与 Token `/var/lib/rancher/k3s/server/node-token`

### 4.2 验证

```bash
# 服务状态
systemctl status k3s
journalctl -u k3s -f          # 实时日志，Ctrl+C 退出

# 集群信息
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl get storageclass

# 版本与内置组件
k3s --version
k3s check-config              # 环境自查
```

预期输出：

```
NAME       STATUS   ROLES                  AGE   VERSION
k3s-node   Ready    control-plane,master   60s   v1.37.1+k3s1
```

### 4.3 配置 kubectl（本机与远程）

```bash
# 方式 A：直接用 k3s 内置 kubectl
sudo k3s kubectl get nodes

# 方式 B：让普通用户使用（安装时已加 --write-kubeconfig-mode 0644）
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# 方式 C：从远程客户端访问（如 Windows 上的 kubectl / Lens）
# 1) 复制 /etc/rancher/k3s/k3s.yaml 到客户端 %USERPROFILE%\.kube\config
# 2) 把 server: https://127.0.0.1:6443 改为 https://192.168.31.101:6443
```

> ⚠️ `k3s.yaml` 中的 `server` 默认是 `127.0.0.1:6443`，远程使用必须改成节点真实 IP，且该 IP 需包含在证书 SAN 中（安装时通过 `--tls-san` 指定），否则会报 `x509: certificate is valid for ...`。

### 4.4 第一个工作负载

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
# 访问 http://<节点IP>:<NodePort>
```

---

## 5. 多节点集群部署

### 5.1 Server 节点安装

```bash
# 在 k3s-server-1 (192.168.31.101) 执行
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --write-kubeconfig-mode 0644 \
  --node-name k3s-server-1 \
  --node-ip 192.168.31.101 \
  --tls-san 192.168.31.101 \
  --flannel-iface eth0" sh -
```

> 💡 多网卡主机（如 NAS、虚拟机、同时有 Docker 网桥）**必须**通过 `--node-ip` 与 `--flannel-iface` 明确指定用于集群通信的网卡，否则容易出现节点 IP 选错、Pod 跨节点不通的问题。

### 5.2 获取 Token

```bash
# Server 节点上查看
sudo cat /var/lib/rancher/k3s/server/node-token
# 输出形如：K10abcdef1234567890::server:9f8e7d6c5b4a3210
```

### 5.3 Agent 节点加入

```bash
# 在每个 Agent 节点执行
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.31.101:6443 \
  K3S_TOKEN=K10abcdef1234567890::server:9f8e7d6c5b4a3210 \
  INSTALL_K3S_EXEC="agent \
    --node-name k3s-agent-1 \
    --node-ip 192.168.31.111 \
    --flannel-iface eth0 \
    --node-label node-role=worker" sh -
```

Agent 侧常用环境变量：

| 变量 | 说明 |
|---|---|
| `K3S_URL` | Server 的 API 地址，**Agent 必填** |
| `K3S_TOKEN` | 集群 Token（同 Server 上的 `node-token`） |
| `K3S_NODE_NAME` | 节点名，也可用 `--node-name` |
| `K3S_DATA_DIR` | 数据目录，默认 `/var/lib/rancher/k3s` |

### 5.4 验证集群

```bash
# 在任意 Server 节点执行
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl top nodes                # 需要 metrics-server，K3s 默认已装

# 查看节点标签
kubectl get nodes --show-labels
```

预期三个（或更多）节点全部 `Ready`，且 `kubectl top nodes` 有数据。

### 5.5 角色与调度控制

```bash
# K3s 的 Agent 默认带标签，Server 默认可调度；
# 若希望 Server 只做控制面，禁止调度业务 Pod：
kubectl taint nodes k3s-server-1 node-role.kubernetes.io/control-plane=:NoSchedule

# 给节点打标签并按标签调度
kubectl label node k3s-agent-1 disktype=ssd
kubectl label node k3s-agent-2 gpu=true
```

```yaml
# Pod 中通过 nodeSelector 指定落点
spec:
  nodeSelector:
    disktype: ssd
```

---

## 6. 高可用（HA）集群部署

### 6.1 HA 方案选择

| 数据存储 | 部署复杂度 | 性能 | 推荐度 | 说明 |
|---|---|---|---|---|
| **嵌入式 etcd** | 低 | 好 | ⭐⭐⭐⭐⭐ | K3s 官方推荐，Server 数量必须为 **奇数**（3/5） |
| 外部 etcd | 高 | 好 | ⭐⭐ | 已有 etcd 集群时复用 |
| 外部 MySQL / PostgreSQL | 中 | 一般 | ⭐ | 需要独立数据库，运维成本高，仅特殊场景使用 |
| SQLite | 低 | 好 | ❌ | 不支持多 Server |

> ⚠️ 嵌入式 etcd 集群的 Server 数量必须是奇数。2 台 Server 的可用性与 1 台相同，不建议。

### 6.2 固定注册地址（关键前置）

Agent 与其他 Server 需要一个**稳定的接入地址**，不能直接指向某一台 Server。两种做法：

**方案 A：kube-vip 提供 VIP（无需额外硬件，推荐）**

kube-vip 以 **Static Pod** 形式运行在每台 Server 上，通过 ARP 宣告 VIP，并借助 leader 选举保证同一时刻只有一个实例持有 VIP。

```bash
# 1) 在 k3s-server-1 上启动首个 Server（此时还没有 VIP，直接用本机 IP）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --cluster-init \
  --tls-san 192.168.31.100 \
  --node-ip 192.168.31.101 \
  --write-kubeconfig-mode 0644" sh -

# 2) 部署 kube-vip 的 RBAC（每个集群一次即可）
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml

# 3) 用官方 CLI 生成 Static Pod manifest，直接写入 K3s 的 manifests 目录
#    （K3s 会自动应用该目录下的清单，无需手动 kubectl apply）
export VIP=192.168.31.100          # 要使用的虚拟 IP
export INTERFACE=eth0              # 承载 VIP 的网卡（务必与实际一致）
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases/latest \
  | grep -o '"tag_name": *"[^"]*"' | cut -d'"' -f4)

sudo k3s ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo k3s ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip \
  manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection \
  | sudo tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml

# 4) 其余的 Server / Agent 安装时，--server / K3S_URL 一律使用 VIP 地址
#    curl -sfL https://get.k3s.io | K3S_TOKEN=xxx INSTALL_K3S_EXEC="server --server https://192.168.31.100:6443 ..." sh -

# 5) 验证 VIP 是否可达
ping -c 2 192.168.31.100
curl -k https://192.168.31.100:6443/healthz       # 期望输出 ok
kubectl get pods -n kube-system | grep kube-vip
```

> ⚠️ kube-vip 生成的 Static Pod 会被 **每个** Server 节点上的 K3s 加载（`manifests` 目录在多 Server 间共享同一份清单），无需重复生成。
> 若 VIP 与节点不在同一二层网络（跨网段），ARP 模式不适用，需改用 BGP 模式。

**方案 B：外部负载均衡（Keepalived + HAProxy / Nginx / F5）**

以 HAProxy + Keepalived 为例，在两台独立主机（或复用 Server）上部署，后端指向三台 Server 的 `6443`，对外暴露 VIP `192.168.31.100`：

```haproxy
# /etc/haproxy/haproxy.cfg 关键片段
frontend k3s_api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k3s_servers

backend k3s_servers
    mode tcp
    balance roundrobin
    option tcp-check
    server k3s-server-1 192.168.31.101:6443 check
    server k3s-server-2 192.168.31.102:6443 check
    server k3s-server-3 192.168.31.103:6443 check
```

> 无论哪种方案，**所有 Server 安装时都必须把 VIP/域名加入 `--tls-san`**，否则证书校验失败。

### 6.3 三 Server 集群部署步骤

```bash
# ── 第 1 台（etcd 初始化节点）──────────────────────────
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --cluster-init \
  --tls-san 192.168.31.100 \
  --tls-san k3s.example.com \
  --node-name k3s-server-1 \
  --node-ip 192.168.31.101 \
  --flannel-iface eth0 \
  --write-kubeconfig-mode 0644" sh -

# 查看 Token
sudo cat /var/lib/rancher/k3s/server/node-token
```

```bash
# ── 第 2、3 台（加入已有 etcd 集群）──────────────────
# 注意：用 --server 指向第 1 台，而不是 --cluster-init
curl -sfL https://get.k3s.io | \
  K3S_TOKEN=K10abcdef1234567890::server:9f8e7d6c5b4a3210 \
  INSTALL_K3S_EXEC="server \
    --server https://192.168.31.101:6443 \
    --tls-san 192.168.31.100 \
    --node-name k3s-server-2 \
    --node-ip 192.168.31.102 \
    --flannel-iface eth0 \
    --write-kubeconfig-mode 0644" sh -
```

```bash
# ── Agent 通过 VIP 加入 ───────────────────────────────
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.31.100:6443 \
  K3S_TOKEN=K10abcdef1234567890::server:9f8e7d6c5b4a3210 \
  INSTALL_K3S_EXEC="agent --node-name k3s-agent-1 --node-ip 192.168.31.111 --flannel-iface eth0" sh -
```

### 6.4 Windows 环境下的替代方案

K3s 无法直接运行在 Windows 上，本机体验有两种途径（均适合验证 YAML，不适合承载生产）：

**方式一：k3d（Docker 中运行 K3s，与 kind 用法相似）**

```powershell
# 依赖本机已安装 Docker Desktop
winget install k3d            # 或 choco install k3d

k3d cluster create myk3s --servers 3 --agents 2
kubectl get nodes
k3d cluster delete myk3s
```

**方式二：WSL2 安装 K3s**

```bash
# WSL2 中开启 systemd（/etc/wsl.conf）
[boot]
systemd=true
# 重启 WSL 后按第 4 章正常安装，注意 systemd 未启用时安装脚本会报错
```

---

## 7. 配置文件驱动部署

命令行参数过多时，推荐改为配置文件方式（可纳入 Git 版本管理）。

### 7.1 配置文件位置与格式

| 项目 | 路径 |
|---|---|
| Server 配置 | `/etc/rancher/k3s/config.yaml` |
| Agent 配置 | `/etc/rancher/k3s/config.yaml`（同一路径，按角色解析） |
| 分片配置目录 | `/etc/rancher/k3s/config.yaml.d/*.yaml`（按文件名顺序合并，后者覆盖前者） |
| 数据目录 | `/var/lib/rancher/k3s` |
| manifests 自动部署目录 | `/var/lib/rancher/k3s/server/manifests/` |

**格式规则：** YAML 的键名 = 命令行参数去掉 `--` 前缀；列表型参数写成数组；布尔参数写 `true`/`false`。

```yaml
# 命令行：k3s server --write-kubeconfig-mode 0644 --tls-san 192.168.31.100 --disable traefik
# 等价配置：
write-kubeconfig-mode: "0644"
tls-san:
  - "192.168.31.100"
disable:
  - traefik
```

### 7.2 部署流程

```bash
# 1) 先写配置（安装脚本会自动读取）
sudo mkdir -p /etc/rancher/k3s
sudo cp config.yaml.example /etc/rancher/k3s/config.yaml
sudo vim /etc/rancher/k3s/config.yaml

# 2) 再执行安装（无需重复传参）
curl -sfL https://get.k3s.io | sh -

# 3) 修改配置后重启生效
sudo systemctl restart k3s          # Server
sudo systemctl restart k3s-agent    # Agent
```

> ⚠️ 修改配置后**重启才能生效**；`/etc/rancher/k3s/config.yaml` 中的参数会与命令行参数合并，冲突时以命令行/环境变量为准。

### 7.3 常用参数映射表

| 命令行参数 | 配置文件键名 | 示例值 |
|---|---|---|
| `--cluster-init` | `cluster-init` | `true` |
| `--server` | `server` | `https://192.168.31.100:6443` |
| `--token` | `token` | `K10xxx::server:xxx` |
| `--node-name` | `node-name` | `k3s-server-1` |
| `--node-ip` | `node-ip` | `192.168.31.101` |
| `--flannel-iface` | `flannel-iface` | `eth0` |
| `--flannel-backend` | `flannel-backend` | `vxlan` / `host-gw` / `wireguard-native` / `none` |
| `--tls-san` | `tls-san` | 数组 |
| `--write-kubeconfig-mode` | `write-kubeconfig-mode` | `"0644"` |
| `--disable` | `disable` | 数组：`traefik` / `servicelb` / `coredns` / `local-storage` / `metrics-server` / `network-policy` |
| `--disable-kube-proxy` | `disable-kube-proxy` | `true` |
| `--disable-network-policy` | `disable-network-policy` | `true` |
| `--disable-cloud-controller` | `disable-cloud-controller` | `true` |
| `--disable-helm-controller` | `disable-helm-controller` | `true` |
| `--selinux` | `selinux` | `true` |
| `--secrets-encryption` | `secrets-encryption` | `true` |
| `--data-dir` | `data-dir` | `/var/lib/rancher/k3s` |
| `--node-label` | `node-label` | 数组 |
| `--node-taint` | `node-taint` | 数组 |
| `--kubelet-arg` | `kubelet-arg` | 数组 |
| `--kube-apiserver-arg` | `kube-apiserver-arg` | 数组 |
| `--etcd-snapshot-schedule-cron` | `etcd-snapshot-schedule-cron` | `"0 */12 * * *"` |
| `--etcd-snapshot-retention` | `etcd-snapshot-retention` | `5` |
| `--etcd-s3` | `etcd-s3` | `true` |

> 📌 完整参数列表：`k3s server --help` / `k3s agent --help`。本仓库配套示例见 [`config.yaml.example`](config.yaml.example) 与 [`agent-config.yaml.example`](agent-config.yaml.example)。

### 7.4 manifests 自动部署（AddOn）

放入 `/var/lib/rancher/k3s/server/manifests/` 的 YAML 会被自动 apply，适合开机即需存在的基础组件：

```bash
sudo cp ~/my-app.yaml /var/lib/rancher/k3s/server/manifests/
# K3s 会在数秒内自动部署，删除文件不会删除已创建的资源
```

> ⚠️ 该机制是「只增不减」：删除 manifest 文件**不会**删除对应资源，需手动 `kubectl delete`。

---

## 8. 镜像仓库与离线安装

### 8.1 registries.yaml 配置

K3s 使用内置 containerd，镜像配置与宿主机 Docker 的 `/etc/docker/daemon.json` **互不影响**。配置文件路径 `/etc/rancher/k3s/registries.yaml`（Server 与 Agent 都要放），改完执行 `systemctl restart k3s`（或 `k3s-agent`）。

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"     # 示例加速地址，可用性会变化，请自行验证
  harbor.example.com:
    endpoint:
      - "https://harbor.example.com"

configs:
  "harbor.example.com":
    auth:
      username: admin
      password: "your-password"
    tls:
      ca_file: "/etc/rancher/k3s/harbor-ca.crt"
      insecure_skip_verify: false
```

完整示例见 [`registries.yaml.example`](registries.yaml.example)。

**验证配置是否生效：**

```bash
# 查看 containerd 生成的 hosts 配置
sudo cat /var/lib/rancher/k3s/agent/etc/containerd/certs.d/docker.io/hosts.toml

# 拉取测试
sudo k3s crictl pull nginx:latest
sudo k3s ctr images ls | grep nginx
```

### 8.2 离线（air-gap）安装

适用于无外网的生产环境：

```bash
# ── 在联网机器上下载（架构按需：amd64 / arm64 / arm）──
VERSION=v1.37.1+k3s1
wget https://github.com/k3s-io/k3s/releases/download/${VERSION}/k3s
wget https://github.com/k3s-io/k3s/releases/download/${VERSION}/k3s-airgap-images-amd64.tar.zst
wget https://get.k3s.io -O install.sh
wget https://github.com/k3s-io/k3s/releases/download/${VERSION}/sha256sum-amd64.txt

# 校验完整性
sha256sum -c sha256sum-amd64.txt --ignore-missing

# ── 传输到目标节点（scp / U 盘）──
```

```bash
# ── 在每个目标节点上执行 ──────────────────────────────
# 1) 放置二进制
sudo install -m 755 k3s /usr/local/bin/k3s

# 2) 放置镜像包（K3s 启动时自动导入该目录下的 tar/tar.gz/tar.zst）
sudo mkdir -p /var/lib/rancher/k3s/agent/images/
sudo cp k3s-airgap-images-amd64.tar.zst /var/lib/rancher/k3s/agent/images/

# 3) 离线安装（跳过下载）
chmod +x install.sh
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh

# 多节点：Server 节点加 --cluster-init，Agent 节点传 K3S_URL / K3S_TOKEN
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://192.168.31.101:6443 K3S_TOKEN=xxx ./install.sh
```

### 8.3 常用镜像工具

```bash
sudo k3s ctr images ls                       # 列出镜像（containerd 原生）
sudo k3s crictl images                       # CRI 视角
sudo k3s crictl pull nginx:latest            # 拉取镜像
sudo k3s crictl ps -a                        # 容器列表
sudo k3s crictl logs <container-id>          # 容器日志
sudo k3s ctr -n k8s.io images ls | grep xxx  # 查看 k8s.io 命名空间镜像
```

---

## 9. 组件定制

### 9.1 内置组件一览

| 组件 | 默认状态 | 禁用参数 | 说明 |
|---|---|---|---|
| CoreDNS | 启用 | `--disable coredns` | 集群 DNS |
| Traefik | 启用 | `--disable traefik` | Ingress Controller |
| ServiceLB（Klipper） | 启用 | `--disable servicelb` | LoadBalancer 类型 Service 实现 |
| local-path-provisioner | 启用 | `--disable local-storage` | 本地存储动态供给 |
| metrics-server | 启用 | `--disable metrics-server` | 资源指标（`kubectl top`） |
| Network Policy Controller | 启用 | `--disable-network-policy` | NetworkPolicy 执行 |
| Flannel | 启用 | `--flannel-backend=none` | CNI |
| kube-proxy | 启用 | `--disable-kube-proxy` | 服务代理（改用 Cilium 时） |
| Helm Controller | 启用 | `--disable-helm-controller` | HelmChart CRD 支持 |

> 完整可禁用列表以 `k3s server --help | grep -A3 disable` 输出为准（版本间会有差异）。

### 9.2 替换 Traefik 为 ingress-nginx

```bash
# 1) 安装时禁用 Traefik 与 ServiceLB（避免 80/443 端口冲突）
#    /etc/rancher/k3s/config.yaml:
#      disable:
#        - traefik
#        - servicelb
sudo systemctl restart k3s

# 2) 用 Helm 安装 ingress-nginx（DaemonSet + hostPort 方式，占用节点 80/443）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.kind=DaemonSet \
  --set controller.hostPort.enabled=true \
  --set controller.service.type=ClusterIP

# 3) 验证
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

> ⚠️ 如果安装时未禁用 Traefik，Traefik 的 ServiceLB 已占用节点 80/443，ingress-nginx 会因端口冲突无法启动。事后补救需删除 `traefik` 的 helmchart 与 `svclb-traefik-*` DaemonSet。

### 9.3 启用 Traefik Dashboard（默认关闭）

```bash
# K3s 通过 HelmChartConfig CRD 定制内置 chart
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    dashboard:
      enabled: true
    ports:
      web:
        exposedPort: 80
EOF

# 生效（Helm controller 会自动重新部署）
kubectl get helmchart -n kube-system
kubectl -n kube-system get pods | grep traefik
```

### 9.4 部署 Kubernetes Dashboard

可复用本仓库的 [`../k8sdashboard.yaml`](../k8sdashboard.yaml)：

```bash
kubectl apply -f ../k8sdashboard.yaml
kubectl -n kubernetes-dashboard get pods,svc

# 创建管理员 token（用于登录）
kubectl -n kubernetes-dashboard create token admin-user

# 访问方式：NodePort 或 ingress
kubectl -n kubernetes-dashboard get svc kubernetes-dashboard
```

### 9.5 使用自建 CNI（Cilium / Calico）

```yaml
# /etc/rancher/k3s/config.yaml
flannel-backend: "none"
disable-network-policy: true
disable-kube-proxy: true        # 由 Cilium 接管 kube-proxy 功能
disable:
  - servicelb
```

```bash
sudo systemctl restart k3s
# 然后用 Helm 安装 Cilium
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.31.100 \
  --set k8sServicePort=6443
```

> ⚠️ CNI 切换会造成集群网络短暂中断，务必在集群初始化阶段完成，不要在已有业务时操作。

---

## 10. 存储方案

### 10.1 local-path（默认）

K3s 默认 StorageClass 为 `local-path`，数据落在 **Pod 所在节点**的 `/var/lib/rancher/k3s/storage`：

```bash
kubectl get storageclass
# local-path (default)  rancher.io/local-path  Delete  WaitForFirstConsumer

# 默认存储路径可通过 HelmChartConfig 修改
cat <<'EOF' | sudo tee /var/lib/rancher/k3s/server/manifests/local-storage-config.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: local-path-provisioner
  namespace: kube-system
spec:
  valuesContent: |-
    storageClass:
      defaultClass: true
    nodePathMap:
      - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
        paths:
          - /data/k3s-storage
EOF
```

| 优点 | 缺点 |
|---|---|
| 开箱即用、本地磁盘性能最好 | **Pod 迁移后数据不再可用**（数据绑定节点） |
| 无需额外组件 | 不支持多节点共享读写 |

> 📌 `local-path` 只适合无状态或可重建的数据；有状态服务（数据库、相册等）建议用 NFS。

### 10.2 NFS 静态 PV（复用本仓库配置）

本仓库已有 [`../nfs-pv.yaml`](../nfs-pv.yaml)、[`../nfs-pvc1.yaml`](../nfs-pvc1.yaml)、[`../nfs-pvc2.yaml`](../nfs-pvc2.yaml)，指向 NFS 服务器 `192.168.31.217:/nfs`。

**前置条件：所有 K8s 节点都要能挂载该 NFS 共享**，需安装 nfs 客户端：

```bash
sudo apt install -y nfs-common        # Debian / Ubuntu
sudo yum install -y nfs-utils         # RHEL 系

# 在「每个」节点上验证（能挂载成功才算就绪）
sudo mount -t nfs 192.168.31.217:/nfs /mnt && ls /mnt && sudo umount /mnt
```

```bash
# 部署 PV / PVC
kubectl apply -f ../nfs-pv.yaml
kubectl apply -f ../nfs-pvc1.yaml

kubectl get pv,pvc
```

> ⚠️ 静态 PV 注意点：
> 1. `nfs-pv.yaml` 未指定 `storageClassName`，PVC 必须同样**不指定或显式写 `storageClassName: ""`**，否则会被 `local-path` 默认类抢走而一直 `Pending`。
> 2. `accessModes: ReadWriteOnce` 是 NFS 的兼容写法，实际 NFS 支持多节点读写（如需多个 Pod 共享，改用 `ReadWriteMany`）。
> 3. `mountOptions` 保留 `hard,nfsvers=4.1`，与 NFS 服务端版本匹配。

### 10.3 NFS 动态供给（推荐）

静态 PV 需要手工管理容量，规模化场景推荐 `nfs-subdir-external-provisioner`，自动按 PVC 创建子目录：

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=192.168.31.217 \
  --set nfs.path=/nfs \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.archiveOnDelete=false

# 验证
kubectl get storageclass
kubectl get pods -n kube-system | grep nfs-client
```

```yaml
# 使用方式：PVC 指定 storageClassName 即可自动创建目录
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-dynamic-pvc
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

### 10.4 存储方案对比

| 方案 | 动态供给 | 多节点共享 | 数据持久性 | 适用 |
|---|---|---|---|---|
| local-path | ✅ | ❌ | 绑定节点 | 无状态、可重建数据 |
| NFS 静态 PV | ❌ | ✅（RWX） | ✅ | 少量固定存储需求 |
| NFS 动态供给 | ✅ | ✅（RWX） | ✅ | 多应用、频繁创建 PVC |
| Longhorn | ✅ | ✅ | ✅（多副本） | 需要块存储与快照的场景 |

---

## 11. 应用部署验证

用本仓库已有的清单做端到端验证（Deployment → NodePort → NFS 持久化）。

### 11.1 部署 nginx（引用仓库现有 YAML）

```bash
# 1) 创建 Deployment（4 副本，挂载 NFS PVC，见 ../nginx-deploy.yaml）
kubectl apply -f ../nginx-deploy.yaml

# 2) 创建 NodePort Service（30080，见 ../nginx-service.yaml）
kubectl apply -f ../nginx-service.yaml

# 3) 查看结果
kubectl get deploy,rs,pod -o wide
kubectl get svc nginx-svc
kubectl rollout status deploy/nginx-deploy
```

访问 `http://192.168.31.111:30080`（任意节点 IP + NodePort）验证。

> ⚠️ `nginx-deploy.yaml` 中的 `claimName: nfs-pvc1` 必须已 `Bound`，否则 Pod 会卡在 `Pending`/`ContainerCreating`。排查：
> ```bash
> kubectl get pvc
> kubectl describe pod <pod-name> | tail -20
> ```

### 11.2 常用对象练习

```bash
# 一次性应用 K8s/yaml 下的全部基础对象
kubectl apply -f ../yaml/

kubectl get all -A
kubectl get pods --field-selector status.phase!=Running -A    # 找出异常 Pod
```

### 11.3 Helm 部署（可复用仓库 values）

```bash
# 仓库内的 Bitnami Jenkins values（密码通过 --set 传入，不写入文件）
helm install jenkins oci://registry-1.docker.io/bitnamicharts/jenkins \
  --namespace jenkins --create-namespace \
  -f ../helm/bitnami-jenkins.yaml \
  --set jenkinsPassword='<你的强密码>'

kubectl -n jenkins get pods,svc,pvc
helm list -A
```

### 11.4 部署自托管应用（与 compose 目录对应）

`Docker/docker-compose` 中的应用（Navidrome、Vaultwarden、IMMICH 等）迁移到 K3s 的思路：

| Compose 概念 | K8s 对应 |
|---|---|
| `services.<name>` | `Deployment`（无状态）/ `StatefulSet`（有状态） |
| `ports: 8080:80` | `Service` + `Ingress`（或 NodePort） |
| `volumes: /host:/data` | `PVC` → `PersistentVolume`（NFS 或 local-path） |
| `environment` / `.env` | `ConfigMap`（普通配置）+ `Secret`（密码） |
| `depends_on` | 无直接对应，用探针 + 重试处理启动顺序 |
| `restart: always` | 默认由控制器保证（无需声明） |
| `networks` | 集群内扁平网络（Service 名即 DNS） |

```bash
# 由 .env 生成 Secret（注意：不要提交到 Git）
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD='<强密码>' \
  --from-env-file=Docker/docker-compose/IMMICH/.env --dry-run=client -o yaml > /tmp/secret.yaml
kubectl apply -f /tmp/secret.yaml
```

---

## 12. 日常运维

### 12.1 服务管理

```bash
# Server 节点
sudo systemctl status k3s
sudo systemctl restart k3s
sudo systemctl stop k3s
sudo journalctl -u k3s -f --since "10 min ago"

# Agent 节点
sudo systemctl status k3s-agent
sudo systemctl restart k3s-agent
sudo journalctl -u k3s-agent -f
```

### 12.2 节点管理

```bash
kubectl get nodes -o wide
kubectl top nodes

# 维护模式：禁止调度并驱逐 Pod（维护前必须执行）
kubectl cordon k3s-agent-1
kubectl drain k3s-agent-1 --ignore-daemonsets --delete-emptydir-data

# 维护完成后恢复
kubectl uncordon k3s-agent-1

# 删除节点（先在节点上卸载 Agent，再删除对象）
kubectl delete node k3s-agent-2
```

### 12.3 证书管理

```bash
# 查看证书有效期（默认 1 年，K3s 会自动轮换）
sudo ls -l /var/lib/rancher/k3s/server/tls/
openssl x509 -in /var/lib/rancher/k3s/server/tls/server-ca.crt -noout -dates

# 手动轮换（Server 上执行，会重启服务）
sudo k3s certificate rotate
sudo systemctl restart k3s

# Agent 客户端证书轮换
sudo k3s certificate rotate-ca --path=/var/lib/rancher/k3s/server
```

> 📌 新增访问地址（如新域名/VIP）时，需在配置文件中补充 `tls-san` 后执行 `k3s certificate rotate` 并重启。

### 12.4 Secret 加密

```yaml
# /etc/rancher/k3s/config.yaml
secrets-encryption: true
```

```bash
sudo systemctl restart k3s

# 查看加密状态与密钥
sudo k3s secrets-encrypt status
sudo k3s secrets-encrypt rotate-keys
```

### 12.5 资源与状态巡检

```bash
# 集群健康
kubectl get --raw='/readyz?verbose'
kubectl get componentstatuses 2>/dev/null    # 老版本可用
kubectl get pods -A | grep -vE 'Running|Completed'

# 资源水位
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head

# 事件（排查异常的首选）
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# etcd 成员状态（HA 集群）
sudo k3s etcd-snapshot list
kubectl -n kube-system exec -it etcd-k3s-server-1 -- etcdctl \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key \
  endpoint status --write-out=table
```

### 12.6 日志与调试

```bash
kubectl logs <pod> -n <ns> --tail=100 -f
kubectl logs <pod> -n <ns> --previous                # 崩溃前的日志
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl debug node/k3s-agent-1 -it --image=busybox   # 节点级排查
kubectl describe pod <pod> -n <ns>                   # 看 Events 段
```

---

## 13. 备份与恢复

### 13.1 自动快照（HA / etcd 模式）

```yaml
# /etc/rancher/k3s/config.yaml
etcd-snapshot-schedule-cron: "0 */12 * * *"        # 每 12 小时一次
etcd-snapshot-retention: 7                          # 保留 7 份
etcd-snapshot-dir: "/var/lib/rancher/k3s/server/db/snapshots"

# 可选：上传到 S3 兼容对象存储（MinIO / 阿里云 OSS 等）
etcd-s3: true
etcd-s3-endpoint: "s3.example.com"
etcd-s3-bucket: "k3s-backup"
etcd-s3-access-key: "..."
etcd-s3-secret-key: "..."
etcd-s3-skip-ssl-verify: false
```

```bash
sudo systemctl restart k3s
sudo k3s etcd-snapshot list
```

### 13.2 手动快照

```bash
# 创建快照
sudo k3s etcd-snapshot save --name manual-$(date +%Y%m%d-%H%M)

# 列出 / 删除
sudo k3s etcd-snapshot list
sudo k3s etcd-snapshot delete --name manual-20261001-1200

# 本地备份文件位置
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/
```

> 💡 **单节点（SQLite）备份**：直接备份整个数据目录即可，注意先停服务保证一致性：
> ```bash
> sudo systemctl stop k3s
> sudo tar czf k3s-sqlite-$(date +%F).tar.gz -C /var/lib/rancher k3s/server/db
> sudo systemctl start k3s
> ```
> 更稳妥的做法是把单节点也切换为 etcd（`--cluster-init`），以使用快照机制。

### 13.3 恢复（cluster-reset）

```bash
# 1) 停服务（仅需在要恢复的 Server 上操作，建议先停全部 Server 防脑裂）
sudo systemctl stop k3s

# 2) 从快照恢复（会重置 etcd 数据）
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/manual-20261001-1200

# 3) 启动服务
sudo systemctl start k3s

# 4) 其余 Server 重新加入（HA 集群）
#    在其他 Server 上先删除旧数据，再用 --server 重新加入：
#    sudo systemctl stop k3s
#    sudo rm -rf /var/lib/rancher/k3s/server/db   # 谨慎！确认已备份
#    重新执行安装脚本加入集群
```

> ⚠️ 恢复操作**会覆盖现有集群数据**，务必先确认快照文件与恢复点，并在维护窗口内操作。

### 13.4 备份策略建议

| 内容 | 方式 | 频率 | 保留 |
|---|---|---|---|
| etcd 数据 | `etcd-snapshot` + S3 上传 | 每 12 小时 | 7 份 |
| 应用 PV 数据 | NFS 侧快照 / rsync 同步 | 每日 | 7~30 天 |
| 集群清单（YAML/Helm values） | Git 仓库（如本仓库） | 变更即提交 | 永久 |
| Secret | `kubectl get secret -A -o yaml`（加密保存） | 每周 | 长期 |

---

## 14. 升级与卸载

### 14.1 升级方式一：安装脚本（单节点/小集群）

```bash
# Server 节点（逐个升级，先 cordon + drain 业务 Pod）
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 sh -

# Agent 节点（升级时不需要 K3S_URL/K3S_TOKEN）
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 sh -
```

K3s 升级是**原地替换二进制并重启服务**，配置与数据保留。

### 14.2 升级方式二：system-upgrade-controller（推荐用于集群）

```bash
# 1) 部署升级控制器
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/crd.yaml

# 2) 创建升级计划（Server 与 Agent 各一份）
```

```yaml
# server-plan.yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.37.1+k3s1
---
# agent-plan.yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  prepare:
    args:
      - prepare
      - server-plan
    image: rancher/k3s-upgrade
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.37.1+k3s1
```

```bash
kubectl apply -f server-plan.yaml
kubectl apply -f agent-plan.yaml
kubectl -n system-upgrade get plans,jobs -w
```

> 📌 升级路径应逐个次版本递进（如 1.35 → 1.36 → 1.37），不要跨多个次版本直接升级。

### 14.3 卸载

```bash
# Server 节点
/usr/local/bin/k3s-uninstall.sh

# Agent 节点
/usr/local/bin/k3s-agent-uninstall.sh

# 彻底清理残留（确认无需保留数据后再执行）
sudo rm -rf /etc/rancher/k3s /var/lib/rancher/k3s
```

> ⚠️ 上述脚本会删除数据目录。如需保留数据，先备份 `/var/lib/rancher/k3s/server/db`。

---

## 15. 故障排查

### 15.1 常见问题速查

| 现象 | 可能原因 | 处理方式 |
|---|---|---|
| `systemctl status k3s` 一直 `activating` | 端口被占用、cgroup 未挂载、swap 未关 | `journalctl -u k3s -n 100` 查看具体报错；确认 6443 未被占用 |
| 节点 `NotReady` | CNI 未就绪、kubelet 异常、磁盘压力 | `kubectl describe node <name>` 看 Conditions；`journalctl -u k3s-agent -f` |
| Pod 卡 `Pending` | 资源不足、PVC 未绑定、污点未容忍 | `kubectl describe pod` 看 Events；检查 `kubectl get pvc`、`kubectl top nodes` |
| Pod 卡 `ContainerCreating` | 镜像拉取失败、NFS 未挂载 | `kubectl describe pod`；节点上手工 `mount -t nfs` 测试 |
| 镜像拉取超时 | 无外网、未配 registry 镜像 | 配置 `/etc/rancher/k3s/registries.yaml` 并重启，或离线导入镜像 |
| Agent 加入失败 `401 Unauthorized` | Token 错误或过期 | 重新从 Server 取 `/var/lib/rancher/k3s/server/node-token` |
| Agent 加入失败 `x509: certificate signed by unknown authority` | 证书 SAN 未含接入地址 | Server 配置补 `tls-san` → `k3s certificate rotate` → 重启 |
| 跨节点 Pod 通信失败 | `flannel-iface` 选错网卡、防火墙拦截 8472 | 指定正确网卡；放行 UDP 8472 |
| `kubectl` 报 `Unable to connect to the server` | KUBECONFIG 未设置或 server 地址是 127.0.0.1 | 设置 `KUBECONFIG`；远程使用时改成节点真实 IP |
| 80/443 被占用，ingress 起不来 | Traefik/ServiceLB 已占用 | 安装时 `disable: [traefik, servicelb]` |
| `kubectl top` 无数据 | metrics-server 未就绪 | `kubectl -n kube-system get pods \| grep metrics`，等待或查日志 |
| 时间偏差导致证书无效 | NTP 未同步 | `sudo timedatectl set-ntp true` |

### 15.2 排查命令序列

```bash
# 第一层：服务与日志
systemctl status k3s
journalctl -u k3s -n 200 --no-pager | grep -iE 'error|fail|fatal'

# 第二层：节点与组件
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -30

# 第三层：具体对象
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl get pvc,pv -A

# 第四层：网络与存储
sudo k3s crictl ps -a
ip a | grep -E 'flannel|cni'
sudo mount -t nfs 192.168.31.217:/nfs /mnt && ls /mnt && sudo umount /mnt
```

### 15.3 环境自查工具

```bash
# K3s 自带检查（cgroup、iptables、内核参数等）
k3s check-config

# 内核与运行时
uname -r
stat -fc %T /sys/fs/cgroup/
sudo iptables -L -n | head
```

---

## 16. 附录

### 16.1 安装脚本环境变量

| 变量 | 说明 | 示例 |
|---|---|---|
| `INSTALL_K3S_VERSION` | 指定版本 | `v1.37.1+k3s1` |
| `INSTALL_K3S_MIRROR` | 使用中国镜像源 | `cn` |
| `INSTALL_K3S_EXEC` | 传给 server/agent 的参数 | `"server --cluster-init"` |
| `INSTALL_K3S_SKIP_DOWNLOAD` | 跳过下载（离线安装） | `true` |
| `INSTALL_K3S_SKIP_START` | 只安装不启动 | `true` |
| `INSTALL_K3S_SYMLINK` | 是否创建 kubectl 等软链接 | `skip` |
| `INSTALL_K3S_CHANNEL` | 指定更新通道 | `stable` / `latest` |
| `K3S_URL` | Server 地址（Agent 必填） | `https://192.168.31.100:6443` |
| `K3S_TOKEN` | 集群 Token | `K10xxx::server:xxx` |
| `K3S_NODE_NAME` | 节点名 | `k3s-agent-1` |
| `K3S_DATA_DIR` | 数据目录 | `/var/lib/rancher/k3s` |

### 16.2 关键路径速查

| 路径 | 说明 |
|---|---|
| `/usr/local/bin/k3s` | K3s 二进制 |
| `/etc/rancher/k3s/config.yaml` | 主配置文件 |
| `/etc/rancher/k3s/registries.yaml` | 镜像仓库配置 |
| `/etc/rancher/k3s/k3s.yaml` | kubeconfig |
| `/var/lib/rancher/k3s/server/node-token` | 集群 Token |
| `/var/lib/rancher/k3s/server/manifests/` | 自动部署清单目录 |
| `/var/lib/rancher/k3s/server/db/` | SQLite / etcd 数据 |
| `/var/lib/rancher/k3s/server/db/snapshots/` | etcd 快照 |
| `/var/lib/rancher/k3s/storage/` | local-path 存储数据 |
| `/var/lib/rancher/k3s/agent/images/` | 离线镜像包（自动导入） |
| `/var/lib/rancher/k3s/agent/etc/containerd/` | containerd 生成配置 |

### 16.3 常用命令速查

```bash
# 安装 / 卸载
curl -sfL https://get.k3s.io | sh -
/usr/local/bin/k3s-uninstall.sh
/usr/local/bin/k3s-agent-uninstall.sh

# 集群信息
k3s --version
k3s check-config
kubectl get nodes -o wide
kubectl cluster-info

# 服务
systemctl restart k3s
journalctl -u k3s -f

# 容器运行时
sudo k3s crictl ps -a
sudo k3s crictl images
sudo k3s ctr images ls

# 快照
sudo k3s etcd-snapshot save --name manual-01
sudo k3s etcd-snapshot list

# 证书 / 加密
sudo k3s certificate rotate
sudo k3s secrets-encrypt status
```

### 16.4 本目录文件说明

| 文件 | 说明 |
|---|---|
| [`README.md`](README.md) | 本文档，K3s 部署全流程指南 |
| [`config.yaml.example`](config.yaml.example) | Server 配置文件模板（复制到 `/etc/rancher/k3s/config.yaml`） |
| [`agent-config.yaml.example`](agent-config.yaml.example) | Agent 配置文件模板 |
| [`registries.yaml.example`](registries.yaml.example) | 镜像仓库（加速 / 私有仓库认证）模板 |

### 16.5 与仓库其他目录的关系

| 目录 | 关系 |
|---|---|
| [`../yaml/`](../yaml) | K8s 基础对象清单，可直接 `kubectl apply` 到 K3s 集群验证 |
| [`../kind/`](../kind) | kind 本地集群（容器化节点），用于免服务器快速验证；K3s 用于真实集群 |
| [`../helm/`](../helm) | Helm values，可原样用于 K3s 上的 Helm 部署 |
| [`../nfs-pv.yaml`](../nfs-pv.yaml) | NFS 静态存储，K3s 集群可直接复用（见第 10 章） |
| [`../../Docker/docker-compose/`](../../Docker/docker-compose) | compose 应用，可按下表迁移为 K8s 清单（见 11.4 节） |

### 16.6 参考链接

- K3s 官方文档：<https://docs.k3s.io/>
- K3s 版本发布：<https://github.com/k3s-io/k3s/releases>
- 国内镜像站：<https://rancher-mirror.rancher.cn/>
- kube-vip：<https://kube-vip.io/>
- system-upgrade-controller：<https://github.com/rancher/system-upgrade-controller>
- NFS Subdir External Provisioner：<https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner>
- K3s 中文社区文档：<https://docs.rancher.cn/>

---

> 📝 本文档基于 K3s `v1.37.1+k3s1` 编写，示例网段沿用 `192.168.31.0/24`（NFS 服务器 `192.168.31.217`），实际部署时请按环境替换 IP、网卡名与密码。
