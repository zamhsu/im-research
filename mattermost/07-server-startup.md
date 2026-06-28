# 伺服器啟動流程

## 程式入口

```
server/cmd/mattermost/main.go → main()
```

---

## 完整啟動流程

```
main.go:main()
    ↓
cmd/mattermost/commands/root.go — Cobra CLI 框架
    └─ 執行 "server" 子命令
    ↓
commands/server.go:serverCmdF()                    [line 39]
    ├─ 解析 CLI flags (--config, --disableconfigwatch...)
    ├─ 載入設定檔 (config.Store)
    └─ 呼叫 runServer()
    ↓
commands/server.go:runServer()                     [line 60]
    ├─ 設定 SIGINT/SIGTERM signal handler
    ├─ 載入 i18n 翻譯檔案
    └─ 初始化主 Server
    ↓
app/server.go:NewServer(options...)                [line 192]
    ├─ 建立 PlatformService（基礎設施）
    │      ├─ 初始化資料庫連線（PostgreSQL/MySQL）
    │      ├─ 執行 DB schema migration
    │      ├─ 初始化 memory cache
    │      ├─ 啟動 cluster 模組（HA 叢集支援）
    │      └─ 設定 Metrics 收集
    │
    ├─ 初始化各 App 模組：
    │      ├─ JobServer（背景工作排程）
    │      ├─ FileStore（本機/S3 檔案儲存）
    │      ├─ SearchEngine（Elasticsearch/Bleve）
    │      └─ EmailService（SMTP）
    │
    └─ 回傳 *Server 實例
    ↓
routes/api 初始化：
    api4/api.go:Init()
    ├─ InitUser()          → POST/GET /api/v4/users/...
    ├─ InitPost()          → POST/GET /api/v4/posts/...
    ├─ InitChannel()       → POST/GET /api/v4/channels/...
    ├─ InitTeam()          → POST/GET /api/v4/teams/...
    ├─ InitWebSocket()     → GET /api/v4/websocket
    ├─ InitFile()          → POST /api/v4/files/...
    ├─ InitOAuth()         → OAuth 端點
    ├─ InitPlugin()        → Plugin API 端點
    └─ ... (30+ Init 函式)
    ↓
wsapi/websocket_handler.go:Init()
    └─ 初始化 WebSocket 訊息路由（typing、status 等）
    ↓
web/web.go:New()
    └─ 設定靜態檔案服務（前端 build 產出）
    ↓
app/server.go:Start()                              [line 923]
    ├─ 啟動 HTTP/HTTPS 伺服器（gorilla/mux）
    │      └─ ListenAndServe on :8065 (預設)
    │
    ├─ 啟動 WebSocket Hub goroutines
    ├─ 啟動 Job Scheduler（定期工作）
    │      ├─ 資料清理
    │      ├─ Email Notification Batching
    │      ├─ Elasticsearch 索引
    │      └─ 其他背景工作
    │
    └─ 輸出啟動完成日誌
    ↓
等待 OS signal (SIGINT / SIGTERM)
    ↓
優雅關閉 (Graceful Shutdown)
    ├─ 停止接受新連線
    ├─ 等待進行中的請求完成
    ├─ 關閉 WebSocket 連線
    ├─ 關閉 Job Scheduler
    └─ 關閉資料庫連線
```

---

## 設定載入

設定檔支援多種來源（優先順序從高到低）：

```
1. 環境變數 (MM_SERVICESETTINGS_LISTENADDRESS 等)
2. config.json 檔案（預設路徑 config/config.json）
3. 資料庫（設定存在 DB 中，支援熱重載）
```

重要設定項目：

| 設定 | 說明 | 預設值 |
|------|------|--------|
| `ServiceSettings.ListenAddress` | 監聽埠 | `:8065` |
| `SqlSettings.DriverName` | 資料庫類型 | `postgres` |
| `SqlSettings.DataSource` | 連線字串 | - |
| `SqlSettings.MaxOpenConns` | 最大連線數 | `300` |
| `FileSettings.DriverName` | 檔案儲存類型 | `local` |
| `ClusterSettings.Enable` | 是否啟用 HA 叢集 | `false` |

---

## 插件（Plugin）載入

Server 啟動後，插件系統也會初始化：

```
app/plugin.go:InitPlugins()
    ├─ 掃描 plugins/ 目錄
    ├─ 載入已啟用的插件
    ├─ 啟動插件 RPC 進程（獨立子進程）
    └─ 插件透過 gRPC 與主進程通訊
```

---

## 叢集模式（High Availability）

啟用 `ClusterSettings.Enable = true` 時：

```
各節點互相發現（Gossip 協議）
    ↓
設定變更 / WebSocket 事件在節點間廣播
    ↓
共享同一個 PostgreSQL 資料庫
```

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/cmd/mattermost/main.go` | 程式入口 |
| `server/cmd/mattermost/commands/server.go:39` | `serverCmdF()` CLI 命令 |
| `server/cmd/mattermost/commands/server.go:60` | `runServer()` |
| `server/channels/app/server.go:192` | `NewServer()` 伺服器初始化 |
| `server/channels/app/server.go:923` | `Start()` 啟動 HTTP 伺服器 |
| `server/channels/api4/api.go` | API 路由初始化 |
| `server/channels/web/web.go` | 靜態檔案服務 |
| `server/channels/wsapi/websocket_handler.go` | WebSocket handler 初始化 |
