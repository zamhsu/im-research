# 訊息（Post）操作流程

## Post 類型

| `Type` 值 | 說明 |
|----------|------|
| `""` (空字串) | 一般使用者訊息 |
| `system_*` | 系統訊息（加入頻道、更改名稱等），使用者不可發送 |
| `ephemeral` | 暫時訊息，只有目標使用者看得到，不存 DB |

---

## 一、建立訊息

### 入口端點

```
POST /api/v4/posts
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 在頻道發文 | `PermissionCreatePost`（channel-scoped） |
| 公開頻道（備用） | `PermissionCreatePostPublic`（team-scoped） |
| 訊息包含附件 | `PermissionUploadFile`（channel-scoped） |

### 呼叫鏈

```
api4/post.go:createPost()                          [HTTP Handler]
    ↓
    解析 JSON body → *model.Post
    ├─ createPostChecks()
    │      ├─ SessionHasPermissionToChannel(PermissionCreatePost)
    │      │      備用：SessionHasPermissionToTeam(PermissionCreatePostPublic)
    │      ├─ 若有 FileIds：SessionHasPermissionToChannel(PermissionUploadFile)
    │      ├─ 驗證 post.Type 合法性
    │      └─ 驗證優先級、burn-on-read 等功能開關
    ↓
    deduplicateCreatePost()                        [去重檢查 - API 層]
    └─ 用 PendingPostId 查 LRU memory cache
       若已存在 → 直接回傳已建立的 post
    ↓
app/post.go:CreatePostAsUser()                     [App Layer - 使用者發文]
    ├─ 驗證頻道未被刪除
    │      Store.Channel().Get(post.ChannelId)
    ├─ 驗證 post.Type 不以 "system_" 開頭
    └─ deduplicateCreatePost()                     [去重檢查 - App 層]
    ↓
app/post.go:CreatePost()                           [App Layer - 核心建立]
    ├─ 前置處理：
    │      ├─ 設定 CreateAt timestamp
    │      ├─ 解析 @mention
    │      └─ 驗證附件 FileIds
    │
    ├─ runGuardedMessageWillBePosted()             [Plugin Hook，可拒絕/修改內容]
    │
    ├─ Store.Post().SaveMultiple([]*model.Post)    [DB 寫入]
    │      INSERT INTO Posts (Id, ChannelId, UserId, Message, ...)
    │      UPDATE Posts SET ReplyCount = ReplyCount + 1  [若為回覆]
    │
    ├─ Store.Channel().UpdateLastPostAt()          [更新頻道元資料]
    │
    ├─ triggerOutgoingWebhooks()                   [觸發 Outgoing Webhook]
    │
    ├─ sendMentionNotifications()                  [處理 @mention 推播]
    │
    ├─ 非同步：RunMultiHook(MessageHasBeenPosted) [Plugin Hook]
    │
    └─ 發布 WebSocket 事件（透過 SendNotifications）：
           WebsocketEventPosted → 所有頻道成員
    ↓
api4/post.go:createPost()
    └─ 回傳 JSON: *model.Post (201 Created)
```

### 去重機制（`deduplicateCreatePost`）

- 客戶端在 `post.PendingPostId` 帶入自訂唯一 ID
- App 層以此 ID 在 LRU cache 中查詢
- 若找到（例如網路重試），直接回傳舊結果，不重複寫入 DB

### CreatePostFlags

`model.CreatePostFlags` 控制 `CreatePost()` 行為：

| 欄位 | 說明 |
|------|------|
| `TriggerWebhooks` | 是否觸發 Outgoing Webhook |
| `SetOnline` | 是否更新使用者線上狀態 |
| `SkipCache` | 是否略過快取 |

---

## 二、查詢訊息

### 入口端點與呼叫鏈

```
GET /api/v4/posts/{post_id}

api4/post.go:getPost()
    ├─ 隱式透過 GetPostIfAuthorized() 確認頻道成員身份
    └─ app.GetSinglePost(postId, includeDeleted)
        ├─ store.Post().GetSingle(postId)
        │      SELECT * FROM Posts WHERE Id = ? AND DeleteAt = 0
        └─ app.PreparePostForClientWithEmbedsAndImages()
               [補齊 metadata：reactions、embeds、file info]
```

```
GET /api/v4/channels/{channel_id}/posts

api4/post.go:getPostsForChannel()
    ├─ SessionHasPermissionToChannel(PermissionReadChannelContent)
    └─ 依參數選擇查詢模式：
       ├─ ?since=<timestamp>   → app.GetPostsSince()
       │      SELECT * FROM Posts WHERE ChannelId=? AND UpdateAt > ?
       │
       ├─ ?before=<post_id>   → app.GetPostsBeforePost()
       │      SELECT * FROM Posts WHERE ChannelId=? AND CreateAt < <post.CreateAt>
       │      ORDER BY CreateAt DESC LIMIT perPage
       │
       └─ ?after=<post_id>    → app.GetPostsAfterPost()
              SELECT * FROM Posts WHERE ChannelId=? AND CreateAt > <post.CreateAt>
              ORDER BY CreateAt ASC LIMIT perPage
       ↓
    app.PreparePostListForClient()
    app.AddCursorIdsForPostList()              [計算 NextPostId / PrevPostId]
    app.SanitizePostListMetadataForUser()
```

```
GET /api/v4/posts/{post_id}/thread

api4/post.go:getPostThread()
    └─ app.GetPostThread(postID, opts, userID)
        ├─ store.Post().Get(postID, options)
        │      [同時取得 root post 與所有 replies]
        ├─ revealBurnOnReadPostsForUser()      [burn-on-read 解密]
        ├─ filterInaccessiblePosts()           [移除無存取權的 post]
        ├─ applyPostsWillBeConsumedHook()      [Plugin Hook]
        ├─ app.PreparePostListForClient()
        └─ app.SanitizePostListMetadataForUser()
```

**支援的查詢參數（thread）：**

| 參數 | 說明 |
|------|------|
| `perPage` | 每頁數量 |
| `fromCreateAt` | 時間戳過濾起點 |
| `fromPost` | 從指定 post ID 開始 |
| `direction` | `up` 或 `down` |
| `updatesOnly` | 僅取更新過的 post |

---

## 三、執行緒（Thread）回覆

回覆訊息時，`post.RootId` 設為根訊息的 ID：

```
POST /api/v4/posts
Body: { "channel_id": "...", "message": "回覆內容", "root_id": "<parent_post_id>" }
```

App 層額外動作：
- `Store.Post().Update()` 更新根訊息的 `ReplyCount + 1`
- 發布 `WebsocketEventThreadUpdated` 事件
- 刪除根訊息時，reply 不連動刪除（RootId 欄位保留）

---

## 四、更新訊息

### 入口端點

```
PUT   /api/v4/posts/{post_id}         ← 完整替換
PATCH /api/v4/posts/{post_id}/patch   ← 部分更新
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 編輯自己的訊息 | `PermissionEditPost`（channel-scoped） |
| 編輯他人的訊息 | `PermissionEditOthersPosts`（channel-scoped） |
| 需同時具備發文能力 | `PermissionCreatePost` |
| 更改附件 | `PermissionUploadFile` + `PermissionEditFileAttachment` |

### 呼叫鏈

```
api4/post.go:updatePost()                          [HTTP Handler]
    ↓
    ├─ 驗證 post.Id 與 URL path 一致
    ├─ app.GetSinglePost()                         [取原始 post]
    ├─ 權限檢查（見上表）
    ├─ 檢查編輯時間限制
    │      若 PostEditTimeLimit > 0 且已超時 → 403
    └─ app.UpdatePost(post, safeUpdate=false)
    ↓
app/post.go:UpdatePost()                           [App Layer 核心]
    ├─ runGuardedMessageWillBeUpdated(new, old)    [Plugin Hook，可拒絕/修改]
    ├─ 設定 EditAt（若 Message 或 FileIds 有變更）
    ├─ store.Post().Update(newPost, oldPost)       [DB 更新]
    │      UPDATE Posts SET Message=?, EditAt=?, Props=?, ...
    │      WHERE Id = ?
    ├─ app.PreparePostForClientWithEmbedsAndImages()
    ├─ 非同步：RunMultiHook(MessageHasBeenUpdated) [Plugin Hook]
    └─ 發布 WebSocket 事件：
           WebsocketEventPostEdited
    ↓
api4/post.go:updatePost()
    └─ 回傳 JSON: *model.Post (200 OK)
```

```
api4/post.go:patchPost()                           [部分更新]
    ├─ postPatchChecks()
    │      ├─ 取原始 post
    │      ├─ 同 updatePost 的權限檢查
    │      └─ 檢查編輯時間限制
    └─ app.PatchPost()
        ├─ post.Patch(patch)                       [僅套用非 nil 欄位]
        └─ app.UpdatePost()                        [委派到完整更新流程]
```

---

## 五、刪除訊息

### 入口端點

```
DELETE /api/v4/posts/{post_id}
DELETE /api/v4/posts/{post_id}?permanent=true   ← 需 PermissionManageSystem
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 刪除自己的訊息 | `PermissionDeletePost`（channel-scoped） |
| 刪除他人的訊息 | `PermissionDeleteOthersPosts`（channel-scoped） |
| 永久刪除 | `PermissionManageSystem` |

### 軟刪除呼叫鏈

```
api4/post.go:deletePost()                          [HTTP Handler]
    ↓
    ├─ 取原始 post，確認存在
    ├─ 權限檢查（見上表）
    └─ app.DeletePost(postId, deleteByUserId)
    ↓
app/post.go:DeletePost()                           [App Layer]
    ├─ store.Post().Delete(postId, deleteAt, userId)
    │      UPDATE Posts SET DeleteAt = <now>, UpdateAt = <now>
    │      WHERE Id = ?                            [資料仍保留在 DB]
    ├─ 若有附件：非同步 deletePostFiles()
    ├─ 若為根訊息：DeletePersistentNotification()
    └─ CleanUpAfterPostDeletion()
           ├─ 發布 WebSocket 事件：WebsocketEventPostDeleted
           ├─ 非同步：RunMultiHook(MessageHasBeenDeleted) [Plugin Hook]
           ├─ RemoveNotifications()
           ├─ deleteDraftsAssociatedWithPost()
           └─ Cache 失效
```

**注意：** 根訊息刪除後，其 reply 不會連動刪除，仍可透過 thread 查詢。

### 永久刪除呼叫鏈

```
app/post.go:PermanentDeletePost()
    ├─ PermanentDeleteFilesByPost()                [刪除實體檔案]
    ├─ store.Post().PermanentDelete(postId)
    │      DELETE FROM Posts WHERE Id = ?
    └─ CleanUpAfterPostDeletion()                  [同軟刪除後處理]
```

---

## 六、釘選 / 取消釘選

### 入口端點

```
POST /api/v4/posts/{post_id}/pin
POST /api/v4/posts/{post_id}/unpin
```

### 呼叫鏈

```
api4/post.go:pinPost() / unpinPost()
    └─ saveIsPinnedPost(isPinned=true/false)
        ├─ SessionHasPermissionToChannel(PermissionReadChannelContent)
        ├─ 若已是目標狀態 → 直接回傳 200（不更新 EditAt）
        ├─ 建立 PostPatch{IsPinned: &isPinned}
        └─ app.PatchPost()
            └─ app.UpdatePost()
                ├─ store.Post().Update()
                │      UPDATE Posts SET IsPinned=?, EditAt=? WHERE Id=?
                ├─ 清除釘選數量快取
                └─ 發布 WebSocket 事件：
                       WebsocketEventPostEdited
```

---

## 七、表情符號回應（Reaction）

### 入口端點

```
POST   /api/v4/reactions                                        ← 新增 reaction
DELETE /api/v4/reactions/{user_id}/{post_id}/{emoji_name}       ← 刪除 reaction
```

### 新增 Reaction

```
api4/reaction.go:saveReaction()                    [HTTP Handler]
    ↓
    ├─ 驗證 user_id 為自己
    ├─ SessionHasPermissionToChannelByPost(PermissionAddReaction)
    └─ app.SaveReactionForPost()
    ↓
app/reaction.go:SaveReactionForPost()
    ├─ 取 post（確認未 burn-on-read 過期）
    ├─ 驗證 emoji 名稱存在（系統 or 自定義）
    ├─ 驗證未超過單一 post 的 reaction 種類上限
    ├─ store.Reaction().Save()
    │      INSERT INTO Reactions (PostId, UserId, EmojiName, ChannelId, CreateAt)
    ├─ 清除頻道快取
    ├─ 非同步：RunMultiHook(ReactionHasBeenAdded) [Plugin Hook]
    └─ 發布 WebSocket 事件：
           WebsocketEventReactionAdded
    ↓
api4/reaction.go:saveReaction()
    └─ 回傳 JSON: *model.Reaction (201 Created)
```

### 刪除 Reaction

```
api4/reaction.go:deleteReaction()                  [HTTP Handler]
    ↓
    ├─ SessionHasPermissionToChannel(PermissionRemoveReaction)
    ├─ 若刪除他人的 reaction：PermissionRemoveOthersReactions
    └─ app.DeleteReactionForPost()
    ↓
app/reaction.go:DeleteReactionForPost()
    ├─ store.Reaction().Delete()
    │      DELETE FROM Reactions
    │      WHERE PostId=? AND UserId=? AND EmojiName=?
    ├─ 清除頻道快取
    ├─ 非同步：RunMultiHook(ReactionHasBeenRemoved) [Plugin Hook]
    └─ 發布 WebSocket 事件：
           WebsocketEventReactionRemoved
```

---

## 八、搜尋訊息

### 入口端點

```
POST /api/v4/posts/search
```

### 呼叫鏈

```
api4/post.go:searchPosts()                         [HTTP Handler]
    ↓
    解析 SearchParameter{ terms, isOrSearch, page, perPage, includeDeletedChannels }
    └─ app.SearchPostsForUser(terms, userId, teamId, ...)
    ↓
app/post.go:SearchPostsForUser()
    ├─ 委派到設定的搜尋後端（SQL / Elasticsearch / Bleve）
    │      store.Post().Search()
    │
    ├─ FilterPostsByChannelPermissions()           [過濾無存取權的 post]
    ├─ metrics: IncrementPostsSearchCounter()
    └─ app.PreparePostListForClient()
```

**搜尋後端：** 依 `ServiceSettings.EnableElasticsearchIndexing` 等設定切換；  
預設為 SQL 全文搜尋（`LIKE` 或 `tsvector`）。

---

## WebSocket 事件彙整

| 事件常數 | 觸發時機 |
|---------|---------|
| `WebsocketEventPosted` | 新訊息建立 |
| `WebsocketEventPostEdited` | 訊息更新、釘選/取消釘選 |
| `WebsocketEventPostDeleted` | 訊息刪除 |
| `WebsocketEventEphemeralMessage` | 暫時訊息（只發給目標使用者） |
| `WebsocketEventReactionAdded` | 新增 Reaction |
| `WebsocketEventReactionRemoved` | 刪除 Reaction |
| `WebsocketEventThreadUpdated` | 回覆導致 Thread 更新 |
| `WebsocketEventPostRevealed` | Burn-on-read 訊息被閱覽 |
| `WebsocketEventPostBurned` | Burn-on-read 訊息過期銷毀 |

---

## Plugin Hook 彙整

| Hook | 時機 | 可否拒絕 | 可否修改內容 | 同步/非同步 |
|------|------|---------|------------|-----------|
| `MessageWillBePosted` | 建立訊息前 | 是 | 是（可替換 post） | 同步（兩階段） |
| `MessageHasBeenPosted` | 建立訊息後 | 否 | 否 | 非同步 |
| `MessageWillBeUpdated` | 更新訊息前 | 是 | 是（可替換 newPost） | 同步（兩階段） |
| `MessageHasBeenUpdated` | 更新訊息後 | 否 | 否 | 非同步 |
| `MessageHasBeenDeleted` | 刪除訊息後 | 否 | 否 | 非同步 |
| `ReactionHasBeenAdded` | 新增 Reaction 後 | 否 | 否 | 非同步 |
| `ReactionHasBeenRemoved` | 刪除 Reaction 後 | 否 | 否 | 非同步 |

**兩階段同步 Hook（MessageWillBePosted / MessageWillBeUpdated）：**
- Phase A：非 guard plugin 先執行（fail-open，任一個 reject 即終止本 Phase）
- Phase B：guard plugin 執行（fail-closed，reject 直接回傳錯誤）

---

## 相關資料結構

### `model.Post` (`server/public/model/post.go`)

| 欄位 | 說明 |
|------|------|
| `Id` | 26 字元 ULID |
| `ChannelId` | 所屬頻道 |
| `UserId` | 發文者 |
| `Message` | 訊息內容（Markdown） |
| `Type` | 訊息類型（空=一般，`system_*`=系統訊息） |
| `RootId` | 執行緒根訊息 ID（回覆時使用） |
| `OriginalId` | 原始訊息 ID（編輯時保留） |
| `FileIds` | 附件檔案 ID 列表 |
| `Props` | 延伸屬性（Webhook、優先級、按鈕等） |
| `Hashtags` | 解析出的 hashtag |
| `IsPinned` | 是否釘選 |
| `PendingPostId` | 客戶端去重用 ID |
| `CreateAt` | 建立時間（毫秒 timestamp） |
| `EditAt` | 最後編輯時間（0 = 從未編輯） |
| `DeleteAt` | 軟刪除時間（0 = 未刪除） |

### `model.Reaction` (`server/public/model/reaction.go`)

| 欄位 | 說明 |
|------|------|
| `PostId` | 所屬 Post ID |
| `UserId` | 使用者 ID |
| `EmojiName` | Emoji 名稱（小寫） |
| `ChannelId` | 所屬頻道（for 索引） |
| `CreateAt` | 建立時間 |

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/post.go` | 所有 Post HTTP handlers |
| `server/channels/api4/post_utils.go` | `createPostChecks()`、權限輔助 |
| `server/channels/api4/reaction.go` | Reaction HTTP handlers |
| `server/channels/app/post.go:39` | `CreatePostAsUser()` |
| `server/channels/app/post.go:161` | `CreatePost()` 核心邏輯 |
| `server/channels/app/post.go:115` | `deduplicateCreatePost()` 去重 |
| `server/channels/app/post.go:1885` | `DeletePost()` 軟刪除 |
| `server/channels/app/post.go:3218` | `PermanentDeletePost()` 永久刪除 |
| `server/channels/app/reaction.go` | Reaction 業務邏輯 |
| `server/channels/app/guarded_hooks.go` | `MessageWillBePosted/Updated` 兩階段 Hook |
| `server/channels/app/notification.go` | `SendNotifications()`、`WebsocketEventPosted` 發布 |
| `server/channels/app/publish.go` | WebSocket 事件廣播 |
| `server/channels/store/sqlstore/post_store.go` | SQL 持久化 |
| `server/public/model/post.go` | Post 資料模型 |
| `server/public/model/reaction.go` | Reaction 資料模型 |
