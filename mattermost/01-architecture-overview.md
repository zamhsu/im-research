# 整體架構概覽

## 分層架構

Mattermost 後端採用嚴格的三層架構，每層職責明確：

```
HTTP 請求
    ↓
┌─────────────────────────────────────┐
│         API Layer (api4/)           │  路由、輸入驗證、權限檢查
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│         App Layer (app/)            │  業務邏輯、資料轉換、事件發布
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│      Store Layer (store/sqlstore/)  │  SQL 查詢、資料持久化
└─────────────────────────────────────┘
    ↓
  資料庫 (PostgreSQL / MySQL)
```

---

## 各層詳細說明

### Layer 1：API 層 (`server/channels/api4/`)

負責 HTTP 請求的入口處理：

- 使用 **Gorilla Mux** 做路由
- 所有端點掛在 `/api/v4/` 路徑下
- 每個功能模組有獨立的路由初始化函式：
  - `InitUser()` → `api4/user.go`
  - `InitPost()` → `api4/post.go`
  - `InitChannel()` → `api4/channel.go`
  - `InitWebSocket()` → `api4/websocket.go`
- Middleware 堆疊：認證 → CSRF 驗證 → 速率限制 → 稽核日誌
- 解析完請求後委派給 App 層執行業務邏輯

### Layer 2：App 層 (`server/channels/app/`)

業務邏輯的核心：

- 實作具體的業務規則與資料驗證
- 透過 Store 介面讀寫資料庫
- 發布 WebSocket 事件通知前端
- 代表性檔案：
  - `app/app.go` — App struct 定義
  - `app/login.go` — 登入流程
  - `app/post.go` — 訊息操作
  - `app/channel.go` — 頻道操作
  - `app/authentication.go` — 密碼/MFA 驗證

### Layer 3：Store 層 (`server/channels/store/`)

資料持久化抽象層：

- `store/store.go` — 定義全部 Store 介面（`PostStore`、`UserStore`、`ChannelStore` 等）
- `store/sqlstore/` — 介面的 SQL 實作（PostgreSQL、MySQL）
- 使用 **Squirrel** 做 SQL 查詢建構
- 使用 **sqlx** 做連線管理
- 支援 Master（寫）/ Replica（讀）分離

### Platform 層 (`server/channels/app/platform/`)

跨服務的基礎設施，不屬於特定業務模組：

- 設定管理
- WebSocket Hub 與廣播
- 快取層
- 授權服務
- 叢集協調
- 儲存服務（檔案上傳）

---

## HTTP 請求完整流程

```
HTTP Request (e.g. POST /api/v4/posts)
    ↓
Gorilla Mux Router
    ↓
api4/api.go — 找到對應路由
    ↓
Middleware 堆疊：
    ├─ APISessionRequired() — 從 header/cookie 提取 session token
    ├─ CSRF 驗證
    ├─ Rate Limiting
    └─ Audit Logging
    ↓
Handler (e.g. createPost in api4/post.go)
    ├─ 解析 JSON body
    ├─ 輸入驗證
    ├─ 權限檢查 (c.App.HasPermissionTo...)
    └─ 呼叫 App 層方法
    ↓
App Layer (e.g. app.CreatePostAsUser in app/post.go)
    ├─ 業務規則驗證
    ├─ 呼叫 Store 方法
    └─ 發布 WebSocket 事件
    ↓
Store Layer (e.g. sqlstore/post_store.go)
    ├─ 建構 SQL 查詢
    ├─ 執行 INSERT/UPDATE
    └─ 返回結果
    ↓
JSON Response → 客戶端
```

---

## 關鍵架構模式

| 模式 | 說明 |
|------|------|
| 介面驅動的 Store | `store.go` 定義介面，`sqlstore/` 實作，方便 mock 測試 |
| Dependency Injection | `Option` 模式注入設定與服務 |
| Context 傳播 | `request.CTX` 貫穿所有層，攜帶 logger、trace ID |
| Master-Replica DB | 寫操作用 Master，讀操作用 Replica |
| WebSocket Hub | 廣播機制，不耦合在業務邏輯中 |
| `model.AppError` | 統一的錯誤類型，包含 HTTP status code |
