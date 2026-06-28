# Mattermost RBAC 權限系統完整設計參考

## 一、核心資料模型

### 1.1 Role 結構 (role.go)

```go
type Role struct {
    Id            string   `json:"id"`              // 角色唯一識別符（如 "system_admin"）
    Name          string   `json:"name"`            // 角色名稱（如 "system_admin"）
    DisplayName   string   `json:"display_name"`    // 前端顯示名稱
    Description   string   `json:"description"`     // 角色描述
    CreateAt      int64    `json:"create_at"`
    UpdateAt      int64    `json:"update_at"`
    DeleteAt      int64    `json:"delete_at"`       // 軟刪除
    Permissions   []string `json:"permissions"`     // 權限 ID 列表
    SchemeManaged bool     `json:"scheme_managed"`  // 是否受 Scheme 管理
    BuiltIn       bool     `json:"built_in"`        // 是否為內建角色
    SchemeId      *string  `json:"scheme_id"`       // 所屬 Scheme ID（可選）
}

type RolePatch struct {
    Permissions *[]string `json:"permissions"`     // 部分更新時僅修改權限
}
```

**關鍵欄位說明**：
- `SchemeManaged`：表示該角色是否受 Scheme 管理。內建角色（BuiltIn=true）通常被 Scheme 管理，允許 Team/Channel 層級自訂角色
- `SchemeId`：當 Role 屬於某個 Team/Channel Scheme 時，記錄其 Scheme ID
- `Permissions`：權限 ID 列表，以空格分隔存儲於 DB，轉換為字串陣列

### 1.2 Scheme 結構 (scheme.go)

```go
type Scheme struct {
    Id                        string `json:"id"`
    Name                      string `json:"name"`
    DisplayName               string `json:"display_name"`
    Description               string `json:"description"`
    CreateAt                  int64  `json:"create_at"`
    UpdateAt                  int64  `json:"update_at"`
    DeleteAt                  int64  `json:"delete_at"`
    Scope                     string `json:"scope"`                       // "team" 或 "channel"
    // Team 層級（Scope="team" 時使用）
    DefaultTeamAdminRole      string `json:"default_team_admin_role"`
    DefaultTeamUserRole       string `json:"default_team_user_role"`
    DefaultTeamGuestRole      string `json:"default_team_guest_role"`
    // Channel 層級
    DefaultChannelAdminRole   string `json:"default_channel_admin_role"`
    DefaultChannelUserRole    string `json:"default_channel_user_role"`
    DefaultChannelGuestRole   string `json:"default_channel_guest_role"`
}

const (
    SchemeScopeTeam    = "team"
    SchemeScopeChannel = "channel"
)
```

**設計要點**：
- Scheme 允許 Team 或 Channel 自訂其角色和權限
- 一個 Scheme 定義該層級的所有預設角色
- Scheme 建立時自動生成相應的 Role 記錄（scheme-managed roles）

### 1.3 成員角色欄位

**User 模型**：
```go
type User struct {
    Roles string  // 系統級角色列表，以空格分隔（如 "system_admin system_user"）
}

func (u *User) GetRoles() []string { return strings.Fields(u.Roles) }
func (u *User) IsSystemAdmin() bool { return IsInRole(u.Roles, SystemAdminRoleId) }
func (u *User) IsGuest() bool       { return IsInRole(u.Roles, SystemGuestRoleId) }
```

**TeamMember 模型**：
```go
type TeamMember struct {
    TeamId        string
    UserId        string
    Roles         string  // 自訂團隊角色列表
    SchemeGuest   bool    // 是否為 Scheme 中的 Guest
    SchemeUser    bool    // 是否為 Scheme 中的 User
    SchemeAdmin   bool    // 是否為 Scheme 中的 Admin
    ExplicitRoles string  // 明確指派的額外角色（覆蓋 Scheme）
    DeleteAt      int64
}
```

**ChannelMember 模型**：
```go
type ChannelMember struct {
    ChannelId string
    UserId    string
    Roles     string  // 自訂頻道角色列表
    // ...通知和計數欄位
}
```

---

## 二、預設角色層級體系

### 2.1 系統級 (System Scope)

| 角色 ID | 說明 | SchemeManaged |
|--------|------|---------------|
| `system_admin` | 系統最高管理員，擁有所有權限 | ✓ |
| `system_user` | 普通系統用戶，預設角色 | ✓ |
| `system_guest` | 訪客用戶，權限受限 | ✓ |
| `system_user_manager` | 專司用戶管理 | ✗ |
| `system_read_only_admin` | 僅讀管理員 | ✗ |
| `system_manager` | 系統管理器（可讀寫） | ✗ |
| `system_custom_group_admin` | 自訂群組管理員 | ✗ |
| `system_post_all` | 允許在系統所有頻道發文 | ✗ |
| `system_post_all_public` | 允許在系統所有公開頻道發文 | ✗ |
| `system_user_access_token` | 用於 API 訪問令牌 | ✗ |

### 2.2 團隊級 (Team Scope)

| 角色 ID | 說明 | 預設權限示例 |
|--------|------|-----------|
| `team_admin` | 團隊管理員 | `manage_team`, `manage_team_roles`, `import_team`, `delete_others_posts` 等 |
| `team_user` | 團隊成員 | `list_team_channels`, `join_public_channels`, `view_team`, `create_public_channel` 等 |
| `team_guest` | 團隊訪客 | `view_team` |
| `team_post_all` | 可在團隊所有頻道發文 | `create_post`, `use_channel_mentions` |

### 2.3 頻道級 (Channel Scope)

| 角色 ID | 說明 | 預設權限示例 |
|--------|------|-----------|
| `channel_admin` | 頻道管理員 | `manage_channel_roles`, `manage_public_channel_properties`, `manage_channel_access_rules` 等 |
| `channel_user` | 頻道成員 | `read_channel`, `create_post`, `edit_post`, `delete_post`, `manage_public_channel_members` 等 |
| `channel_guest` | 頻道訪客 | `read_channel`, `create_post`, `upload_file` 等 |

---

## 三、權限常數清單

### 3.1 權限結構

```go
type Permission struct {
    Id          string `json:"id"`
    Name        string `json:"name"`        // i18n key
    Description string `json:"description"`
    Scope       string `json:"scope"`       // 適用層級
}

const (
    PermissionScopeSystem  = "system_scope"
    PermissionScopeTeam    = "team_scope"
    PermissionScopeChannel = "channel_scope"
    PermissionScopeGroup   = "group_scope"
)
```

### 3.2 系統級權限 (PermissionScopeSystem)

**使用者管理**：
- `manage_system` — 系統管理總權限
- `assign_system_admin_role` — 指派系統管理員角色
- `edit_other_users` — 編輯其他用戶
- `promote_guest` — 從訪客升級為用戶
- `demote_to_guest` — 降級為訪客
- `create_user_access_token` — 建立用戶訪問令牌
- `permanent_delete_user` — 永久刪除用戶

**團隊管理**：
- `create_team` — 建立新團隊
- `list_public_teams` / `list_private_teams` — 列出團隊
- `join_public_teams` / `join_private_teams` — 加入團隊

**直接/群組訊息**：
- `create_direct_channel` — 建立直接訊息
- `create_group_channel` — 建立群組訊息
- `view_members` — 查看系統成員

**系統控制台** (sysconsole_read/write_* 形式)：超過 100 個，涵蓋設置、環境、認證、合規、報告等

### 3.3 團隊級權限 (PermissionScopeTeam)

**團隊管理**：
- `manage_team`, `manage_team_roles`, `manage_team_access_rules`
- `remove_user_from_team`, `invite_user`, `add_user_to_team`

**頻道管理**：
- `create_public_channel`, `create_private_channel`
- `list_team_channels`, `join_public_channels`, `read_public_channel`
- `view_team`

**Webhook & 斜杠命令**：
- `manage_own_slash_commands`, `manage_others_slash_commands`
- `manage_own_incoming_webhooks`, `manage_others_incoming_webhooks`
- `manage_own_outgoing_webhooks`, `manage_others_outgoing_webhooks`

### 3.4 頻道級權限 (PermissionScopeChannel)

**讀寫**：
- `read_channel`, `read_channel_content`
- `create_post`, `create_post_public`, `create_post_ephemeral`
- `edit_post`, `edit_others_posts`
- `delete_post`, `delete_others_posts`
- `upload_file`

**管理**：
- `manage_public_channel_members`, `manage_private_channel_members`
- `manage_public_channel_properties`, `manage_private_channel_properties`
- `manage_channel_roles`, `manage_channel_access_rules`
- `delete_public_channel`, `delete_private_channel`
- `convert_public_channel_to_private`, `convert_private_channel_to_public`

**互動**：
- `add_reaction`, `remove_reaction`, `remove_others_reactions`
- `use_channel_mentions`, `use_group_mentions`

---

## 四、業務邏輯層實現

### 4.1 核心權限檢查函數 (authorization.go)

```go
// 基於 Session 的權限檢查（API 層推薦）
func (a *App) SessionHasPermissionTo(session model.Session, permission *model.Permission) bool
func (a *App) SessionHasPermissionToTeam(session model.Session, teamID string, permission *model.Permission) bool
func (a *App) SessionHasPermissionToChannel(rctx request.CTX, session model.Session, channelID string, permission *model.Permission) (hasPermission bool, isMember bool)

// 基於用戶 ID 的權限檢查（應用層推薦）
func (a *App) HasPermissionTo(askingUserId string, permission *model.Permission) bool
func (a *App) HasPermissionToTeam(rctx request.CTX, askingUserId string, teamID string, permission *model.Permission) bool
func (a *App) HasPermissionToChannel(rctx request.CTX, askingUserId string, channelID string, permission *model.Permission) (hasPermission bool, isMember bool)

// 核心角色-權限檢查
func (a *App) RolesGrantPermission(roleNames []string, permissionId string) bool
    // 1. 根據角色名稱從 DB 查詢 Role 對象
    // 2. 遍歷 Role.Permissions 是否包含目標權限
    // 3. 若任一角色包含該權限，返回 true
```

### 4.2 HasPermissionToChannel 的檢查順序

```go
func (a *App) SessionHasPermissionToChannel(...) bool {
    // 1. 檢查 channel member 的角色
    if a.RolesGrantPermission(channelRoles, permission.Id) {
        return true
    }
    // 2. 若無，向上檢查 Team 權限
    if channel.TeamId != "" {
        return a.SessionHasPermissionToTeam(session, channel.TeamId, permission)
    }
    // 3. 若無，檢查系統級權限
    return a.SessionHasPermissionTo(session, permission)
}
```

### 4.3 發文權限的兩階段 Fallback 模式

```go
func userCreatePostPermissionCheckWithApp(rctx request.CTX, a *App, userId, channelId string) *model.AppError {
    hasPermission := false

    // 第一步：檢查 Channel 級 create_post 權限
    if ok, _ := a.HasPermissionToChannel(rctx, userId, channelId, model.PermissionCreatePost); ok {
        hasPermission = true
    } else if channel, err := a.GetChannel(rctx, channelId); err == nil {
        // 第二步：Fallback — 公開頻道可用 create_post_public 權限
        if channel.Type == model.ChannelTypeOpen &&
            a.HasPermissionToTeam(rctx, userId, channel.TeamId, model.PermissionCreatePostPublic) {
            hasPermission = true
        }
    }

    if !hasPermission {
        return model.MakePermissionErrorForUser(userId, []*model.Permission{model.PermissionCreatePost})
    }
    return nil
}
```

### 4.4 Role CRUD

```go
func (a *App) CreateRole(role *model.Role) (*model.Role, *model.AppError)
    // BuiltIn = false, SchemeManaged = false（手動建立的角色）

func (a *App) UpdateRole(role *model.Role) (*model.Role, *model.AppError)
    // 驗證角色有效性、驗證每個 Permission ID 存在

func (a *App) PatchRole(role *model.Role, patch *model.RolePatch) (*model.Role, *model.AppError)
    // 僅更新 Permissions 欄位，發送角色更新 WebSocket 事件
```

---

## 五、Store 層資料結構

### 5.1 Roles 表

```
Id              STRING PRIMARY KEY
Name            STRING UNIQUE        -- 如 "system_admin"
DisplayName     STRING
Description     STRING
Permissions     TEXT                 -- 以空格分隔的權限 ID
SchemeManaged   BOOLEAN
BuiltIn         BOOLEAN
SchemeId        STRING FK            -- 所屬 Scheme（可為空）
CreateAt        BIGINT
UpdateAt        BIGINT
DeleteAt        BIGINT               -- 軟刪除
```

### 5.2 Schemes 表

```
Id                        STRING PRIMARY KEY
Name                      STRING
DisplayName               STRING
Description               STRING
Scope                     STRING         -- "team" 或 "channel"
DefaultTeamAdminRole      STRING FK → Roles.Id
DefaultTeamUserRole       STRING FK → Roles.Id
DefaultTeamGuestRole      STRING FK → Roles.Id
DefaultChannelAdminRole   STRING FK → Roles.Id
DefaultChannelUserRole    STRING FK → Roles.Id
DefaultChannelGuestRole   STRING FK → Roles.Id
CreateAt                  BIGINT
UpdateAt                  BIGINT
DeleteAt                  BIGINT
```

### 5.3 TeamMembers 表

```
TeamId          STRING
UserId          STRING
Roles           TEXT           -- 自訂角色列表（空格分隔）
SchemeAdmin     BOOLEAN        -- 屬於 Scheme 的 Admin 角色
SchemeUser      BOOLEAN        -- 屬於 Scheme 的 User 角色
SchemeGuest     BOOLEAN        -- 屬於 Scheme 的 Guest 角色
ExplicitRoles   TEXT           -- 明確指派的額外角色
DeleteAt        BIGINT
CreateAt        BIGINT
PRIMARY KEY (TeamId, UserId)
```

### 5.4 ChannelMembers 表

```
ChannelId       STRING
UserId          STRING
Roles           TEXT           -- 自訂 Channel 角色
LastViewedAt    BIGINT
MsgCount        BIGINT
DeleteAt        BIGINT
PRIMARY KEY (ChannelId, UserId)
```

---

## 六、Scheme 機制深入解析

### 6.1 多層級覆蓋關係

```
系統預設角色（System Default Roles）
    ↓ 當 Team 指派 Scheme 時
Team Scheme 自訂角色
    ↓ 當 Channel 指派 Scheme 時
Channel Scheme 自訂角色
    ↓
用戶在 Channel 的實際有效權限
```

### 6.2 調節權限 (Moderated Permissions)

**目的**：讓 Channel 管理員可微調 Channel 級的操作權限，但不能超越 Team 層級的限制。

```go
func (r *Role) MergeChannelHigherScopedPermissions(higherScopedPermissions *RolePermissions)
    // 合併更高層級（Team）的權限到 Channel 角色
    // 規則：
    // - 若權限是 ChannelModeratedPermissions（可調節類），則
    //   必須在 Channel 角色 AND 更高層級都存在，才保留
    // - 若不是調節類，則由更高層級決定
```

**調節權限列表**（Channel 級有效性受 Team 級約束）：
- `create_post` / `create_post_moderated`
- `edit_post` / `edit_post_moderated`
- `delete_post` / `delete_post_moderated`
- `manage_channel_members` / `manage_channel_members_moderated`

### 6.3 Scheme 指派流程

**Team 指派 Scheme**：
```
Team(team_A) 指派 Scheme(scheme_custom)
    ↓
自動為該 Team 建立 6 個 Scheme-managed Roles
    ↓
TeamMember 的有效角色 = SchemeAdmin/User/Guest 標誌對應角色 + ExplicitRoles
```

**Channel 指派 Scheme**：
```
Channel(channel_B) 指派 Scheme(scheme_channel)
    ↓
ChannelMember 的有效權限 = Channel Scheme 角色 ∩ Team 角色（調節類）
                           ∪ Team 角色（非調節類）
```

---

## 七、API 層權限檢查模式

### 7.1 建立頻道

```go
func createChannel(c *Context, w http.ResponseWriter, r *http.Request) {
    // 根據頻道類型檢查不同 Team 級權限
    if channel.Type == model.ChannelTypeOpen {
        if !c.App.SessionHasPermissionToTeam(*c.AppContext.Session(), channel.TeamId,
            model.PermissionCreatePublicChannel) {
            c.SetPermissionError(model.PermissionCreatePublicChannel)
            return
        }
    } else if channel.Type == model.ChannelTypePrivate {
        if !c.App.SessionHasPermissionToTeam(*c.AppContext.Session(), channel.TeamId,
            model.PermissionCreatePrivateChannel) {
            c.SetPermissionError(model.PermissionCreatePrivateChannel)
            return
        }
    }
    // ...執行操作
}
```

### 7.2 更新頻道

```go
func updateChannel(c *Context, w http.ResponseWriter, r *http.Request) {
    // 基於頻道類型檢查 Channel 級權限
    switch oldChannel.Type {
    case model.ChannelTypeOpen:
        if ok, _ := c.App.SessionHasPermissionToChannel(c.AppContext,
            *c.AppContext.Session(), c.Params.ChannelId,
            model.PermissionManagePublicChannelProperties); !ok {
            c.SetPermissionError(model.PermissionManagePublicChannelProperties)
            return
        }
    case model.ChannelTypePrivate:
        if ok, _ := c.App.SessionHasPermissionToChannel(c.AppContext,
            *c.AppContext.Session(), c.Params.ChannelId,
            model.PermissionManagePrivateChannelProperties); !ok {
            c.SetPermissionError(model.PermissionManagePrivateChannelProperties)
            return
        }
    }
}
```

---

## 八、訪客 (Guest) 與機器人 (Bot) 角色

### 8.1 訪客用戶

```go
func (u *User) IsGuest() bool {
    return IsInRole(u.Roles, SystemGuestRoleId)
}
```

**system_guest 預設權限**：
- `create_direct_channel`
- `create_group_channel`

**訪客限制**：
- 不能建立團隊
- 不能自行加入公開團隊（需明確邀請）
- 受邀加入特定頻道後才能訪問
- 某些高級功能（如 Persistent Notifications）可配置是否允許訪客使用

### 8.2 機器人帳號

```go
type User struct {
    IsBot             bool    // 標記為機器人
    BotDescription    string
    BotLastIconUpdate int64
}
```

**Bot 特性**：
- 由整合或插件建立，可分配特定系統角色（`create_bot`, `manage_bots`）
- 通常不被視為普通用戶（很多 API 過濾掉 Bot）
- 可在所有頻道發文（取決於分配的權限）

---

## 九、資料關係圖

```
┌──────────────────────────────────────────────────────────┐
│                       System                             │
│  User (Roles: "system_admin system_user")                │
└──────────────┬───────────────────────────────────────────┘
               │ belongs to
               ↓
┌──────────────────────────────────────────────────────────┐
│                      Teams                               │
│  Team (SchemeId → Scheme)                                │
└──────────────┬───────────────────────────────────────────┘
               │ has members
               ↓
┌──────────────────────────────────────────────────────────┐
│                   TeamMembers                            │
│  Roles, SchemeAdmin/User/Guest, ExplicitRoles            │
│  effective_roles = SchemeRole + ExplicitRoles            │
└──────────────┬───────────────────────────────────────────┘
               │ contains
               ↓
┌──────────────────────────────────────────────────────────┐
│                    Channels                              │
│  Channel (TeamId, SchemeId, Type)                        │
└──────────────┬───────────────────────────────────────────┘
               │ has members
               ↓
┌──────────────────────────────────────────────────────────┐
│                ChannelMembers                            │
│  Roles (合併 Team 調節權限後的有效權限)                   │
└──────────────┬───────────────────────────────────────────┘
               │ checks against
               ↓
┌──────────────────────────────────────────────────────────┐
│                     Roles                                │
│  Id, Name, Permissions (空格分隔), SchemeManaged, BuiltIn │
└──────────────┬───────────────────────────────────────────┘
               │ part of
               ↓
┌──────────────────────────────────────────────────────────┐
│                    Schemes                               │
│  Scope ("team" | "channel")                              │
│  DefaultTeam/ChannelAdminRole/UserRole/GuestRole         │
│  [各欄位均指向 Roles 表]                                  │
└──────────────────────────────────────────────────────────┘
```

---

## 十、關鍵設計模式

### 10.1 三層級 Cascading Authorization

```
Channel 級權限（ChannelMember.Roles + Scheme）
    ↓ 若無
Team 級權限（TeamMember.Roles + Scheme）
    ↓ 若無
System 級權限（User.Roles）
```

### 10.2 Session vs User 級權限檢查

| 情境 | 推薦函數 | 原因 |
|------|----------|------|
| HTTP API handler | `SessionHasPermissionTo*` | 基於當前 Session 狀態，含 `IsUnrestricted` 短路 |
| 後台任務 / 事件處理 | `HasPermissionTo*` | 永遠從 DB 查最新狀態 |

### 10.3 性能優化策略

1. **批量查詢**：`GetRolesByNames(names []string)`、`GetAllChannelMembersForUser()`
2. **緩存**：Channel member 角色映射被快取，減少重複 DB 查詢
3. **短路邏輯**：System Admin 預檢查，直接返回 true，避免不必要的 DB 查詢
4. **Session 緩存**：Session 內含角色快照，HTTP 請求不需每次 DB 查詢

### 10.4 SchemeManaged 的含義

| 值 | 含義 | 典型角色 |
|----|------|---------|
| `true` | 受 Scheme 管理，遵循調節權限規則 | `system_user`, `team_admin`, `channel_user` |
| `false` | 獨立定義，不受 Scheme 約束 | `system_admin`, `system_post_all` |

---

## 十一、給實作者的設計啟示

### 11.1 多層級 + 多維度的設計

- **垂直（層級）**：System → Team → Channel，下層可被上層約束（調節權限）
- **水平（維度）**：系統預設角色 vs Scheme 自訂角色 vs ExplicitRoles

### 11.2 調節權限機制值得引入

讓 Channel 管理員可微調頻道行為，但上限受 Team 層級約束，防止「頻道權限高於團隊限制」的邏輯漏洞。

### 11.3 審計的重要性

所有角色建立/修改/刪除、成員角色指派、Scheme 指派都應記錄完整的前後狀態，搭配 `AuditRecord` 機制。

### 11.4 軟刪除的代價

Mattermost 全面使用 `DeleteAt` 欄位軟刪除，換來完整審計追蹤，但所有查詢必須加 `WHERE DeleteAt = 0` 條件，需注意 index 設計。

### 11.5 實作優先順序建議

1. **先定義 Permission 常數與 Permission Scope** — 影響所有 API 設計
2. **建立系統預設角色體系** — 三層級各自的預設角色與預設權限
3. **實作 RolesGrantPermission + 三層 Cascading** — 核心檢查引擎
4. **加入 Scheme 機制** — 讓 Team/Channel 可自訂角色（企業彈性需求）
5. **加入調節權限** — 讓 Channel 管理員可微調但不逾越 Team 限制
6. **完善審計日誌** — 企業採購合規必備

---

## 十二、實作檢查清單

- [ ] 定義三層角色結構（System / Team / Channel）
- [ ] 列舉所有核心操作為 Permission 常數，標記 Scope
- [ ] 建立系統預設角色及其預設 Permission 映射表
- [ ] 實作 `RolesGrantPermission` + 三層 Cascading 檢查
- [ ] 實作 Scheme CRUD（自動生成 scheme-managed roles）
- [ ] 實作調節權限合併邏輯
- [ ] 批量查詢 + 緩存角色查詢
- [ ] 所有 API handler 加入權限 guard（`SetPermissionError`）
- [ ] 所有角色變更寫入 Audit Log
- [ ] 搜尋 + 列表 API 過濾掉 DeleteAt > 0 的記錄
- [ ] Session 緩存角色快照，減少 DB 壓力
- [ ] 單元測試：System Admin 短路、Guest 限制、Scheme 覆蓋邏輯
