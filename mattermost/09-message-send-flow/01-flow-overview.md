# 整體流程概覽

## 高層架構

Mattermost 的訊息發送流程跨越四個主要層次：

```
┌─────────────┐     HTTP POST      ┌──────────────────────────────────────┐
│   Client A  │ ─────────────────► │  API Layer (api4/post.go)            │
│  (sender)   │                    │  createPost() handler                │
└─────────────┘                    └─────────────────┬────────────────────┘
                                                     │
                                                     ▼
                                   ┌──────────────────────────────────────┐
                                   │  App Layer (app/post.go)             │
                                   │  CreatePost() → DB Save → Events     │
                                   └─────────────────┬────────────────────┘
                                                     │
                                                     ▼
                                   ┌──────────────────────────────────────┐
                                   │  Platform Layer                      │
                                   │  Publish() → Hub.Broadcast()         │
                                   └─────────────────┬────────────────────┘
                                                     │
                                      ┌──────────────┴──────────────┐
                                      ▼                             ▼
                              ┌───────────────┐           ┌───────────────┐
                              │  Hub 0        │           │  Hub N        │
                              │ (CPU 0 conn.) │    ...    │ (CPU N conn.) │
                              └───────┬───────┘           └───────────────┘
                                      │
                          ┌───────────┴───────────┐
                          ▼           ▼            ▼
                    ┌──────────┐ ┌──────────┐ ┌──────────┐
                    │ WebConn  │ │ WebConn  │ │ WebConn  │
                    │ Client B │ │ Client C │ │ Client D │
                    └──────────┘ └──────────┘ └──────────┘
```

## 完整時序

```
時間軸
  │
  │  1. Client A 送出 HTTP POST /api/v4/posts
  │
  │  2. createPost() 驗證 session 與權限
  │
  │  3. CreatePostAsUser() → CreatePost()
  │
  │  4. ★ DB 寫入 (Store().Post().Save())
  │      └→ 訊息持久化完成
  │
  │  5. PreparePostForClient() 組裝 metadata
  │
  │  6. handlePostEvents()
  │
  │  7. SendNotifications()
  │      ├→ 計算 push / desktop 通知
  │      └→ NewWebSocketEvent("posted", channelId=...)
  │
  │  8. publishWebsocketEventForPost()
  │      └→ 將 post JSON 加入 event payload
  │
  │  9. ps.Publish()
  │      ├→ IncrementWebsocketEvent (metrics)
  │      ├→ PublishSkipClusterSend()  (本機 Hub)
  │      └→ clusterIFace.SendClusterMessage() (叢集其他節點，Reliable TCP)
  │
  │  10. hub.Broadcast()
  │       └→ 放入 hub.broadcast channel (buffer 4096)
  │
  │  11. Hub main loop 接收 msg
  │       ├→ PrecomputeJSON() (只序列化一次)
  │       ├→ 根據 broadcast scope 選擇目標 connections
  │       │    ├ ConnectionId → 單一 connection
  │       │    ├ UserId       → 該 user 的所有 connections
  │       │    ├ ChannelId    → 頻道成員 connections (fast path)
  │       │    └ (無)         → 所有 connections
  │       └→ 對每個 webConn:
  │            ├ ShouldSendEvent() 檢查
  │            └ webConn.send <- msg
  │
  │  12. writePump() goroutine (每個 connection 獨立)
  │       ├→ SetSequence() 加上序號
  │       ├→ Encode() 序列化
  │       ├→ addToDeadQueue() (斷線重連緩衝)
  │       └→ WebSocket.WriteMessage() 寫出
  │
  ▼  Client B/C/D 接收 "posted" 事件
```

## 關鍵時機：DB 寫入 vs 事件發送

**DB 寫入優先**：`Store().Post().Save()` 在 `handlePostEvents()` 之前完成（`post.go:357` vs `post.go:466`）。

這意味著：
- 事件發送失敗不影響訊息持久化
- 客戶端刷新後一定能看到訊息（即使 WebSocket 事件遺失）
- Dead Queue 機制作為補充，支援斷線重連後補發

## 叢集行為

對於 `WebsocketEventPosted`、`WebsocketEventPostEdited` 等重要事件，叢集傳遞使用 **Reliable TCP** 而非 Best Effort UDP（`cluster.go:207-214`），確保多節點環境下事件不遺失。
