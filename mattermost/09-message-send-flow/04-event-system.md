# WebSocket 事件的建立與發布

## 事件類型常數

**檔案**：`server/public/model/websocket_message.go:17-19`

```go
WebsocketEventPosted      WebsocketEventType = "posted"
WebsocketEventPostEdited  WebsocketEventType = "post_edited"
WebsocketEventPostDeleted WebsocketEventType = "post_deleted"
```

## WebSocketEvent 資料結構

**檔案**：`server/public/model/websocket_message.go:237-244`

```go
type WebSocketEvent struct {
    event           WebsocketEventType      // 事件類型
    data            map[string]any          // 事件 payload
    broadcast       *WebsocketBroadcast     // 發送目標範圍
    sequence        int64                   // 客戶端接收序號
    precomputedJSON *precomputedWebSocketEventJSON  // 預先序列化快取
    rejected        bool                    // 是否已被 hook 拒絕
}
```

## WebsocketBroadcast：發送範圍控制

**檔案**：`server/public/model/websocket_message.go:146-169`

```go
type WebsocketBroadcast struct {
    OmitUsers              map[string]bool  // 排除這些 user IDs
    UserId                 string           // 只發給這個 user
    ChannelId              string           // 發給頻道所有成員
    TeamId                 string           // 發給 team 所有成員
    ConnectionId           string           // 只發給這個 connection
    OmitConnectionId       string           // 排除這個 connection
    ContainsSanitizedData  bool             // 只給非管理員
    ContainsSensitiveData  bool             // 只給管理員
    ReliableClusterSend    bool             // 叢集用 Reliable 模式傳送
    BroadcastHooks         []string         // 每個 connection 執行的 hooks
    BroadcastHookArgs      []map[string]any
}
```

## SendNotifications() 中的事件建立

**檔案**：`server/channels/app/notification.go:689-740`

```go
// 建立事件：目標範圍 = 整個頻道
message := model.NewWebSocketEvent(
    model.WebsocketEventPosted,
    "",            // teamId（空 = 不限 team）
    post.ChannelId, // channelId（頻道所有成員）
    "",            // userId（空 = 不限單一 user）
    nil,           // omitUsers
    "",            // omitConnectionId
)

// 加入頻道 metadata
message.Add("channel_type", channel.Type)
message.Add("channel_display_name", channel.DisplayName)
message.Add("channel_name", channel.Name)
message.Add("sender_name", senderName)
message.Add("team_id", teamId)
message.Add("set_online", setOnline)

// 組裝 post JSON 並發布
appErr := a.publishWebsocketEventForPost(rctx, post, message)
```

### NewWebSocketEvent() 建構子

```go
func NewWebSocketEvent(event WebsocketEventType, teamId, channelId, userId string,
    omitUsers map[string]bool, omitConnectionId string) *WebSocketEvent {
    return &WebSocketEvent{
        event: event,
        data:  make(map[string]any),
        broadcast: &WebsocketBroadcast{
            TeamId:           teamId,
            ChannelId:        channelId,
            UserId:           userId,
            OmitUsers:        omitUsers,
            OmitConnectionId: omitConnectionId,
        },
    }
}
```

## publishWebsocketEventForPost()

**檔案**：`server/channels/app/post.go:1010-1066`

```go
func (a *App) publishWebsocketEventForPost(rctx request.CTX, post *model.Post,
    message *model.WebSocketEvent) *model.AppError {

    // 序列化 post 為 JSON
    postJSON, jsonErr := post.ToJSON()

    // 將 post JSON 加入 event data
    message.Add("post", postJSON)

    // 發布事件
    a.Publish(message)

    return nil
}
```

## Thread 更新事件

除了 `"posted"` 事件，若訊息是回覆（有 `RootId`），也會發送 `WebsocketEventThreadUpdated` 事件：

**檔案**：`server/channels/app/notification.go:769`

```go
message := model.NewWebSocketEvent(
    model.WebsocketEventThreadUpdated,
    team.Id,
    "",
    uid,  // 只發給特定 user
    nil, "",
)
```

這個事件的 broadcast scope 是 **per-user**（`UserId != ""`），而非整個頻道。
