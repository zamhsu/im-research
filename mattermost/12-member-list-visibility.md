# 成員清單顯示規則

## 一、誰能查詢成員清單

### Team 成員清單

```
GET /api/v4/teams/{team_id}/members
```

需要 `PermissionViewTeam`（team-scoped）。**只有已加入該 team 的成員才能查詢**，非成員直接 403。

**程式碼位置：** `api4/team.go:774`
```go
c.App.SessionHasPermissionToTeam(*c.AppContext.Session(), c.Params.TeamId, model.PermissionViewTeam)
```

### Channel 成員清單

```
GET /api/v4/channels/{channel_id}/members
```

需要 `PermissionReadChannel`（channel-scoped）。**只有已加入該 channel 的成員才能查詢**，公開頻道的非成員也看不到。

**程式碼位置：** `api4/channel.go:1843`
```go
c.App.SessionHasPermissionToChannel(c.AppContext, *c.AppContext.Session(), c.Params.ChannelId, model.PermissionReadChannel)
```

---

## 二、清單裡能看到哪些人（ViewUsersRestrictions）

通過查詢權限後，還有第二層過濾：**只能看到「與自己有交集」的人**。

### 規則邏輯（`app/user.go:GetViewUsersRestrictions`）

```
若 user 具備系統級 PermissionViewMembers
    → restrictions = nil（看得到所有人）

否則
    → restrictions.Teams    = 自己有 PermissionViewMembers 的所有 team ID
    → restrictions.Channels = 自己所屬的所有 channel ID

Store 層以 JOIN TeamMembers / ChannelMembers 過濾，
只回傳「在上述任一 team 或 channel 中出現過」的使用者
```

### 各角色實際可見範圍

| 角色 | 看得到誰 |
|------|---------|
| 系統管理員 | 所有人（restrictions = nil，跳過所有限制） |
| 一般成員（system_user） | 與自己**共用至少一個 team 或 channel** 的人 |
| Guest（system_guest） | 僅與自己**共用 channel** 的人（無 PermissionViewMembers，restrictions.Teams 為空） |

### 適用範圍

ViewUsersRestrictions 同樣套用於：
- `GET /api/v4/teams/{team_id}/members`（`team_store.go:1710`）
- `POST /api/v4/users/search`（`api4/user.go:1344`）
- `GET /api/v4/users/autocomplete`（`api4/user.go:1409`）

**Channel 成員清單例外：** `GET /api/v4/channels/{channel_id}/members` 不額外套用 ViewUsersRestrictions，頻道成員資格本身已是足夠的限制。

---

## 三、回傳資料的欄位過濾

### User 物件欄位（搜尋、autocomplete 結果）

**程式碼位置：** `public/model/user.go:696`（`SanitizeProfile`）

| 欄位 | 可見性規則 |
|------|-----------|
| `password`、`mfa_secret`、`mfa_used_timestamps` | 永遠清除，任何人都看不到 |
| `email` | 由 `PrivacySettings.ShowEmailAddress` 控制；管理員永遠可見 |
| `first_name`、`last_name` | 由 `PrivacySettings.ShowFullName` 控制；管理員永遠可見 |
| `auth_data`、`auth_service` | 非管理員清除 |
| `notify_props` | 非管理員清除 |
| `last_password_update` | 非管理員清除 |
| `allow_marketing` | 永遠清除 |

### TeamMember 角色欄位

若呼叫者**沒有** `PermissionManageTeamRoles`，回傳的 `TeamMember` 會呼叫 `SanitizeRoleData()`：

**程式碼位置：** `api4/team.go:800`、`public/model/team_member.go`

| 欄位 | 清除條件 |
|------|---------|
| `roles`、`explicit_roles` | 無 PermissionManageTeamRoles 時清除 |
| `scheme_admin`、`scheme_user`、`scheme_guest` | 同上 |
| `delete_at` | 同上 |

### ChannelMember 時間欄位

**程式碼位置：** `api4/channel.go:1857`（`SanitizeForCurrentUser`）

| 欄位 | 對自己 | 對他人 |
|------|--------|--------|
| `last_viewed_at` | 保留實際值 | 設為 `-1` |
| `last_update_at` | 保留實際值 | 設為 `-1` |

---

## 四、隱私相關設定

**程式碼位置：** `public/model/config.go:2346`（`PrivacySettings`）

| 設定 | 預設值 | 說明 |
|------|--------|------|
| `ShowEmailAddress` | `true` | 控制非管理員是否看得到其他人的 email |
| `ShowFullName` | `true` | 控制非管理員是否看得到其他人的 first_name / last_name |

兩項設定對系統管理員無效（管理員永遠看得到）。

---

## 五、UserCanSeeOtherUser 單點判斷

除了清單 API，系統內部也有單點判斷函式決定「我能不能看到這個人」。

**程式碼位置：** `app/user.go:2630`

```
UserCanSeeOtherUser(viewerId, targetId):
    1. viewerId == targetId           → 永遠可見（自己）
    2. restrictions == nil            → 可見（管理員或系統級 PermissionViewMembers）
    3. targetId 在 restrictions.Teams
       或 restrictions.Channels 中   → 可見
    4. 否則                           → 不可見
```

---

## 六、流程摘要

```
呼叫 GET /teams/{id}/members 或 /channels/{id}/members
    ↓
[第一層] 自己是否為成員？
    PermissionViewTeam / PermissionReadChannel
    否 → 403
    ↓
[第二層] 清單裡回傳哪些人？（ViewUsersRestrictions）
    管理員          → 全部
    一般成員        → 共用 team 或 channel 的人
    Guest           → 共用 channel 的人
    ↓
[第三層] 每個人顯示哪些欄位？（SanitizeProfile / SanitizeRoleData）
    密碼 / MFA      → 永遠隱藏
    Email / 全名    → 依 PrivacySettings 隱藏（管理員例外）
    TeamMember 角色 → 無 ManageTeamRoles 時隱藏
    ChannelMember 時間 → 他人的 last_viewed_at 設為 -1
```

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/team.go:764` | `getTeamMembers()` handler |
| `server/channels/api4/channel.go:1837` | `getChannelMembers()` handler |
| `server/channels/api4/user.go:1259` | `searchUsers()` / `autocompleteUsers()` |
| `server/channels/app/user.go:2676` | `GetViewUsersRestrictions()` |
| `server/channels/app/user.go:2630` | `UserCanSeeOtherUser()` |
| `server/channels/store/sqlstore/team_store.go:1710` | ViewUsersRestrictions SQL 過濾（Team） |
| `server/channels/store/sqlstore/user_store.go:1800` | `applyViewRestrictionsFilter()` |
| `server/public/model/user.go:696` | `SanitizeProfile()` |
| `server/public/model/team_member.go` | `SanitizeRoleData()` |
| `server/public/model/config.go:2346` | `PrivacySettings` |
