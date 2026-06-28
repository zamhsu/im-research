# WebSocket 即時通訊流程

## 概覽

Mattermost 使用 WebSocket 推送即時事件（新訊息、頻道變更、使用者狀態等）。  
客戶端在頁面載入後主動建立 WebSocket 連線，伺服器透過 Hub 廣播機制將事件推送給相關連線。

---

## 連線建立流程

### 入口端點

```
GET /api/v4/websocket   (HTTP Upgrade → WebSocket)
```

### 呼叫鏈

```
api4/websocket.go:connectWebSocket()               [HTTP Handler]
    ↓
    從 query param 取得：
    ├─ connection_id — 重連時的舊連線 ID
    └─ sequence_number — 斷線前的最後事件序號
    ↓
    HTTP Upgrade → WebSocket (使用 gorilla/websocket)
    ↓
    建立 WebConnConfig：
    ├─ Session
    ├─ IPAddress
    ├─ XForwardedFor
    └─ ConnectionID
    ↓
app/platform/service.go:NewWebConn()               [Platform Layer]
    ├─ 建立 WebConn 結構：
    │      ├─ Send channel (buffered, 256 個事件)
    │      ├─ Session
    │      └─ Pump goroutines
    │
    ├─ 向 Hub 註冊：HubRegister(wc)
    └─ 啟動雙向 Pump：wc.Pump()
```

---

## Hub 架構

```
┌─────────────────────────────────────────────────────┐
│                     Hub 陣列                         │
│  Hub[0]    Hub[1]    Hub[2]  ...  Hub[N-1]           │
│   │         │         │               │              │
│  User→Hub 對應由 userId hash 決定                    │
└─────────────────────────────────────────────────────┘
        │
        ├─ 每個 Hub 管理一批 WebConn
        ├─ 每個 Hub 在獨立的 goroutine 中運行
        └─ 廣播訊息透過 channel 傳入 Hub
```

- 預設 Hub 數量 = CPU 核心數
- 一個使用者的所有連線（多分頁、多裝置）都在同一個 Hub

---

## 訊息廣播流程

當業務邏輯產生需要即時推送的事件（如新訊息），廣播流程如下：

```
app/publish.go:Publish(message)                    [App Layer]
    ↓
platform.PlatformService.Publish(message)
    ↓
    依 message.Broadcast 決定目標：
    ├─ ChannelId → 廣播給頻道所有成員
    ├─ UserId → 廣播給特定使用者
    └─ TeamId → 廣播給 Team 所有成員
    ↓
Hub.Broadcast(message)                             [Platform Layer]
    ↓
    遍歷 Hub 中的 WebConn：
    ├─ 過濾不符合條件的連線（頻道成員檢查等）
    └─ WebConn.Send <- message
    ↓
WebConn 的 writePump goroutine
    └─ conn.WriteMessage(websocket.TextMessage, jsonPayload)
    ↓
客戶端接收 WebSocket 訊息
```

---

## 客戶端 → 伺服器的訊息

客戶端也可主動送訊息給伺服器（例如打字中狀態）：

```
客戶端 WebSocket 送出：
{
    "action": "user_typing",
    "seq": 1,
    "data": { "channel_id": "...", "parent_id": "" }
}
    ↓
WebConn 的 readPump goroutine
    ↓
wsapi/websocket_handler.go:ServeWebSocket()        [wsapi Layer]
    ↓
    依 action 分發到對應 handler：
    ├─ "user_typing"      → userTyping()
    ├─ "get_statuses"     → getStatuses()
    └─ "authentication_challenge" → authenticationChallenge()
    ↓
handler 執行業務邏輯
    └─ 必要時再透過 Hub 廣播回應事件
```

---

## 常見 WebSocket 事件列表

| 事件名稱 | 觸發時機 |
|---------|---------|
| `posted` | 新訊息發送 |
| `post_edited` | 訊息被編輯 |
| `post_deleted` | 訊息被刪除 |
| `channel_created` | 頻道建立 |
| `channel_deleted` | 頻道刪除 |
| `user_added` | 使用者加入頻道 |
| `user_removed` | 使用者離開頻道 |
| `user_updated` | 使用者資料更新 |
| `typing` | 使用者正在輸入 |
| `status_change` | 使用者線上狀態變更 |
| `thread_updated` | 執行緒（Thread）更新 |
| `channel_viewed` | 頻道已讀 |
| `new_mention` | 收到 @mention |

---

## WebSocket 訊息格式

### 伺服器 → 客戶端（Event）

```json
{
    "event": "posted",
    "data": {
        "channel_display_name": "...",
        "channel_name": "...",
        "channel_type": "O",
        "post": "{...}"
    },
    "broadcast": {
        "omit_users": null,
        "user_id": "",
        "channel_id": "channel123",
        "team_id": ""
    },
    "seq": 42
}
```

### 客戶端 → 伺服器（Request）

```json
{
    "action": "user_typing",
    "seq": 1,
    "data": {
        "channel_id": "channel123",
        "parent_id": ""
    }
}
```

---

## 重連機制

斷線重連時，客戶端帶上 `connection_id` 和 `sequence_number`：

```
GET /api/v4/websocket?connection_id=xxx&sequence_number=42
```

伺服器會將斷線期間遺漏的事件（序號 > 42）補送給客戶端，確保不漏訊息。

---

## 相關資料結構

### `model.WebSocketEvent`

| 欄位 | 說明 |
|------|------|
| `Event` | 事件名稱（如 `posted`） |
| `Data` | 事件內容（map） |
| `Broadcast` | 廣播目標（頻道/使用者/Team） |
| `Sequence` | 序號，用於重連補送 |

### `platform.WebConn`

| 欄位 | 說明 |
|------|------|
| `Send` | buffered channel，存放待送訊息 |
| `Session` | 對應的使用者 Session |
| `T` | 連線建立時間 |
| `Sequence` | 最後發出的序號 |

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/websocket.go:57` | `connectWebSocket()` HTTP handler |
| `server/channels/app/platform/websocket_router.go` | Hub 管理與廣播 |
| `server/channels/app/platform/websocket.go` | WebConn 實作 |
| `server/channels/wsapi/websocket_handler.go` | 客戶端訊息路由 |
| `server/channels/app/publish.go` | 業務層事件發布入口 |
| `server/public/model/websocket_message.go` | WebSocket 訊息模型 |
