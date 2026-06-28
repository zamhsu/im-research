# API 層：HTTP Handler

## 路由註冊

**檔案**：`server/channels/api4/post.go:24`

```go
api.BaseRoutes.Posts.Handle("", api.APISessionRequired(createPost)).Methods(http.MethodPost)
```

- 路由：`POST /api/v4/posts`
- Middleware：`APISessionRequired` — 強制需要有效 session token

## createPost() Handler

**檔案**：`server/channels/api4/post.go:96-181`

### 執行步驟

```go
func createPost(c *Context, w http.ResponseWriter, r *http.Request) {
    // 1. 解析 JSON body 為 model.Post
    post := model.PostFromJSON(r.Body)

    // 2. 移除圖片 proxy URL (for re-processing)
    post = c.App.PostWithProxyRemovedFromImageURLs(&post)

    // 3. 強制設定 UserId = 目前 session 的 UserId（防止偽造）
    post.UserId = c.AppContext.Session().UserId

    // 4. 執行發送前檢查（見下方）
    createPostChecks("Api4.createPost", c, &post)

    // 5. 解析 ?set_online= 查詢參數
    setOnlineBool := r.URL.Query().Get("set_online") != "false"

    // 6. 呼叫 App 層
    rpost, err := c.App.CreatePostAsUser(
        c.AppContext, &post,
        c.AppContext.Session().Id,
        setOnlineBool,
    )

    // 7. 回傳 HTTP 201 Created
    w.WriteHeader(http.StatusCreated)
    if err := json.NewEncoder(w).Encode(rpost); err != nil { ... }
}
```

## createPostChecks() 權限驗證

**檔案**：`server/channels/api4/post.go:59-94`

依序檢查：

| 檢查項目 | 說明 |
|----------|------|
| `PermissionCreatePost` | 使用者在該頻道有發文權限 |
| `PermissionUseChannelMentions` | 有無 `@channel` / `@all` 權限 |
| `PermissionUseGroupMentions` | 有無 `@group` 提及權限 |
| Channel 類型 | DM/GM 頻道的額外限制 |

## 重要設計

- **UserId 強制覆寫**（`post.go:104`）：即使客戶端傳入錯誤的 UserId，server 端一律用 session 中的 ID 覆寫，防止身份偽造。
- **set_online 參數**：控制此次發文是否將發送者狀態設為 online，可以用 `?set_online=false` 靜默發文（例如 bot）。
