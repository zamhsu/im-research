# 資料庫層設計

## 架構概覽

```
App Layer
    ↓ 使用介面
store/store.go — Store 介面定義（全部 Sub-store）
    ↓ 注入實作
store/sqlstore/store.go — SQL 實作的主結構
    ├─ SqlPostStore
    ├─ SqlUserStore
    ├─ SqlChannelStore
    ├─ SqlSessionStore
    └─ ... (70+ 個 sub-store)
    ↓ 使用
sqlx / database/sql
    ↓
PostgreSQL / MySQL
```

---

## Store 介面設計 (`store/store.go`)

```go
type Store interface {
    User()           UserStore
    Channel()        ChannelStore
    Post()           PostStore
    Session()        SessionStore
    Team()           TeamStore
    Bot()            BotStore
    FileInfo()       FileInfoStore
    Webhook()        WebhookStore
    Job()            JobStore
    // ... 70+ sub-store 介面
    Close()
    DropAllTables()
    TotalMasterDbConnections() int
    TotalReadDbConnections() int
}
```

每個 Sub-store 也是介面，例如：

```go
type PostStore interface {
    Save(post *model.Post) (*model.Post, error)
    SaveMultiple(posts []*model.Post) ([]*model.Post, int, error)
    Get(ctx context.Context, id string, opts GetPostsOptions, ...) (*model.PostList, error)
    Update(newPost *model.Post, oldPost *model.Post) (*model.Post, error)
    Delete(postId string, time int64, deleteByID string) error
    // ...
}
```

這種介面設計的優點：
- App 層不依賴 SQL 實作細節
- 測試時可注入 mock
- 可替換底層資料庫引擎

---

## SQL Store 連線管理 (`store/sqlstore/store.go`)

### 主從分離

```go
type SqlStore struct {
    masterX *sqlxDBWrapper    // 主庫（寫操作）
    ReplicaXs []*sqlxDBWrapper // 從庫陣列（讀操作）
    // ...
}
```

取得連線的方式：

```go
// 寫操作
s.GetMaster()

// 讀操作（自動 Round-robin 選擇 Replica）
s.GetReplica()
// 若無 Replica 設定，自動 fallback 到 Master
```

### 連線池設定

透過設定檔控制：
- `MaxOpenConns` — 最大連線數
- `MaxIdleConns` — 閒置連線數
- `ConnMaxLifetimeMilliseconds` — 連線最大存活時間

---

## SQL 查詢模式

Mattermost 使用 **Squirrel** 做 SQL 查詢建構，搭配 **sqlx** 執行：

### Insert 範例（Post 建立）

```go
// store/sqlstore/post_store.go
func (s *SqlPostStore) SaveMultiple(posts []*model.Post) ([]*model.Post, int, error) {
    builder := s.getQueryBuilder().
        Insert("Posts").
        Columns("Id", "CreateAt", "UpdateAt", "UserId", "ChannelId", "Message", ...)

    for _, post := range posts {
        builder = builder.Values(post.Id, post.CreateAt, post.UpdateAt, ...)
    }

    query, args, err := builder.ToSql()
    _, err = s.GetMaster().ExecContext(ctx, query, args...)
}
```

### Select 範例（查詢頻道訊息）

```go
func (s *SqlPostStore) GetPostsPage(options GetPostsOptions) (*model.PostList, error) {
    query := s.getQueryBuilder().
        Select("p.*").
        From("Posts p").
        Where(sq.Eq{"p.ChannelId": options.ChannelId, "p.DeleteAt": 0}).
        OrderBy("p.CreateAt DESC").
        Limit(uint64(options.PerPage)).
        Offset(uint64(options.Page * options.PerPage))

    rows, err := s.GetReplica().QueryContext(ctx, query, args...)
    // ...
}
```

---

## 資料庫 Schema 遷移

使用 **Morph** 工具管理 schema 版本：

```
server/channels/db/migrations/
├─ postgres/
│   ├─ 000001_create_users_table.up.sql
│   ├─ 000002_create_channels_table.up.sql
│   └─ ... (依序編號的 migration 檔案)
└─ mysql/
    └─ ...
```

伺服器啟動時自動執行待套用的 migration。

---

## 快取層

Mattermost 在 Store 層之上有多層快取：

```
App 請求
    ↓
localcachelayer (store/localcachelayer/)
    ├─ 記憶體 LRU cache（go-cache）
    ├─ 分散式快取（透過 cluster 廣播失效通知）
    └─ 快取未命中 → 呼叫底層 sqlstore
    ↓
sqlstore
```

常見快取項目：
- Session（每個請求都需查詢）
- User（頻繁查詢）
- Channel Members（成員列表）
- 設定檔（Config）

---

## 支援的資料庫

| 資料庫 | 狀態 | 驅動 |
|--------|------|------|
| PostgreSQL 12+ | 生產推薦 | `lib/pq` |
| MySQL 8.0+ | 支援 | `go-sql-driver/mysql` |
| SQLite | 僅供本機測試 | `mattn/go-sqlite3` |

---

## 核心資料表

| 資料表 | 對應 Model | 說明 |
|--------|-----------|------|
| `Users` | `model.User` | 使用者帳號 |
| `Sessions` | `model.Session` | 登入 Session |
| `Channels` | `model.Channel` | 頻道 |
| `ChannelMembers` | `model.ChannelMember` | 頻道成員關係 |
| `Posts` | `model.Post` | 訊息 |
| `Teams` | `model.Team` | 工作區（Team） |
| `TeamMembers` | `model.TeamMember` | Team 成員關係 |
| `FileInfo` | `model.FileInfo` | 上傳檔案 metadata |
| `Tokens` | `model.Token` | 一次性 token（邀請、密碼重設等） |
| `Jobs` | `model.Job` | 背景工作排程 |

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/store/store.go` | 所有 Store 介面定義 |
| `server/channels/store/sqlstore/store.go` | SQL Store 主結構與連線管理 |
| `server/channels/store/sqlstore/post_store.go` | Post 資料存取 |
| `server/channels/store/sqlstore/user_store.go` | User 資料存取 |
| `server/channels/store/sqlstore/channel_store.go` | Channel 資料存取 |
| `server/channels/store/sqlstore/session_store.go` | Session 資料存取 |
| `server/channels/store/localcachelayer/` | 記憶體快取層 |
| `server/channels/db/migrations/` | Schema migration 檔案 |
