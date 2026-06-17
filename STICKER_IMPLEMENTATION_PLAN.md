# Sticker（贴纸）功能实现方案

> 基于 Telegram 官方 API 文档 `https://core.telegram.org/api/stickers` 及 Teamgram 社区版现有代码结构分析。

---

## 一、现状分析

### 1.1 协议层 —— 已完备

`mtproto` 依赖包（`github.com/teamgram/proto v0.223.2`）中已经包含了 Sticker 相关的全部类型定义：
- `StickerSet`、`StickerPack`、`InputStickerSet`
- `DocumentAttributeSticker`、`MaskCoords`
- `Messages_AllStickers`、`Messages_StickerSet`、`Messages_RecentStickers` 等返回类型

### 1.2 BFF 假响应层 —— 已占位

`app/bff/bff/client/fake_rpc_result.go` 中已为 **12 个** Sticker 查询接口提供空响应，避免客户端崩溃：

| 方法 | 当前行为 |
|------|---------|
| `messages.getAllStickers` | 返回空 `Sets` 列表 |
| `messages.getArchivedStickers` | 返回空 `Sets` 列表 |
| `messages.getFavedStickers` | 返回空 `Packs` + `Stickers` |
| `messages.getMaskStickers` | 返回空 `Sets` 列表 |
| `messages.getEmojiStickers` | 返回空 `Sets` 列表 |
| `messages.getOldFeaturedStickers` | 返回空 `Sets` 列表 |
| `messages.getRecentStickers` | 返回空 `Packs` + `Stickers` |
| `messages.getStickers` | 返回空 `Stickers` 列表 |
| `messages.getFeaturedStickers` | 返回空 `Sets` 列表 |
| `messages.getFeaturedEmojiStickers` | 返回空 `Sets` 列表 |
| `messages.getStickerSet` | **被注释掉**，会走 `ErrMethodNotImpl` |
| `messages.installStickerSet` 等写操作 | **未注册**，会返回 `not found method` |

**机制说明**：`bff_proxy_client.go` 中，当找不到对应微服务时，会自动调用 `TryReturnFakeRpcResult()` 返回空数据。这是客户端目前能正常打开但不显示贴纸的原因。

### 1.3 企业版锁定点

以下核心接口被硬编码拦截：

```go
// app/service/media/internal/core/media.uploadStickerFile_handler.go
func (c *MediaCore) MediaUploadStickerFile(...) (*mtproto.Document, error) {
    c.Logger.Errorf("media.uploadStickerFile blocked, License key from https://teamgram.net required...")
    return nil, mtproto.ErrEnterpriseIsBlocked
}

// app/bff/chats/internal/core/channels.setEmojiStickers_handler.go
func (c *ChatsCore) ChannelsSetEmojiStickers(...) (*mtproto.Bool, error) {
    c.Logger.Errorf("channels.setEmojiStickers blocked, License key from https://teamgram.net required...")
    return nil, mtproto.ErrEnterpriseIsBlocked
}
```

### 1.4 数据库层 —— 完全空白

当前数据库中**无任何 Sticker 相关表**。所有贴纸数据需要从零设计。

---

## 二、数据库设计

### 2.1 核心表结构

```sql
-- 贴纸集表（对应 StickerSet）
CREATE TABLE IF NOT EXISTS `sticker_sets` (
  `id` BIGINT NOT NULL,
  `access_hash` BIGINT NOT NULL DEFAULT 0,
  `title` VARCHAR(255) NOT NULL DEFAULT '',
  `short_name` VARCHAR(255) NOT NULL DEFAULT '',
  `set_type` INT NOT NULL DEFAULT 0 COMMENT '0=普通 1=mask 2=emoji',
  `count` INT NOT NULL DEFAULT 0,
  `hash` INT NOT NULL DEFAULT 0,
  `masks` BOOLEAN NOT NULL DEFAULT FALSE,
  `emojis` BOOLEAN NOT NULL DEFAULT FALSE,
  `official` BOOLEAN NOT NULL DEFAULT FALSE,
  `animated` BOOLEAN NOT NULL DEFAULT FALSE,
  `videos` BOOLEAN NOT NULL DEFAULT FALSE,
  `archived` BOOLEAN NOT NULL DEFAULT FALSE,
  `thumb_document_id` BIGINT NOT NULL DEFAULT 0,
  `creator_user_id` BIGINT NOT NULL DEFAULT 0,
  `created_at` BIGINT NOT NULL DEFAULT 0,
  `updated_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `short_name` (`short_name`),
  KEY `creator_user_id` (`creator_user_id`),
  KEY `set_type` (`set_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 贴纸表（对应 Document + DocumentAttributeSticker）
CREATE TABLE IF NOT EXISTS `stickers` (
  `id` BIGINT NOT NULL,
  `sticker_set_id` BIGINT NOT NULL DEFAULT 0,
  `document_id` BIGINT NOT NULL DEFAULT 0,
  `emoji` VARCHAR(64) NOT NULL DEFAULT '',
  `alt` VARCHAR(255) NOT NULL DEFAULT '',
  `mask_coords` VARCHAR(255) NOT NULL DEFAULT '',
  `sort_order` INT NOT NULL DEFAULT 0,
  `created_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `sticker_set_id` (`sticker_set_id`),
  KEY `document_id` (`document_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户已安装贴纸集
CREATE TABLE IF NOT EXISTS `user_sticker_sets` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT NOT NULL DEFAULT 0,
  `sticker_set_id` BIGINT NOT NULL DEFAULT 0,
  `installed` BOOLEAN NOT NULL DEFAULT TRUE,
  `archived` BOOLEAN NOT NULL DEFAULT FALSE,
  `order_index` INT NOT NULL DEFAULT 0,
  `installed_date` BIGINT NOT NULL DEFAULT 0,
  `created_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `user_set` (`user_id`, `sticker_set_id`),
  KEY `user_id` (`user_id`, `archived`, `order_index`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户最近使用贴纸
CREATE TABLE IF NOT EXISTS `user_recent_stickers` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT NOT NULL DEFAULT 0,
  `document_id` BIGINT NOT NULL DEFAULT 0,
  `is_attached` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '是否是附加贴纸',
  `created_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `user_doc` (`user_id`, `document_id`, `is_attached`),
  KEY `user_id` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 用户收藏贴纸
CREATE TABLE IF NOT EXISTS `user_faved_stickers` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT NOT NULL DEFAULT 0,
  `document_id` BIGINT NOT NULL DEFAULT 0,
  `created_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  UNIQUE KEY `user_doc` (`user_id`, `document_id`),
  KEY `user_id` (`user_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 推荐/精选贴纸集
CREATE TABLE IF NOT EXISTS `featured_sticker_sets` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `sticker_set_id` BIGINT NOT NULL DEFAULT 0,
  `unread` BOOLEAN NOT NULL DEFAULT TRUE,
  `created_at` BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (`id`),
  KEY `sticker_set_id` (`sticker_set_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 三、微服务架构设计

### 3.1 服务分工

```
┌─────────────────────────────────────────────────────────────┐
│                        Telegram Client                       │
└───────────────────────────┬─────────────────────────────────┘
                            │ MTProto RPC
┌───────────────────────────▼─────────────────────────────────┐
│              BFF - messages 服务（现有）                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐   │
│  │ getStickers │ │installStickerSet│ │ createStickerSet │   │
│  └──────┬──────┘ └──────┬──────┘ └──────────┬──────────┘   │
│         │               │                     │              │
│  ┌──────▼──────┐ ┌──────▼──────┐ ┌───────────▼──────────┐  │
│  │   DAO查询    │ │  DAO写操作   │ │ 调用 media.upload    │  │
│  │  sticker表   │ │  user_sticker│ │  + 持久化 sticker元数据│  │
│  └─────────────┘ └─────────────┘ └──────────────────────┘  │
└───────────────────────────┬─────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼───────┐  ┌────────▼────────┐  ┌──────▼──────┐
│  MySQL        │  │  Media Service  │  │  MinIO      │
│ (sticker元数据)│  │ (uploadSticker) │  │ (文件存储)   │
└───────────────┘  └─────────────────┘  └─────────────┘
```

**设计理由**：
- Sticker 查询/管理类接口属于 `messages.*` 命名空间，直接扩展现有 `app/bff/messages` 服务最自然
- 文件上传复用现有 `Media Service`，只需解锁 `uploadStickerFile`
- 不需要新增独立微服务（ sticker 逻辑相对集中，不需要单独进程）

---

## 四、分阶段实现路线图

### Phase 1：数据库 + 基础查询（建议优先，1-2 周）

**目标**：让客户端能显示已安装的贴纸集和贴纸内容。

**需要实现的接口**：

| 接口 | 难度 | 说明 |
|------|------|------|
| `messages.getStickerSet` | 低 | 根据 `InputStickerSet` 查询贴纸集详情 + 贴纸列表 |
| `messages.getAllStickers` | 低 | 查询用户 `user_sticker_sets` 中 `installed=true` 且 `archived=false` 的记录 |
| `messages.getRecentStickers` | 低 | 查询 `user_recent_stickers` |
| `messages.getStickers` | 低 | 按 emoji/关键词匹配 `stickers` 表 |

**需要修改的文件**：
1. `app/bff/messages/internal/core/` 下新建 4 个 handler
2. `app/bff/messages/internal/dal/` 下新增 DAO/DO/Table（可用 dalgen 自动生成）
3. `app/bff/messages/internal/dao/` 下新增 sticker DAO 封装
4. `app/bff/messages/internal/server/grpc/service/messages_service_impl.go` 注册 handler
5. `app/bff/bff/client/fake_rpc_result.go` 移除对应 case（让请求走真实 handler）

### Phase 2：安装/卸载/收藏（1-2 周）

**目标**：让用户能安装、卸载、收藏贴纸。

| 接口 | 难度 | 说明 |
|------|------|------|
| `messages.installStickerSet` | 低 | `user_sticker_sets` 插入记录 |
| `messages.uninstallStickerSet` | 低 | `user_sticker_sets` 删除记录 |
| `messages.reorderStickerSets` | 低 | 更新 `order_index` |
| `messages.getFavedStickers` | 低 | 查询 `user_faved_stickers` |
| `messages.saveRecentSticker` | 低 | 插入 `user_recent_stickers` |
| `messages.clearRecentStickers` | 低 | 清空 `user_recent_stickers` |

### Phase 3：上传/创建/管理（2-3 周）

**目标**：让用户能创建自己的贴纸集并上传贴纸。

| 接口 | 难度 | 说明 |
|------|------|------|
| `media.uploadStickerFile` | 中 | **解锁企业版锁定**，复用现有 Document 上传流程 |
| `messages.createStickerSet` | 中 | 创建 `sticker_sets` + 多个 `stickers` 记录 |
| `messages.addStickerToSet` | 中 | 新增 sticker 到已有集合 |
| `messages.removeStickerFromSet` | 低 | 删除 sticker 记录 |
| `messages.setStickerPositionInSet` | 低 | 更新 `sort_order` |

**技术要点**：
- `uploadStickerFile` 需要返回 `Document` 对象，复用 `app/service/media` 中现有的 `document` 创建逻辑
- 创建贴纸集时需要为每个 sticker 生成独立的 `document_id`

### Phase 4：高级功能（1-2 周）

| 接口 | 难度 | 说明 |
|------|------|------|
| `messages.getFeaturedStickers` | 低 | 查询 `featured_sticker_sets` |
| `messages.readFeaturedStickers` | 低 | 标记 `unread=false` |
| `messages.getArchivedStickers` | 低 | 查询 `user_sticker_sets` 中 `archived=true` |
| `messages.getMaskStickers` | 低 | 查询 `sticker_sets` 中 `masks=true` |
| `messages.getEmojiStickers` | 低 | 查询 `sticker_sets` 中 `emojis=true` |
| `channels.setEmojiStickers` | 中 | **解锁企业版锁定**，绑定频道与贴纸集 |

---

## 五、参考代码结构（以 `messages.getStickerSet` 为例）

### 5.1 BFF Handler 模板

```go
// app/bff/messages/internal/core/messages.getStickerSet_handler.go
package core

import (
    "github.com/teamgram/proto/mtproto"
    "github.com/teamgram/teamgram-server/app/bff/messages/internal/svc"
)

// MessagesGetStickerSet
// messages.getStickerSet#2619a90e stickerset:InputStickerSet hash:int = messages.StickerSet;
func (c *MessagesCore) MessagesGetStickerSet(in *mtproto.TLMessagesGetStickerSet) (*mtproto.Messages_StickerSet, error) {
    // 1. 解析 InputStickerSet（可能是 ID 或 ShortName）
    // 2. 查询 sticker_sets 表
    // 3. 查询 stickers 表获取贴纸列表
    // 4. 查询 Document 信息（调用 Media Service）
    // 5. 组装 Messages_StickerSet 返回
    
    set, err := c.svcCtx.Dao.StickerSetDAO.SelectByInput(c.ctx, in.Stickerset)
    if err != nil {
        c.Logger.Errorf("messages.getStickerSet - error: %v", err)
        return nil, err
    }
    if set == nil {
        return nil, mtproto.ErrStickerSetInvalid
    }
    
    // ... 组装返回
    return mtproto.MakeTLMessagesStickerSet(&mtproto.Messages_StickerSet{
        Set: set.ToStickerSet(),
        // Packs: ...,  // emoji -> document_id 映射
        // Documents: ..., // 调用 media 获取 Document 列表
    }).To_Messages_StickerSet(), nil
}
```

### 5.2 需要注册到 gRPC Service

```go
// app/bff/messages/internal/server/grpc/service/messages_service_impl.go
func (s *Service) MessagesGetStickerSet(ctx context.Context, request *mtproto.TLMessagesGetStickerSet) (*mtproto.Messages_StickerSet, error) {
    c := svc.NewMessagesCore(ctx, s.svcCtx)
    r, err := c.MessagesGetStickerSet(request)
    return r, err
}
```

### 5.3 移除 Fake RPC Result

```go
// app/bff/bff/client/fake_rpc_result.go
// 删除以下 case：
// case "TLMessagesGetStickerSet": ...
// case "TLMessagesGetAllStickers": ...
// 等已经实现的接口
```

---

## 六、关键风险与注意事项

### 6.1 企业版锁定的处理

对于 `media.uploadStickerFile` 和 `channels.setEmojiStickers` 这类被锁定的接口：
- **开发阶段**：直接修改对应 handler 文件，将 `ErrEnterpriseIsBlocked` 替换为真实逻辑
- **升级注意**：如果后续升级 teamgram 版本，这些修改可能会被覆盖，建议做好 Git 分支管理

### 6.2 与 Media Service 的交互

Sticker 本质上是一种 `Document`，其文件存储复用现有的 `document` + `MinIO` 体系。上传 sticker 后需要：
1. 调用 Media Service 生成 `Document` 对象（获得 `document_id`）
2. 将 `document_id` 与 sticker 元数据（emoji、alt 等）存入 `stickers` 表

### 6.3 Hash 缓存机制

Telegram 客户端使用 `hash` 参数做增量同步（如果服务器 hash 与本地一致，返回 `NotModified`）。
- `sticker_sets.hash` 字段需要在数据变更时更新
- 返回类型需要支持 `Messages_AllStickersNotModified` 等变体

### 6.4 与现有消息发送的集成

在 `messages.sendMessage` 中，`update_stickersets_order:flags.15?true` 字段当前被注释掉了。当用户通过 sticker 面板发送贴纸后，可能需要触发贴纸集排序更新。

---

## 七、总结

Sticker 功能是**社区版自研的最佳切入点**：

1. **协议层已完备**：不需要写任何 protobuf/序列化代码
2. **客户端兼容性好**：`fake_rpc_result` 已让客户端不崩溃，实现后能直接看到效果
3. **不涉及复杂分布式逻辑**：主要是单用户的 CRUD + 文件上传，不需要改造消息投递、同步等核心链路
4. **有明确参考**：Telegram 官方文档对每个接口的输入输出描述非常清晰

**推荐起步顺序**：
1. 执行 Phase 1 的数据库 SQL
2. 先用 `messages.getStickerSet` 练手（只需查一张表）
3. 再实现 `messages.getAllStickers` + `messages.installStickerSet`
4. 有了基础后，逐步解锁 `uploadStickerFile` 实现创建功能
