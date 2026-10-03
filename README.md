# LearnDocker

Docker 与 Kubernetes 学习仓库：包含常用自托管应用的 docker-compose 编排实战、Dockerfile 参考源码，以及 K8s 基础对象、kind 本地集群、K3s 集群部署与 Helm 的入门练习。

## 📁 目录结构

```
LearnDocker/
├── Docker/
│   ├── docker-compose/          # 21 个自托管应用的 compose 配置
│   └── dockerfile/              # Dockerfile 学习资料
│       ├── jenkins/             # Bitnami Jenkins 完整 Dockerfile 源码树
│       ├── jenkins-agent/       # Jenkins Agent 镜像
│       └── containers-main/     # （不入库）Bitnami containers 仓库快照
└── K8s/
    ├── yaml/                    # K8s 基础对象练习：Pod / Deployment / StatefulSet / DaemonSet / CronJob / PV
    ├── kind/                    # kind 本地集群：部署文档 + 集群配置（多节点、HA）
    ├── K3s/                     # K3s 集群部署文档与配置模板（单节点 / 多节点 / HA）
    ├── helm/                    # Jenkins 的 Helm values 自定义
    ├── charts-main/             # （不入库）Bitnami charts 仓库快照
    ├── nfs-pv.yaml / nfs-pvc*.yaml   # NFS 存储练习
    └── nginx-*.yaml             # nginx 部署与服务暴露练习
```

## 🐳 Docker Compose 应用清单

| 类别 | 应用 | 说明 |
|---|---|---|
| 下载 / 媒体 | qbittorrent、navidrome、TinyMediaManager、music_tag_web | 下载器、音乐服务器、影视刮削 |
| NAS 工具 | nas-tools、CloudDriver2、cookiecloud、SMB | 媒体自动化、网盘挂载、Cookie 同步、文件共享 |
| 智能家居 / 门户 | homeassistant、homepage、caddy | 智能家居、导航页、反向代理 |
| 效率工具 | IT-Tools、vaultwarden、syncthing、IMMICH | 开发工具箱、密码管理、文件同步、相册备份 |
| 监控 / DevOps | netdata、gitlab、jenkins-compose、obsandian | 监控、代码托管、CI、笔记 |

## 🚀 快速开始

### 使用 compose 启动应用

部分应用（jenkins-compose、mysql、SMB、IMMICH）的敏感配置已迁移到 `.env` 文件：

```bash
cd Docker/docker-compose/<应用名>
cp .env.example .env   # 然后编辑 .env 填入自己的密码等信息
docker compose up -d
```

> ⚠️ `.env` 已加入 `.gitignore`，不会被提交；仓库中只保留 `.env.example` 模板。

### 第三方参考源码（体积大，默认不入库）

`Docker/dockerfile/containers-main/` 与 `K8s/charts-main/` 是 Bitnami 官方仓库快照，仅作学习参考，已从 Git 移除跟踪。如需获取：

```bash
git clone --depth 1 https://github.com/bitnami/containers.git Docker/dockerfile/containers-main
git clone --depth 1 https://github.com/bitnami/charts.git K8s/charts-main
```

## ☸️ Kubernetes 练习

### kind 本地集群

用 Docker 容器模拟 K8s 节点，无需真实服务器即可搭建多节点集群，详细文档见 **[K8s/kind/README.md](K8s/kind/README.md)**。

```bash
# 1 控制面 + 3 worker
kind create cluster --config K8s/kind-4nodes.yaml

# 高可用集群（多控制面 + 端口映射）
kind create cluster --config K8s/kind/kind-ha.yaml

# 单节点 + Ingress 端口映射
kind create cluster --config K8s/kind/kind-config-single.yaml

# 查看 / 删除
kind get clusters
kind delete cluster --name kind-ha
```

可用集群配置：

| 文件 | 拓扑 |
|---|---|
| [K8s/kind-4nodes.yaml](K8s/kind-4nodes.yaml) | 1 控制面 + 3 worker（最简） |
| [K8s/kind/kind-ha.yaml](K8s/kind/kind-ha.yaml) | 2 控制面 + 3 worker（多控制面） |
| [K8s/kind/kind-config-single.yaml](K8s/kind/kind-config-single.yaml) | 单节点 + 80/443 映射 |
| [K8s/kind/kind-config-registry.yaml](K8s/kind/kind-config-registry.yaml) | 本地镜像仓库 + 2 worker |
| [K8s/kind/kind-test.yaml](K8s/kind/kind-test.yaml) | 自定义网段 / ipvs / 固定 API 端口 |

### 基础对象

```bash
kubectl apply -f K8s/yaml/                 # Pod / Deployment / StatefulSet / DaemonSet / CronJob
kubectl apply -f K8s/nfs-pv.yaml           # NFS PV/PVC 存储
```

### K3s 集群部署

轻量级生产可用集群，支持单节点 / 多节点 / 高可用（嵌入式 etcd）三种拓扑，详细文档见 **[K8s/K3s/README.md](K8s/K3s/README.md)**。

```bash
# 单节点快速起步（Linux 主机执行）
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes

# 高可用集群的第 1 台 Server
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init --tls-san <VIP>" sh -
```

配套配置模板：

| 文件 | 用途 |
|---|---|
| [K8s/K3s/config.yaml.example](K8s/K3s/config.yaml.example) | Server 配置（HA、组件开关、etcd 快照） |
| [K8s/K3s/agent-config.yaml.example](K8s/K3s/agent-config.yaml.example) | Agent 配置（加入集群、节点标签） |
| [K8s/K3s/registries.yaml.example](K8s/K3s/registries.yaml.example) | 镜像加速与私有仓库认证 |

### Helm 部署 Jenkins

```bash
helm install jenkins oci://registry-1.docker.io/bitnamicharts/jenkins \
  -f K8s/helm/bitnami-jenkins.yaml \
  --set jenkinsPassword=<你的密码>   # 密码通过 --set 传入，勿写入 values 文件
```

## 🔐 安全约定

- 所有密码、密钥统一放在各服务目录下的 `.env` 文件中（已被 `.gitignore` 忽略），模板见对应的 `.env.example`。
- Helm values 中不写真实密码，安装时用 `--set` 覆盖。
- 集群证书（`kubecfg.crt/key/p12`）不入库。
- 若历史上曾提交过敏感信息，请及时更换对应密码。

## 📄 License

[Apache License 2.0](LICENSE)
