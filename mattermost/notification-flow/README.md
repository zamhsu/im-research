# Mattermost 通知廣播流程研究

> 研究問題：在 channel 發送訊息後，如何大量發送通知到客戶端？

---

## 整體架構圖

```
用戶發送訊息
     │
     ▼
[API 層] POST /api/v4/posts
     │  api4/post.go: createPost()
     │
     ▼
[App 層] CreatePostAsUser() → CreatePost()
     │  app/post.go
     │  ├─ 儲存到資料庫
     │  ├─ 觸發 Plugin hooks
     │  └─ handlePostEvents()
     │
     ▼
[事件分發] SendNotifications()
     │  app/notification.go
     │
     ├──────────────────────────────────────────────┐
     │                                              │
     ▼                                              ▼
[Path A] WebSocket 通知                    [Path B] Push 通知（手機）
app/platform/cluster.go                    app/notification_push.go
     │                                              │
     ▼                                              ▼
[Hub 廣播系統]                              [PushNotificationsHub]
app/platform/web_hub.go                    HTTP POST → Push Proxy
     │
     ▼
[WebConn 逐一投遞]
app/platform/web_conn.go
```

---

## 詳細流程說明

### 1. API 入口

**檔案：** `server/channels/api4/post.go:96`

```go
func createPost(c *Context, w http.ResponseWriter, r *http.Request) {
    // 解析 POST body
    // 驗證權限
    c.App.CreatePostAsUser(rctx, post, sessionId, setOnline)
}
```

---

### 2. Post 建立流程

**檔案：** `server/channels/app/post.go`

```
CreatePostAsUser()  [line 39]
  └─> CreatePost()  [line 161]
        ├─ 驗證 channel 成員資格
        ├─ Store().Post().Save()  → 寫入資料庫
        ├─ Plugin hooks: MessageHasBeenPosted
        └─ handlePostEvents()  [line 466]
```

**handlePostEvents()** `[line 640]`
```
handlePostEvents()
  ├─> SendNotifications()       ← 主要通知邏輯
  ├─> SendAutoResponseIfNecessary()
  └─> handleWebhookEvents()
```

---

### 3. 通知分發核心：SendNotifications()

**檔案：** `server/channels/app/notification.go:54`

這是整個系統的核心，負責決定誰要收到通知以及用什麼方式發送。

#### 3A. WebSocket 通知（即時推播給在線客戶端）

```go
// line 689
message := model.NewWebSocketEvent(model.WebsocketEventPosted, ...)
message.Add("channel_type", channel.Type)
message.Add("sender_name", senderName)
// ... 附上 mentions、file info 等

a.publishWebsocketEventForPost(rctx, post, message)  // line 736
```

**publishWebsocketEventForPost()** `[app/post.go:1010]`
```
publishWebsocketEventForPost()
  ├─ 移除敏感 metadata（安全考量）
  ├─ 預先序列化 JSON（所有用戶共用一份，避免重複序列化）
  ├─ 掛載 broadcast hooks（每人個別過濾）：
  │   ├─ addMentionsBroadcastHook
  │   ├─ addFollowersBroadcastHook
  │   ├─ permalinkBroadcastHook
  │   ├─ channelMentionsBroadcastHook
  │   ├─ burnOnReadBroadcastHook
  │   └─ abacFilesBroadcastHook
  └─ a.Publish(message)  ← 廣播出去
```

#### 3B. Push 通知（手機離線推播）

```go
// line 530
if a.canSendPushNotifications() {
    for _, id := range mentionedUsersList {
        status = a.GetStatus(id)
        if a.ShouldSendPushNotification(user, channelMemberNotifyProps, ...) {
            a.sendPushNotification(notification, user, ...)
        }
    }
}
```

---

### 4. Hub 廣播系統（WebSocket 核心）

**檔案：** `server/channels/app/platform/cluster.go:189`

```go
func (ps *PlatformService) Publish(message *WebSocketEvent) {
    // 1. 更新 metrics
    // 2. PublishSkipClusterSend(message)
    //    └─ 根據 userID 計算 hash → 選擇對應的 Hub
    //    └─ hub.Broadcast(event)  → 塞入 hub 的 broadcast channel
    // 3. 如果是 cluster 環境：傳給其他節點
}
```

#### Hub 路由設計

**檔案：** `server/channels/app/platform/web_hub.go:157`

```
用戶 ID → hash → Hub[i]   (i = hash % runtime.NumCPU())
```

- 每個 CPU core 對應一個 Hub
- 每個 Hub 獨立運行，平行處理廣播事件
- 達到水平擴展效果

#### Hub.Start() 廣播迴圈

**檔案：** `server/channels/app/platform/web_hub.go:538`

```go
case msg := <-h.broadcast:
    // 取出 broadcast hooks
    // 預先序列化 JSON（只做一次）
    for _, webConn := range h.connections {
        if webConn.ShouldSendEvent(msg) {
            // 執行該用戶的 broadcast hooks（個別化內容）
            // 送入 webConn.send channel
        }
    }
```

#### 連線索引（高效查詢）

```go
// hubConnectionIndex 支援多種查詢模式
hubConnectionIndex.ForUser(userId)          // 針對特定用戶
hubConnectionIndex.ForChannel(channelId)    // 針對 channel 成員
hubConnectionIndex.ForAll()                 // 全部連線
hubConnectionIndex.ForConnection(connId)    // 針對單一連線
```

---

### 5. WebConn：最後一哩投遞

**檔案：** `server/channels/app/platform/web_conn.go`

每個 WebSocket 連線對應一個 `WebConn`：

```go
type WebConn struct {
    send      chan *model.WebSocketEvent  // 發送佇列（buffer 256）
    deadQueue [128]*model.WebSocketEvent  // 斷線重連用的循環緩衝區
    WebSocket *websocket.Conn             // 實際 TCP 連線
    Sequence  int64                       // 訊息序號（用於重連恢復）
}
```

**Write Pump**（把訊息寫入 WebSocket）：
- 從 `send` channel 讀取事件
- 序列化後寫入 WebSocket frame
- 維護 ping/pong 心跳（每 60% pongWait 發一次 ping）

**Read Pump**（接收客戶端訊息）：
- 監聽客戶端指令
- 更新用戶活躍時間
- 路由到對應的 request handler

---

### 6. Push Notification Hub（手機推播）

**檔案：** `server/channels/app/notification_push.go:440`

```
PushNotificationsHub
├─ notificationsChan  ← 緩衝 channel，接收推播任務
└─ start() 迴圈
     ├─ 取得 semaphore（最大並行數 = NumCPU * 8）
     └─ sendPushNotificationSync()
          └─ sendPushNotificationToAllSessions()
               └─ 對每個手機 session：
                    ├─ 建立 PushNotification（含 JWT 簽名）
                    └─ sendToPushProxy()  → HTTP POST 到 Push Proxy
```

Push Proxy 再轉送到 APNs（iOS）或 FCM（Android）。

---

## 關鍵資料結構

### WebSocketEvent

```go
type WebSocketEvent struct {
    EventType string                  // e.g., "posted"
    Broadcast WebsocketBroadcast      // 路由資訊
    Data      map[string]interface{}  // 訊息 payload
    Sequence  int64                   // 訊息序號
}
```

### WebsocketBroadcast（路由控制）

```go
type WebsocketBroadcast struct {
    UserId             string  // 只送給特定用戶
    ChannelId          string  // 送給 channel 所有成員
    TeamId             string  // 送給 team 所有成員
    ConnectionId       string  // 只送給單一連線
    IgnoreConnectionId string  // 排除發送者自己
}
```

---

## 效能設計重點

| 機制 | 說明 |
|------|------|
| **多 Hub 平行處理** | Hub 數量 = CPU 核心數，各自獨立廣播 |
| **一次序列化** | JSON 只序列化一次，所有 WebConn 共用 |
| **Broadcast Hooks** | 個人化內容在 Hub 內懶執行，避免預先計算所有組合 |
| **Dead Queue** | WebConn 保留最近 128 筆，斷線重連可恢復訊息 |
| **Push Semaphore** | 最大並行推播 = NumCPU × 8，避免 goroutine 爆炸 |
| **Channel 路由索引** | O(1) 查詢某 channel 的所有連線 |
| **Cluster 傳播** | 多節點部署時，事件透過 cluster message 廣播到所有節點 |

---

## 完整呼叫鏈總結

```
POST /api/v4/posts
  → api4/post.go: createPost()
  → app/post.go: CreatePostAsUser() → CreatePost()
  → app/post.go: handlePostEvents()
  → app/notification.go: SendNotifications()
      ├─[WebSocket]─→ app/post.go: publishWebsocketEventForPost()
      │                → app/platform/cluster.go: Publish()
      │                → app/platform/web_hub.go: Hub.Broadcast()
      │                → app/platform/web_hub.go: Hub.Start() broadcast loop
      │                → app/platform/web_conn.go: WebConn.send channel
      │                → WebSocket frame → 客戶端
      │
      └─[Push]──────→ app/notification_push.go: sendPushNotification()
                       → PushNotificationsHub.notificationsChan
                       → sendToPushProxy()
                       → Push Proxy → APNs / FCM → 手機
```

---

## 涉及的關鍵檔案

| 元件 | 檔案路徑 | 主要函式 |
|------|----------|----------|
| API 入口 | `server/channels/api4/post.go` | `createPost()` |
| Post 建立 | `server/channels/app/post.go` | `CreatePost()`, `handlePostEvents()`, `publishWebsocketEventForPost()` |
| 通知分發 | `server/channels/app/notification.go` | `SendNotifications()` |
| Push 通知 | `server/channels/app/notification_push.go` | `sendPushNotification()`, `sendToPushProxy()` |
| Cluster 廣播 | `server/channels/app/platform/cluster.go` | `Publish()` |
| Hub 廣播 | `server/channels/app/platform/web_hub.go` | `Hub.Start()`, `Broadcast()` |
| WebSocket 連線 | `server/channels/app/platform/web_conn.go` | `WebConn` struct, write/read pump |
| Broadcast Hooks | `server/channels/app/web_broadcast_hooks.go` | 各種個人化過濾 hook |
