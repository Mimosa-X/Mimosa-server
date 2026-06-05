# Teamgram 数据库结构深度分析文档

> 本文档基于 `mimosa.sql`（由 `teamgramd/deploy/sql/` 下全部 SQL 文件合并而成）进行全面解析，涵盖 38 张核心数据表，结合 Telegram（MTProto）即时通讯架构特性，阐述各表的业务职责、字段含义及技术实现要点。

---

## 一、数据库整体概览

| 属性 | 说明 |
|------|------|
| 数据库名 | `teamgram` |
| 字符集 | `utf8mb4`（支持完整 Unicode 及 Emoji） |
| 存储引擎 | `InnoDB`（支持事务、行级锁、外键） |
| 数据规模 | 1,376 行 SQL，覆盖建表、索引、迁移脚本 |
| 表数量 | **38 张** |

### 表分类总览

| 模块 | 表名 | 数量 |
|------|------|------|
| 认证与安全 | `auths`, `auth_keys`, `auth_key_infos`, `auth_seq_updates`, `auth_users`, `encrypted_files` | 6 |
| 用户管理 | `users`, `username`, `user_contacts`, `imported_contacts`, `unregistered_contacts`, `popular_contacts`, `predefined_users`, `phone_books` | 8 |
| 群组与聊天 | `chats`, `chat_invites`, `chat_invite_participants`, `chat_participants` | 4 |
| 消息与对话 | `messages`, `dialogs`, `dialog_filters`, `hash_tags` | 4 |
| 媒体与文件 | `documents`, `photos`, `photo_sizes`, `video_sizes` | 4 |
| Bot 系统 | `bots`, `bot_commands` | 2 |
| 推送与设备 | `devices` | 1 |
| 隐私与设置 | `user_privacies`, `user_global_privacy_settings`, `user_notify_settings`, `user_peer_blocks`, `user_peer_settings`, `user_presences`, `user_settings` | 7 |
| 状态同步 | `user_pts_updates` | 1 |
| 其他 | `auth_seq_updates`（已在认证模块统计） | - |

---

## 二、认证与安全模块

本模块是 Telegram MTProto 协议的基石，负责客户端身份认证、密钥交换、会话管理及端到端加密文件存储。

### 2.1 `auth_keys` —— 认证密钥主表

**职责**：存储每个客户端与服务器之间通过 Diffie-Hellman 交换生成的 **256 字节 Auth Key**。这是 MTProto 协议安全通信的核心，所有后续通信都基于此密钥进行 AES-IGE 加密。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_key_id` | bigint(20) | Auth Key 的 64 位哈希标识，用于快速索引（唯一索引） |
| `body` | varchar(512) | Auth Key 原始二进制数据经 Base64 编码后的字符串，注释明确说明"原始数据为 256 字节二进制" |
| `deleted` | tinyint(1) | 软删除标记 |
| `created_at` | timestamp | 创建时间 |

**业务价值**：`auth_key_id` 是 MTProto 握手成功后客户端发送的每个请求包头中的必备字段，服务器通过它快速定位对应的完整 Auth Key 进行解密验证。

### 2.2 `auth_key_infos` —— 认证密钥类型信息

**职责**：记录 Auth Key 的类型及关联关系。Telegram 支持多种密钥类型（永久密钥、临时密钥、媒体临时密钥），以优化性能与安全性。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_key_id` | bigint(20) | 关联 `auth_keys` 表（唯一索引） |
| `auth_key_type` | int(11) | 密钥类型枚举（如永久、临时、绑定临时） |
| `perm_auth_key_id` | bigint(20) | 绑定的永久密钥 ID（默认 0） |
| `temp_auth_key_id` | bigint(20) | 绑定的临时密钥 ID（默认 0） |
| `media_temp_auth_key_id` | bigint(20) | 绑定的媒体临时密钥 ID（默认 0） |

**业务价值**：Telegram 的 PFS（完美前向保密）机制依赖临时密钥。当临时密钥泄露时，历史消息不会受影响。媒体文件传输使用独立的媒体密钥进一步隔离风险。

### 2.3 `auths` —— 授权会话详情

**职责**：记录每次成功认证后的会话元数据，包括设备信息、协议层版本、代理配置等。Telegram 允许一个账号在多台设备同时登录，此表追踪每台设备的环境信息。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_key_id` | bigint(20) | 关联密钥（唯一索引） |
| `layer` | int(11) | MTProto 协议层版本号，决定客户端支持的功能集 |
| `api_id` | int(11) | 应用 ID，标识是哪个第三方客户端（如官方客户端、第三方客户端） |
| `device_model` | varchar(255) | 设备型号（如 iPhone13,2） |
| `system_version` | varchar(255) | 操作系统版本（如 iOS 16.0） |
| `app_version` | varchar(255) | App 版本号 |
| `system_lang_code` | varchar(255) | 系统语言代码 |
| `lang_pack` | varchar(255) | 语言包标识 |
| `lang_code` | varchar(255) | 应用界面语言 |
| `system_code` | varchar(256) | 系统附加代码 |
| `proxy` | varchar(512) | 代理配置（JSON 或字符串），记录用户是否通过 SOCKS5/MTProto 代理连接 |
| `params` | json | 扩展参数，灵活存储额外的设备配置信息 |
| `client_ip` | varchar(32) | 客户端公网 IP |
| `date_active` | bigint(20) | 最后活跃时间戳（Unix 时间） |
| `deleted` | tinyint(1) | 软删除标记 |

**业务价值**：
- **多设备管理**：用户可在"活跃会话"列表中看到所有登录设备，并远程终止某台设备的会话（将 `deleted` 置 1）。
- **协议兼容**：`layer` 字段确保新旧客户端都能与服务器正确交互，服务器根据 layer 值决定返回什么格式的数据。

### 2.4 `auth_users` —— 认证用户绑定

**职责**：建立 `auth_key_id` 与 `user_id` 的绑定关系，同时记录更详细的设备平台信息。与 `auths` 表相比，此表更侧重于**用户维度**的追踪。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_key_id` | bigint(20) | 认证密钥 ID |
| `user_id` | bigint(20) | 绑定的用户 ID |
| `hash` | bigint(20) | 设备哈希值，用于快速比对设备是否已存在 |
| `layer` | int(11) | 协议层版本 |
| `device_model` | varchar(128) | 设备型号 |
| `platform` | varchar(64) | 平台（如 iOS, Android, Desktop, macOS） |
| `system_version` | varchar(64) | 系统版本 |
| `api_id` | int(11) | 应用 ID |
| `app_name` | varchar(64) | 应用名称 |
| `app_version` | varchar(64) | 应用版本 |
| `date_created` | bigint(20) | 首次登录时间 |
| `date_actived` | bigint(20) | 最后活跃时间 |
| `ip` | varchar(64) | IP 地址 |
| `country` | varchar(64) | 国家（通过 GeoIP 解析） |
| `region` | varchar(64) | 地区 |
| `state` | int(11) | 状态（默认 0，迁移脚本新增） |

**业务价值**：
- **安全审计**：当用户账号异常时，可通过此表追溯所有登录过的设备、IP、地理位置。
- **登录通知**：新设备登录时，服务器向其他活跃设备推送"新登录"提醒，依赖此表获取现有设备列表。

### 2.5 `auth_seq_updates` —— 认证维度更新序列

**职责**：为每个认证会话（auth_id）维护独立的更新序列号（seq），确保每个设备都能按顺序接收状态更新。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_id` | bigint(20) | 认证会话 ID |
| `user_id` | bigint(20) | 用户 ID |
| `seq` | int(11) | 序列号，单调递增 |
| `update_type` | int(11) | 更新类型枚举（如消息、已读、删除等） |
| `update_data` | json | 更新内容（序列化后的 TL 对象） |
| `date2` | bigint(20) | 业务时间戳 |

**业务价值**：Telegram 的同步机制要求每个设备维护独立的 `seq`。当设备离线后重新上线，服务器通过比较客户端上报的 `seq` 与表中的最大值，推送遗漏的更新，实现增量同步。

### 2.6 `encrypted_files` —— 端到端加密文件

**职责**：存储通过 Secret Chat（端到端加密聊天）传输的加密文件元数据。这些文件使用客户端生成的密钥加密，服务器仅作为中转存储。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `encrypted_file_id` | bigint(20) | 加密文件唯一标识 |
| `access_hash` | bigint(20) | 访问哈希，防止枚举攻击 |
| `dc_id` | int(11) | 数据中心 ID |
| `file_size` | bigint(20) | 文件大小（迁移后改为 bigint） |
| `key_fingerprint` | int(11) | 加密密钥指纹，用于客户端校验密钥正确性 |
| `md5_checksum` | varchar(32) | 文件 MD5 校验值 |
| `file_path` | varchar(255) | 文件在存储系统中的物理路径 |

**业务价值**：Secret Chat 的文件不存储在普通 `documents` 表中，而是独立存放于此表，确保即使服务端被攻破，攻击者也无法解密文件内容（密钥仅存于客户端）。

---

## 三、用户管理模块

本模块是 IM 系统的核心，管理用户账号、联系人关系、用户名及通讯录导入逻辑。

### 3.1 `users` —— 用户主表

**职责**：存储平台所有注册用户的核心档案信息。此表是整个系统中数据量最大、访问最频繁的表之一。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 用户唯一 ID（自增） |
| `user_type` | int(11) | 用户类型（默认 2，可能区分普通用户、服务账号等） |
| `access_hash` | bigint(20) | 访问哈希，用于 peer 引用时验证 |
| `secret_key_id` | bigint(20) | 与 Secret Chat 相关的密钥 ID |
| `first_name` | varchar(64) | 名 |
| `last_name` | varchar(64) | 姓 |
| `username` | varchar(64) | 用户名（@username），可空 |
| `phone` | varchar(32) | 手机号（唯一索引），注册和登录的核心凭证 |
| `country_code` | varchar(3) | 国家代码（如 CN, US） |
| `verified` | tinyint(1) | 是否官方认证（蓝勾） |
| `support` | tinyint(1) | 是否为支持账号 |
| `scam` | tinyint(1) | 是否标记为诈骗账号 |
| `fake` | tinyint(1) | 是否标记为虚假账号 |
| `about` | varchar(128) | 个人简介 |
| `state` | int(11) | 账号状态（如正常、冻结、删除中） |
| `is_bot` | tinyint(1) | 是否为 Bot 账号 |
| `account_days_ttl` | int(11) | 账号自动删除 TTL（默认 180 天），Telegram 特色功能 |
| `photo_id` | bigint(20) | 头像照片 ID |
| `restricted` | tinyint(1) | 是否被限制 |
| `restriction_reason` | varchar(128) | 限制原因 |
| `archive_and_mute_new_noncontact_peers` | tinyint(1) | 是否自动归档并静音非联系人消息（隐私设置同步字段） |
| `deleted` | tinyint(1) | 软删除标记 |
| `delete_reason` | varchar(128) | 删除原因 |

**迁移脚本新增字段**：
- `premium`（boolean）：是否为 Premium 订阅用户
- `emoji_status_document_id` / `emoji_status_until`：Emoji 状态（如自定义状态表情）
- `stories_max_id`：Stories 功能的最大 ID
- `color` / `color_background_emoji_id` / `profile_color` / `profile_color_background_emoji_id`：个人资料颜色主题

**业务价值**：
- **Phone-Centric 设计**：以手机号为主键索引，支持通过手机号查找好友。
- **软删除机制**：`deleted` + `delete_reason` 实现账号注销流程，保留数据一段时间后可彻底清理。
- **Premium 体系**：通过 `premium` 字段控制功能访问权限，是 Telegram 商业化核心。

### 3.2 `username` —— 用户名注册表

**职责**：独立维护用户名与 peer（用户或频道）的映射关系。Telegram 的用户名具有全局唯一性，且支持回收机制。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `username` | varchar(32) | 用户名（唯一索引，不区分大小写） |
| `peer_type` | int(11) | 持有者类型（0=用户, 2=群组, 3=频道） |
| `peer_id` | bigint(20) | 持有者 ID |
| `deleted` | tinyint(1) | 软删除（用户名被释放后可被他人重新注册） |

**业务价值**：用户名释放机制（`deleted=1`）允许用户修改用户名后，原用户名进入冷却期，之后可被其他用户注册。`peer_type` 设计使未来频道（Channel）也能拥有用户名。

### 3.3 `user_contacts` —— 用户联系人关系

**职责**：存储每个用户的联系人列表。Telegram 的联系人关系是**单向的**：A 把 B 加为联系人，不代表 B 也把 A 加为联系人。

| 字段 | 类型 | 说明 |
|------|------|
| `id` | bigint(20) | 自增主键 |
| `owner_user_id` | bigint(20) | 联系人列表所有者 |
| `contact_user_id` | bigint(20) | 被添加的用户 ID |
| `contact_phone` | varchar(32) | 添加时记录的手机号 |
| `contact_first_name` | varchar(255) | 添加时记录的别名（可覆盖对方真实姓名） |
| `contact_last_name` | varchar(255) | 添加时记录的别名 |
| `mutual` | tinyint(1) | 是否为双向联系人 |
| `is_deleted` | tinyint(1) | 是否已从联系人列表删除 |
| `close_friend` | tinyint(1) | 是否为密友（迁移脚本新增，用于 Stories 可见性） |
| `date2` | bigint(20) | 添加时间戳 |

**业务价值**：
- **联系人覆盖**：`contact_first_name` 允许用户给好友设置本地备注名，不影响对方的公开资料。
- **双向检测**：当 B 也把 A 加为联系人时，系统将双方的 `mutual` 更新为 1，用于显示" mutual contact"标识。

### 3.4 `imported_contacts` —— 已导入联系人

**职责**：记录用户通过"导入通讯录"功能发现的好友关系。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 导入者 |
| `imported_user_id` | bigint(20) | 被发现的好友用户 ID |
| `deleted` | tinyint(1) | 是否已删除此导入记录 |

**业务价值**：避免重复导入。当用户再次导入通讯录时，服务器比对已有记录，只返回新增的匹配结果。

### 3.5 `unregistered_contacts` —— 未注册联系人

**职责**：存储用户通讯录中尚未注册 Telegram 的手机号。当这些手机号后续注册时，系统可自动通知导入者"您的好友已加入 Telegram"。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `phone` | varchar(32) | 手机号 |
| `importer_user_id` | bigint(20) | 导入者 |
| `import_first_name` | varchar(64) | 通讯录中的名字 |
| `import_last_name` | varchar(64) | 通讯录中的姓氏 |
| `imported` | tinyint(1) | 该联系人是否已注册并通知 |

**业务价值**：这是 Telegram 用户增长的核心机制之一。通过追踪未注册联系人，实现"好友加入通知"推送，驱动自然增长。

### 3.6 `popular_contacts` —— 热门联系人

**职责**：统计每个手机号被多少用户导入，用于推荐"可能认识的人"或反垃圾。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `phone` | varchar(32) | 手机号（唯一索引） |
| `importers` | int(11) | 导入人数计数 |
| `deleted` | tinyint(1) | 软删除 |

### 3.7 `predefined_users` —— 预定义用户

**职责**：管理通过管理后台或脚本预设的账号，常用于企业内部部署、测试账号或客服账号的批量创建。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `phone` | varchar(32) | 预设手机号 |
| `first_name` / `last_name` | varchar(255) | 预设姓名 |
| `username` | varchar(255) | 预设用户名 |
| `code` | varchar(32) | 预设验证码（绕过短信验证） |
| `verified` | tinyint(1) | 是否认证 |
| `registered_user_id` | bigint(20) | 实际注册后的用户 ID |

**业务价值**：私有化部署场景下，管理员可预创建账号并分配固定验证码，用户首次登录无需短信网关。

### 3.8 `phone_books` —— 手机通讯录

**职责**：存储用户上传的原始通讯录内容，用于匹配和推荐好友。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 上传者 |
| `auth_key_id` | bigint(20) | 上传设备 |
| `client_id` | bigint(20) | 客户端生成的通讯录条目 ID |
| `phone` | varchar(32) | 手机号 |
| `first_name` / `last_name` | varchar(64) | 姓名 |

---

## 四、群组与聊天模块

Telegram 的群组（Chat）是小规模群聊的基础单元（最多支持 20 万成员），区别于频道（Channel）。

### 4.1 `chats` —— 群组主表

**职责**：存储群组（Group/Chat）的基础信息。注意：此表仅管理**基础群组**（legacy group），超级群组（supergroup）和频道通常以 channel 形式存储在其他系统中。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 群组 ID（自增） |
| `creator_user_id` | bigint(20) | 创建者 |
| `access_hash` | bigint(20) | 访问哈希 |
| `random_id` | bigint(20) | 创建时生成的随机数，防重放 |
| `participant_count` | int(11) | 成员数量 |
| `title` | varchar(255) | 群组标题 |
| `about` | varchar(255) | 群组简介 |
| `photo_id` | bigint(20) | 群组头像 |
| `default_banned_rights` | bigint(20) | 默认禁言权限位掩码（控制谁可以发消息、添加成员等） |
| `migrated_to_id` | bigint(20) | 升级后的超级群组/频道 ID |
| `migrated_to_access_hash` | bigint(20) | 升级后的访问哈希 |
| `deactivated` | tinyint(1) | 是否已停用 |
| `version` | int(11) | 群组版本号（成员变动时递增） |
| `date` | bigint(20) | 创建时间 |

**迁移脚本新增字段**：
- `noforwards`（boolean）：是否禁止转发消息
- `ttl_period`（int）：消息自动销毁周期
- `available_reactions` / `available_reactions_type`：允许的 reaction 类型

**业务价值**：
- **群组升级**：当基础群组成员超过阈值时，可"升级"为超级群组，`migrated_to_id` 记录升级后的新 ID，保持历史消息可追溯。
- **权限系统**：`default_banned_rights` 使用位掩码高效存储多种权限状态。

### 4.2 `chat_invites` —— 群邀请链接

**职责**：管理群组的所有邀请链接，包括永久链接、临时链接、需审批链接。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `chat_id` | bigint(20) | 所属群组 |
| `admin_id` | bigint(20) | 创建链接的管理员 |
| `migrated_to_id` | bigint(20) | 升级后的群组 ID |
| `link` | varchar(64) | 邀请链接标识（如 `t.me/+AbCdEf` 的后缀，唯一索引） |
| `permanent` | tinyint(1) | 是否为永久链接 |
| `revoked` | tinyint(1) | 是否已撤销 |
| `request_needed` | tinyint(1) | 是否需要管理员审批才能加入 |
| `start_date` | bigint(20) | 生效时间 |
| `expire_date` | bigint(20) | 过期时间 |
| `usage_limit` | int(11) | 最大使用次数 |
| `usage2` | int(11) | 已使用次数 |
| `requested` | int(11) | 已申请加入的人数 |
| `title` | varchar(64) | 链接自定义标题（便于管理员管理多个链接） |
| `date2` | bigint(20) | 创建时间 |
| `state` | int(11) | 状态 |

**业务价值**：
- **链接管理**：管理员可创建多个邀请链接，分别投放到不同渠道（如微博、微信），通过 `title` 和 `usage2` 统计各渠道的拉新效果。
- **安全控制**：`revoked` 和 `expire_date` 支持链接的时效管理，防止旧链接长期有效导致垃圾用户涌入。

### 4.3 `chat_invite_participants` —— 邀请参与记录

**职责**：记录通过邀请链接加入群组的用户。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `link` | varchar(32) | 使用的邀请链接 |
| `user_id` | bigint(20) | 加入的用户 |
| `date2` | bigint(20) | 加入时间 |
| `chat_id` | bigint(20) | 群组 ID（迁移脚本新增） |
| `deleted` | tinyint(1) | 软删除 |

**业务价值**：当发现某个链接被恶意传播时，管理员可批量踢出通过该链接加入的所有用户（`chat_id + link` 索引支持快速查询）。

### 4.4 `chat_participants` —— 群成员关系

**职责**：存储群组与用户的成员关系，是群组成员管理的核心表。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `chat_id` | bigint(20) | 群组 ID |
| `user_id` | bigint(20) | 成员用户 ID（联合唯一索引） |
| `participant_type` | int(11) | 成员类型（如普通成员、管理员、创建者） |
| `link` | varchar(64) | 加入时使用的邀请链接 |
| `usage2` | int(11) | 辅助计数 |
| `admin_rights` | int(11) | 管理员权限位掩码 |
| `inviter_user_id` | bigint(20) | 邀请人 |
| `invited_at` | bigint(20) | 被邀请时间 |
| `kicked_at` | bigint(20) | 被踢出时间 |
| `left_at` | bigint(20) | 主动退群时间 |
| `state` | int(11) | 成员状态 |
| `date2` | bigint(20) | 记录时间 |
| `groupcall_default_join_as_peer_type` / `groupcall_default_join_as_peer_id` | 迁移新增 | 群语音默认加入身份 |
| `is_bot` | boolean | 是否为 Bot |

**业务价值**：
- **成员生命周期管理**：`invited_at`, `kicked_at`, `left_at` 完整追踪成员的加入、踢出、退群时间线。
- **权限粒度**：`admin_rights` 使用位掩码，可精细控制（修改群信息、删除消息、禁言用户、邀请成员等）。

---

## 五、消息与对话模块

这是 IM 系统最核心的业务模块，处理消息的存储、对话列表管理及搜索索引。

### 5.1 `messages` —— 消息主表

**职责**：存储所有用户发送的消息。Telegram 采用**消息多副本存储**策略：同一条消息在发送者和每个接收者的消息箱中各存一条记录，通过 `dialog_id1/dialog_id2` 区分归属。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 消息所属用户（消息箱所有者） |
| `user_message_box_id` | int(11) | 该用户视角下的消息本地 ID（每个用户独立递增） |
| `dialog_id1` | bigint(20) | 对话标识分量 1（通常为用户 ID） |
| `dialog_id2` | bigint(20) | 对话标识分量 2（通常为对方 ID 或群 ID） |
| `dialog_message_id` | bigint(20) | 对话级别的消息 ID（用于排序和去重） |
| `sender_user_id` | bigint(20) | 实际发送者 |
| `peer_type` | int(11) | 对话类型（1=用户, 2=群组, 3=频道） |
| `peer_id` | bigint(20) | 对话对象 ID |
| `random_id` | bigint(20) | 客户端生成的随机数，用于去重和幂等性校验 |
| `message_filter_type` | int(11) | 消息过滤类型（如文本、图片、视频、文件） |
| `message_data` | json | 消息完整内容的 JSON 序列化（包含媒体、按钮、转发信息等） |
| `message` | varchar(6000) | 纯文本内容（便于全文搜索和摘要显示） |
| `mentioned` | tinyint(1) | 是否 @ 提及当前用户 |
| `media_unread` | tinyint(1) | 媒体是否未读 |
| `pinned` | tinyint(1) | 是否置顶消息 |
| `has_reaction` | boolean | 是否有 reaction（迁移新增） |
| `reaction` | varchar(16) | reaction 内容（迁移新增） |
| `reaction_date` | bigint(20) | reaction 时间（迁移新增） |
| `reaction_unread` | boolean | reaction 是否未读（迁移新增） |
| `ttl_period` | int(11) | 消息自毁周期（迁移新增） |
| `date2` | bigint(20) | 发送时间戳 |
| `deleted` | tinyint(1) | 软删除 |

**业务价值**：
- **消息箱模型**：`user_id + user_message_box_id` 唯一索引确保每个用户的消息 ID 独立递增，这是 Telegram 客户端显示消息列表的基础。
- **去重机制**：`random_id` 防止用户重复点击发送导致消息重复。
- **多维度索引**：`user_id + dialog_id1 + dialog_id2` 支持按对话快速拉取历史消息。
- **Reaction 支持**：迁移脚本新增的 reaction 字段使系统支持消息表情回应功能。

### 5.2 `dialogs` —— 对话列表

**职责**：存储每个用户的对话列表（即聊天列表中的每个会话卡片）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `peer_type` | int(11) | 对话类型 |
| `peer_id` | bigint(20) | 对话对象 ID |
| `peer_dialog_id` | bigint(20) | 对话唯一标识（联合唯一索引） |
| `pinned` | bigint(20) | 置顶排序值（0=未置顶，越大越靠前） |
| `top_message` | int(11) | 最新消息的 `user_message_box_id` |
| `pinned_msg_id` | int(11) | 当前置顶消息 ID |
| `read_inbox_max_id` | int(11) | 收件箱已读最大 ID |
| `read_outbox_max_id` | int(11) | 发件箱已读最大 ID |
| `unread_count` | int(11) | 未读消息数 |
| `unread_mentions_count` | int(11) | 未读 @ 提及数 |
| `unread_mark` | tinyint(1) | 是否手动标记为未读 |
| `draft_type` | int(11) | 草稿类型 |
| `draft_message_data` | json | 草稿内容 |
| `folder_id` | int(11) | 所属文件夹（0=默认, 1=归档） |
| `folder_pinned` | bigint(20) | 文件夹内置顶排序 |
| `has_scheduled` | tinyint(1) | 是否有定时消息 |
| `theme_emoticon` | varchar(64) | 主题表情（迁移新增） |
| `ttl_period` | int(11) | 消息自毁周期（迁移新增） |
| `date2` | bigint(20) | 最后更新时间 |
| `deleted` | tinyint(1) | 软删除 |

**业务价值**：
- **已读回执**：`read_inbox_max_id` 和 `read_outbox_max_id` 实现双向已读状态追踪。当对方已读时，客户端能看到消息状态从"一个勾"变为"两个勾"。
- **多文件夹**：`folder_id` 支持"所有聊天"与"归档"分类，未来可扩展自定义文件夹。
- **草稿同步**：`draft_message_data` 以 JSON 存储，实现多端草稿实时同步。

### 5.3 `dialog_filters` —— 对话筛选器（文件夹）

**职责**：存储用户自定义的对话筛选规则，对应客户端的"聊天文件夹"功能。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `dialog_filter_id` | int(11) | 筛选器 ID（联合唯一索引） |
| `dialog_filter` | json | 筛选规则（包含包含/排除的群组、联系人、标签等） |
| `order_value` | bigint(20) | 排序值 |
| `deleted` | tinyint(1) | 软删除 |

**业务价值**：用户可创建如"工作"、"家人"、"未读"等自定义文件夹，`dialog_filter` JSON 中定义包含哪些 peer，服务器据此推送增量更新。

### 5.4 `hash_tags` —— 话题标签索引

**职责**：为消息中的 `#话题` 建立反向索引，支持按标签搜索消息。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `peer_type` | int(11) | 对话类型 |
| `peer_id` | bigint(20) | 对话对象 ID |
| `hash_tag` | varchar(128) | 标签内容（如 "teamgram"） |
| `hash_tag_message_id` | int(11) | 关联的消息 ID |
| `deleted` | tinyint(1) | 软删除 |

**业务价值**：当用户在某个对话中点击 `#话题` 时，服务器通过 `user_id + peer_type + peer_id + hash_tag` 索引快速返回该话题下的所有消息。

---

## 六、媒体与文件模块

Telegram 的媒体存储采用"元数据+文件分离"架构：数据库存储文件的元数据（尺寸、路径、属性），实际文件内容存储在 MinIO 或分布式文件系统中。

### 6.1 `documents` —— 文件/文档主表

**职责**：存储所有上传的文件、文档、音频、视频等通用文件的元数据。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `document_id` | bigint(20) | 文件唯一标识（唯一索引） |
| `access_hash` | bigint(20) | 访问哈希 |
| `dc_id` | int(11) | 数据中心 ID |
| `file_path` | varchar(255) | 物理存储路径 |
| `file_size` | bigint(20) | 文件大小（迁移后改为 bigint，支持大文件） |
| `uploaded_file_name` | varchar(255) | 原始文件名 |
| `ext` | varchar(32) | 扩展名 |
| `mime_type` | varchar(128) | MIME 类型（如 video/mp4） |
| `thumb_id` | bigint(20) | 缩略图照片 ID |
| `video_thumb_id` | bigint(20) | 视频缩略图 ID |
| `version` | int(11) | 文件版本 |
| `attributes` | json | 文件属性（如音频时长、视频分辨率、文件名等 TL 对象序列化） |
| `date2` | bigint(20) | 上传时间 |

**业务价值**：
- **多端下载**：客户端通过 `document_id + access_hash` 向对应 `dc_id` 的数据中心请求下载文件。
- **大文件支持**：`file_size` 改为 bigint 后可支持超过 4GB 的文件（如高清视频）。

### 6.2 `photos` —— 照片主表

**职责**：存储照片（图片）的元数据。一张照片可包含多个尺寸版本（缩略图、原图等）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `photo_id` | bigint(20) | 照片唯一标识（唯一索引） |
| `access_hash` | bigint(20) | 访问哈希 |
| `has_stickers` | tinyint(1) | 是否包含贴纸标注 |
| `dc_id` | int(11) | 数据中心 ID |
| `date2` | bigint(20) | 上传时间 |
| `has_video` | tinyint(1) | 是否包含视频版本（如动态照片/Live Photo） |
| `size_id` | bigint(20) | 静态尺寸集合 ID |
| `video_size_id` | bigint(20) | 视频尺寸集合 ID |
| `input_file_name` | varchar(128) | 输入文件名 |
| `ext` | varchar(32) | 扩展名 |

**业务价值**：`size_id` 和 `video_size_id` 分别关联 `photo_sizes` 和 `video_sizes` 表，实现"一张照片、多种尺寸、自动适配网络"的功能。

### 6.3 `photo_sizes` —— 照片尺寸详情

**职责**：存储照片各尺寸版本的物理信息。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `photo_size_id` | bigint(20) | 所属照片 |
| `size_type` | char(1) | 尺寸类型标识（如 'a', 'b', 'c', 'x', 'y' 等，按字母顺序表示从小到大） |
| `width` / `height` | int(11) | 宽高像素 |
| `file_size` | int(11) | 文件大小 |
| `file_path` | varchar(255) | 存储路径 |
| `cached_type` / `cached_bytes` | 迁移新增 | 缓存类型及缩略图字节 |

**迁移删除字段**：早期版本包含 `volume_id`, `local_id`, `secret`, `has_stripped`, `stripped_bytes`，后因存储架构重构被移除。

**业务价值**：客户端根据网络状况（WiFi/4G/弱网）自动选择合适 `size_type` 的图片下载，节省流量并提升加载速度。

### 6.4 `video_sizes` —— 视频缩略图/预览尺寸

**职责**：存储视频文件的各种预览尺寸（如动画缩略图、进度条预览图）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `video_size_id` | bigint(20) | 所属视频/文档 |
| `size_type` | char(1) | 尺寸类型 |
| `width` / `height` | int(11) | 宽高 |
| `file_size` | int(11) | 文件大小 |
| `video_start_ts` | double | 视频起始时间戳（用于视频中的某一帧截图） |
| `file_path` | varchar(255) | 存储路径 |

**业务价值**：`video_start_ts` 支持从视频中提取特定时间点的帧作为缩略图，常用于视频消息和圆形视频消息的预览。

---

## 七、Bot 系统模块

Telegram Bot 是平台生态的重要组成部分，支持命令交互、内联查询、支付等功能。

### 7.1 `bots` —— Bot 主表

**职责**：存储 Bot 账号的专属配置和行为属性。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `bot_id` | bigint(20) | Bot 用户 ID（唯一索引） |
| `bot_type` | int(11) | Bot 类型 |
| `creator_user_id` | bigint(20) | 创建者 |
| `token` | varchar(128) | Bot API Token（唯一索引，格式如 `123456:ABC-DEF...`） |
| `description` | varchar(10240) | Bot 简介（支持长文本） |
| `bot_chat_history` | tinyint(1) | 是否可访问群历史消息 |
| `bot_nochats` | tinyint(1) | 是否禁止加入群组（默认 1=禁止，需手动开启） |
| `verified` | tinyint(1) | 是否认证 Bot |
| `bot_inline_geo` | tinyint(1) | 内联模式是否可请求地理位置 |
| `bot_info_version` | int(11) | Bot 信息版本号（修改描述时递增，客户端据此刷新缓存） |
| `bot_inline_placeholder` | varchar(128) | 内联查询输入框占位提示文字 |

**业务价值**：
- **安全隔离**：`bot_nochats` 默认禁止 Bot 加入群组，防止恶意 Bot 泛滥，开发者需显式开启。
- **Token 管理**：`token` 作为 Bot 调用 HTTP API 的凭证，采用唯一索引防止重复。

### 7.2 `bot_commands` —— Bot 命令定义

**职责**：存储每个 Bot 注册的斜杠命令列表（如 `/start`, `/help`）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `bot_id` | bigint(20) | 所属 Bot |
| `command` | varchar(128) | 命令名 |
| `description` | varchar(10240) | 命令描述 |

**业务价值**：当用户在聊天中输入 `/` 时，客户端向服务器请求该 Bot 的命令列表，服务器从此表查询并返回，实现命令自动补全。

---

## 八、推送与设备模块

### 8.1 `devices` —— 推送设备令牌

**职责**：管理移动设备的推送令牌（APNs for iOS, FCM for Android），用于离线消息推送。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `auth_key_id` | bigint(20) | 认证密钥 |
| `user_id` | bigint(20) | 用户 ID |
| `token_type` | int(11) | 推送服务类型（1=APNs, 2=FCM, 3=WebPush 等） |
| `token` | varchar(512) | 设备推送令牌 |
| `no_muted` | tinyint(1) | 是否不推送静音消息（仅推送非静音消息） |
| `locked_period` | int(11) | 锁定周期 |
| `app_sandbox` | tinyint(1) | 是否为沙盒环境（iOS 开发测试用） |
| `secret` | varchar(1024) | WebPush 密钥等额外安全信息 |
| `other_uids` | varchar(1024) | 该设备上登录的其他用户 ID 列表 |
| `state` | tinyint(1) | 设备状态 |

**业务价值**：
- **多端推送**：一个用户可能有多个设备（iPhone + iPad + Android），`other_uids` 支持多账号切换场景下的推送管理。
- **静音策略**：`no_muted` 确保用户开启"静音"的对话不会打扰用户，但重要对话仍能收到推送。

---

## 九、隐私与设置模块

Telegram 以隐私保护著称，本模块支撑了丰富的隐私控制功能。

### 9.1 `user_privacies` —— 用户隐私规则

**职责**：存储用户对各项隐私设置的访问控制规则（谁可以看到我的手机号、最后上线时间、头像等）。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `key_type` | int(11) | 隐私项类型（如 1=手机号, 2=最后上线, 3=头像, 4=转发, 5=通话等） |
| `rules` | json | 规则列表（如 {"allow_all": false, "allow_contacts": true, "allow_users": [123, 456]}） |

**业务价值**：Telegram 的隐私规则非常灵活，支持"所有人"、"仅联系人"、"无人"、"仅特定用户"四种模式，`rules` JSON 字段灵活存储这些组合规则。

### 9.2 `user_global_privacy_settings` —— 全局隐私设置

**职责**：存储全局级隐私开关。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID（唯一索引） |
| `archive_and_mute_new_noncontact_peers` | tinyint(1) | 是否自动归档并静音陌生人消息 |

**业务价值**：此设置帮助用户减少骚扰。开启后，非联系人发来的第一条消息会自动进入"归档"文件夹且不会推送通知。

### 9.3 `user_notify_settings` —— 通知设置

**职责**：存储用户对每个对话（或全局）的通知偏好。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `peer_type` / `peer_id` | int/bigint | 对话对象（若为全局设置，peer 可能为特殊值） |
| `show_previews` | int(11) | 是否显示消息预览（-1=默认, 0=不显示, 1=显示） |
| `silent` | int(11) | 是否静音（-1=默认, 0=正常, 1=静音） |
| `mute_until` | int(11) | 静音截止时间戳（-1=永久静音, 0=不静音） |
| `sound` | varchar(255) | 提示音文件名（默认 'default'） |

**业务价值**：支持"8 小时静音"、"永久静音"、"自定义铃声"等精细化通知控制，提升用户体验。

### 9.4 `user_peer_blocks` —— 用户黑名单

**职责**：记录用户屏蔽的 Peer（用户或群组）。被屏蔽后，对方无法发送消息或看到用户的在线状态。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 屏蔽者 |
| `peer_type` / `peer_id` | int/bigint | 被屏蔽对象 |
| `date` | bigint(20) | 屏蔽时间 |
| `deleted` | tinyint(1) | 是否已解除屏蔽 |

### 9.5 `user_peer_settings` —— Peer 级用户设置

**职责**：存储用户对某个特定对话对象的个性化设置。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `peer_type` / `peer_id` | int/bigint | 对话对象 |
| `hide` | tinyint(1) | 是否隐藏对话 |
| `report_spam` | tinyint(1) | 是否已举报垃圾消息 |
| `add_contact` | tinyint(1) | 是否已添加联系人 |
| `block_contact` | tinyint(1) | 是否已屏蔽 |
| `share_contact` | tinyint(1) | 是否共享联系人 |
| `need_contacts_exception` | tinyint(1) | 是否需要联系人例外 |
| `report_geo` | tinyint(1) | 是否举报地理位置 |
| `autoarchived` | tinyint(1) | 是否自动归档 |
| `invite_members` | tinyint(1) | 是否邀请成员 |
| `geo_distance` | int(11) | 地理位置距离（用于附近的人功能） |

**业务价值**：当用户举报某个对话为垃圾消息时，`report_spam` 标记触发服务器的反垃圾审查流程。

### 9.6 `user_presences` —— 用户在线状态

**职责**：记录用户的最后在线时间，是"Online/Recently/Long ago"显示的数据源。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID（唯一索引） |
| `last_seen_at` | bigint(20) | 最后活跃时间戳 |
| `expires` | int(10) | 在线状态过期时间 |

**业务价值**：
- **隐私兼容**：根据 `user_privacies` 中的"最后上线时间"设置，服务器决定是否向查询者返回精确时间或模糊状态（如"最近三天内"）。
- **实时同步**：用户每次操作（发送消息、打开 App）都会更新此表，并通过 `sync` 服务推送给订阅者。

### 9.7 `user_settings` —— 用户通用设置

**职责**：以 Key-Value 形式存储用户的各种杂项设置。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `key2` | varchar(64) | 设置项键名 |
| `value` | varchar(512) | 设置值 |

**业务价值**：Key-Value 设计提供了极大的扩展性，新增设置项无需改表结构。例如可存储主题、字体大小、自动下载策略等。

---

## 十、状态同步模块

### 10.1 `user_pts_updates` —— 用户 PTS 更新队列

**职责**：为每个用户维护一个**PTS（Point in Time Sequence）**更新序列。这是 Telegram 状态同步的核心机制，比 `auth_seq_updates` 更侧重于**用户维度**的全局状态变更。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint(20) | 自增主键 |
| `user_id` | bigint(20) | 用户 ID |
| `pts` | int(11) | 序列号（单调递增） |
| `pts_count` | int(11) | 该更新消耗的 PTS 点数（通常为 1，批量操作可能大于 1） |
| `update_type` | tinyint(4) | 更新类型（1=消息, 2=已读, 3=删除, 4=设置变更等） |
| `update_data` | json | 更新内容 |
| `date2` | bigint(20) | 时间戳 |

**业务价值**：
- **增量同步**：客户端本地缓存当前 `pts`，断线重连后向服务器请求 `pts + 1` 之后的所有更新，实现轻量同步。
- **顺序保证**：`user_id + pts` 索引确保更新严格按序处理，防止乱序导致的状态不一致（如先收到删除、后收到消息）。
- **多端一致**：同一用户的所有设备共享同一套 `pts`，确保手机、平板、电脑上的对话状态完全一致。

---

## 十一、索引设计分析

数据库的索引设计直接决定了 IM 系统的查询性能。以下是关键索引及其业务意义：

| 表 | 索引 | 类型 | 业务意义 |
|----|------|------|----------|
| `auth_keys` | `auth_key_id` | 唯一 | MTProto 请求解密时的 O(1) 查询 |
| `auth_users` | `auth_key_id + user_id` | 唯一 | 快速定位某设备绑定的用户 |
| `bots` | `token` | 唯一 | Bot HTTP API 调用时的凭证校验 |
| `chat_invites` | `link` | 唯一 | 邀请链接解析（用户点击链接时） |
| `chat_participants` | `chat_id + user_id` | 唯一 | 判断用户是否在群内、获取成员身份 |
| `dialogs` | `user_id + peer_type + peer_id` | 唯一 | 打开某个对话时的快速定位 |
| `dialogs` | `user_id + peer_dialog_id` | 唯一 | 对话列表排序与去重 |
| `hash_tags` | `user_id + peer_type + peer_id + hash_tag` | 普通 | 按话题搜索消息 |
| `messages` | `user_id + user_message_box_id` | 唯一 | 客户端按消息 ID 拉取单条消息 |
| `messages` | `user_id + dialog_id1 + dialog_id2` | 普通 | 按对话拉取历史消息列表 |
| `phone_books` | `auth_key_id + client_id` | 唯一 | 去重上传的通讯录条目 |
| `popular_contacts` | `phone` | 唯一 | 统计手机号被导入次数 |
| `users` | `phone` | 唯一 | 手机号注册与登录 |
| `username` | `username` | 唯一 | 用户名查找与 @ 提及解析 |
| `user_contacts` | `owner_user_id + contact_user_id` | 唯一 | 判断是否已为联系人 |
| `user_notify_settings` | `user_id + peer_type + peer_id` | 唯一 | 获取某对话的通知设置 |
| `user_peer_blocks` | `user_id + peer_type + peer_id` | 唯一 | 判断某用户是否被屏蔽 |
| `user_privacies` | `user_id + key_type` | 唯一 | 获取某项隐私设置 |
| `user_profile_photos` | `user_id + photo_id` | 唯一 | 获取用户头像列表 |
| `user_pts_updates` | `user_id + pts` | 普通 | 按 PTS 范围拉取更新 |

---

## 十二、设计特点与关键技术洞察

### 12.1 软删除（Soft Delete）

绝大多数表都包含 `deleted` 字段。IM 系统的数据删除通常是逻辑删除：
- **消息删除**：用户删除消息时，`messages.deleted=1`，但服务器保留一段时间以便多端同步删除指令。
- **账号注销**：`users.deleted=1` 后，账号进入冻结期，之后可彻底清除。
- **对话清理**：`dialogs.deleted=1` 将对话从列表隐藏，但历史消息仍保留在 `messages` 表中。

### 12.2 JSON 字段的灵活扩展

`message_data`, `attributes`, `rules`, `dialog_filter`, `params`, `draft_message_data` 等字段采用 JSON 类型：
- **优点**：适应 Telegram 协议频繁迭代的特性，新增消息类型或设置项无需 ALTER 表。
- **权衡**：JSON 字段无法直接建立高效索引，因此同时抽取关键字段（如 `message` 文本、`peer_type`）作为独立列用于查询。

### 12.3 大整数时间戳（`date2`, `last_seen_at`）

所有业务时间使用 `bigint(20)` 存储 Unix 时间戳（秒级或毫秒级），而非 MySQL 的 `datetime`：
- **时区无关**：避免服务器、数据库、客户端时区不一致导致的问题。
- **语言无关**：Go、移动端、Web 端都能无歧义解析。

### 12.4 位掩码权限（`admin_rights`, `default_banned_rights`）

群组权限使用整数位掩码存储，每一位代表一种权限（如第 1 位=修改群信息，第 2 位=删除消息）。这种设计：
- 将多种布尔权限压缩为单个整数，节省存储。
- 权限检查时通过位运算（`rights & PERMISSION != 0`）实现，效率高。

### 12.5 消息多副本（Message Fan-out）

`messages` 表采用"写扩散"模型：一条群消息发给 1000 人，就在 `messages` 表中插入 1000 条记录（`user_id` 不同，但 `message_data` 相同）。
- **优点**：每个用户的消息箱完全独立，查询历史消息时无需 JOIN，性能极高。
- **代价**：存储成本增加，但现代存储成本远低于计算成本，这是 IM 系统的经典取舍。

### 12.6 用户名与 Phone 的双主键设计

- 用户注册时以 `phone` 为唯一标识（`users.phone` 唯一索引）。
- 注册后可设置 `username`（`username.username` 唯一索引），之后可通过任一方式被查找。
- `users.username` 与 `username` 独立表的双轨设计，支持用户名的释放与再分配，而手机号一旦绑定通常不变。

---

## 十三、总结

Teamgram 的数据库设计充分体现了 Telegram 架构的核心思想：

1. **安全性优先**：Auth Key、端到端加密、Secret Chat 独立存储，多层密钥体系。
2. **隐私可控**：丰富的隐私规则表、黑名单、精细化通知设置。
3. **性能导向**：消息多副本、PTS 同步、位掩码权限、丰富的联合索引。
4. **扩展灵活**：JSON 字段应对协议迭代、Key-Value 设置表、迁移脚本机制。
5. **生态完整**：Bot 系统、邀请链接、通讯录导入、Stories（迁移脚本可见）等功能均有对应表支撑。

这 38 张表构成了一个功能完备、可支撑千万级用户的即时通讯后端数据基石。
