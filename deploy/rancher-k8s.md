# Rancher 表单部署 RocketMQ Studio（复用已有 MySQL）

本文按 [`docker-compose.yml`](./docker-compose.yml) 中的 `rocketmq-server` 和
`rocketmq-web` 服务，说明如何在 Rancher 网页端创建 Kubernetes 资源。所有部署和验证步骤均使用 Rancher
页面表单。MySQL 使用已有实例，不在 Kubernetes 中创建 MySQL
工作负载或存储卷。

```text
浏览器 -> Ingress -> rocketmq-web Service:80
                         | Nginx 转发 /api、/readyz 等
                         v
                   rocketmq-server Service:8888 -> 已有 MySQL
                                               -> RocketMQ NameServer / Brokers
```

后端 Service 默认仍可命名为 `rocketmq-server`。也可以使用其他名称；此时在
`rocketmq-web` 工作负载中设置 `BACKEND_URL=http://<后端Service名>:8888`。
两个工作负载和 Service 必须部署在同一个 namespace，除非使用其他 namespace 可达的完整 DNS 名称。

## 1. 准备信息和镜像

提前准备：

| 配置 | 示例 | 说明 |
|---|---|---|
| Rancher namespace | `rocketmq-studio` | 可替换为组织约定的 namespace |
| 镜像仓库 | `registry.example.com/team` | K8s 节点需要能拉取镜像 |
| MySQL 地址、端口、库名 | `mysql.example.internal:3306/rocketmq` | 必须能从 Pod 网络访问，不能使用 `localhost` |
| MySQL 用户和密码 | 独立的 Studio 用户 | 需要访问目标数据库；首次初始化/升级还需要 DDL 权限 |
| RocketMQ NameServer | `rmq-ns.example.internal:9876` | 必须能从 server Pod 访问 |
| Web 访问域名 | `studio.example.com` | Ingress 使用的域名 |
| TLS 证书 | Rancher 中的 TLS Secret | HTTPS 部署需要 |

集群网络策略和防火墙还需要允许 server Pod 访问 MySQL、NameServer，以及 RocketMQ
Broker 对外公布的管理地址/端口。

如果 Compose 中的 `registry.cn-hangzhou.aliyuncs.com/yuanzw/rocketmq-studio-server:3.0.0`
和对应的 Web 镜像可供你的集群拉取，也可以直接使用这两个镜像；否则在仓库根目录构建
并推送到 Rancher 集群可访问的镜像仓库：

```bash
docker login registry.example.com

docker build \
  --build-arg VITE_GIT_COMMIT="$(git rev-parse --short HEAD)" \
  -f Dockerfile \
  -t registry.example.com/team/rocketmq-studio-server:3.0.0 .

docker build \
  -t registry.example.com/team/rocketmq-studio-web:3.0.0 \
  web/

docker push registry.example.com/team/rocketmq-studio-server:3.0.0
docker push registry.example.com/team/rocketmq-studio-web:3.0.0
```

Server 使用仓库根目录的 [`Dockerfile`](../Dockerfile)，构建上下文必须是仓库根目录；
Web 的构建上下文是 `web/`。将仓库地址和版本标签替换为实际值。镜像架构必须与
Kubernetes 节点兼容。Server Dockerfile 默认执行许可证检查；正式镜像应使用通过检查
的发布构建。

## 2. 准备已有 MySQL

Compose 中 MySQL 的初始化脚本只对**全新数据目录**执行。复用已有 MySQL 时，需要先在
数据库管理工具（例如 MySQL Workbench、DBeaver 或云厂商控制台 SQL 窗口）中准备数据库
和用户：

```sql
CREATE DATABASE IF NOT EXISTS rocketmq
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER IF NOT EXISTS 'rocketmq_studio'@'<允许访问的来源>'
  IDENTIFIED BY '<强密码>';
GRANT ALL PRIVILEGES ON rocketmq.* TO 'rocketmq_studio'@'<允许访问的来源>';
```

把 `<允许访问的来源>` 限定为实际 Pod 出口地址或组织允许的网段；不要在生产环境中
不加评估地使用 `'%'`。

### 全新 Studio 数据库

在数据库管理工具中选择 `rocketmq` 数据库，执行仓库文件
[`schema.sql`](../server/src/main/resources/db/schema.sql) 的全部内容。此文件是当前
表结构的权威来源；MyBatis 不会自动建表。初始化需要 `CREATE`、`ALTER` 等表结构权限
以及应用所需的读写权限。

### 已有 Studio 数据库

先备份，再确认数据库结构与当前版本兼容。不要把当前 Schema 当作通用升级脚本，也不要
导入 `upgrade-demo-*.sql`：这些脚本是开发演示数据，不是生产迁移脚本。存量版本按实际
版本检查 [`deploy/mysql/`](./mysql/) 中对应的迁移脚本；部分旧版本迁移会修改列、主键
或清理重复记录。启动时的数据库迁移也可能需要 DDL 权限。

## 3. 在 Rancher 网页端创建资源

Rancher 不同版本的菜单名称可能略有差异。以下按常见的 **Cluster Explorer** 页面描述；
表单中的名称、namespace、端口和标签请保持一致。

### 3.1 创建 namespace 和镜像仓库凭据

1. 在目标集群进入 **Projects/Namespaces**，创建 namespace，例如 `rocketmq-studio`。
2. 如果镜像仓库需要登录，在该 namespace 的 **Storage / Secrets** 页面新建
   **Registry** 类型 Secret，填写仓库地址、用户名和密码，例如命名为 `regcred`。
3. 如果使用公开镜像仓库，可跳过 Registry Secret；创建工作负载时也不需要配置
   `imagePullSecrets`。

### 3.2 创建应用配置 ConfigMap

在 namespace 的 **Storage / ConfigMaps** 页面创建 ConfigMap，名称：
`rocketmq-studio-config`。逐项添加以下键值，并把示例主机、域名换成实际值：

| Key | 示例 Value |
|---|---|
| `TZ` | `Asia/Shanghai` |
| `SPRING_PROFILES_ACTIVE` | `prod` |
| `STUDIO_AUTH_LOGIN_REQUIRED` | `true` |
| `STUDIO_AUTH_SESSION_COOKIE_SECURE` | `true` |
| `STUDIO_CORS_ALLOWED_ORIGINS` | `https://studio.example.com` |
| `STUDIO_ROCKETMQ_NAMESRV_ADDR` | `rmq-ns.example.internal:9876` |
| `SPRING_DATASOURCE_URL` | `jdbc:mysql://mysql.example.internal:3306/rocketmq?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai&characterEncoding=UTF-8&connectionCollation=utf8mb4_unicode_ci` |

`STUDIO_ROCKETMQ_NAMESRV_ADDR` 必须填写 server Pod 能访问的 NameServer 地址；这也是
Studio 默认 RocketMQ 管理连接使用的地址。NameServer 返回的 Broker 地址也必须能由
server Pod 访问。

### 3.3 创建密码 Secret

在 **Storage / Secrets** 页面创建 **Opaque** 类型 Secret，名称：
`rocketmq-studio-secret`。添加以下键值：

| Key | Value |
|---|---|
| `SPRING_DATASOURCE_USERNAME` | MySQL Studio 用户名 |
| `SPRING_DATASOURCE_PASSWORD` | MySQL Studio 用户密码 |
| `STUDIO_AUTH_ADMIN_USERNAME` | 初始管理员用户名，例如 `admin` |
| `STUDIO_AUTH_ADMIN_PASSWORD` | 强随机密码 |
| `STUDIO_AI_TOKEN_SECRET` | 随机长密钥；不使用需要签名确认的 AI 工具时可不添加 |

真实密码和 Token 只填写在 Rancher Secret 中，不要放入 ConfigMap，也不要提交到 Git。
`STUDIO_AUTH_ADMIN_USERNAME` 和 `STUDIO_AUTH_ADMIN_PASSWORD` 用于数据库中尚无用户时的
首次管理员登录引导。

HTTPS 部署保持 `STUDIO_AUTH_SESSION_COOKIE_SECURE=true`。若只用 HTTP 做临时测试，需要
在 ConfigMap 中改为 `false` 并重建/重启后端工作负载；生产环境建议启用 HTTPS。

### 3.4 创建 Web DNS ConfigMap

Web 镜像自带的 DNS 脚本针对 Docker/Podman；Kubernetes Pod 应使用自身
`/etc/resolv.conf` 中的 CoreDNS 地址。否则 Nginx 可能无法解析后端 Service 名称。

在 ConfigMaps 页面新建 `rocketmq-studio-web-resolver`，添加一个 Key：

- **Key**：`15-resolver.envsh`
- **Value**：以下多行脚本（整段粘贴，保留换行）

```sh
RESOLVER="$(awk '$1 == "nameserver" { print $2; exit }' /etc/resolv.conf)"
if [ -z "$RESOLVER" ]; then
  echo "Unable to find a Kubernetes DNS nameserver in /etc/resolv.conf" >&2
  exit 1
fi
export RESOLVER
echo "15-resolver.envsh: resolver=$RESOLVER"
```

### 3.5 创建后端 Deployment

在 **Workloads / Deployments** 页面点击 **Create**，填写：

| 表单项 | 值 |
|---|---|
| Name | `rocketmq-server` |
| Namespace | `rocketmq-studio` |
| Replicas | `1` |
| Container name | `server` |
| Image | `registry.example.com/team/rocketmq-studio-server:3.0.0` |
| Image pull policy | `IfNotPresent`；使用可变标签时选 `Always` |
| Container port | 名称 `http`，端口 `8888`，协议 `TCP` |
| Pod label | `app=rocketmq-server` |
| Image pull secret | 私有仓库选 `regcred`；公开仓库留空 |

在工作负载的 **Environment Variables** 区域，使用 Rancher 的 ConfigMap/Secret 引用功能
导入：

- 来自 ConfigMap `rocketmq-studio-config` 的全部变量；
- 来自 Secret `rocketmq-studio-secret` 的全部变量。

如果当前 Rancher 版本没有“导入全部变量”选项，逐项添加变量引用，值来源选择相应的
ConfigMap Key 或 Secret Key；不要把密码直接填成明文环境变量。

在 **Health Check / Probes** 中配置以下 HTTP 探针（某些 Rancher 版本会分别显示
Startup、Readiness、Liveness）：

| Probe | Path | Port | 周期 / 超时 / 失败阈值 |
|---|---|---:|---|
| Startup | `/livez` | `8888` | `10s / 5s / 60` |
| Readiness | `/readyz` | `8888` | `10s / 5s / 6` |
| Liveness | `/livez` | `8888` | `20s / 5s / 3` |

`/readyz` 会检查数据库可用性。若 Rancher 版本不提供 Startup Probe，可先只配置
Readiness 和 Liveness。

### 3.6 创建后端 ClusterIP Service

进入 **Service Discovery / Services**，创建内部服务：

| 表单项 | 值 |
|---|---|
| Name | **`rocketmq-server`** |
| Namespace | `rocketmq-studio` |
| Type | `ClusterIP` / Internal |
| Selector | `app=rocketmq-server` |
| Port | 名称 `http`，协议 `TCP`，Service Port `8888`，Target Port `8888` |

确认 Service 的 Endpoints 中出现后端 Pod。不要将后端服务设为 NodePort 或 LoadBalancer；
浏览器只应访问 Web。

### 3.7 创建前端 Deployment

再次进入 **Workloads / Deployments** 创建工作负载：

| 表单项 | 值 |
|---|---|
| Name | `rocketmq-web` |
| Namespace | `rocketmq-studio` |
| Replicas | `1` |
| Container name | `web` |
| Image | `registry.example.com/team/rocketmq-studio-web:3.0.0` |
| Image pull policy | `IfNotPresent`；使用可变标签时选 `Always` |
| Container port | 名称 `http`，端口 `80`，协议 `TCP` |
| Pod label | `app=rocketmq-web` |
| Environment | 后端名称非默认时设置 `BACKEND_URL=http://<后端Service名>:8888`；默认可不设置 |
| Image pull secret | 私有仓库选 `regcred`；公开仓库留空 |

在 **Volumes / Volume Mounts** 中，将 ConfigMap `rocketmq-studio-web-resolver` 的文件
挂载到 Nginx 启动脚本位置：

| 配置 | 值 |
|---|---|
| Volume 类型 | ConfigMap |
| ConfigMap | `rocketmq-studio-web-resolver` |
| Key / 文件路径 | `15-resolver.envsh` |
| Mount Path | `/docker-entrypoint.d/15-resolver.envsh` |
| SubPath | `15-resolver.envsh` |
| Read-only | 开启 |

必须使用**单文件挂载并设置 SubPath**，不要把 ConfigMap 挂载到整个
`/docker-entrypoint.d` 目录，否则会遮蔽镜像内其他 Nginx 启动脚本。

为 Web 工作负载添加 HTTP 检查：

| Probe | Path | Port | 周期 / 失败阈值 |
|---|---|---:|---|
| Startup | `/` | `80` | `5s / 3s / 30` |
| Readiness | `/` | `80` | `10s / 5s / 3` |
| Liveness | `/` | `80` | `20s / 5s / 3` |

### 3.8 创建前端 ClusterIP Service

在 **Service Discovery / Services** 创建：

| 表单项 | 值 |
|---|---|
| Name | `rocketmq-web` |
| Namespace | `rocketmq-studio` |
| Type | `ClusterIP` / Internal |
| Selector | `app=rocketmq-web` |
| Port | 名称 `http`，协议 `TCP`，Service Port `80`，Target Port `80` |

### 3.9 创建 Ingress

进入 **Service Discovery / Ingresses** 或 Rancher 的 **Ingresses** 页面创建：

| 表单项 | 值 |
|---|---|
| Namespace | `rocketmq-studio` |
| Host | `studio.example.com` |
| Path | `/` |
| Path Type | `Prefix` |
| Target Service | `rocketmq-web` |
| Target Port | `80` |
| Ingress Class | 集群实际使用的 Ingress Controller，例如 `nginx` |
| TLS | 选择覆盖该域名的 TLS Secret，例如 `studio-tls` |

如果 Rancher 集群没有 Ingress Controller，需先由集群管理员安装/配置；也可以按平台规范
将 `rocketmq-web` 暴露为 NodePort。对外只暴露 Web 入口。

## 4. 验证与排障（Rancher 网页端）

1. 在 **Workloads** 页面确认 `rocketmq-server` 和 `rocketmq-web` 都显示
   `1/1 Available` 或 Ready。
2. 如果 Pod 未 Ready，进入对应 Pod 的 **Events** 和 **Logs** 查看原因：
   - `rocketmq-server` 日志中的 JDBC 连接错误：检查 MySQL 地址、网络策略、账号权限和
     Schema。
   - 镜像拉取失败：检查镜像标签、节点架构和 Registry Secret。
   - `rocketmq-web` Nginx 启动或 API 502：确认 DNS ConfigMap 文件挂载到了单文件路径，
     SubPath 正确；再确认 `rocketmq-server` Service 名称、端口和 Endpoints。
3. 在 **Service Discovery / Services** 检查两个 Service 的 Endpoints 均指向相应 Pod。
4. 在 Ingress 页面确认 Host、TLS 和后端 Service 配置正确，然后通过域名访问并登录。

## 5. 更新版本与回滚

构建并推送新的、不可变的镜像标签后，在 Rancher **Workloads / Deployments** 页面分别
编辑 `rocketmq-server` 和 `rocketmq-web`，更新容器 Image 并保存。等待新 Replica Ready
后确认页面和 API 正常。回滚时在工作负载的 **Revision History / Rollback** 中选择上一
个可用版本。

升级前备份已有 MySQL。镜像回滚不一定能回滚已经执行的数据库结构迁移。

## 与 Docker Compose 的差异

- Compose 的 `mysql` 服务、`mysql-data` 卷和 `depends_on` 不部署到 K8s；数据库由外部
  MySQL 提供，通过 `SPRING_DATASOURCE_URL`、`SPRING_DATASOURCE_USERNAME` 和
  `SPRING_DATASOURCE_PASSWORD` 连接。
- Compose 的 Server `expose: 8888` 对应后端 ClusterIP Service；Web 的
  `PUBLIC_PORT:80` 对应 Web ClusterIP Service 加 Ingress。
- Compose 容器网络别名不能直接照搬到 Kubernetes。K8s 内部互访使用 Service DNS；
  NameServer 和 MySQL 地址则使用 Pod 网络可达的 DNS 名或 IP。
- Compose 的 `depends_on` 不会迁移为 K8s 启动顺序；后端使用 `/livez` 和 `/readyz`
  探针，Web 通过 Kubernetes DNS 访问后端 Service。
