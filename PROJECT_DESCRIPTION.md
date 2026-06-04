# Teamgram Server 项目技术描述文档

> 本文档基于对项目代码库的全面分析编写，旨在帮助新团队成员快速理解项目全貌、技术架构及核心实现细节。

---

## 一、项目整体定位与主要功能

### 1.1 项目定位

**Teamgram Server** 是一款基于 Go 语言开发的开源 **MTProto 即时通讯（IM）服务端**，完整实现了 Telegram 官方 MTProto 2.0 协议，兼容各类 Telegram 官方及第三方客户端。项目采用微服务架构设计，支持私有化部署，适用于构建高并发、高可用的即时通讯基础设施。

- **协议兼容性**：支持 MTProto 2.0 全部传输模式（Abridged / Intermediate / Padded Intermediate / Full）
- **API 层版本**：Layer 223
- **开源协议**：Apache License 2.0
- **官方仓库**：`github.com/teamgram/teamgram-server`

### 1.2 主要功能

| 功能域 | 说明 |
|--------|------|
| **私聊** | 一对一文本、多媒体消息收发，已读回执，消息编辑与删除 |
| **基础群组** | 群组创建、成员管理、权限控制、邀请链接 |
| **联系人** | 通讯录同步、联系人推荐、黑名单管理 |
| **文件与媒体** | 图片、视频、文档、语音消息的上传、下载与转码 |
| **会话管理** | 多设备登录、AuthKey 生命周期管理、消息确认机制 |
| **状态同步** | 跨设备实时同步消息、阅读状态、设置变更 |
| **Web 接入** | 支持 HTTP / WebSocket 接入方式 |

---

## 二、技术栈选型与架构设计

### 2.1 技术栈选型

| 层级 | 技术组件 | 选型说明 |
|------|----------|----------|
| **编程语言** | Go 1.23 | 高并发原生支持，编译部署高效 |
| **微服务框架** | go-zero v1.10 | 内置 gRPC、服务发现、熔断、限流、日志、监控 |
| **网关层** | gnet v2.9.1 | 基于事件驱动的高性能网络框架，替代传统 net 处理百万级长连接 |
| **RPC 通信** | gRPC + Protocol Buffers | 内部服务间标准通信协议 |
| **服务发现** | etcd | 服务注册与发现、配置中心 |
| **消息队列** | Kafka (Sarama) | 异步消息投递、事件广播、削峰填谷 |
| **主存储** | MySQL 5.7+ / 8.0 | 关系型数据持久化 |
| **缓存层** | Redis | 会话缓存、热点数据、去重、分布式锁 |
| **对象存储** | MinIO | 兼容 S3 协议的文件、图片、视频存储 |
| **媒体处理** | FFmpeg | 服务端音视频转码、缩略图生成 |
| **序列化** | jsoniter / protobuf | 高性能序列化 |
| **监控与日志** | Prometheus + Grafana + Jaeger + Filebeat + ELK |  Metrics、链路追踪、日志收集 |

### 2.2 整体架构设计

项目采用经典的分层微服务架构，自上而下分为 **接入层（Interface）**、**业务编排层（BFF）**、**消息核心层（Messenger）**、**基础服务层（Service）**，各层通过 gRPC 进行通信，关键异步流程通过 Kafka 解耦。

```
┌─────────────────────────────────────────────────────────────┐
│                      Client (Android/iOS/Web/Desktop)        │
└──────────────────────────┬──────────────────────────────────┘
                           │ MTProto 2.0 / HTTP / WebSocket
┌──────────────────────────▼──────────────────────────────────┐
│  Interface Layer（接入层）                                   │
│  ├─ gnetway      : 高性能 TCP/WS/HTTP 网关，协议握手、AuthKey │
│  ├─ session      : 会话状态机、RPC 路由、消息确认、Push 队列   │
│  └─ httpserver   : HTTP API 网关（Bot / Web 场景）            │
└──────────────────────────┬──────────────────────────────────┘
                           │ gRPC
┌──────────────────────────▼──────────────────────────────────┐
│  BFF Layer（业务编排层）                                     │
│  ├─ bff          : BFF 聚合网关，注册全部子服务 RPC           │
│  ├─ account / authorization / messages / contacts / chats   │
│  ├─ files / dialogs / drafts / updates / users / premium    │
│  └─ ...（共 20+ 个业务子服务）                                │
└──────────────────────────┬──────────────────────────────────┘
                           │ gRPC / Kafka
┌──────────────────────────▼──────────────────────────────────┐
│  Messenger Layer（消息核心层）                               │
│  ├─ msg          : 消息发件箱/收件箱、历史记录、已读回执       │
│  │   └─ inbox    : 收件箱 MQ 消费者，通过 Kafka 异步投递      │
│  └─ sync         : 更新广播、多端同步、Push 推送              │
└──────────────────────────┬──────────────────────────────────┘
                           │ gRPC / Kafka
┌──────────────────────────▼──────────────────────────────────┐
│  Service Layer（基础服务层）                                 │
│  ├─ authsession  : AuthKey、授权状态、FutureSalt 管理        │
│  ├─ biz          : 业务原子服务（user / chat / dialog / code）│
│  ├─ dfs          : 分布式文件存储（对接 MinIO）               │
│  ├─ media        : 媒体元数据管理、照片/视频尺寸信息          │
│  ├─ idgen        : 分布式 ID 生成（Snowflake）               │
│  └─ status       : 用户在线状态、最近上线时间                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 架构核心原则

1. **单一职责**：每个微服务只负责一个业务域，独立编译、独立部署。
2. **无状态设计**：业务服务本身无状态，状态外迁至 Redis / MySQL，便于水平扩展。
3. **异步解耦**：消息投递、更新广播等流程通过 Kafka 异步化，降低核心链路延迟。
4. **协议一致性**：全链路基于自研的 `teamgram/proto` MTProto 协议库，确保前后端语义一致。
5. **代码生成驱动**：大量 `.tl.pb.go`、`_handler.go`、`mysql_dao` 由 `mtprotoc` 和 `dalgen` 工具自动生成，保证协议与数据模型严格同步。

---

## 三、完整目录结构说明

```
teamgram-server/
├── app/                          # 主应用代码，按分层微服务组织
│   ├── interface/                # 接入层（Interface Layer）
│   │   ├── gnetway/              # 高性能网关服务
│   │   ├── session/              # 会话管理服务
│   │   └── httpserver/           # HTTP API 网关
│   ├── bff/                      # 业务编排层（BFF Layer）
│   │   ├── bff/                  # BFF 聚合网关入口
│   │   ├── account/              # 账户管理
│   │   ├── authorization/        # 登录授权、验证码
│   │   ├── messages/             # 消息业务
│   │   ├── contacts/             # 联系人
│   │   ├── chats/                # 群组/频道
│   │   ├── dialogs/              # 会话列表
│   │   ├── files/                # 文件操作
│   │   ├── updates/              # 更新推送
│   │   ├── users/                # 用户资料
│   │   └── ...（其他业务子服务）
│   ├── messenger/                # 消息核心层（Messenger Layer）
│   │   ├── msg/                  # 消息服务（发件箱/收件箱/历史）
│   │   │   └── inbox/            # 收件箱 MQ 消费者
│   │   └── sync/                 # 同步广播服务
│   └── service/                  # 基础服务层（Service Layer）
│       ├── authsession/          # 认证会话
│       ├── biz/                  # 业务原子服务
│       │   ├── biz/              # biz 聚合入口
│       │   ├── user/             # 用户原子服务
│       │   ├── chat/             # 群组原子服务
│       │   ├── dialog/           # 会话原子服务
│       │   └── code/             # 验证码服务
│       ├── dfs/                  # 分布式文件系统（MinIO）
│       ├── media/                # 媒体元数据
│       ├── idgen/                # 分布式 ID 生成
│       └── status/               # 用户在线状态
├── pkg/                          # 公共工具库
│   ├── code/                     # 验证码发送抽象（短信/邮件/无验证码）
│   ├── conf/                     # 公共配置结构
│   ├── deduplication/            # Redis 消息去重
│   ├── goffmpeg/                 # FFmpeg 封装与媒体转码
│   ├── hashx/                    # 哈希工具
│   ├── httpx/                    # HTTP 渲染工具
│   ├── mention/                  # @提及解析
│   ├── net2/                     # 网络工具（含 brpc）
│   ├── phonenumber/              # 手机号格式化与校验
│   └── pubsub/                   # 发布订阅抽象
├── teamgramd/                    # 部署与运维配置
│   ├── bin/                      # 启动/停止脚本
│   ├── deploy/                   # 监控、日志、SQL 迁移
│   │   ├── sql/                  # 数据库初始化与迁移脚本
│   │   ├── prometheus/           # 监控配置
│   │   ├── filebeat/             # 日志收集配置
│   │   └── go-stash/             # 日志处理配置
│   ├── etc/                      # 服务运行配置文件（开发环境）
│   ├── etc2/                     # 服务运行配置文件（生产环境）
│   ├── docker/                   # Docker 入口脚本
│   └── third_party/              # 第三方依赖部署说明
├── docs/                         # 项目文档（安装、架构、监控）
├── specs/                        # 架构规格说明书
├── go.mod                        # Go 模块依赖
├── Makefile                      # 编译构建脚本
├── Dockerfile                    # 容器镜像构建
├── docker-compose.yaml           # 应用容器编排
├── docker-compose-env.yaml       # 基础设施容器编排（MySQL/Redis/Kafka/...）
└── README.md                     # 项目说明
```

---

## 四、各层模块详细说明

### 4.1 接入层（Interface Layer）

#### 4.1.1 `app/interface/gnetway` —— 高性能网关

**职责**：作为客户端与服务端交互的第一道关卡，负责维护海量长连接、MTProto 协议握手、AuthKey 缓存、消息编解码及路由分发。

**关键技术实现**：
- 基于 `gnet/v2` 实现事件驱动的 TCP/WebSocket/HTTP 多协议接入。
- 实现完整的 MTProto 2.0 编解码器族：
  - `mtproto_abridged_codec.go`：紧凑模式
  - `mtproto_intermediate_codec.go`：中间模式
  - `mtproto_padded_intermediate_codec.go`：填充中间模式
  - `mtproto_full_codec.go`：完整模式
  - `mtproto_obfuscated_codec.go`：混淆模式
- 通过 RSA 公钥完成 DH 密钥交换（`handshake.go`），生成永久 AuthKey 与临时 AuthKey。
- `auth_session_manager.go` 管理连接级别的会话映射；`timeout_wheel.go` 实现连接超时与心跳检测。
- 与 `session` 服务通过 gRPC Stream 双向通信，将解密后的业务请求转发至会话层。

**目录结构**：
```
gnetway/
├── cmd/gnetway/main.go           # 服务入口
├── etc/gnetway.yaml              # 配置文件（监听地址、RSA 密钥、Session 客户端）
├── internal/
│   ├── config/config.go          # 配置结构（含 GnetwayServer、RSAKey）
│   ├── server/
│   │   ├── gnet/                 # gnet 服务器实现
│   │   │   ├── codec/            # MTProto 编解码器
│   │   │   ├── server.go         # 事件引擎（OnOpen/OnClose/OnTraffic/OnTick）
│   │   │   ├── handshake.go      # DH 握手流程
│   │   │   ├── auth_key_util.go  # AuthKey 指纹与缓存
│   │   │   └── ...
│   │   └── grpc/                 # gRPC 服务端（供 session 回调）
│   ├── dao/                      # 数据访问（Session 分片路由）
│   └── svc/                      # 服务上下文
├── gateway/                      # protobuf 协议定义与生成代码
└── helper.go                     # 公共辅助函数
```

#### 4.1.2 `app/interface/session` —— 会话管理

**职责**：维护用户与会话的映射关系，处理 RPC 请求的路由与限流，管理消息确认（ACK）、重发、Push 队列，实现多端登录的同步与互斥。

**关键技术实现**：
- `MainAuthWrapper`：每个用户-AuthKey 维度的核心状态机，管理登录态、Layer 版本、设备信息。
- `session_inbound_queue.go` / `session_outgoing_queue.go`：入站/出站消息队列，保证时序。
- `session_invoke.go`：将客户端 RPC 请求通过 gRPC 转发至 BFF 层，并缓存结果等待客户端拉取。
- `session_push_queue.go`：管理服务端主动推送（Updates、RPC Result）。
- `takeout_guard.go`：实现数据导出模式下的请求拦截。
- 支持 **Stream Gateway** 模式：与 gnetway 建立双向流，减少传统 gRPC 调用的连接开销。

**与其他模块交互**：
- 接收来自 `gnetway` 的解密消息 → 调用 `bff` 或 `messenger` 服务 → 将结果/更新通过 `gnetway` 推回客户端。
- 通过 `sync` 服务接收广播事件，推送到当前在线会话。

#### 4.1.3 `app/interface/httpserver` —— HTTP 网关

**职责**：为 Web 端或 Bot 提供基于 HTTP 的 API 接入能力，将 HTTP 请求转换为内部 gRPC 调用。

**关键技术实现**：
- 实现 MTProto 的 HTTP 传输模式（`http_codec.go`）。
- `routes.go` 定义 HTTP 路由，将请求映射到对应的 BFF RPC。
- 同样复用 `AuthKey` 认证体系，支持长轮询（Long Polling）获取 Updates。

---

### 4.2 业务编排层（BFF Layer）

BFF 层由 20+ 个业务子服务组成，每个子服务遵循 **go-zero** 的标准工程结构：`cmd`（入口）、`etc`（配置）、`internal/config`（配置结构）、`internal/core`（业务核心）、`internal/dao`（数据访问）、`internal/server`（RPC 服务端注册）、`client`（RPC 客户端封装）。

#### 4.2.1 `app/bff/bff` —— BFF 聚合网关

**职责**：作为所有 BFF 子服务的统一 gRPC 入口，将不同业务域的 RPC 服务注册到同一台 gRPC Server 上，对外暴露统一的 `mtproto.RPC*` 接口。

**关键技术实现**：
- `server/server.go` 中通过 `zrpc.MustNewServer` 创建 gRPC 服务器，逐一注册各 BFF 子服务的 Server 实现：
  ```go
  mtproto.RegisterRPCTosServer(grpcServer, ...)
  mtproto.RegisterRPCAccountServer(grpcServer, ...)
  mtproto.RegisterRPCMessagesServer(grpcServer, ...)
  // ... 共 20+ 个服务注册
  ```
- 客户端通过 `bff_proxy_client.go` 进行智能路由或 fallback。

#### 4.2.2 典型子服务示例

| 子服务 | 职责 | 依赖的底层服务 |
|--------|------|----------------|
| `authorization` | 手机号校验、验证码发送/校验、登录/注册/登出 | `biz/code`, `authsession`, `biz/user` |
| `messages` | 发送消息、获取历史、删除消息、媒体消息组装 | `msg`, `media`, `user`, `chat`, `sync` |
| `contacts` | 导入通讯录、添加/删除联系人、获取用户状态 | `biz/user`, `status` |
| `chats` | 创建/编辑/删除群组、成员管理、邀请链接 | `biz/chat`, `biz/user` |
| `files` | 上传/下载文件、获取文件元数据 | `dfs`, `media` |
| `updates` | 获取差量更新、设置推送设备 | `sync`, `msg` |

**代码组织模式**：
```
messages/
├── cmd/messages/main.go
├── etc/messages.yaml
├── internal/
│   ├── config/config.go          # 依赖的 RPC 客户端配置（MsgClient、MediaClient...）
│   ├── core/
│   │   ├── core.go               # Core 上下文封装
│   │   ├── messages.sendMessage_handler.go   # 具体业务 Handler
│   │   └── ...
│   ├── dao/dao.go                # DAO 组装（调用下游 Service 的 Client）
│   └── server/
│       ├── server.go             # gRPC Server 注册
│       └── grpc/grpc.go          # 具体 Server 实现
├── client/messages_client.go     # 对外暴露的 RPC 客户端
├── plugin/plugin.go              # 插件接口（业务扩展点）
└── helper.go
```

每个 `_handler.go` 文件对应一个 TL RPC 方法，采用 **显式命令模式**：接收 protobuf 请求 → 参数校验 → 调用 DAO/Client → 组装响应。这种设计使得业务逻辑高度内聚，便于单元测试与性能分析。

---

### 4.3 消息核心层（Messenger Layer）

#### 4.3.1 `app/messenger/msg` —— 消息服务

**职责**：处理消息的全生命周期，包括发件箱（Outbox）、收件箱（Inbox）、历史记录、已读回执、消息编辑与删除。

**关键技术实现**：
- **发件箱逻辑**（`msg.sendMessageV2_handler.go`）：
  1. 参数校验（Peer 类型、消息内容、权限检查）。
  2. 通过 `idgen` 获取全局唯一 Message ID。
  3. 将消息写入发送者的 Outbox（`outbox.go`）。
  4. 根据 Peer 类型（USER/CHAT/CHANNEL）分别处理：
     - **私聊**：直接投递到接收者 Inbox。
     - **群聊**：遍历成员列表，批量投递 Inbox。
  5. 调用 `sync` 服务推送 `Updates` 到相关用户的在线设备。
- **收件箱逻辑**（`inbox/` 子服务）：
  - 作为独立的 Kafka Consumer 运行，消费 `InboxClient` 发送的 MQ 消息。
  - 实现最终一致性：即使接收方服务瞬时不可用，消息也能通过 MQ 重试最终到达。
- **PTS 机制**（`pts.go`）：为每个用户维护 `PTS`（Sequence Number）与 `PTS Count`，客户端通过 `updates.getState` / `updates.getDifference` 拉取差量更新，保证不丢消息、不乱序。
- **消息分片**（`MessageSharding`）：当用户消息量巨大时，可按 `PeerId` 进行数据库分片，配置于 `msg.yaml`。

**数据模型**：
- `messages`：消息主体（Outbox + Inbox 复用或分表）。
- `dialogs`：会话列表（最近一条消息、未读数、置顶状态）。
- `message_read_outbox`：已读回执记录。
- `user_pts_updates`：PTS 增量事件队列。

#### 4.3.2 `app/messenger/sync` —— 同步广播服务

**职责**：将服务端产生的各类更新（新消息、已读状态、设置变更、群组信息变更等）实时推送到用户的所有在线设备。

**关键技术实现**：
- **同步类型**（`SyncType`）：
  - `syncTypeUser`：推送给该用户所有设备。
  - `syncTypeUserNotMe`：推送给该用户除当前操作设备外的其他设备。
  - `syncTypeUserMe`：仅推送给指定设备。
- **广播流程**（`sync.broadcastUpdates_handler.go`）：
  1. 接收上游服务的 `Updates` 对象。
  2. 通过 `status` 服务查询用户当前在线会话列表。
  3. 遍历会话，通过 gRPC Stream 或 MQ 将更新推送到 `session` 服务。
  4. `session` 服务再将更新写入对应连接的 Push Queue。
- **Kafka 解耦**：`sync_mq_client.go` 支持将更新事件先写入 Kafka，由 `sync` 服务消费后再分发，避免上游服务阻塞。

---

### 4.4 基础服务层（Service Layer）

#### 4.4.1 `app/service/authsession` —— 认证会话

**职责**：管理 AuthKey 的全生命周期，包括生成、绑定用户、解绑、查询、FutureSalt 维护等。是所有安全相关操作的根基。

**关键技术实现**：
- `SetAuthKeyV2`：将客户端公钥指纹与 AuthKey 关联写入 MySQL，同时缓存到 Redis。
- `BindAuthKeyUser` / `UnbindAuthKeyUser`：将 AuthKey 与用户 ID 绑定，实现登录态切换。
- `GetFutureSalts`：为每个 AuthKey 维护一组未来盐值（Future Salt），客户端用于生成消息校验和。
- `AuthKeyStateData`：记录 AuthKey 状态（New / PasswordNeeded / LoggedIn / LoggedOut）。

#### 4.4.2 `app/service/biz` —— 业务原子服务

**职责**：提供高度内聚、可复用的业务原子能力，被 BFF 层及 Messenger 层调用。

| 子服务 | 核心功能 |
|--------|----------|
| `user` | 用户资料 CRUD、用户名占用查询、隐私设置、用户状态 |
| `chat` | 群组元数据、成员列表、权限矩阵、邀请链接验证 |
| `dialog` | 会话元数据、置顶、归档、草稿、文件夹分类 |
| `code` | 验证码生成、校验、过期管理（支持短信/邮件/无验证码模式） |

**数据访问模式**：
- 每个原子服务内置 `dal/` 目录，包含：
  - `tables/*.xml`：表结构定义与 SQL 模板。
  - `mysql_dao/`：由 `dalgen` 生成的 MySQL DAO 代码。
  - `dataobject/`：数据库对象（DO）结构体。
- 服务启动时，DAO 层自动初始化 MySQL 连接池与 Redis 缓存（`cache.CacheConf`）。

#### 4.4.3 `app/service/dfs` —— 分布式文件系统

**职责**：封装对象存储（MinIO）操作，提供文件上传、下载、分片、预签名 URL 等能力。

**关键技术实现**：
- `MinioUtil`：封装 MinIO Client，支持桶管理、对象上传（`PutObject`）、分片上传、临时 URL 生成。
- `IDGenClient2`：通过 `idgen` 服务获取全局唯一文件 ID，作为对象存储中的 Key。
- `ssdb`（KV Store）：缓存热点文件的元数据，减少 MinIO 访问延迟。
- 支持图片缩略图、视频首帧抽取（与 `media` 服务协同）。

#### 4.4.4 `app/service/media` —— 媒体元数据

**职责**：管理照片、视频、文档的元数据（尺寸、时长、缩略图、文件引用关系）。

**关键技术实现**：
- `photo.go` / `video.go`：维护媒体属性与文件 ID 的映射。
- 集成 `goffmpeg` 调用 FFmpeg 进行服务端转码，生成多分辨率缩略图或适配视频。
- 与 `dfs` 解耦：`media` 只记录 "用什么文件 ID 表示什么媒体内容"，`dfs` 负责实际存储。

#### 4.4.5 `app/service/idgen` —— 分布式 ID 生成

**职责**：为消息、文件、会话等提供全局唯一、趋势递增的 ID。

**关键技术实现**：
- 基于 `bwmarrin/snowflake` 实现分布式雪花算法。
- 每个服务实例在启动时从 etcd 获取唯一 Worker ID，避免 ID 冲突。
- 提供 gRPC 接口 `idgen.getNextId` / `idgen.getNextIdList`，支持批量预取。

#### 4.4.6 `app/service/status` —— 用户在线状态

**职责**：记录用户最近上线时间、在线状态，为 `sync` 的推送路由与 `contacts` 的状态展示提供数据。

---

### 4.5 公共库（pkg）

| 目录 | 功能 |
|------|------|
| `pkg/code` | 验证码发送策略抽象，支持 `none`（默认 12345）、`me`（自定义 HTTP 接口）、短信平台扩展 |
| `pkg/conf` | 公共配置结构，如 `BFFProxyClients`、`ZRpcServerConf` |
| `pkg/deduplication` | 基于 Redis 的消息/请求去重，防止重复处理 |
| `pkg/goffmpeg` | FFmpeg 封装，提供转码命令构建、进度解析、错误处理 |
| `pkg/hashx` | 一致性哈希等算法工具 |
| `pkg/mention` | UTF-16 偏移计算、@用户名/链接解析 |
| `pkg/net2` | 网络辅助，含 `brpc`（百度 RPC 协议兼容实验） |
| `pkg/phonenumber` | 基于 `nyaruka/phonenumbers` 的手机号格式化与区域判断 |
| `pkg/pubsub` | 发布订阅抽象封装 |

---

## 五、核心业务流程详解

### 5.1 客户端登录流程

```
Client                          gnetway                         session                    authsession           biz/code / biz/user
  |                               |                               |                          |                         |
  |-- (1) TCP/WS 连接 ----------->|                               |                          |                         |
  |                               |-- (2) DH 握手，生成 auth_key_id -->|                        |                         |
  |<-- (3) server_public_key, nonce --|                             |                          |                         |
  |-- (4) 加密请求: auth.sendCode -->|                              |                          |                         |
  |                               |-- (5) 解密后转发到 session ------->|                         |                         |
  |                               |                               |-- (6) RPC 调用 bff/authorization ->|               |
  |                               |                               |                          |-- (7) code.createPhoneCode ->|
  |                               |                               |                          |<-- 返回验证码 (默认 12345) --|
  |<-- (8) 返回 auth.SentCode ----|                               |                          |                         |
  |-- (9) auth.signIn (phone, code) ->|                           |                          |                         |
  |                               | ... 类似转发 ...               |                          |                         |
  |                               |                               |                          |-- (10) 校验验证码、查询/创建用户 |
  |                               |                               |                          |-- (11) authsession.bindAuthKeyUser |
  |<-- (12) 返回 auth.Authorization |                             |                          |                         |
```

**要点**：
- 步骤 (2) 的 DH 交换完全在 `gnetway` 完成，不依赖后端状态。
- 验证码默认值为 `12345`，生产环境需在 `bff.yaml` 中接入真实短信平台。
- AuthKey 与用户绑定后，`session` 建立 `MainAuthWrapper`，后续请求均在此会话上下文中执行。

### 5.2 消息发送与同步流程

```
Client A        session A       bff/messages      msg (outbox)     inbox (Kafka)    session B       Client B
  |                |                |                |                |                |                |
  |-- sendMessage ->|               |                |                |                |                |
  |                |-- RPC ------->|                |                |                |                |
  |                |               |-- msg.sendMessageV2 ----------->|                |                |
  |                |               |                |-- (1) 写入 A 的 Outbox                |                |
  |                |               |                |-- (2) 投递 Kafka (InboxTopic) -------->|                |
  |                |               |                |-- (3) 返回 Updates (含 Outbox)         |                |
  |                |<-- Updates ---|                |                |                |                |
  |<-- Updates ----|               |                |                |                |                |
  |                |               |                |                |-- (4) 消费 MQ，写入 B 的 Inbox       |
  |                |               |                |                |-- (5) 调用 sync.pushUpdates ------->|
  |                |               |                |                |                |-- (6) 查找 B 的在线会话 |
  |                |               |                |                |                |-- (7) 推送 Updates ->|
  |                |               |                |                |                |                |<-- 收到新消息
```

**要点**：
- **Outbox 同步返回**：发送者通过 `msg` 服务的同步调用立即收到包含自身 Outbox 的 `Updates`，保证 UI 即时反馈。
- **Inbox 异步投递**：接收者的 Inbox 写入通过 Kafka 异步化，避免群聊场景下大量收件人导致的阻塞。
- **PTS 与差量同步**：每条消息递增 PTS，离线客户端重新上线后通过 `updates.getDifference` 补齐遗漏消息。
- **多端同步**：`sync` 服务通过 `syncTypeUserNotMe` 将更新推送到发送者的其他设备，保证多端状态一致。

### 5.3 文件上传与下载流程

```
Client                  httpserver / gnetway          bff/files                dfs                  MinIO
  |                         |                           |                      |                      |
  |-- (1) 上传文件分片 ------>|                           |                      |                      |
  |                         |-- (2) RPC 调用 ------------>|                      |                      |
  |                         |                           |-- (3) idgen 获取 file_id                |
  |                         |                           |-- (4) dfs.uploadFile ------------------->|
  |                         |                           |                      |-- (5) PutObject ---->|
  |                         |                           |                      |<-- 返回存储路径 ------|
  |                         |                           |<-- 返回 file_id -----|                      |
  |<-- (6) 返回 InputFile ---|                           |                      |                      |
  |-- (7) sendMedia (引用 file_id) ->|                   |                      |                      |
  |                         | ... 消息流程 ...            |                      |                      |
  |                         |                           |                      |                      |
  |-- (8) 下载请求 ---------->|                           |                      |                      |
  |                         |-- (9) RPC 调用 dfs.downloadFile ----------------->|
  |                         |                           |                      |-- (10) 生成预签名 URL ->|
  |                         |                           |                      |<-- 返回 URL ----------|
  |<-- (11) 重定向或直接流式返回 --|                       |                      |                      |
```

**要点**：
- `dfs` 不直接暴露给客户端，所有文件操作由 `bff/files` 鉴权后代理。
- 小文件可直接通过 gRPC 流传输；大文件建议由 `dfs` 生成临时 URL，客户端直传 MinIO，减轻服务端带宽压力。
- `media` 服务在消息发送时关联 `file_id` 与媒体属性（如图片宽高、视频时长），便于客户端展示。

---

## 六、关键技术实现要点

### 6.1 MTProto 协议栈实现

项目不依赖 Telegram 官方服务端代码，而是从零实现了完整的 MTProto 2.0 协议栈：
- **传输层**：`gnetway/codec` 支持 4 种传输模式及 Obfuscated 混淆，适配不同网络环境。
- **加密层**：AES-256-IGE 用于 Payload 加密，AES-CTR-128 用于 Obfuscated 传输。
- **消息层**：`MsgId`、`SeqNo`、`Salt` 的生成与校验，防止重放攻击。
- **RPC 层**：全量 TL Schema（Layer 223）通过 `mtprotoc` 生成 Go 结构体与 gRPC 接口，位于各 `*.tl.pb.go` 文件中。

### 6.2 高性能网关与连接管理

- **gnet 事件驱动**：利用 `epoll/kqueue`（Linux/macOS）或 `IOCP`（Windows）实现单线程 Reactor 模型，轻松支撑数十万并发连接。
- **AuthKey LRU 缓存**：`gnetway` 本地缓存热点 AuthKey，减少 Redis 查询，提升解密速度。
- **连接分片**：`session` 服务支持按 `auth_key_id` 哈希分片，多个 `session` 实例可横向扩展，通过 etcd 动态发现。

### 6.3 数据访问与代码生成

- **DAL 生成（dalgen）**：开发者在 `tables/*.xml` 中定义表结构与 SQL 模板，执行 `dalgen.sh` 后自动生成：
  - `mysql_dao/*.go`：带缓存的 CRUD 方法。
  - `dataobject/*.go`：与表一一对应的 DO 结构体。
- **TL 生成（mtprotoc）**：从 Telegram 官方 `scheme.tl` 自动生成：
  - `*.tl.pb.go`：protobuf 消息结构 + CRC32 校验。
  - `*_handler.go`：RPC Handler 骨架（需开发者填充业务逻辑）。
  - `rpc_client_registers.go`：客户端调用注册表。

### 6.4 消息可靠性与顺序性

- **ACK 机制**：客户端对每条收到的消息发送 `msgs_ack`，`session` 清理已确认的发送队列，超时未确认则重发。
- **Container 与批处理**：支持 `msg_container` 将多条消息打包发送，减少网络往返。
- **PTS 与 QTS**：
  - `PTS`（Points）：用于普通消息、已读状态等有序事件。
  - `QTS`（Qts）：用于弱序事件（如状态变更），允许客户端按需处理。

### 6.5 扩展与插件机制

部分 BFF 服务（如 `account`、`chats`、`messages`）定义了 `plugin/plugin.go` 接口，允许在不修改核心代码的情况下注入自定义逻辑：
- 消息发送前的内容审核（Anti-Spam）。
- 用户注册时的额外校验（邀请码、IP 限制）。
- 群组操作的外部审计日志上报。

---

## 七、部署与运维

### 7.1 编译构建

项目根目录 `Makefile` 定义了全部服务的编译目标：
```bash
make all      # 编译全部服务
make idgen    # 单独编译 idgen
make gnetway  # 单独编译网关
make clean    # 清理编译产物
```
编译产物输出至 `teamgramd/bin/`。

### 7.2 配置文件

- `teamgramd/etc/*.yaml`：各服务的 go-zero 标准配置，包含：
  - `ListenOn`：gRPC 监听地址。
  - `Etcd`：服务注册中心地址与 Key。
  - `Mysql` / `Cache` / `KV`：数据存储连接信息。
  - `Log` / `Prometheus` / `Telemetry`：可观测性配置。
- 生产环境建议使用 `teamgramd/etc2/*.yaml`，并配合配置中心做热更新。

### 7.3 数据库迁移

- `teamgramd/deploy/sql/1_teamgram.sql`：初始建库脚本。
- `migrate-*.sql`：历次版本迭代的数据库变更脚本，需按日期顺序执行。

### 7.4 容器化

- `docker-compose-env.yaml`：一键启动 MySQL、Redis、etcd、Kafka、MinIO、Jaeger 等基础设施。
- `docker-compose.yaml`：启动 Teamgram Server 主应用容器，暴露 10+ 个服务端口。
- `Dockerfile`：基于多阶段构建，生成精简的运行时镜像。

---

## 八、总结

Teamgram Server 是一款工程化程度极高的开源 IM 服务端，其核心价值在于：

1. **协议级兼容**：完整实现 MTProto 2.0，可直接复用 Telegram 生态的客户端资源。
2. **微服务化架构**：基于 go-zero 构建，天然具备服务治理、熔断、限流、监控能力。
3. **高并发接入**：gnet 网关 + 会话分片 + Redis 缓存，支撑百万级在线用户。
4. **消息高可靠**：Kafka 异步解耦 + PTS 差量同步 + ACK 重发机制，确保消息必达。
5. **工程自动化**：TL Schema 与数据库表定义均通过代码生成工具转化为可维护的 Go 代码，大幅降低协议迭代成本。

对于新团队成员，建议从以下路径逐步深入：
1. 阅读 `app/interface/gnetway` 与 `app/interface/session`，理解连接与会话模型。
2. 阅读 `app/bff/authorization` 与 `app/bff/messages`，掌握核心业务逻辑。
3. 阅读 `app/messenger/msg` 与 `app/messenger/sync`，理解消息流转与同步机制。
4. 阅读 `app/service/biz` 下的原子服务，理解数据模型与 DAO 生成模式。
5. 结合 `docs/` 与 `specs/` 中的架构图与部署文档，完成本地环境搭建与调试。
