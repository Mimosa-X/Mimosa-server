# Teamgram Server Docker 部署指南

> 本文档面向需要快速在本地或测试环境部署 Teamgram Server 的开发者。通过 Docker Compose，可在几分钟内一键启动完整的 IM 服务端及全部依赖基础设施。

---

## 一、前置条件

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| Docker | 20.10+ | 容器运行时 |
| Docker Compose | V2 (`docker compose`) | 编排工具，旧版 `docker-compose` 亦可兼容 |
| Git | 任意 | 克隆源码仓库 |
| 系统内存 | 建议 ≥ 8GB | 同时运行 MySQL、Kafka、Elasticsearch 等较为吃内存 |

**Windows 用户特别提示**：
- 请确保 Docker Desktop 已启动，并且设置为 **Linux 容器模式**（默认）。
- 建议在 WSL2 环境下执行命令，或在 PowerShell 中运行。

---

## 二、部署架构概览

本项目采用 **分离式 Compose 设计**：

| Compose 文件 | 作用 | 镜像来源 |
|--------------|------|----------|
| `docker-compose-env.yaml` | 启动全部基础设施：MySQL、Redis、etcd、Kafka、MinIO、Jaeger、Prometheus、Grafana、Elasticsearch、Kibana、Filebeat、Go-Stash | 拉取 Docker Hub 官方镜像（无需本地编译） |
| `docker-compose.yaml` | 启动 Teamgram 应用本身（gnetway、session、bff、msg、sync 等全部微服务） | **基于本地源码构建**（`build: .`） |

**为什么要分成两个文件？**
- 基础设施镜像成熟稳定，可直接拉取，启动速度快。
- Teamgram 应用镜像需要根据你的本地代码定制编译（如修改业务逻辑、配置插件等），因此必须现场构建。

---

## 三、快速开始（推荐方式）

### 步骤 1：克隆仓库

```bash
git clone https://github.com/teamgram/teamgram-server.git
cd teamgram-server
```

### 步骤 2：配置环境变量（可选但强烈建议）

项目提供了 `.env.example` 模板，复制为 `.env` 后可根据需求修改密码等敏感项：

```bash
cp .env.example .env
```

默认配置如下（**生产环境请务必修改**）：

```ini
# MySQL
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=teamgram
MYSQL_USER=teamgram
MYSQL_PASSWORD=teamgram

# MinIO
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=miniostorage

# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
GRAFANA_ROOT_URL=http://localhost:3000
```

> `.env` 文件会被 `docker-compose-env.yaml` 自动读取，无需手动注入。

### 步骤 3：启动基础设施

```bash
docker compose -f docker-compose-env.yaml up -d
```

**首次启动时，系统会自动完成以下初始化**：
- **MySQL**：挂载 `teamgramd/deploy/sql/` 目录，自动执行 `1_teamgram.sql` 建库建表，以及所有 `migrate-*.sql` 迁移脚本。
- **MinIO**：`minio-mc` 容器自动创建所需的存储桶（`documents`、`encryptedfiles`、`photos`、`videos`）。
- **Kafka**：自动创建预设 Topic（`Inbox-T`、`Sync-T`、`teamgram-log`）。

**等待约 30~60 秒**，待所有容器健康检查通过后再进行下一步。可通过以下命令查看状态：

```bash
docker compose -f docker-compose-env.yaml ps
```

### 步骤 4：构建并启动 Teamgram 应用

```bash
docker compose up -d --build
```

参数说明：
- `--build`：强制根据当前目录的 `Dockerfile` 重新编译镜像。第一次运行时必须加此参数。
- `-d`：后台运行。

**构建过程简述**：
1. `Dockerfile` 第一阶段（`builder`）：基于 `golang:1.23.0` 镜像，将本地源码复制进容器，执行 `./build.sh` 编译所有微服务二进制文件。
2. `Dockerfile` 第二阶段（`runtime`）：基于 `ubuntu:latest`，安装 FFmpeg 等运行时依赖，从第一阶段拷贝编译产物。
3. 容器启动后执行 `entrypoint.sh` → 调用 `runall-docker.sh` 按顺序启动所有服务。

> 首次构建可能需要 **5~15 分钟**（取决于网络与机器性能），因为需要下载 Go 依赖并编译。后续若未修改代码，可直接 `docker compose up -d` 复用已有镜像。

---

## 四、验证部署是否成功

### 4.1 查看容器状态

```bash
# 查看基础设施容器
docker compose -f docker-compose-env.yaml ps

# 查看 Teamgram 应用容器
docker compose ps
```

所有容器状态应为 `Up (healthy)` 或 `Up`。

### 4.2 查看应用日志

Teamgram 应用的所有微服务运行在同一个容器内，日志输出到标准输出：

```bash
docker logs -f teamgram-server-teamgram-1
```

> 容器名可能因目录名不同而有差异，可用 `docker ps` 查看实际名称。

### 4.3 测试端口连通性

Teamgram 应用对外暴露的主要端口：

| 端口 | 用途 |
|------|------|
| 10443 / 11443 | gnetway MTProto 网关 |
| 5222 | gnetway 备用端口 |
| 8801 | HTTP/WebSocket 网关 |
| 20450 / 20420 | session 服务 |
| 20010 | bff 服务 |

基础设施管理界面：

| 服务 | 地址 | 默认账号 |
|------|------|----------|
| MinIO Console | http://127.0.0.1:9001 | minio / miniostorage |
| Grafana | http://127.0.0.1:3000 | admin / admin |
| Jaeger UI | http://127.0.0.1:16686 | - |
| Kibana | http://127.0.0.1:5601 | - |
| Prometheus | http://127.0.0.1:9090 | - |

---

## 五、常用运维命令

### 5.1 停止服务

```bash
# 停止 Teamgram 应用（不影响基础设施）
docker compose down

# 停止基础设施（数据默认保留在 ./data/ 目录）
docker compose -f docker-compose-env.yaml down

# 彻底清理（删除容器 + 卷 + 镜像）
docker compose down --rmi all -v
docker compose -f docker-compose-env.yaml down --rmi all -v
```

### 5.2 修改代码后重新部署

```bash
# 修改源码后，强制重新构建镜像并重启容器
docker compose up -d --build
```

### 5.3 进入容器排查问题

```bash
# 进入 Teamgram 应用容器
docker exec -it teamgram-server-teamgram-1 /bin/bash

# 查看容器内进程
docker exec teamgram-server-teamgram-1 ps aux

# 查看容器内配置
docker exec teamgram-server-teamgram-1 ls -la /app/etc2/
```

### 5.4 单独重启某个微服务（容器内操作）

由于所有微服务跑在同一个容器内，推荐直接在容器内操作：

```bash
docker exec -it teamgram-server-teamgram-1 /bin/bash

# 进入工作目录
cd /app/bin

# 例如：单独重启 gnetway
pkill gnetway
./gnetway -f=../etc2/gnetway.yaml &
```

> **生产环境建议**：将每个微服务拆分为独立容器或独立 Pod，便于单独扩缩容与故障隔离。

### 5.5 查看构建的镜像

```bash
docker images
```

你会发现类似以下的镜像记录：

```
REPOSITORY             TAG       IMAGE ID       CREATED          SIZE
teamgram-server        latest    abc123def456   10 minutes ago   1.2GB
golang                 1.23.0    xxx            2 weeks ago      1GB
```

- `teamgram-server` 就是根据你本地代码构建的运行时镜像。
- 镜像实际存储在 Docker 本地仓库（Windows 下位于 WSL2 虚拟磁盘内），不在项目目录中。

---

## 六、关键文件说明

| 文件 | 说明 |
|------|------|
| `Dockerfile` | 多阶段构建定义：阶段一编译 Go 二进制，阶段二生成带 FFmpeg 的运行镜像 |
| `build.sh` | Go 编译脚本，依次编译 idgen、status、dfs、media、authsession、biz、msg、sync、bff、session、gnetway |
| `teamgramd/docker/entrypoint.sh` | 容器启动入口，设置环境变量后调用 `runall-docker.sh` |
| `teamgramd/bin/runall-docker.sh` | 按顺序启动所有微服务进程（后台运行） |
| `teamgramd/etc/*.yaml` | 开发环境配置文件（监听 127.0.0.1） |
| `teamgramd/etc2/*.yaml` | 容器内实际使用的配置文件（已替换为容器网络地址） |
| `docker-compose-env.yaml` | 基础设施编排：数据库、缓存、消息队列、对象存储、监控 |
| `docker-compose.yaml` | 应用编排：构建并运行 Teamgram 主容器 |
| `.env` | 环境变量文件，控制 MySQL/MinIO/Grafana 的初始密码 |

---

## 七、常见问题排查

### Q1：MySQL 启动失败或不断重启

**可能原因**：
- `./data/mysql` 目录权限不足（Linux/macOS 常见）。
- 端口 3306 已被本地 MySQL 占用。

**解决**：
```bash
# 检查端口占用
sudo lsof -i :3306

# 清理旧数据后重启（注意会丢失数据）
sudo rm -rf ./data/mysql
docker compose -f docker-compose-env.yaml up -d mysql
```

### Q2：Teamgram 应用容器启动后马上退出

**可能原因**：
- 基础设施尚未完全就绪（如 MySQL 还在初始化）。
- `build.sh` 编译失败（网络问题导致 Go 依赖拉取失败）。

**解决**：
```bash
# 查看具体报错
docker logs teamgram-server-teamgram-1

# 若编译失败，尝试手动进入 builder 阶段排查
docker build --target builder -t teamgram-builder .
```

### Q3：Kafka 连接不上

**可能原因**：Kafka 容器启动较慢，Teamgram 应用启动时 Kafka 尚未就绪。

**解决**：
- 确保启动 Teamgram 应用前，`docker compose -f docker-compose-env.yaml ps` 中 kafka 状态为 `healthy`。
- 或在 `docker-compose-env.yaml` 中为 Kafka 增加更长的 `healthcheck` 间隔。

### Q4：如何修改微服务的配置？

当前 Docker 部署使用的是容器内的 `/app/etc2/*.yaml`。由于 `entrypoint.sh` 中相关配置生成逻辑已被注释，目前这些配置文件是直接从 `teamgramd/etc2/` 复制进镜像的。

**如需修改**：
1. 修改本地 `teamgramd/etc2/` 下的对应 YAML 文件。
2. 重新构建镜像：`docker compose up -d --build`。

---

## 八、生产环境部署建议

当前单容器多进程的模式适合 **开发测试** 与 **快速体验**。若用于生产，建议进行以下改造：

1. **服务拆分**：将每个微服务（gnetway、session、bff、msg 等）拆分为独立的 Docker 服务或 Kubernetes Deployment，便于独立扩缩容。
2. **配置外置**：将 `etc2/*.yaml` 挂载为外部 ConfigMap 或卷，避免每次改配置都重新构建镜像。
3. **数据库与 Kafka 外置**：使用云厂商的 RDS、Redis、Kafka、对象存储服务，替代容器化的中间件。
4. **健康检查**：为每个微服务添加 Docker `HEALTHCHECK` 指令，配合负载均衡自动剔除故障实例。
5. **日志采集**：Filebeat 已包含在 `docker-compose-env.yaml` 中，确保应用日志写入 `/app/logs/` 以便采集。

---

## 九、卸载与清理

如需彻底删除所有容器、镜像与数据：

```bash
# 停止并删除所有服务
docker compose down --rmi all -v
docker compose -f docker-compose-env.yaml down --rmi all -v

# 清理 Docker 系统缓存（可选）
docker system prune -a

# 删除本地持久化数据（彻底清空数据库与文件）
rm -rf ./data
```

> **警告**：`rm -rf ./data` 会删除所有 MySQL 数据、MinIO 文件、Redis 持久化数据，操作前请确认无需保留。
