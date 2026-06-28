# 用 NATS 取代 Mattermost 通知系統的可行性分析

---

## 結論先說

**可以取代，但只有一部分，且有明確的邊界。**

Mattermost 的通知廣播系統分成兩層，NATS 對這兩層的適用性完全不同：

| 層級 | 元件 | 能否用 NATS 取代？ | 說明 |
|------|------|-------------------|------|
| **節點內** | Hub（Go channel 廣播） | ❌ 不該取代 | 零網路延遲，替換無益 |
| **跨節點** | ClusterInterface | ✅ 可以取代 | 目前是可插拔介面，NATS 完全適合 |

---

## 現有架構回顧

```
訊息發送
   ↓
ps.Publish(WebSocketEvent)
   ├─ [本機] PublishSkipClusterSend()
   │     └─ Hub.Broadcast()  ← Go channel，不走網路
   │         └─ WebConn.send → WebSocket frame
   │
   └─ [跨節點] ClusterInterface.SendClusterMessage()
         └─ 目前由企業版 Gossip 實作
             └─ 其他節點收到後再走各自的 Hub
```

### Hub 是什麼？

- **每個 CPU core 一個 Hub goroutine**，透過 `userID hash` 決定用哪個 Hub
- Hub 內部維護所有 WebSocket 連線的 index（按 userId / channelId 索引）
- 廣播時直接 `for loop` 遍歷符合條件的 WebConn，送入其 `send channel`
- **完全在記憶體內，沒有網路 I/O**

### ClusterInterface 是什麼？

定義在 `server/einterfaces/cluster.go`，是一個 **可插拔介面**：

```go
type ClusterInterface interface {
    SendClusterMessage(msg *model.ClusterMessage)
    SendClusterMessageToNode(nodeID string, msg *model.ClusterMessage) error
    RegisterClusterMessageHandler(event model.ClusterEvent, crm ClusterMessageHandler)
    IsLeader() bool
    GetClusterInfos() ([]*model.ClusterInfo, error)
    // ... 共 16 個方法
}
```

**目前由企業版的 Gossip 協議實作**，開源版單節點時此介面為 nil。

---

## NATS 可以取代哪裡？

### 替換目標：ClusterInterface

NATS 完全具備取代 Gossip 叢集實作的能力，對應關係如下：

#### 訊息廣播（49 種 ClusterEvent）

NATS 可用 **Subject 層次結構** 對應所有事件類型：

```
mattermost.cluster.publish                    → WebSocket 事件轉發
mattermost.cluster.cache.channel.<channelId>  → Channel cache 失效
mattermost.cluster.cache.user.<userId>        → User cache 失效
mattermost.cluster.cache.roles                → Roles cache 失效
mattermost.cluster.status.update             → 用戶狀態更新
mattermost.cluster.session.clear.<userId>    → Session cache 清除
mattermost.cluster.config.changed            → 設定變更通知
mattermost.cluster.plugin.<pluginId>         → Plugin 事件
```

每個節點訂閱這些 subject，收到後呼叫對應的 handler（目前由 `RegisterClusterMessageHandler` 登記的同一批 handler）。

#### Request-Reply（Gossip 查詢類操作）

目前這些方法走 Gossip 的 request/response：
- `GetLogs()` / `QueryLogs()`
- `GetClusterStats()`
- `GetPluginStatuses()`
- `WebConnCountForUser()`
- `GetWSQueues()`

NATS **原生支援 Request-Reply**，一行就能實作：

```go
// 查詢特定 userID 的 WebConn 數量
resp, err := nc.Request("mattermost.cluster.query.webconn_count", userIDBytes, 2*time.Second)
```

#### Leader Election

NATS JetStream 的 **Raft-based leader election** 可直接取代 Gossip 的 `IsLeader()` 機制。

#### 節點成員發現

NATS 本身就有內建的叢集拓樸管理，可取代 `GetClusterInfos()`。

---

## NATS 不適合取代哪裡？

### 不該替換：Hub 內部的 Go channel

**原因很簡單：Hub 的設計目標是零延遲本機廣播。**

如果把 Hub 改成每條訊息都透過 NATS（即使是 localhost）：

| 指標 | 現有 Go channel | 替換為 NATS |
|------|----------------|-------------|
| 延遲 | ~100ns（記憶體操作） | ~1-5ms（TCP loopback） |
| 吞吐 | 受 CPU memory bandwidth 限制 | 受 NATS server 容量限制 |
| 連線過濾 | Hub 直接遍歷記憶體索引 | 需要 NATS subject per channel/user |
| Broadcast Hooks | 在 Hub goroutine 內執行 | 需要額外機制 |
| Dead Queue | WebConn 記憶體內保存 | 需要 JetStream consumer |

**結論：Hub 的本機廣播用 Go channel 是最優解，NATS 無法帶來好處，只有壞處。**

---

## 完整替換後的架構設計

如果用 NATS 取代 ClusterInterface，架構會變成：

```
多節點部署

Node A                         Node B                         Node C
──────                         ──────                         ──────
App.Publish(event)             App.Publish(event)             App.Publish(event)
  │                              │                              │
  ├─[本機] Hub broadcast          ├─[本機] Hub broadcast          ├─[本機] Hub broadcast
  │   └─ 本節點的 WebConn          │   └─ 本節點的 WebConn          │   └─ 本節點的 WebConn
  │                              │                              │
  └─[跨節點] NATSCluster.Publish   │                              │
       │                         │                              │
       ▼                         ▼                              ▼
   ┌────────────────────────────────────────────────────────────┐
   │                   NATS JetStream Cluster                   │
   │  Subject: mattermost.cluster.publish                       │
   │  Subject: mattermost.cluster.cache.*                       │
   │  Subject: mattermost.cluster.status.*                      │
   └────────────────────────────────────────────────────────────┘
       │                         │                              │
       ▼                         ▼                              ▼
  Node A 收到          Node B 收到                 Node C 收到
  (自己忽略)            Hub.Broadcast()              Hub.Broadcast()
                       本節點 WebConn               本節點 WebConn
```

### NATS ClusterInterface 實作骨架

```go
type NATSCluster struct {
    nc   *nats.Conn
    js   nats.JetStreamContext
    handlers map[model.ClusterEvent]einterfaces.ClusterMessageHandler
    nodeID   string
}

func (c *NATSCluster) SendClusterMessage(msg *model.ClusterMessage) {
    subject := clusterEventToSubject(msg.Event)
    data, _ := json.Marshal(msg)
    
    if msg.SendType == model.ClusterSendReliable {
        // JetStream：保證投遞，有 ACK
        c.js.PublishAsync(subject, data)
    } else {
        // NATS Core：best effort，低延遲
        c.nc.Publish(subject, data)
    }
}

func (c *NATSCluster) RegisterClusterMessageHandler(
    event model.ClusterEvent, handler einterfaces.ClusterMessageHandler,
) {
    subject := clusterEventToSubject(event)
    c.nc.QueueSubscribe(subject, "mattermost-cluster", func(msg *nats.Msg) {
        var cm model.ClusterMessage
        json.Unmarshal(msg.Data, &cm)
        handler(&cm)
    })
}
```

---

## NATS Core vs JetStream 的選擇

| 事件類型 | 使用 NATS Core 或 JetStream | 原因 |
|---------|--------------------------|------|
| WebSocket 事件轉發（posted、edited） | **JetStream** | 訊息丟失影響用戶體驗 |
| Cache 失效 | **NATS Core** | 冪等操作，丟失可接受（下次讀取自動重查） |
| 用戶狀態更新 | **NATS Core** | 高頻、短暫，最終一致即可 |
| Session 清除 | **JetStream** | 安全相關，必須保證送達 |
| 設定變更 | **JetStream** | 影響全節點行為 |
| Plugin 事件 | 依需求而定 | 由 plugin 決定重要性 |

這對應現有的 `ClusterSendReliable` vs `ClusterSendBestEffort` 語意，可以直接對應。

---

## 使用 NATS 的優缺點

### 優點

1. **去除企業版依賴**：目前 ClusterInterface 的 Gossip 實作只在企業版，NATS 實作可以讓開源版也有多節點能力
2. **更好的可觀測性**：NATS 提供原生的 monitoring endpoint、message tracing、consumer lag 監控
3. **水平擴展**：NATS JetStream cluster 本身支援 horizontal scaling
4. **訊息持久化**：JetStream 可保存訊息，新節點加入時可 replay 錯過的事件
5. **Subject 路由**：比 Gossip 的全廣播更精準，可以只訂閱需要的事件類型
6. **Request-Reply 原生支援**：不需要自行實作 gossip 的 query/response 機制

### 缺點與風險

1. **引入外部依賴**：NATS server 成為 critical path，需要額外的 HA 配置
2. **Broadcast Hooks 問題**：目前 Hub 在投遞前執行 per-user 的 broadcast hooks（mentions、followers、ABAC 等），這些邏輯仍然需要在節點本機執行，NATS 無法幫忙
3. **Dead Queue 機制要額外設計**：目前 WebConn 有 128 筆的 dead queue 供斷線重連用，改用 NATS 後斷線重連的訊息恢復需要靠 JetStream consumer seek
4. **序列化開銷**：ClusterMessage 目前已經是序列化過的，送到 NATS 前不需要多一層，但 subject routing 的設計需要仔細規劃
5. **Cluster 節點發現的整合**：需要讓 NATS 的節點成員資訊和 Mattermost 的 `GetClusterInfos()` 保持同步

---

## 實作路徑建議

如果要做，建議分三個階段：

### Phase 1：替換 ClusterInterface（風險最低）

只替換跨節點的 Gossip 層，Hub 完全不動。

```
實作 NATSCluster struct，實作 ClusterInterface 的 16 個方法
└─ 用 RegisterClusterInterface() 注入
```

影響範圍：`server/einterfaces/cluster.go` 的所有實作方

### Phase 2：Cache 失效改用 NATS（選擇性）

把 `localcachelayer` 中約 40 種 cache invalidation 事件的 SendClusterMessage 呼叫，改走 NATS 的輕量級 pub/sub。

### Phase 3：加入訊息持久化（可選）

對 `ClusterEventPublish`（WebSocket 事件轉發）啟用 JetStream，讓節點短暫斷線重連後可以 replay 錯過的事件。

---

## 不需要改動的部分

以下這些完全不受影響：

- `Hub` 及其 broadcast 迴圈
- `WebConn` 及其 send/read pump
- `SendNotifications()` 的邏輯
- Broadcast Hooks（`addMentionsBroadcastHook` 等）
- Push Notification Hub（走 HTTP 到 Push Proxy，與此無關）
- Email 通知

---

## 總結

```
可以替換：ClusterInterface（跨節點 Gossip）
不應替換：Hub（節點內 Go channel 廣播）

替換後的好處：
  - 開源版也能做多節點
  - 更好的可觀測性與持久化
  - NATS Request-Reply 簡化 gossip 查詢

替換的代價：
  - 引入 NATS server 作為基礎設施依賴
  - 需要實作完整的 ClusterInterface（16 個方法）
  - Dead queue 重連恢復需要額外設計
```
