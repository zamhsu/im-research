# Hub 循序圖

## 1. 客戶端連線建立（register）

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant M as WebSocket Middleware
    participant HR as HubRouter
    participant H as HubShard<br/>(hash(userID) % NumCPU)
    participant CI as ConnectionIndex

    C->>M: WebSocket 握手
    M->>M: 驗證 session / userId
    M->>HR: HubRegister(webConn)
    HR->>HR: hash(userID) % NumCPU → 選 HubShard
    HR->>H: register channel ← webConn
    H->>CI: connIndex.Add(webConn)
    CI-->>H: ok
    H->>C: send ← HelloMessage(connectionId)
    H-->>HR: err = nil
    HR-->>M: ok
    Note over M,C: ReadPump / WritePump 開始運行
```

---

## 2. 廣播事件（broadcast）— 以發送訊息為例

```mermaid
sequenceDiagram
    participant App as App Layer<br/>(SendNotifications)
    participant PS as PlatformService<br/>Publish()
    participant HR as HubRouter
    participant H as HubShard
    participant CI as ConnectionIndex
    participant WC as WebConn<br/>(send channel)
    participant C as 客戶端

    App->>PS: Publish(WebSocketEvent{posted})
    PS->>HR: PublishSkipClusterSend(event)

    alt event.Broadcast.UserId != ""
        HR->>H: GetHubForUserId(userId)
        H->>CI: ForUser(userId)
    else event.Broadcast.ChannelId != ""
        HR->>H: 所有 HubShard
        H->>CI: ForChannel(channelId)
    else 全廣播
        HR->>H: 所有 HubShard
        H->>CI: All()
    end

    CI-->>H: []WebConn

    H->>H: msg.PrecomputeJSON()<br/>（只序列化一次）

    loop 每個符合條件的 WebConn
        H->>WC: ShouldSendEvent(msg)?
        alt 有權限
            H->>H: runBroadcastHooks(msg, webConn)<br/>加入 mentions / followers / ABAC
            H->>WC: send channel ← finalMsg
            WC->>C: WebSocket frame
        else 無權限
            H->>H: 跳過此連線
        end
    end

    PS->>PS: ClusterInterface.SendClusterMessage()<br/>（廣播給其他節點）
```

---

## 3. 廣播佇列滿（send channel full）

```mermaid
sequenceDiagram
    participant H as HubShard
    participant WC as WebConn<br/>(send buffer 256)
    participant CI as ConnectionIndex
    participant C as 客戶端

    H->>WC: send channel ← msg（non-blocking）
    alt send channel 未滿
        WC->>C: WritePump 寫出 WebSocket frame
    else send channel 已滿
        H->>H: 記錄警告 log
        H->>WC: close(send)
        H->>CI: connIndex.Remove(webConn)
        Note over WC,C: WritePump 偵測到 channel 關閉<br/>WebSocket 連線終止
    end
```

---

## 4. 客戶端斷線（unregister）

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant WC as WebConn
    participant M as Middleware
    participant HR as HubRouter
    participant H as HubShard
    participant CI as ConnectionIndex
    participant CL as ClusterInterface

    C->>WC: WebSocket 關閉 / 網路中斷
    WC->>M: ReadPump / WritePump 結束
    M->>HR: HubUnregister(webConn)
    HR->>H: unregister channel ← webConn
    H->>WC: Active = false
    H->>CI: connIndex.Remove(webConn)

    H->>CI: ForUser(userId) — 還有其他連線？

    alt 該 user 還有其他 active 連線
        H->>H: 更新 lastUserActivityAt
        H->>H: isUserAway? → SetStatusLastActivityAt
    else 本節點已無此 user 的連線
        H->>CL: Cluster.WebConnCountForUser(userId)<br/>詢問其他節點
        alt 其他節點仍有連線
            H->>H: 不動作（user 仍在線）
        else 所有節點都沒有連線
            H->>H: QueueSetStatusOffline(userId)
        end
    end
```

---

## 5. Session 失效（invalidateUser）

```mermaid
sequenceDiagram
    participant App as App Layer
    participant PS as PlatformService
    participant CL as ClusterInterface
    participant H as HubShard
    participant CI as ConnectionIndex
    participant WC as WebConn

    App->>PS: InvalidateCacheForUser(userId)
    PS->>PS: invalidateWebConnSessionCacheForUser(userId)
    PS->>H: invalidateUser channel ← userId

    H->>CI: ForUser(userId)
    CI-->>H: []WebConn

    loop 每個 WebConn
        H->>WC: webConn.InvalidateCache()<br/>清除 session token / 權限快取
    end

    PS->>CL: SendClusterMessage<br/>(ClusterEventInvalidateWebConnCacheForUser)
    Note over CL: 其他節點收到後<br/>同樣對各自的 Hub 執行 invalidateUser
```

---

## 6. 跨節點廣播（叢集架構）

```mermaid
sequenceDiagram
    participant App as Node A<br/>App Layer
    participant PS_A as Node A<br/>PlatformService
    participant H_A as Node A<br/>HubShard
    participant CL as ClusterInterface<br/>(Gossip / NATS)
    participant PS_B as Node B<br/>PlatformService
    participant H_B as Node B<br/>HubShard
    participant WC_B as Node B<br/>WebConn（目標用戶）

    App->>PS_A: Publish(WebSocketEvent)

    par 本節點投遞
        PS_A->>H_A: PublishSkipClusterSend(event)
        H_A->>H_A: 投遞給本節點連線
    and 跨節點傳播
        PS_A->>CL: SendClusterMessage(event)
        CL->>PS_B: ClusterPublishHandler(msg)
        PS_B->>H_B: PublishSkipClusterSend(event)
        H_B->>WC_B: send channel ← msg
        WC_B->>WC_B: WritePump → WebSocket frame
    end
```

---

## 7. 完整生命週期總覽

```mermaid
sequenceDiagram
    participant C as 客戶端
    participant MW as Middleware
    participant H as HubShard
    participant CI as ConnectionIndex
    participant WC as WebConn
    participant App as App Layer
    participant CL as Cluster

    Note over C,CL: 連線建立
    C->>MW: WebSocket connect
    MW->>WC: new WebSocketConnection
    MW->>H: register(webConn)
    H->>CI: Add(webConn)
    H->>WC: send ← HelloMessage
    WC->>C: HelloMessage frame

    Note over C,CL: 正常運作期間
    C->>WC: 客戶端訊息（ReadPump）
    WC->>App: route to handler
    App->>H: Broadcast(WebSocketEvent)
    H->>CI: 查詢目標連線
    H->>WC: send ← event（含 broadcast hooks）
    WC->>C: WebSocket frame（WritePump）

    Note over C,CL: 斷線處理
    C->>WC: 連線中斷
    WC->>H: unregister(webConn)
    H->>CI: Remove(webConn)
    H->>CL: WebConnCountForUser？
    CL-->>H: 0（全部節點都離線）
    H->>App: QueueSetStatusOffline(userId)
```
