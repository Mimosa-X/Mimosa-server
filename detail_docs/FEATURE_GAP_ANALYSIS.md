# Teamgram 社区版功能缺失清单

> 基于 README.md、specs/roadmap.md 及全库代码检索分析得出。以下功能在官方文档中被明确标注为**企业版独占**，或在代码层面仅存在协议/数据库占位、缺乏核心业务逻辑。

---

## 一、官方明确标注为企业版的功能（README / Roadmap）

| 功能大类 | 具体功能 | 社区版状态 |
|---------|---------|-----------|
| **内容表达** | Sticker（贴纸） | 仅支持文件上传接口，无贴纸商店/管理逻辑 |
| | Theme / Chat Theme（主题） | 仅支持文件上传接口，无主题商店/应用逻辑 |
| | Wallpaper（壁纸） | 仅支持文件上传接口，无壁纸商店/应用逻辑 |
| | Reactions（消息表情回应） | 数据库表已预留字段，核心交互逻辑不完整 |
| **安全与认证** | Secret Chat（端到端加密聊天） | 基本未实现，仅存在加密文件上传相关代码 |
| | 2FA（两步验证） | 仅存在 protobuf 协议定义，无设置/校验 handler |
| | SMS（真实短信网关） | 验证码框架存在，但默认返回固定码 `12345`，未接入真实 SMS 提供商 |
| **推送与通知** | Push（APNs / Web / FCM） | 无独立 Push 服务，代码中仅在 session 层有零散引用 |
| **客户端形态** | Web（Web 版客户端支持） | 未发现相关实现（非 WebSocket 网关，指 Web App / WebK / WebZ） |
| | Miniapp（小程序） | **完全未找到任何代码** |
| **消息特性** | Scheduled（定时消息） | 数据库 `dialogs.has_scheduled` 字段存在，无独立业务逻辑 |
| | Autodelete / TTL（自动销毁） | `dialog.setHistoryTTL` 已实现，可设置 TTL，但自动清理调度逻辑待确认 |
| **群组形态** | Channels（频道） | 无 `createChannel` handler，无独立 `channels` 表，核心创建/订阅/广播逻辑缺失 |
| | Megagroups（超级群组） | 与 Channels 同属企业版，社区版仅支持 Basic Group |
| **实时通讯** | Audio / Video Call（语音/视频通话） | 无 `startCall` / `acceptCall` 等核心信令，无 `phone_calls` 表 |
| | Group Call / Conference（群组/会议通话） | `chat_participants` 表预留了 `groupcall` 字段，但无媒体服务器/SFU 逻辑 |
| **机器人** | Bots（机器人完整生态） | 基础数据管理（`bots` 表、`updateBotData`）存在，但 **Bot API / Webhook / Inline Query / 支付** 等高级功能缺失 |

---

## 二、代码层面详细验证结果

### 1. Channels / Megagroups（频道 / 超级群组）—— 核心逻辑缺失

- **缺失点**：
  - 未找到 `createChannel`、`joinChannel`、`leaveChannel`、`setChannelAdmin` 等核心 handler。
  - 数据库中**无独立 `channels` 表**，仅有 `chats` 表通过 `migrated_to_access_hash` 等字段做基础群聊支持。
- **已有占位**：
  - `updateNewChannelMessage` 在更新合并助手（`merge_updates_helper.go`）中有协议层面的 case 处理。
  - `user.updatePersonalChannel`、`getChannelUsername`、`deactivateAllChannelUsernames` 存在，说明**频道用户名管理**有局部实现，但依附于 User 服务而非独立 Channel 服务。

### 2. Secret Chat（加密聊天）—— 基本未实现

- **缺失点**：
  - 未找到密钥交换（DH）、加密会话生命周期管理、前向保密等核心逻辑。
- **已有占位**：
  - `messages.uploadEncryptedFile` handler 存在，说明加密文件上传通道已预留。

### 3. 2FA（两步验证）—— 仅协议定义

- **缺失点**：
  - 未找到 `account.updatePasswordSettings`、`account.confirmPasswordEmail` 等核心 handler。
- **已有占位**：
  - `user.tl.proto` 等 protobuf 文件中有相关 class name 和 CRC32 注册，属于协议兼容性占位。

### 4. Audio / Video / Group / Conference Call —— 信令与媒体服务器缺失

- **缺失点**：
  - 无 `phoneCallRequested` / `phoneCallAccepted` 等完整信令 handler。
  - 无 `phone_calls` 表，无 SFU / MCU 媒体服务器逻辑。
- **已有占位**：
  - `messages.SelectPhoneCallList` DAO 方法存在，支持按通话类型筛选消息记录。
  - `msg.deletePhoneCallHistory_handler` 存在，仅用于删除通话历史。
  - `user_privacy_key_phone_call` 隐私规则已实现。
  - `chat_participants` 表有 `groupcall_default_join_as_peer_type` 等字段，说明群通话有数据库预留。

### 5. Bots（机器人）—— 高级功能缺失

- **缺失点**：
  - 未找到 **Bot API Server**（接收 Webhook、处理 `/command` 的 HTTP 服务）。
  - 未找到 **Inline Query**、**Keyboard**、**Payment**、**Game** 等交互 handler。
- **已有占位**：
  - `bots` 表、`bot_commands` 表完整存在。
  - `user.updateBotData_handler` 已实现，可更新 bot 的基本属性（`bot_inline_geo`、`bot_inline_placeholder` 等）。
  - protobuf 中 `BotInfoData` 结构完整，支持序列化/反序列化。

### 6. Miniapp（小程序）—— 完全未实现

- 全库搜索 `miniapp`、`mini_app`、`web_app` 等业务关键词，**无任何匹配结果**。

### 7. Push（APNs / Web / FCM）—— 推送服务缺失

- **缺失点**：
  - 无独立 `push` 微服务。
  - 未找到对接 Apple APNs、Google FCM、Web Push 的 SDK 集成代码。
- **已有占位**：
  - `session` 模块中有 Push 相关的 session 同步逻辑引用，但仅为框架级调用，无真实推送能力。

### 8. Scheduled Message（定时消息）—— 业务逻辑缺失

- **缺失点**：
  - 未找到 `messages.sendScheduledMessage`、`messages.getScheduledHistory` 等 handler。
- **已有占位**：
  - `dialogs` 表有 `has_scheduled` 字段，用于标记对话框是否存在定时消息。

### 9. Stories（快拍）—— 仅数据库预留

- **缺失点**：
  - 未找到 `stories.*` 相关 handler、独立表或媒体处理逻辑。
- **已有占位**：
  - `users` 表有 `stories_max_id` 字段。
  - `user_contacts` 表有 `stories_hidden` 字段。
  - protobuf 定义中有 Stories 相关 class name 注册。

### 10. Sticker / Theme / Wallpaper —— 仅文件上传

- **缺失点**：
  - 无贴纸包管理、主题应用、壁纸商店的业务逻辑。
- **已有占位**：
  - `media` 服务提供 `uploadStickerFile`、`uploadThemeFile`、`uploadWallPaperFile` handler，仅完成文件落盘。

### 11. Reactions（消息表情回应）—— 数据库支持，逻辑不完整

- **已有实现**：
  - `messages` 表：`reaction`、`reaction_date`、`reaction_unread`、`has_reaction`。
  - `dialogs` 表：`unread_reactions_count`。
  - `chats` 表：`available_reactions`、`available_reactions_type`。
- **待确认缺失**：
  - `messages.sendReaction`、`messages.getMessagesReactions` 等核心 handler 是否完整实现需进一步验证 BFF 层。数据库层已完备。

### 12. Web（Web 版客户端）—— 未实现

- 项目中的 `gnetway` 虽然支持 WebSocket 作为 MTProto 传输层，但这只是**协议网关**。
- 官方文档中提到的 "web" 指完整的 **Web 客户端应用**（如 Telegram WebK / WebZ），在代码库中未发现对应的静态资源服务或前端适配逻辑。

---

## 三、社区版**已实现**的亮点功能（作为对比）

| 功能 | 说明 |
|------|------|
| **QR Code 登录** | 完整的 `qrcode` 微服务，支持 `auth.exportLoginToken`、`auth.importLoginToken`、`auth.acceptLoginToken` |
| **Basic Group（基础群聊）** | 完整的创建、邀请、踢人、设置管理员等逻辑 |
| **Private Chat（私聊）** | 完整的端到端消息收发、已读回执、历史消息拉取 |
| **Contacts（联系人）** | 导入、同步、删除、隐私设置 |
| **Media（媒体文件）** | 图片、视频、文档、语音的上传、下载、缩略图、FFmpeg 转码 |
| **Message TTL（消息自动销毁）** | `dialog.setHistoryTTL` 已实现，支持设置对话级 TTL |
| **Username（用户名）** | 完整的用户名注册、解析、占用检查 |
| **Auth（认证）** | 手机注册/登录、验证码（本地模拟）、Session 管理、多端同步 |

---

## 四、结论

社区版 Teamgram 已经实现了 **Telegram 最核心的 IM 能力**（私聊、基础群、联系人、媒体、认证），但以下商业化/高级功能被明确限制在企业版：

1. **频道与超级群组**（Channels / Megagroups）
2. **端到端加密**（Secret Chat）
3. **两步验证**（2FA）
4. **音视频通话**（Audio / Video / Group / Conference Call）
5. **机器人完整生态**（Bots 高级能力）
6. **小程序**（Miniapp）
7. **真实推送**（Push）
8. **真实短信**（SMS 网关）
9. **Web 客户端**
10. **Stories、Scheduled、Sticker/Theme 商店** 等周边功能

> **评估建议**：如果你需要一个**自托管的即时通讯基础平台**，社区版足够支撑私聊、小群、文件传输等核心场景。若需要**频道运营、机器人平台、音视频通话、端到端加密**等能力，则需联系作者获取企业版授权。
