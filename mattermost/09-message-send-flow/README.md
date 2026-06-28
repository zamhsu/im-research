# Mattermost 訊息發送流程分析

## 概覽

本資料夾分析 Mattermost server 中從客戶端發送訊息到其他客戶端接收事件的完整流程。

## 文件列表

| 文件 | 說明 |
|------|------|
| [01-flow-overview.md](./01-flow-overview.md) | 整體流程概覽與時序圖 |
| [02-api-layer.md](./02-api-layer.md) | API 入口層：HTTP handler |
| [03-app-layer.md](./03-app-layer.md) | App 層：CreatePost 邏輯 |
| [04-event-system.md](./04-event-system.md) | WebSocket 事件的建立與發布 |
| [05-hub-architecture.md](./05-hub-architecture.md) | Hub 架構與事件分發 |
| [06-client-delivery.md](./06-client-delivery.md) | 最終寫入客戶端 WebSocket |
| [07-permission-check.md](./07-permission-check.md) | ShouldSendEvent 權限過濾 |

## 快速參考：關鍵呼叫路徑

```
HTTP POST /api/v4/posts
  → createPost()                              api4/post.go:96
  → app.CreatePostAsUser()                    app/post.go:39
  → app.CreatePost()                          app/post.go:161
      → Store().Post().Save()                 (DB 寫入)
      → app.handlePostEvents()                app/post.go:466
          → app.SendNotifications()           app/notification.go:54
              → NewWebSocketEvent("posted")   app/notification.go:689
              → publishWebsocketEventForPost  app/post.go:1010
                  → ps.Publish()             platform/cluster.go:189
                      → hub.Broadcast()      platform/web_hub.go:442
                          → ShouldSendEvent  platform/web_conn.go:884
                          → webConn.send ←   platform/web_hub.go:731
                              → writePump()  platform/web_conn.go:565
                                  → WebSocket.WriteMessage()
```
