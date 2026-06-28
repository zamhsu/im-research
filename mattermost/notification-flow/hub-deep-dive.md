# Hub 深度解析

## Hub 是什麼

**Hub 是這個節點上所有 WebSocket 連線的管理中心**，負責兩件事：

1. **維護連線 registry**（知道哪個 user / channel 有哪些 WebSocket 連線在線）
2. **把事件廣播給正確的連線**

Hub 不只是「本機事件傳遞」，更精確的說法是：

> Hub 是 WebSocket 連線的本機管理者，知道「這台機器上哪些用戶在線、有幾條連線、在哪些 channel」，並且負責把事件高效地投遞給正確的連線。

跨節點的事件（其他節點上的用戶）不經過這個 Hub，由 ClusterInterface 先傳到對方節點，再由對方的 Hub 處理。

---

## Hub 的規模

```go
// web_hub.go:128
numberOfHubs := runtime.NumCPU()
```

- 每個 CPU core 一個 Hub goroutine，平行運作
- 透過 `userID hash` 決定用哪個 Hub：`hash(userID) % NumCPU`
- 廣播佇列 buffer 大小為 4096（`broadcastQueueSize`）

---

## Hub 管理的六種操作

`Hub.Start()` 是一個無限 select 迴圈（`web_hub.go:538`），處理六類訊號：

| 訊號 channel | 觸發時機 | 做什麼 |
|---|---|---|
| `register` | 客戶端建立 WebSocket 連線 | 登記到連線索引，送出 hello 訊息 |
| `unregister` | 客戶端斷線 | 從索引移除；若該 user 全部離線則設 status offline |
| `broadcast` | 業務事件發生（如發訊息） | 找出符合條件的 WebConn，逐一投遞 |
| `directMsg` | 點對點送訊息 | 送給特定單一 WebConn |
| `invalidateUser` | session 或權限變更 | 讓某 user 的所有 WebConn 清除 session cache |
| `activity` | 客戶端有操作 | 更新某 user 的最後活躍時間 |

---

## 連線索引結構

`hubConnectionIndex`（`web_hub.go:836`）同時維護三個 map，支援多種查詢模式：

```go
type hubConnectionIndex struct {
    byUserId      map[string]map[*WebConn]struct{}  // userID → 該 user 的所有連線
    byChannelID   map[string]map[*WebConn]struct{}  // channelID → 在這 channel 的所有連線
    byConnection  map[*WebConn][]string             // 所有連線（value 是該 conn 的 channel 列表）
    byConnectionId map[string]*WebConn              // connectionID → 單一連線
}
```

---

## 廣播邏輯

收到 `broadcast` 事件時，Hub 根據 `WebSocketEvent.Broadcast` 決定查哪個 index（`web_hub.go:744`）：

```
msg.Broadcast.ConnectionId != ""  →  ForConnection()   只送給單一連線
msg.Broadcast.UserId != ""        →  ForUser()         只送給特定 user 的所有連線
msg.Broadcast.ChannelId != ""     →  ForChannel()      送給 channel 所有在線成員
（以上皆無）                       →  All()             全廣播
```

每條連線送出前會執行兩個檢查：

1. **`webConn.ShouldSendEvent(msg)`**：驗證該連線是否有權收到此事件（session 有效、用戶有 channel 存取權等）
2. **`runBroadcastHooks()`**：執行個人化內容的過濾 hook（加入 mentions、followers、ABAC 檔案權限等）

```go
// web_hub.go:729
if webConn.ShouldSendEvent(msg) {
    select {
    case webConn.send <- h.runBroadcastHooks(msg, webConn, broadcastHooks, broadcastHookArgs):
    default:
        // send channel 滿了，關閉這條連線
        closeAndRemoveConn(connIndex, webConn)
    }
}
```

JSON 序列化只做一次（`msg.PrecomputeJSON()`），所有 WebConn 共用，避免重複序列化的開銷。

---

## 連線斷開的處理

當 `unregister` 被觸發時（`web_hub.go:607`）：

```
webConn.Active = false
↓
檢查該 user 是否還有其他 active 連線
├─ 有 → 更新 lastUserActivityAt，判斷是否設為 away
└─ 無 → 詢問 Cluster 其他節點是否還有連線
         ├─ 有 → 不動作
         └─ 無 → QueueSetStatusOffline()
```

這保證在 HA 部署下，用戶在其他節點仍有連線時不會被誤設為 offline。

---

## Hub 與 ClusterInterface 的邊界

```
業務事件（如發訊息）
        ↓
   ps.Publish(event)
        │
        ├─── [本節點] PublishSkipClusterSend()
        │         └─ GetHubForUserId() → Hub.Broadcast()
        │             └─ 投遞給本節點上的 WebConn
        │
        └─── [其他節點] ClusterInterface.SendClusterMessage()
                  └─ 傳到其他節點
                      └─ 各節點再走自己的 Hub.Broadcast()
```

Hub 只負責**本節點**的連線，跨節點由 ClusterInterface 處理後，再各自進入對方的 Hub。

---

## 關鍵檔案

| 檔案 | 內容 |
|---|---|
| `server/channels/app/platform/web_hub.go` | Hub struct、Start()、hubConnectionIndex |
| `server/channels/app/platform/web_conn.go` | WebConn struct、send/read pump |
| `server/channels/app/platform/cluster.go` | Publish()、PublishSkipClusterSend() |
| `server/channels/app/web_broadcast_hooks.go` | 各種 broadcast hook 實作 |
