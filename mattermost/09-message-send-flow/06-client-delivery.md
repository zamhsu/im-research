# 最終寫入客戶端 WebSocket

## WebConn 結構

每個 WebSocket 連線對應一個 `WebConn`，擁有獨立的 `send` channel 和 goroutines：

```
WebConn
├── send channel (buffered)    ← Hub 寫入事件到這裡
├── readPump goroutine         ← 讀取客戶端訊息
├── writePump goroutine        ← 從 send 讀取並寫出 WebSocket
├── Sequence counter           ← 事件序號，供重連對齊
└── deadQueue [128]ring buffer ← 斷線重連補發緩衝
```

## writePump() 核心邏輯

**檔案**：`server/channels/app/platform/web_conn.go:516-639`

```go
func (wc *WebConn) writePump() {
    // 2KB 預分配 buffer（涵蓋 98.5% 的訊息大小）
    var buf bytes.Buffer
    buf.Grow(1024 * 2)
    enc := json.NewEncoder(&buf)

    for {
        select {
        case msg, ok := <-wc.send:
            if !ok {
                // channel 已關閉 → 送 WebSocket Close frame
                wc.writeMessageBuf(websocket.CloseMessage, []byte{})
                return
            }

            evt, evtOk := msg.(*model.WebSocketEvent)

            // 被 BroadcastHook 拒絕的事件直接跳過
            if evtOk && evt.IsRejected() {
                continue
            }

            buf.Reset()
            if evtOk {
                // ★ 加上序號（每個 connection 獨立遞增）
                evt = evt.SetSequence(wc.Sequence)
                err = evt.Encode(enc, &buf)
                wc.Sequence++
            } else {
                err = enc.Encode(msg)
            }

            // 佇列警告（send channel 超過閾值時 log）
            if wc.Active.Load() && len(wc.send) >= sendFullWarn {
                wc.Platform.logger.Warn("websocket.full", ...)
            }

            // ★ 存入 dead queue（支援斷線後重傳）
            if evtOk {
                wc.addToDeadQueue(evt)
            }

            // ★ 實際寫出 WebSocket frame
            if err := wc.writeMessageBuf(websocket.TextMessage, buf.Bytes()); err != nil {
                wc.logSocketErr("websocket.send", err)
                return  // 寫入失敗 → 結束 pump，觸發 connection 關閉
            }

            // Metrics 計數
            m.IncrementWebSocketBroadcast(msg.EventType())

        case <-ticker.C:
            // 定期 Ping（keepalive）
            wc.writeMessageBuf(websocket.PingMessage, []byte{})

        case <-wc.endWritePump:
            return
        }
    }
}
```

## writeMessageBuf()

**檔案**：`server/channels/app/platform/web_conn.go:643-647`

```go
func (wc *WebConn) writeMessageBuf(msgType int, data []byte) error {
    // 設定寫入 deadline（30 秒，超時視為連線問題）
    wc.WebSocket.SetWriteDeadline(time.Now().Add(writeWaitTime))
    return wc.WebSocket.WriteMessage(msgType, data)
}
```

## 序號機制（Sequence）

每個 `WebSocketEvent` 發送給客戶端前，都會加上 per-connection 的遞增序號：

```
Client A 斷線重連時，告訴 server：「我最後收到序號 42」
Server 從 dead queue 找到 seq 43 以後的事件，重新傳送
```

這讓短暫斷線後不需要完整刷新頁面，直接補發遺失的事件。

## Dead Queue（斷線緩衝）

**相關常數**（`web_conn.go:42`）：
- 大小：128 個事件的環形 buffer
- 只存 `*model.WebSocketEvent`（非 WebSocket response 類訊息）

當重連時，`writeMessageBuf` 之前已經把事件存入 dead queue，所以即使寫入 WebSocket 失敗，事件仍然在 dead queue 中，等下次重連補發。

## 連線過慢保護

**檔案**：`server/channels/app/platform/web_conn.go:893-909`

當 `send` channel 接近滿時，低優先級事件會被直接丟棄（在 `ShouldSendEvent` 中）：

```go
if len(wc.send) >= sendSlowWarn {
    switch msg.EventType() {
    case model.WebsocketEventTyping,
         model.WebsocketEventStatusChange,
         model.WebsocketEventMultipleChannelsViewed:
        return false  // 直接丟棄非重要事件
    }
}
```

`"posted"` 事件**不在**此清單，因此即使連線慢，也會繼續嘗試送出。若 `send` channel 完全滿（`Hub.broadcast` 中的 `default` 分支），Hub 會關閉這個 connection（`closeAndRemoveConn`）。

## 完整的 send channel 容量鏈

```
Hub.broadcast channel  [4096]  ← Publish() 放入
       ↓
WebConn.send channel   [?]     ← Hub main loop 放入
       ↓
WebSocket write        (無 buffer) ← writePump() 寫出
```
