# 使用者認證流程

## 入口端點

```
POST /api/v4/users/login
```

## 完整呼叫鏈

```
api4/user.go:login()                              [HTTP Handler]
    ↓
    解析 JSON body → model.LoginOptions
    呼叫 c.App.AuthenticateUserForLogin()
    ↓
app/login.go:AuthenticateUserForLogin()           [App Layer]
    ├─ 驗證密碼不為空（非 CWS 登入時）
    ├─ GetUserForLogin(rctx, id, loginId)
    │      └─ Store.User().GetForLogin()          [DB 查詢]
    │             SELECT * FROM Users WHERE ...
    │
    ├─ authenticateUser()                         [app/authentication.go]
    │      ├─ checkUserPassword()
    │      │      └─ bcrypt 雜湊比對
    │      └─ CheckUserMfa()
    │             └─ TOTP token 驗證
    │
    └─ 返回 *model.User
    ↓
app/login.go:DoLogin()                            [App Layer]
    ├─ 建立 model.Session 物件
    │      ├─ Token = 隨機 26 字元
    │      ├─ UserId
    │      ├─ CreateAt, ExpiresAt
    │      └─ Props（platform、browser、device）
    ├─ Store.Session().Save()                     [DB 寫入]
    ├─ Store.User().UpdateLastLogin()             [DB 更新]
    └─ 設定 response header: Token
    ↓
api4/user.go:login()                              [HTTP Handler]
    └─ 回傳 JSON: *model.User
       header: Token: <session_token>
```

---

## 各步驟詳細說明

### 1. HTTP Handler：`login()` (`api4/user.go:2093`)

```go
func login(c *Context, w http.ResponseWriter, r *http.Request) {
    // 解析 JSON body
    // 取得 loginId, password, mfaToken, deviceId
    // 呼叫 c.App.AuthenticateUserForLogin(...)
    // 成功後呼叫 c.App.DoLogin(...)
    // 設定 Token header
    // 回傳 User JSON
}
```

### 2. 使用者查詢：`AuthenticateUserForLogin()` (`app/login.go:29`)

```go
func (a *App) AuthenticateUserForLogin(
    rctx request.CTX,
    id, loginId, password, mfaToken, cwsToken string,
    ldapOnly bool,
) (user *model.User, err *model.AppError)
```

支援的登入方式：
- 一般密碼登入（email / username）
- MFA（TOTP）
- LDAP
- SAML SSO
- CWS（Cloud Web Service）單次 token

### 3. 密碼驗證：`authenticateUser()` (`app/authentication.go`)

```go
func (a *App) authenticateUser(
    rctx request.CTX,
    user *model.User,
    password, mfaToken string,
) *model.AppError
```

- 使用 `bcrypt` 比對密碼雜湊
- MFA 使用 TOTP 演算法驗證

### 4. Session 建立：`DoLogin()` (`app/login.go`)

```go
func (a *App) DoLogin(
    rctx request.CTX,
    w http.ResponseWriter,
    r *http.Request,
    user *model.User,
    deviceID string,
    isMobile, isOAuthUser, isSaml bool,
) (*model.Session, *model.AppError)
```

Session token 透過 `response header Token` 回傳給客戶端，後續請求須在 `Authorization: Bearer <token>` 或 cookie 中攜帶。

---

## 後續請求認證

所有需要登入的端點都包在 `APISessionRequired()` middleware：

```
Request Header: Authorization: Bearer <token>
    ↓
api4/context.go: GetSession()
    ↓
app/session.go: GetSession(token)
    ├─ 優先從 memory cache 取得
    └─ 快取失效時 → Store.Session().Get(token)  [DB 查詢]
    ↓
將 session 綁定到 request context
```

---

## 相關資料結構

### `model.User` (`server/public/model/user.go`)

| 欄位 | 說明 |
|------|------|
| `Id` | 26 字元 ULID |
| `Username` | 使用者名稱 |
| `Email` | 電子郵件 |
| `Password` | bcrypt 雜湊值 |
| `AuthService` | 認證來源（ldap / saml / 空=本機） |
| `Roles` | 系統角色字串 |
| `MfaActive` | 是否啟用 MFA |

### `model.Session` (`server/public/model/session.go`)

| 欄位 | 說明 |
|------|------|
| `Id` | Session ID |
| `Token` | 26 字元隨機 token |
| `UserId` | 所屬使用者 |
| `CreateAt` | 建立時間（毫秒 timestamp） |
| `ExpiresAt` | 到期時間 |
| `Props` | 平台、瀏覽器、裝置資訊 |

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/user.go:2093` | `login()` HTTP handler |
| `server/channels/app/login.go:29` | `AuthenticateUserForLogin()` |
| `server/channels/app/authentication.go` | 密碼與 MFA 驗證 |
| `server/channels/app/session.go` | Session 建立與管理 |
| `server/channels/store/sqlstore/user_store.go` | User DB 操作 |
| `server/channels/store/sqlstore/session_store.go` | Session DB 操作 |
