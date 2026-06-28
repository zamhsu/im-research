# 團隊（Team）操作流程

## 團隊類型

| 類型常數 | 值 | 說明 |
|---------|-----|------|
| `TeamOpen` | `O` | 公開團隊，任何人可申請加入 |
| `TeamInvite` | `I` | 私密團隊，只能透過邀請加入 |

---

## 一、建立團隊

### 入口端點

```
POST /api/v4/teams
```

### 權限檢查

| 權限 | 時機 |
|------|------|
| `PermissionCreateTeam` | 必要，建立團隊前 |
| `PermissionInviteUser` | 若設定 `AllowOpenInvite` 或 `AllowedDomains` |
| `PermissionSysconsoleWriteUserManagementPermissions` | 若同時指定 `SchemeId` |

### 呼叫鏈

```
api4/team.go:createTeam()                              [HTTP Handler]
    ↓
    解析 JSON body → *model.Team
    ├─ SessionHasPermissionTo(PermissionCreateTeam)
    ├─ Cloud 上限檢查（active teams count）
    └─ 呼叫 c.App.CreateTeamWithUser()
    ↓
app/team.go:CreateTeamWithUser()                       [App Layer]
    ├─ GetUser(userId)                                 [取建立者資料]
    ├─ teamService.IsTeamEmailAllowed(user, team)      [AllowedDomains 驗證]
    └─ 呼叫 CreateTeam() + JoinUserToTeam()
    ↓
app/team.go:CreateTeam()                               [App Layer - 核心]
    └─ ch.srv.teamService.CreateTeam(rctx, team)
        └─ store.Team().Save(team)                     [DB 寫入]
               INSERT INTO Teams (...)
    ↓
app/team.go:JoinUserToTeam()                           [建立者加入]
    ├─ ABAC 存取評估（非公開團隊）
    ├─ RunMultiHook(TeamMemberWillBeAdded)             [Plugin Hook]
    ├─ teamService.JoinUserToTeam()
    │      └─ store.Team().SaveMember()                [DB 寫入]
    │             INSERT INTO TeamMembers (...)
    ├─ createInitialSidebarCategories()                [建立預設 Sidebar 分類]
    ├─ JoinDefaultChannels()                           [自動加入 town-square 等]
    ├─ RunMultiHook(UserHasJoinedTeam)                 [Plugin Hook]
    └─ 發布 WebSocket 事件：
           WebsocketEventAddedToTeam
    ↓
api4/team.go:createTeam()
    └─ 回傳 JSON: *model.Team (201 Created)
```

---

## 二、加入團隊（新增成員）

### 入口端點

```
POST /api/v4/teams/{team_id}/members          ← 直接新增（需管理員）
POST /api/v4/teams/members/invite             ← 透過 invite token 或 invite_id
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 使用者加入公開團隊（自己） | `PermissionJoinPublicTeams` |
| 使用者加入私密團隊（自己） | `PermissionJoinPrivateTeams` |
| 管理員新增其他使用者 | `PermissionAddUserToTeam` |
| 邀請訪客帳號 | `PermissionInviteGuest` |

### 呼叫鏈

```
api4/team.go:addTeamMember()                           [HTTP Handler]
    ↓
    ├─ 權限檢查（依情境，見上表）
    ├─ Group-constrained 檢查：FilterNonGroupTeamMembers()
    └─ 呼叫 c.App.AddTeamMember()
    ↓
app/team.go:AddTeamMember()
    └─ AddUserToTeam(rctx, teamID, userID, "")
    ↓
app/team.go:AddUserToTeam()
    ├─ 平行取得 store.Team().Get(teamID)
    ├─ 平行取得 store.User().Get(userID)
    └─ JoinUserToTeam(rctx, team, user, requestorId)
    ↓
app/team.go:JoinUserToTeam()                           [核心加入邏輯]
    ├─ [ABAC] TeamAccessControlled(rctx, teamID)
    │      └─ 若啟用存取控制：
    │             acs.AccessEvaluation(
    │               resource=TeamId, action=Membership
    │             )                                    [拒絕則 403]
    ├─ RunMultiHook(TeamMemberWillBeAdded)             [Plugin Hook，可拒絕]
    ├─ teamService.JoinUserToTeam()
    │      └─ store.Team().SaveMember(member, maxUsersPerTeam)
    │             INSERT INTO TeamMembers (TeamId, UserId, Roles, ...)
    ├─ store.User().UpdateUpdateAt(userID)
    ├─ createInitialSidebarCategories(rctx, userID, teamID)
    ├─ JoinDefaultChannels(rctx, teamID, user, ...)   [加入預設頻道]
    ├─ Cache 失效：InvalidateCacheForUser(), ClearTeamMembersCache()
    ├─ RunMultiHook(UserHasJoinedTeam)                 [Plugin Hook，非同步]
    └─ 發布 WebSocket 事件：
           WebsocketEventAddedToTeam
    ↓
api4/team.go:addTeamMember()
    └─ 回傳 JSON: *model.TeamMember (201 Created)
```

### 批次新增成員

```
POST /api/v4/teams/{team_id}/members/batch

api4/team.go:addTeamMembers()
    └─ app.AddTeamMembers(rctx, teamID, userIDs, requestorId, graceful)
        ├─ 逐一呼叫 AddTeamMember()
        └─ graceful=true 時：收集錯誤但繼續處理其餘成員
```

### 透過邀請連結加入

```
POST /api/v4/teams/members/invite

api4/team.go:addUserToTeamFromInvite()
    ├─ 解析 invite_token 或 invite_id
    ├─ 若 invite_token（email invite）：
    │      store.Token().GetByValue(token)
    │      驗證 token 未過期
    │      app.AddUserToTeamByToken(rctx, userID, token)
    │          ├─ 拒絕 group-constrained 團隊
    │          └─ JoinUserToTeam()
    └─ 若 invite_id（invite link）：
           app.AddUserToTeamByInviteId(rctx, inviteId, userID)
               ├─ store.Team().GetByInviteId(inviteId)
               ├─ 拒絕 group-constrained 團隊
               └─ JoinUserToTeam()
```

---

## 三、查詢團隊

### 入口端點與呼叫鏈

```
GET /api/v4/teams/{team_id}

api4/team.go:getTeam()
    ├─ 公開團隊：SessionHasPermissionTo(PermissionListPublicTeams)
    ├─ 私密團隊：SessionHasPermissionToTeam(PermissionViewTeam)
    └─ app.GetTeam(teamID)
        └─ teamService.GetTeam(teamID)
            └─ store.Team().Get(teamID)
                   SELECT * FROM Teams WHERE Id = ? AND DeleteAt = 0
```

```
GET /api/v4/users/{user_id}/teams

api4/team.go:getTeamsForUser()
    └─ app.GetTeamsForUser(userID)
        └─ store.Team().GetTeamsByUserId(userID)
               SELECT Teams.*
               FROM Teams
               INNER JOIN TeamMembers ON Teams.Id = TeamMembers.TeamId
               WHERE TeamMembers.UserId = ? AND TeamMembers.DeleteAt = 0
               AND Teams.DeleteAt = 0
```

```
GET /api/v4/teams/{team_id}/members

api4/team.go:getTeamMembers()
    └─ app.GetTeamMembers(teamID, offset, limit, opts)
        └─ store.Team().GetMembers(teamID, offset, limit, opts)
               SELECT TeamMembers.*
               FROM TeamMembers
               LEFT JOIN Schemes ON TeamMembers.SchemeId = Schemes.Id
               WHERE TeamId = ? AND DeleteAt = 0
               LIMIT ? OFFSET ?
```

```
GET /api/v4/teams/{team_id}/stats

api4/team.go:getTeamStats()
    └─ app.GetTeamStats(teamID, restrictions)
        └─ store.Team().GetActiveMemberCount(teamID)
               SELECT COUNT(DISTINCT UserId)
               FROM TeamMembers
               WHERE TeamId = ? AND DeleteAt = 0
```

---

## 四、更新團隊

### 入口端點

```
PUT  /api/v4/teams/{team_id}         ← 完整替換
PUT  /api/v4/teams/{team_id}/patch   ← 部分更新（Patch）
PUT  /api/v4/teams/{team_id}/privacy ← 切換公開 / 私密
```

### 呼叫鏈

```
api4/team.go:updateTeam()                              [HTTP Handler]
    ↓
    ├─ SessionHasPermissionToTeam(PermissionManageTeam)
    ├─ 若更改 AllowOpenInvite 或 AllowedDomains：
    │      SessionHasPermissionToTeam(PermissionInviteUser)
    └─ 呼叫 c.App.UpdateTeam()
    ↓
app/team.go:UpdateTeam()
    ├─ teamService.UpdateTeam(team, UpdateOptions{Sanitized: true})
    │      └─ store.Team().Update(team)                [DB 更新]
    │             UPDATE Teams SET DisplayName=?, Name=?, ...
    │             WHERE Id = ?
    └─ sendTeamEvent(oldTeam, WebsocketEventUpdateTeam)
           WebsocketEvent 廣播給所有團隊成員
    ↓
api4/team.go:updateTeam()
    └─ 回傳 JSON: *model.Team (200 OK)
```

```
api4/team.go:patchTeam()                               [部分更新]
    ├─ SessionHasPermissionToTeam(PermissionManageTeam)
    └─ c.App.PatchTeam(teamID, patch)
        ├─ GetTeam(teamID)
        ├─ team.Patch(patch)                           [僅更新非 nil 欄位]
        ├─ UpdateTeam(patched)
        └─ 若團隊為 group-constrained：
               DeleteGroupConstrainedTeamMemberships() [移除不在 Group 的成員]
```

---

## 五、移除成員 / 離開團隊

### 入口端點

```
DELETE /api/v4/teams/{team_id}/members/{user_id}
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 使用者自己離開 | 無（隱性允許） |
| 移除其他成員 | `PermissionRemoveUserFromTeam` |
| Group-constrained 限制 | 只有本人或 Bot 可移除 |

### 呼叫鏈

```
api4/team.go:removeTeamMember()                        [HTTP Handler]
    ↓
    ├─ 若移除他人：SessionHasPermissionToTeam(PermissionRemoveUserFromTeam)
    ├─ Group-constrained 檢查：若非自己且非 bot → 403
    └─ 呼叫 c.App.RemoveUserFromTeam()
    ↓
app/team.go:RemoveUserFromTeam()
    ├─ 平行取得 store.Team().Get(teamID)
    ├─ 平行取得 store.User().Get(userID)
    └─ LeaveTeam(rctx, team, user, requestorId)
    ↓
app/team.go:LeaveTeam()                                [核心離開邏輯]
    ├─ GetTeamMember(rctx, teamID, userID)
    ├─ 取得使用者在該團隊所有頻道
    ├─ 逐一移除非 DM 頻道成員：
    │      removeChannelMembership(rctx, userID, channelID)
    ├─ 若開啟系統訊息：
    │      postLeaveTeamMessage() 或 postRemoveFromTeamMessage()
    ├─ teamService.RemoveTeamMember(rctx, teamMember)
    │      └─ store.Team().RemoveMember(teamID, userID)
    │             DELETE FROM TeamMembers
    │             WHERE TeamId = ? AND UserId = ?
    └─ postProcessTeamMemberLeave()
           ├─ RunMultiHook(UserHasLeftTeam)            [Plugin Hook]
           ├─ store.User().UpdateUpdateAt(userID)
           ├─ store.Channel().ClearSidebarOnTeamLeave(userID, teamID)
           ├─ store.Preference().DeleteCategory(userID, teamID)
           └─ Cache 失效：ClearSessionCacheForUser(), InvalidateCacheForUser()
    ↓
api4/team.go:removeTeamMember()
    └─ 回傳 204 No Content
```

---

## 六、更新成員角色

### 入口端點

```
PUT /api/v4/teams/{team_id}/members/{user_id}/roles
PUT /api/v4/teams/{team_id}/members/{user_id}/schemeRoles
```

### 呼叫鏈

```
api4/team.go:updateTeamMemberRoles()                   [HTTP Handler]
    ↓
    ├─ SessionHasPermissionToTeam(PermissionManageTeamRoles)
    └─ app.UpdateTeamMemberRoles(rctx, teamID, userID, newRoles)
    ↓
app/team.go:UpdateTeamMemberRoles()
    ├─ store.Team().GetMember(rctx, teamID, userID)
    ├─ 驗證 newRoles 合法性
    ├─ store.Team().UpdateMember(rctx, member)
    │      UPDATE TeamMembers
    │      SET Roles=?, SchemeUser=?, SchemeAdmin=?, SchemeGuest=?
    │      WHERE TeamId=? AND UserId=?
    └─ 發布 WebSocket 事件：
           WebsocketEventMemberroleUpdated
```

---

## 七、刪除與還原團隊

### 軟刪除

```
DELETE /api/v4/teams/{team_id}

api4/team.go:deleteTeam()
    ├─ SessionHasPermissionToTeam(PermissionManageTeam)
    └─ app.SoftDeleteTeam(teamID)
        ├─ store.Team().SoftDelete(teamID)
        │      UPDATE Teams SET DeleteAt = ? WHERE Id = ?
        ├─ 清除所有頻道（軟刪除）
        └─ 發布 WebSocket 事件：WebsocketEventDeleteTeam
```

### 永久刪除

```
DELETE /api/v4/teams/{team_id}?permanent=true

app.PermanentDeleteTeam(rctx, team)
    ├─ 刪除所有頻道（永久）
    ├─ store.Team().RemoveAllMembersByTeam(teamID)
    │      DELETE FROM TeamMembers WHERE TeamId = ?
    └─ store.Team().PermanentDelete(teamID)
           DELETE FROM Teams WHERE Id = ?
```

### 還原

```
POST /api/v4/teams/{team_id}/restore

app.RestoreTeam(teamID)
    ├─ store.Team().Get(teamID)
    ├─ store.Team().Update(team with DeleteAt=0)
    └─ 發布 WebSocket 事件：WebsocketEventRestoreTeam
```

---

## WebSocket 事件彙整

| 事件常數 | 觸發時機 |
|---------|---------|
| `WebsocketEventAddedToTeam` | 使用者加入團隊 |
| `WebsocketEventUpdateTeam` | 團隊資料更新、隱私切換 |
| `WebsocketEventUpdateTeamScheme` | 權限方案變更 |
| `WebsocketEventMemberroleUpdated` | 成員角色更新 |
| `WebsocketEventDeleteTeam` | 團隊軟刪除 |
| `WebsocketEventRestoreTeam` | 團隊還原 |

---

## Plugin Hook 彙整

| Hook | 時機 | 可否拒絕 |
|------|------|---------|
| `TeamMemberWillBeAdded` | 成員加入前 | 是（回傳錯誤即拒絕） |
| `UserHasJoinedTeam` | 成員加入後（非同步） | 否 |
| `UserHasLeftTeam` | 成員離開後（非同步） | 否 |

---

## 相關資料結構

### `model.Team` (`server/public/model/team.go`)

| 欄位 | 說明 |
|------|------|
| `Id` | 26 字元 ULID |
| `DisplayName` | 顯示名稱（最多 64 字元） |
| `Name` | URL slug（唯一，2-64 字元） |
| `Type` | `O`（公開）/ `I`（私密） |
| `Description` | 團隊描述 |
| `Email` | 聯絡 Email |
| `InviteId` | 邀請連結 token |
| `AllowOpenInvite` | 是否開放邀請連結 |
| `AllowedDomains` | Email 網域白名單（逗號分隔） |
| `SchemeId` | 權限方案 ID（可空） |
| `GroupConstrained` | 是否綁定 LDAP Group（可空） |
| `DeleteAt` | 軟刪除時間戳（0 = 未刪除） |

### `model.TeamMember` (`server/public/model/team_member.go`)

| 欄位 | 說明 |
|------|------|
| `TeamId` | 所屬 Team ID |
| `UserId` | 使用者 ID |
| `Roles` | 合併後的角色字串（SchemeRoles + ExplicitRoles） |
| `SchemeGuest` | 方案訪客角色 flag |
| `SchemeUser` | 方案一般使用者角色 flag |
| `SchemeAdmin` | 方案管理員角色 flag |
| `ExplicitRoles` | 額外自定義角色（非方案繼承） |
| `DeleteAt` | 軟刪除時間戳（0 = 有效成員） |
| `CreateAt` | 加入時間戳 |

---

## 角色與權限

### 內建角色常數

| 常數 | 說明 |
|------|------|
| `TeamUserRoleId` | 一般成員角色 |
| `TeamAdminRoleId` | 團隊管理員角色 |
| `TeamGuestRoleId` | 訪客角色 |

### 角色解析順序

```
TeamMember.Roles（查詢時動態組合）
    = 方案角色（SchemeGuest / SchemeUser / SchemeAdmin）
    + 明確角色（ExplicitRoles）
```

方案角色透過 `Teams.SchemeId → Schemes` 表取得預設角色名稱，  
Store 層在 `GetMember()` / `GetMembers()` 時以 LEFT JOIN 一併查詢。

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/team.go` | 所有 Team HTTP handlers |
| `server/channels/app/team.go` | 業務邏輯（CreateTeam、JoinUserToTeam、LeaveTeam 等） |
| `server/channels/store/sqlstore/team_store.go` | SQL 持久化（Teams / TeamMembers 表） |
| `server/public/model/team.go` | Team 資料模型與常數 |
| `server/public/model/team_member.go` | TeamMember 資料模型 |
| `server/channels/app/team_service.go` | teamService 介面（包裝 store） |
