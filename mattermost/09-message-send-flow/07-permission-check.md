# ShouldSendEvent 權限過濾

## 位置與時機

**檔案**：`server/channels/app/platform/web_conn.go:884`

`ShouldSendEvent()` 在 Hub main loop 中對**每個 WebConn** 個別執行，決定這個連線是否應該收到某個事件。這是事件送達前最後一道過濾關卡。

## 過濾規則（依序）

### 1. 認證檢查

```go
if !wc.IsAuthenticated() {
    return false  // 未完成認證（含 MFA）的連線不收任何事件
}
```

### 2. 連線過慢保護

```go
if len(wc.send) >= sendSlowWarn {
    switch msg.EventType() {
    case model.WebsocketEventTyping,
         model.WebsocketEventStatusChange,
         model.WebsocketEventMultipleChannelsViewed:
        return false  // 丟棄低優先級事件
    }
}
```

### 3. 資料敏感度分流

這是一個「事件分流」設計：同一份資料會建立兩個事件版本：
- `ContainsSanitizedData = true`：已清理版本 → 只給**非管理員**
- `ContainsSensitiveData = true`：原始版本 → 只給**管理員**

```go
var hasReadPrivateDataPermission *bool

// 清理版 → 管理員不能收（他們有更詳細的版本）
if msg.GetBroadcast().ContainsSanitizedData {
    hasReadPrivateDataPermission = checkPermission(PermissionManageSystem)
    if *hasReadPrivateDataPermission {
        return false
    }
}

// 敏感版 → 非管理員不能收
if msg.GetBroadcast().ContainsSensitiveData {
    if !*hasReadPrivateDataPermission {
        return false
    }
}
```

### 4. 特定 Connection

```go
if msg.GetBroadcast().ConnectionId != "" {
    return wc.GetConnectionID() == msg.GetBroadcast().ConnectionId
}
```

### 5. 排除特定 Connection

```go
if wc.GetConnectionID() == msg.GetBroadcast().OmitConnectionId {
    return false
}
```

### 6. 特定 User

```go
if msg.GetBroadcast().UserId != "" {
    return wc.UserId == msg.GetBroadcast().UserId
}
```

### 7. 排除特定 Users

```go
if len(msg.GetBroadcast().OmitUsers) > 0 {
    if _, ok := msg.GetBroadcast().OmitUsers[wc.UserId]; ok {
        return false
    }
}
```

### 8. 頻道成員驗證

```go
if chID := msg.GetBroadcast().ChannelId; chID != "" {
    // 若啟用 fast iteration，Hub 已在路由階段確保只發給頻道成員
    if *wc.Platform.Config().ServiceSettings.EnableWebHubChannelIteration {
        return true  // 已由 Hub 層過濾，直接通過
    }

    // Fallback：查詢 user 的所有頻道成員資格
    // 結果快取 30 分鐘
    if model.GetMillis()-wc.lastAllChannelMembersTime > webConnMemberCacheTime {
        wc.allChannelMembers = nil
    }
    if wc.allChannelMembers == nil {
        result, _ := wc.Platform.Store.Channel().GetAllChannelMembersForUser(...)
        wc.allChannelMembers = result
        wc.lastAllChannelMembersTime = model.GetMillis()
    }

    _, ok := wc.allChannelMembers[chID]
    return ok  // 是頻道成員才能收
}
```

### 9. Team 成員驗證

```go
if msg.GetBroadcast().TeamId != "" {
    return wc.isMemberOfTeam(msg.GetBroadcast().TeamId)
}
```

### 10. Guest 使用者限制

```go
if wc.GetSession().Props[model.SessionPropIsGuest] == "true" {
    return wc.ShouldSendEventToGuest(msg)
}
```

Guest 使用者有額外的事件過濾規則（例如不能看見某些系統事件）。

## 特殊過濾：WebSocketEventScope Feature Flag

**檔案**：`web_conn.go:966-973`

當 `WebSocketEventScope` feature flag 開啟時，`typing`、`reaction_added`、`reaction_removed` 事件**只發給**目前有開啟該頻道或 thread 的 connection：

```go
if wc.Platform.Config().FeatureFlags.WebSocketEventScope &&
    slices.Contains([]model.WebsocketEventType{
        model.WebsocketEventTyping,
        model.WebsocketEventReactionAdded,
        model.WebsocketEventReactionRemoved,
    }, msg.EventType()) && wc.notInChannel(chID) && wc.notInThread(chID) {
    return false
}
```

這是一個效能優化：減少傳送「使用者沒在看」的即時事件。

## 快取機制

| 快取項目 | 欄位 | TTL |
|----------|------|-----|
| 所有頻道成員資格 | `wc.allChannelMembers` | 30 分鐘（`webConnMemberCacheTime`）|
| Team 成員資格 | `wc.teamMembers` | 30 分鐘 |
| 目前開啟的頻道 | `wc.activeChannelId` | 即時（由客戶端 update 訊息更新）|

快取讓每次事件不需要 DB 查詢，但代價是成員變更後最多 30 分鐘才生效（成員被踢出後仍可能短暫收到事件）。`invalidateUser` channel 可以強制清除快取。
