# App 層：CreatePost 邏輯

## CreatePostAsUser()

**檔案**：`server/channels/app/post.go:39`

薄包裝層，設定 session context 後呼叫 `CreatePost()`：

```go
func (a *App) CreatePostAsUser(rctx request.CTX, post *model.Post, currentSessionId string, setOnline bool) (*model.Post, *model.AppError) {
    // 設定發送者身份到 context
    // 呼叫 CreatePost()
}
```

## CreatePost() 核心流程

**檔案**：`server/channels/app/post.go:161-481`

### 階段 1：前置處理（161-356）

```
✓ 重複訊息去重檢查（Pending Post ID）
✓ 載入 Channel 與 User 資料
✓ 驗證 channel 狀態（未封存、未關閉）
✓ 處理 @mentions 解析
✓ 處理 hashtags
✓ 解析訊息中的 URL embed
✓ 套用字數限制
```

### 階段 2：DB 寫入（357）

```go
rpost, nErr := a.Srv().Store().Post().Save(rctx, post)
```

**這是整個流程中最關鍵的同步點**：
- 寫入成功後才繼續後續步驟
- 失敗則直接回傳錯誤給 API，不發送任何事件

### 階段 3：後處理（381-465）

```
✓ 附加檔案（FileInfo 關聯）       post.go:381
✓ PreparePostForClient()           post.go:412
    └ 加入 reactions、files、embeds metadata
✓ 自動翻譯（若啟用）              post.go:417
```

### 階段 4：事件觸發（466）

```go
if err := a.handlePostEvents(rctx, rpost, user, channel,
    flags.TriggerWebhooks, parentPostList, flags.SetOnline); err != nil {
    // 記錄 error 但不回傳 — 訊息已儲存成功
}
```

注意：`handlePostEvents` 的錯誤**不會**造成 `CreatePost` 回傳失敗。訊息已持久化，事件發送是盡力而為。

### 階段 5：回傳（481）

回傳已儲存的 `*model.Post` 給 API 層，API 回 HTTP 201。

## handlePostEvents()

**檔案**：`server/channels/app/post.go:640-688`

```go
func (a *App) handlePostEvents(rctx request.CTX, post *model.Post, user *model.User,
    channel *model.Channel, triggerWebhooks bool, parentPostList *model.PostList,
    setOnline bool) error {

    // 1. 清除快取
    a.invalidateCacheForChannel(channel)          // line 660
    a.invalidateCacheForChannelPosts(channel.Id)  // line 661

    // 2. 發送通知（含 WebSocket 事件）
    _, err := a.SendNotifications(rctx, post, team, channel, user, parentPostList, setOnline) // line 666

    // 3. 觸發 outgoing webhooks（如啟用）
    if triggerWebhooks {
        a.handleWebhookEvents(rctx, post, team, channel, user) // line 679
    }
}
```

## 重複訊息防護（Pending Post ID）

**檔案**：`server/channels/app/post.go:172-186`

客戶端發送時可附帶 `PendingPostId`（UUID）。Server 用此 ID 作為冪等鍵：

```go
if post.PendingPostId != "" {
    if cached, ok := a.Srv().Store().Post().GetPostByPendingPostId(post.PendingPostId); ok {
        return cached, nil  // 直接回傳已存在的 post
    }
}
```

這防止了網路重試造成的重複訊息。
