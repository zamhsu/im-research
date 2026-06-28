# Hub 架構與事件分發

## Hub 是什麼

Hub 是 WebSocket 連線的管理核心。每個 Hub 在獨立 goroutine 中運行，負責：
- 管理一批 WebSocket connections 的生命週期
- 接收廣播事件並分發給目標 connections
- 處理 connection 的註冊 / 登出

## Hub 數量與 User 分配

**檔案**：`server/channels/app/platform/web_hub.go:124-145`

```go
func (ps *PlatformService) hubStart(broadcastHooks map[string]BroadcastHook) {
    // Hub 數量 = CPU 核心數
    numberOfHubs := runtime.NumCPU()
    hubs := make([]*Hub, numberOfHubs)
    for i := range numberOfHubs {
        hubs[i] = newWebHub(ps)
        hubs[i].Start()
    }
}
```

User 被分配到哪個 Hub 由 hash 決定（`GetHubForUserId`），確保同一個 user 的所有 connections 都在同一個 Hub 中。

## Hub 結構

**檔案**：`server/channels/app/platform/web_hub.go:78-101`

```go
type Hub struct {
    connectionCount int64
    platform        *PlatformService
    connectionIndex int                          // 第幾個 Hub
    register        chan *webConnRegisterMessage  // 新 connection 加入
    unregister      chan *WebConn                // connection 離開
    broadcast       chan *model.WebSocketEvent   // 廣播事件（buffer 4096）
    stop            chan struct{}
    invalidateUser  chan string                  // 清除 user 快取
    broadcastHooks  map[string]BroadcastHook     // per-connection 處理 hooks
    // ...
}
```

## Publish() → Hub 路徑

**檔案**：`server/channels/app/platform/cluster.go:189-234`

```go
func (ps *PlatformService) Publish(message *model.WebSocketEvent) {
    // 1. Metrics 計數
    ps.metricsIFace.IncrementWebsocketEvent(message.EventType())

    // 2. 本機 Hub 分發
    ps.PublishSkipClusterSend(message)

    // 3. 叢集分發
    if ps.clusterIFace != nil {
        cm := &model.ClusterMessage{
            Event:    model.ClusterEventPublish,
            SendType: model.ClusterSendBestEffort,  // 預設盡力傳送
            Data:     data,
        }

        // posted / post_edited 等重要事件升級為 Reliable TCP
        if message.EventType() == model.WebsocketEventPosted ||
            message.EventType() == model.WebsocketEventPostEdited ||
            message.EventType() == model.WebsocketEventDirectAdded ||
            message.GetBroadcast().ReliableClusterSend {
            cm.SendType = model.ClusterSendReliable
        }
        ps.clusterIFace.SendClusterMessage(cm)
    }
}
```

```go
func (ps *PlatformService) PublishSkipClusterSend(event *model.WebSocketEvent) {
    if event.GetBroadcast().UserId != "" {
        // 只發給特定 user → 找到他的 Hub，只廣播給那個 Hub
        hub := ps.GetHubForUserId(event.GetBroadcast().UserId)
        hub.Broadcast(event)
    } else {
        // 頻道/team 範圍 → 廣播給所有 Hub（各自再過濾）
        for _, hub := range ps.hubs {
            hub.Broadcast(event)
        }
    }

    // 通知 Shared Channel 同步服務
    ps.SharedChannelSyncHandler(event)
}
```

## Hub.Broadcast()

**檔案**：`server/channels/app/platform/web_hub.go:442-460`

```go
func (h *Hub) Broadcast(message *model.WebSocketEvent) {
    if h == nil || message == nil {
        return
    }
    if m := h.platform.metricsIFace; m != nil {
        m.IncrementWebSocketBroadcastBufferSize(strconv.Itoa(h.connectionIndex), 1)
    }
    select {
    case h.broadcast <- message:   // 非阻塞放入 buffer channel
    case <-h.stop:
    }
}
```

**注意**：`h.broadcast` channel 的 buffer 大小為 4096（`web_hub.go:24` 的常數）。若滿了，訊息會被丟棄（select 中沒有 default，但有 `<-h.stop` 作為退出路徑）。

## Hub Main Loop：事件分發

**檔案**：`server/channels/app/platform/web_hub.go:715-774`

```go
case msg := <-h.broadcast:
    // 移除 hook 資訊（不納入 precomputed JSON）
    msg, broadcastHooks, broadcastHookArgs := msg.WithoutBroadcastHooks()

    // ★ 只序列化一次 JSON，所有收件人共用
    msg = msg.PrecomputeJSON()

    broadcast := func(webConn *WebConn) {
        if !connIndex.Has(webConn) { return }
        if webConn.ShouldSendEvent(msg) {  // 權限過濾
            select {
            case webConn.send <- h.runBroadcastHooks(msg, webConn, broadcastHooks, broadcastHookArgs):
            default:
                // send channel 滿了 → 關閉這個 connection
                closeAndRemoveConn(connIndex, webConn)
            }
        }
    }

    // 路由選擇（優先順序由上往下）
    if webConn := connIndex.ForConnection(msg.GetBroadcast().ConnectionId); webConn != nil {
        broadcast(webConn)  // 特定 connection
        continue
    }

    fastIteration := *h.platform.Config().ServiceSettings.EnableWebHubChannelIteration
    var targetConns iter.Seq[*WebConn]

    if userID := msg.GetBroadcast().UserId; userID != "" {
        targetConns = connIndex.ForUser(userID)  // 特定 user 的所有 connections
    } else if channelID := msg.GetBroadcast().ChannelId; channelID != "" && fastIteration {
        targetConns = connIndex.ForChannel(channelID)  // 頻道成員（快速路徑）
    }

    if targetConns != nil {
        for webConn := range targetConns {
            broadcast(webConn)
        }
        continue
    }

    // 若 fast iteration 開啟但沒找到頻道成員，跳過（其他 Hub 會處理）
    if channelID := msg.GetBroadcast().ChannelId; channelID != "" && fastIteration {
        continue
    }

    // Fallback：遍歷所有 connections，由 ShouldSendEvent 做個別過濾
    for webConn := range connIndex.All() {
        broadcast(webConn)
    }
```

## 事件路由優先順序

| 優先序 | 條件 | 行為 |
|--------|------|------|
| 1 | `ConnectionId != ""` | 只發給該 connection |
| 2 | `UserId != ""` | 發給該 user 的所有 connections |
| 3 | `ChannelId != ""` + fast iteration 開啟 | 發給頻道成員的 connections |
| 4 | Fallback | 發給 Hub 內所有 connections（由 `ShouldSendEvent` 過濾）|

## 效能設計重點

- **PrecomputeJSON()**：JSON 只序列化一次，所有收件人共用同一份 bytes，避免重複計算
- **CPU-count Hubs**：減少鎖競爭，每個 Hub 獨立處理自己的 connections
- **channel-based routing**（`EnableWebHubChannelIteration`）：可選功能，讓 Hub 直接索引頻道成員，避免遍歷所有 connections
