# Jenkins packaged by Bitnami

## 什么是 Jenkins？

> Jenkins 是一个开源的持续集成和持续交付（CI/CD）服务器，旨在自动化任何软件项目的构建、测试和部署。

[Jenkins 概述](http://jenkins-ci.org/)
商标声明：此软件列表由 Bitnami 打包。产品中提到的相应商标归相应公司所有，使用这些商标并不意味着任何关联或认可。

## 快速开始

```console
curl -sSL https://raw.githubusercontent.com/bitnami/containers/main/bitnami/jenkins/docker-compose.yml > docker-compose.yml
docker-compose up -d
```

您可以在[环境变量](#environment-variables)部分找到默认凭据和可用的配置选项。

## 为什么使用 Bitnami 镜像？

* Bitnami 密切跟踪上游源代码更改，并使用我们的自动化系统及时发布此镜像的新版本。
* 使用 Bitnami 镜像，最新的错误修复和功能可以尽快获得。
* Bitnami 容器、虚拟机和云镜像使用相同的组件和配置方法 - 使您可以根据项目需求轻松在不同格式之间切换。
* 我们所有的镜像都基于 [minideb](https://github.com/bitnami/minideb)，这是一个极简的基于 Debian 的容器镜像，为您提供小型基础容器镜像和领先 Linux 发行版的熟悉感。
* Docker Hub 中提供的所有 Bitnami 镜像都使用 [Docker Content Trust (DCT)](https://docs.docker.com/engine/security/trust/content_trust/) 进行签名。您可以使用 `DOCKER_CONTENT_TRUST=1` 来验证镜像的完整性。
* Bitnami 容器镜像定期发布，包含最新的发行版软件包。

想在生产环境中使用 Jenkins？试试 [VMware Tanzu Application Catalog](https://bitnami.com/enterprise)，这是 Bitnami Application Catalog 的企业版。

## 如何在 Kubernetes 中部署 Jenkins？

将 Bitnami 应用程序部署为 Helm Charts 是在 Kubernetes 上开始使用我们应用程序的最简单方法。在 [Bitnami Jenkins Chart GitHub 仓库](https://github.com/bitnami/charts/tree/master/bitnami/jenkins) 中阅读有关安装的更多信息。

Bitnami 容器可以与 [Kubeapps](https://kubeapps.dev/) 一起使用，用于在集群中部署和管理 Helm Charts。

## 为什么使用非 root 容器？

非 root 容器镜像增加了额外的安全层，通常建议用于生产环境。但是，因为它们以非 root 用户身份运行，特权任务通常是不允许的。在我们的文档中了解更多关于非 root 容器的信息[在我们的文档中](https://docs.bitnami.com/tutorials/work-with-non-root-containers/)。

## 支持的标签和相应的 `Dockerfile` 链接

在我们的文档页面中了解更多关于 Bitnami 标签策略以及滚动标签和不可变标签之间的区别[在我们的文档页面中](https://docs.bitnami.com/tutorials/understand-rolling-tags-containers/)。

您可以通过查看分支文件夹中的 `tags-info.yaml` 文件来查看不同标签之间的等价关系，即 `bitnami/ASSET/BRANCH/DISTRO/tags-info.yaml`。

通过关注 [bitnami/containers GitHub 仓库](https://github.com/bitnami/containers) 订阅项目更新。

## 获取此镜像

获取 Bitnami Jenkins Docker 镜像的推荐方法是从 [Docker Hub Registry](https://hub.docker.com/r/bitnami/jenkins) 拉取预构建的镜像。

```console
docker pull bitnami/jenkins:latest
```

要使用特定版本，您可以拉取版本化标签。您可以在 Docker Hub Registry 中查看[可用版本列表](https://hub.docker.com/r/bitnami/jenkins/tags/)。

```console
docker pull bitnami/jenkins:[TAG]
```

如果您愿意，也可以通过克隆仓库、切换到包含 Dockerfile 的目录并执行 `docker build` 命令来自己构建镜像。请记住在下面的示例命令中将 `APP`、`VERSION` 和 `OPERATING-SYSTEM` 路径占位符替换为正确的值。

```console
git clone https://github.com/bitnami/containers.git
cd bitnami/APP/VERSION/OPERATING-SYSTEM
docker build -t bitnami/APP:latest .
```

## 如何使用此镜像

### 使用 Docker Compose

此仓库的主文件夹包含一个功能性的 [`docker-compose.yml`](https://github.com/bitnami/containers/blob/main/bitnami/jenkins/docker-compose.yml) 文件。如下所示使用它运行应用程序：

```console
curl -sSL https://raw.githubusercontent.com/bitnami/containers/main/bitnami/jenkins/docker-compose.yml > docker-compose.yml
docker-compose up -d
```

### 使用 Docker 命令行

如果您想手动运行应用程序而不是使用 `docker-compose`，这些是您需要运行的基本步骤：

#### 步骤 1：创建网络

```console
docker network create jenkins-network
```

#### 步骤 2：为 Jenkins 持久化创建卷并启动容器

```console
$ docker volume create --name jenkins_data
docker run -d -p 80:8080 --name jenkins \
  --network jenkins-network \
  --volume jenkins_data:/bitnami/jenkins \
  bitnami/jenkins:latest
```

在 `http://your-ip/` 访问您的应用程序

## 持久化您的应用程序

如果您删除容器，所有数据和配置都将丢失，下次运行镜像时数据库将重新初始化。为避免这种数据丢失，您应该挂载一个即使在容器被删除后仍会持久存在的卷。

对于持久化，您应该在 `/bitnami/jenkins` 路径挂载一个卷。上面的示例定义了一个名为 `jenkins_data` 的 docker 卷。只要不删除此卷，Jenkins 应用程序状态就会持久存在。

为避免意外删除此卷，您可以[将主机目录挂载为数据卷](https://docs.docker.com/engine/tutorials/dockervolumes/)。或者，您可以使用卷插件来托管卷数据。

### 使用 Docker Compose 将主机目录挂载为数据卷

这需要对此仓库中存在的 [`docker-compose.yml`](https://github.com/bitnami/containers/blob/main/bitnami/jenkins/docker-compose.yml) 文件进行小幅更改：

```diff
  ...
  services:
    jenkins:
    ...
    volumes:
-     - 'jenkins_data:/bitnami/jenkins
+     - /path/to/jenkins-persistence:/bitnami/jenkins
- volumes:
-   jenkins_data:
-     driver: local
```

> 注意：由于这是一个非 root 容器，挂载的文件和目录必须具有 UID `1001` 的适当权限。

### 使用 Docker 命令行将主机目录挂载为数据卷

#### 步骤 1：创建网络（如果不存在）

```console
docker network create jenkins-network
```

#### 步骤 2. 使用主机卷创建 Jenkins 容器

```console
docker run -d -p 80:8080 --name jenkins \
  --network jenkins-network \
  --volume /path/to/jenkins-persistence:/bitnami/jenkins \
  bitnami/jenkins:latest
```

## 配置

### 环境变量

当您启动 Jenkins 镜像时，可以通过在 docker-compose 文件或 `docker run` 命令行上传递一个或多个环境变量来调整实例的配置。如果您想添加新的环境变量：

* 对于 docker-compose，在此仓库中存在的 [`docker-compose.yml`](https://github.com/bitnami/containers/blob/main/bitnami/jenkins/docker-compose.yml) 文件的应用程序部分下添加变量名和值：

    ```yaml
    jenkins:
      ...
      environment:
        - JENKINS_PASSWORD=my_password
      ...
    ```

* 对于手动执行，为每个变量和值添加一个 `--env` 选项：

    ```console
    $ docker run -d -p 80:8080 --name jenkins \
      --env JENKINS_PASSWORD=my_password \
      --network jenkins-network \
      --volume /path/to/jenkins-persistence:/bitnami/jenkins \
      bitnami/jenkins:latest
    ```

可用环境变量：

#### 用户和站点配置

* `JENKINS_USERNAME`: Jenkins 管理员用户名。默认值：**user**
* `JENKINS_PASSWORD`: Jenkins 管理员密码。默认值：**bitnami**
* `JENKINS_EMAIL`: Jenkins 管理员邮箱。默认值：**user@example.com**
* `JENKINS_HOME`: Jenkins 主目录。默认值：**/bitnami/jenkins/home**
* `JENKINS_HTTP_PORT_NUMBER`: Jenkins 用于 HTTP 的端口。默认值：**8080**
* `JENKINS_HTTPS_PORT_NUMBER`: Jenkins 用于 HTTPS 的端口。默认值：**8443**
* `JENKINS_EXTERNAL_HTTP_PORT_NUMBER`: Jenkins 在使用 HTTP 访问时用于生成 URL 和链接的端口。默认值：**80**
* `JENKINS_EXTERNAL_HTTPS_PORT_NUMBER`: Jenkins 在使用 HTTPS 访问时用于生成 URL 和链接的端口。默认值：**443**
* `JENKINS_JNLP_PORT_NUMBER`: Jenkins 用于 JNLP 的端口。默认值：**50000**
* `JENKINS_FORCE_HTTPS`: 仅通过 HTTPS 提供 Jenkins 服务。默认值：**no**
* `JENKINS_SKIP_BOOTSTRAP`: 跳过执行初始引导。默认值：**no**

#### JAVA 配置

* `JAVA_OPTS`: 自定义 JVM 参数。无默认值。

## 日志记录

Bitnami Jenkins Docker 镜像将容器日志发送到 `stdout`。要查看日志：

```console
docker logs jenkins
```

或使用 Docker Compose：

```console
docker-compose logs jenkins
```

如果您希望以不同方式使用容器日志，可以使用 `--log-driver` 选项配置容器的[日志驱动程序](https://docs.docker.com/engine/admin/logging/overview/)。在默认配置中，docker 使用 `json-file` 驱动程序。

## 维护

### 备份您的容器

要备份您的数据、配置和日志，请按照以下简单步骤操作：

#### 步骤 1：停止当前运行的容器

* 对于 docker-compose：`$ docker-compose stop jenkins`
* 对于手动执行：`$ docker stop jenkins`

#### 步骤 2：运行备份命令

我们需要在用于创建备份的容器中挂载两个卷：主机上的一个目录用于存储备份，以及我们刚刚停止的容器的卷，以便我们可以访问数据。

```console
docker run --rm -v /path/to/jenkins-backups:/backups --volumes-from jenkins bitnami/os-shell \
  cp -a /bitnami/jenkins /backups/latest
```

### 恢复备份

恢复备份就像在容器中将备份挂载为卷一样简单。

```diff
 $ docker run -d --name jenkins \
   ...
-  --volume /path/to/jenkins-persistence:/bitnami/jenkins \
+  --volume /path/to/jenkins-backups/latest:/bitnami/jenkins \
   bitnami/jenkins:latest
```

### 升级 Jenkins

Bitnami 提供 Jenkins 的最新版本，包括安全补丁，在上游发布后不久就会发布。我们建议您按照以下步骤升级容器。我们将在这里介绍 Jenkins 容器的升级。

### 步骤 1. 获取更新的镜像

```console
docker pull bitnami/jenkins:latest
```

### 步骤 2. 停止您的容器

* 对于 docker-compose：`$ docker-compose stop jenkins`
* 对于手动执行：`$ docker stop jenkins`

### 步骤 3. 拍摄应用程序状态的快照

按照[备份您的容器](#backing-up-your-container)中的步骤拍摄当前应用程序状态的快照。

### 步骤 4. 删除已停止的容器

* 对于 docker-compose：`$ docker-compose rm -v jenkins`
* 对于手动执行：`$ docker rm -v jenkins`

### 步骤 5. 运行新镜像

* 对于 docker-compose：`$ docker-compose up jenkins`
* 对于手动执行（如果需要，挂载目录）：`docker run --name jenkins bitnami/jenkins:latest`

## 自定义此镜像

对于自定义，请注意此镜像默认是一个使用 `uid=1001` 的 `jenkins` 用户的非 root 容器。

### 扩展此镜像

要扩展 bitnami 原始镜像，您可以使用以下格式的 Dockerfile 创建自己的镜像：

```Dockerfile
FROM bitnami/jenkins
## Put your customizations below
...
```

以下是一个扩展镜像的示例，包含以下修改：

* 安装 `vim` 编辑器

```Dockerfile
FROM bitnami/jenkins

## Change user to perform privileged actions
USER 0
## Install 'vim'
RUN install_packages vim
## Revert to the original non-root user
USER 1001
```

### 安装插件

要下载并安装一组插件及其依赖项，请使用 [Plugin Installation Manager tool](https://github.com/jenkinsci/plugin-installation-manager-tool)。您可以在以下指南中找到有关如何使用此工具的信息：

* [Plugin Installation Manager tool 入门](https://github.com/jenkinsci/plugin-installation-manager-tool#getting-started)

或者，可以使用以下环境变量安装插件：

* `JENKINS_PLUGINS`: 在首次启动时要安装的 Jenkins 插件的逗号分隔列表。
* `JENKINS_PLUGINS_LATEST`: 如果设置为 false，则安装 `JENKINS_PLUGINS` 中插件的最低必需版本。默认值：**true**
* `JENKINS_PLUGINS_LATEST_SPECIFIED`: 如果设置为 true，则安装任何请求最新版本的插件的最新依赖项。默认值：**false**
* `JENKINS_OVERRIDE_PLUGINS`: 如果设置为 true，将删除持久卷中的现有插件，并强制重新安装插件。默认值：**false**
* `JENKINS_SKIP_IMAGE_PLUGINS`: 如果设置为 true，则跳过镜像内置插件的安装。默认值：**false**

### 传递 JVM 参数

您可能需要自定义运行 Jenkins 的 JVM，通常是为了传递系统属性或调整堆内存设置。为此目的使用 `JAVA_OPTS` 环境变量：

```console
docker run -d --name jenkins -p 80:8080 \
  --env JAVA_OPTS=-Dhudson.footerURL=http://mycompany.com \
  bitnami/jenkins:latest
```

### 使用自定义启动器参数

为了使用 Jenkins 启动器的自定义参数，例如，如果您需要在反向代理后面安装 Jenkins，前缀为 mycompany.com/jenkins，您可以使用 "JENKINS_OPTS" 环境变量：

```console
docker run -d --name jenkins -p 8080:8080 \
  --env JENKINS_OPTS="--prefix=/jenkins" \
  bitnami/jenkins:latest
```

### 跳过 Bitnami 初始化

默认情况下，运行此镜像时，Bitnami 会实现一些逻辑以配置它以便开箱即用。此初始化包括创建用户和密码、准备要持久化的数据、配置权限、创建 `JENKINS_HOME` 等。您可以通过两种方式跳过它：

* 将 `JENKINS_SKIP_BOOTSTRAP` 环境变量设置为 `yes`。
* 附加一个包含功能完整的 Jenkins 安装的自定义 `JENKINS_HOME` 的卷。

### 向镜像添加文件/目录

您可以自动将文件包含到镜像中。位于 `/usr/share/jenkins/ref` 的所有文件/目录都会复制到 `/bitnami/jenkins/home`（默认 Jenkins 主目录）。

#### 示例

##### 在 Jenkins 启动时运行 groovy 脚本

您可以创建自定义 groovy 脚本并让 Jenkins 在启动时运行它们。

但是，使用此功能将禁用 Bitnami 脚本完成的默认配置。这旨在通过代码自定义 Jenkins 配置。

```console
$ mkdir jenkins-init.groovy.d
$ echo "println '--> hello world'" > jenkins-init.groovy.d/AA_hello.groovy
$ echo "println '--> bye world'" > jenkins-init.groovy.d/BA_bye.groovy

docker run -d -p 80:8080 --name jenkins \
  --env "JENKINS_SKIP_BOOTSTRAP=yes" \
  --volume "$(pwd)/jenkins-init.groovy.d:/usr/share/jenkins/ref/init.groovy.d" \
  bitnami/jenkins:latest

$ docker logs jenkins | grep world
--> hello world!
--> bye world!
```

##### 运行自定义 `config.xml`

您可以使用自己的 `config.xml` 文件。但是，使用此功能将禁用 Bitnami 脚本生成的默认配置。这旨在通过代码自定义 Jenkins 配置。

```console
docker run -d -p 80:8080 --name jenkins \
  --env "JENKINS_SKIP_BOOTSTRAP=yes" \
  --volume "$(pwd)/config.xml:/usr/share/jenkins/ref/config.xml" \
  bitnami/jenkins:latest
```

> 注意：使用此设置，默认的 `admin` 用户将不会被创建。应该单独完成。

## 重要变更

### 2.346.3-debian-11-r3

* 移除了预安装的插件。

### 2.332.2-debian-10-r21

* 默认启用 HTTPS 和 HTTP 支持。
* `JENKINS_ENABLE_HTTPS` 已重命名为 `JENKINS_FORCE_HTTPS`。

### 2.277.4-debian-10-r19

* 容器镜像的大小已减小。
* 配置逻辑现在基于 *rootfs/* 文件夹中的 Bash 脚本。
* 仅持久化 Jenkins Home 目录。
* `install-plugins.sh` 脚本已被弃用。相反，请使用 [安装插件](#installing-plugins) 部分中说明的 Plugin Installation Manager Tool。
* `DISABLE_JENKINS_INITIALIZATION` 环境变量已重命名为 `JENKINS_SKIP_BOOTSTRAP`。

### 2.263.3-debian-10-rXX

* 以下已弃用的插件不再默认包含在镜像中：
  * [GitHub Organization Folder](https://plugins.jenkins.io/github-organization-folder)。
  * [Pipeline: Declarative Agent API](https://plugins.jenkins.io/pipeline-model-declarative-agent)。

### 2.222.1-debian-10-r17

* Java 发行版已从 AdoptOpenJDK 迁移到 OpenJDK Liberica。作为 VMware 的一部分，我们与 Bell Software 达成协议，分发 OpenJDK 的 Liberica 发行版。这样，我们可以为 Java 提供支持和最新版本以及安全版本。

### 2.204.4-debian-10-r3

* Jenkins 容器已迁移到"非 root"用户方法。以前，容器以 `root` 用户身份运行，Jenkins 服务以 `jenkins` 用户身份启动。从现在开始，容器和 Jenkins 服务都以用户 `jenkins`（`uid=1001`）运行。您可以通过在 Dockerfile 中将 `USER 1001` 更改为 `USER root` 来恢复此行为。
* 后果：
  * 当使用 docker 或 docker-compose 持久化数据时，不保证向后兼容性。我们强烈建议迁移您的 Jenkins 数据，确保 `jenkins` 用户具有适当的权限。
  * 不再允许"特权"操作。

### 2.121.2-ol-7-r14 / 2.121.2-debian-9-r18

* 使用 Jetty 而不是 Tomcat 作为 Web 服务器。

### 2.107.1-r0

* Jenkins 容器已迁移到 LTS 版本。从现在开始，此仓库将仅跟踪来自 [Jenkins](https://jenkins.io/changelog-stable/) 的长期支持版本。

## 贡献

我们很乐意让您为这个容器做出贡献。您可以通过创建 [issue](https://github.com/bitnami/containers/issues) 或提交 [pull request](https://github.com/bitnami/containers/pulls) 来请求新功能。

## 问题

如果您在运行此容器时遇到问题，可以提交 [issue](https://github.com/bitnami/containers/issues/new/choose)。为了我们能够提供更好的支持，请务必填写问题模板。

## 许可证

Copyright &copy; 2023 VMware, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
