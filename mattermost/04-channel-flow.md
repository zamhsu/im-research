# 頻道操作流程

## 頻道類型

| 類型常數 | 值 | 說明 |
|---------|-----|------|
| `ChannelTypeOpen` | `O` | 公開頻道，任何人可加入 |
| `ChannelTypePrivate` | `P` | 私人頻道，需要邀請 |
| `ChannelTypeDirect` | `D` | 兩人私訊 |
| `ChannelTypeGroup` | `G` | 群組私訊（3-8 人） |

---

## 一、建立頻道

### 入口端點

```
POST /api/v4/channels
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 建立公開頻道 | `PermissionCreatePublicChannel`（team-scoped） |
| 建立私人頻道 | `PermissionCreatePrivateChannel`（team-scoped） |
| 設定可探索私人頻道 | `PermissionManagePrivateChannelDiscoverability`（team-scoped） |

### 呼叫鏈

```
api4/channel.go:createChannel()                    [HTTP Handler]
    ↓
    解析 JSON body → *model.Channel
    ├─ 確認 TeamId 有效
    ├─ SessionHasPermissionToTeam(CreatePublicChannel / CreatePrivateChannel)
    └─ 呼叫 c.App.CreateChannelWithUser()
    ↓
app/channel.go:CreateChannelWithUser()             [App Layer]
    └─ 呼叫 CreateChannel(channel, addMember=true)
    ↓
app/channel.go:CreateChannel()                     [App Layer - 核心]
    ├─ 驗證 Team 存在
    ├─ 確認未超過最大頻道數限制
    ├─ Store.Channel().Save()                      [DB 寫入]
    │      INSERT INTO Channels (...)
    │
    ├─ 若 addMember=true：
    │      AddUserToChannel(creator)
    │      ├─ Store.ChannelMember().Save()         [DB 寫入]
    │      └─ Store.ChannelMemberHistory().LogJoinEvent()
    │
    ├─ 將頻道加入預設 sidebar 分類：
    │      addChannelToDefaultCategory()
    │
    ├─ 發布系統訊息（加入頻道通知）：
    │      postJoinChannelMessage()
    │
    ├─ RunMultiHook(ChannelHasBeenCreated)         [Plugin Hook，非同步]
    │
    └─ 發布 WebSocket 事件：
           WebsocketEventChannelCreated
    ↓
api4/channel.go:createChannel()                    [HTTP Handler]
    └─ 回傳 JSON: *model.Channel (201 Created)
```

---

## 二、加入頻道

### 入口端點

```
POST /api/v4/channels/{channel_id}/members
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 加入公開頻道（自己） | `PermissionJoinPublicChannels`（team-scoped） |
| 管理員新增他人至公開頻道 | `PermissionManagePublicChannelMembers`（channel-scoped） |
| 管理員新增他人至私人頻道 | `PermissionManagePrivateChannelMembers`（channel-scoped） |
| 可探索私人頻道 | 禁止直接 API 新增，需走申請加入流程 |

### 呼叫鏈

```
api4/channel.go:addChannelMember()                 [HTTP Handler]
    ↓
    解析 user_id
    ├─ 確認 channel 存在
    ├─ 權限檢查（依情境，見上表）
    └─ 呼叫 c.App.AddUserToChannel()
    ↓
app/channel.go:AddUserToChannel()                  [App Layer]
    ├─ 確認使用者已加入對應的 Team
    ├─ 確認使用者未被封鎖
    ├─ Store.ChannelMember().SaveMember()           [DB 寫入]
    │      INSERT INTO ChannelMembers (...)
    │
    ├─ Store.ChannelMemberHistory().LogJoinEvent() [DB 寫入]
    │
    ├─ 更新頻道成員數快取
    ├─ 發送加入通知訊息
    ├─ RunMultiHook(UserHasJoinedChannel)           [Plugin Hook，非同步]
    └─ 發布 WebSocket 事件：
           WebsocketEventUserAdded
    ↓
api4/channel.go:addChannelMember()
    └─ 回傳 JSON: *model.ChannelMember (201 Created)
```

---

## 三、標記頻道已讀（ViewChannel）

### 入口端點

```
POST /api/v4/channels/members/{user_id}/view
```

### 呼叫鏈

```
api4/channel.go:viewChannel()                      [HTTP Handler]
    ↓
app/channel.go:ViewChannel()                       [App Layer]
    ├─ Store.ChannelMember().UpdateLastViewedAt()  [DB 更新]
    ├─ 清除 MentionCount（未讀提及數）
    └─ 發布 WebSocket 事件：
           WebsocketEventMultipleChannelsViewed
```

---

## 四、建立私訊（Direct Message）

### 入口端點

```
POST /api/v4/channels/direct
```

### 特殊邏輯

```
api4/channel.go:createDirectChannel()             [HTTP Handler]
    ↓
app/channel.go:GetOrCreateDirectChannel()         [App Layer]
    ├─ 計算唯一 DM 頻道名稱（兩個 userId 排序後串接）
    ├─ Store.Channel().GetByName() — 嘗試找現有頻道
    │      若已存在 → 直接回傳
    │
    └─ 若不存在 → CreateChannel()
           ├─ Type = "D"
           ├─ 自動加入兩位使用者為成員
           ├─ RunMultiHook(ChannelHasBeenCreated)   [Plugin Hook]
           └─ 發布 WebSocket 事件：
                  WebsocketEventDirectAdded
```

---

## 五、查詢頻道

### 入口端點與呼叫鏈

```
GET /api/v4/channels/{channel_id}

api4/channel.go:getChannel()
    ├─ 公開頻道：SessionHasPermissionToTeam(PermissionReadPublicChannel)
    │            或 SessionHasPermissionToChannel(PermissionReadChannel)
    ├─ 私人頻道：SessionHasPermissionToChannel(PermissionReadChannel)
    └─ app.GetChannel(channelId)
        ├─ store.Channel().Get(channelId)
        │      SELECT Channels.*, ChannelScheme.*, TeamScheme.*
        │      FROM Channels
        │      LEFT JOIN Schemes ChannelScheme ON Channels.SchemeId = ChannelScheme.Id
        │      LEFT JOIN Teams ON Channels.TeamId = Teams.Id
        │      LEFT JOIN Schemes TeamScheme ON Teams.SchemeId = TeamScheme.Id
        │      WHERE Channels.Id = ?
        └─ app.FillInChannelProps()                [補齊 DisplayName 等衍生欄位]
```

```
GET /api/v4/teams/{team_id}/channels

api4/channel.go:getPublicChannelsForTeam()
    ├─ SessionHasPermissionToTeam(PermissionListTeamChannels)
    └─ app.GetPublicChannelsForTeam(teamId, page, perPage)
        └─ store.Channel().GetPublicChannelsForTeam()
               SELECT * FROM Channels
               WHERE TeamId = ? AND Type = 'O' AND DeleteAt = 0
               ORDER BY DisplayName
               LIMIT ? OFFSET ?
```

```
GET /api/v4/users/{user_id}/teams/{team_id}/channels

api4/channel.go:getChannelsForTeamForUser()
    ├─ SessionHasPermissionTo(PermissionEditOtherUsers)   [查其他使用者時]
    ├─ SessionHasPermissionToTeam(PermissionViewTeam)
    └─ app.GetChannelsForTeamForUser(teamId, userId, opts)
        └─ store.Channel().GetChannelsForTeamForUser()
               SELECT * FROM Channels
               INNER JOIN ChannelMembers ON Channels.Id = ChannelMembers.ChannelId
               WHERE Channels.TeamId = ? AND ChannelMembers.UserId = ?
               AND Channels.DeleteAt = 0
```

```
GET /api/v4/channels/{channel_id}/members

api4/channel.go:getChannelMembers()
    ├─ SessionHasPermissionToChannel(PermissionReadChannel)
    └─ app.GetChannelMembersPage(channelId, page, perPage)
        └─ store.Channel().GetMembers(opts)
               SELECT ChannelMembers.*,
                      TeamScheme.DefaultChannelGuestRole, ...,
                      ChannelScheme.DefaultChannelGuestRole, ...
               FROM ChannelMembers
               INNER JOIN Channels ON ChannelMembers.ChannelId = Channels.Id
               LEFT JOIN Schemes ChannelScheme ON Channels.SchemeId = ChannelScheme.Id
               LEFT JOIN Teams ON Channels.TeamId = Teams.Id
               LEFT JOIN Schemes TeamScheme ON Teams.SchemeId = TeamScheme.Id
               WHERE ChannelMembers.ChannelId = ?
               LIMIT ? OFFSET ?
```

```
GET /api/v4/channels/{channel_id}/stats

api4/channel.go:getChannelStats()
    ├─ SessionHasPermissionToChannel(PermissionReadChannel)
    └─ app.GetChannelMemberCount() + GetChannelGuestCount()
       + GetChannelPinnedPostCount() + GetChannelFileCount()
       回傳 ChannelStats{MemberCount, GuestCount, PinnedPostCount, FilesCount}
```

---

## 六、更新頻道

### 入口端點

```
PUT  /api/v4/channels/{channel_id}          ← 完整替換
PUT  /api/v4/channels/{channel_id}/patch    ← 部分更新（Patch）
PUT  /api/v4/channels/{channel_id}/privacy  ← 切換公開 / 私人
```

### 呼叫鏈

```
api4/channel.go:updateChannel()                    [HTTP Handler]
    ↓
    ├─ 公開頻道：SessionHasPermissionToChannel(PermissionManagePublicChannelProperties)
    ├─ 私人頻道：SessionHasPermissionToChannel(PermissionManagePrivateChannelProperties)
    └─ app.UpdateChannel(channel)
        ├─ runGuardedChannelWillBeUpdated()        [Plugin Hook 前置檢查]
        ├─ ChannelAccessControlled()               [ABAC 存取評估]
        ├─ store.Channel().Update(channel)         [DB 更新]
        │      UPDATE Channels SET DisplayName=?, Header=?, Purpose=?, ...
        │      WHERE Id = ?
        ├─ InvalidateCacheForChannel()
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelUpdated
    ↓
api4/channel.go:updateChannel()
    └─ 回傳 JSON: *model.Channel (200 OK)
```

```
api4/channel.go:patchChannel()                     [部分更新]
    ├─ 公開頻道屬性：  PermissionManagePublicChannelProperties
    ├─ 私人頻道屬性：  PermissionManagePrivateChannelProperties
    ├─ 公開頻道自動翻譯：PermissionManagePublicChannelAutoTranslation
    ├─ 私人頻道自動翻譯：PermissionManagePrivateChannelAutoTranslation
    ├─ 可探索設定：    PermissionManagePrivateChannelDiscoverability
    └─ app.PatchChannel(channel, patch, userId)
        ├─ runGuardedChannelWillBeUpdated()
        ├─ store.Channel().Update(patched)
        ├─ 若 Header / Purpose / DisplayName 有變更：
        │      postSystemMessage(HeaderChange / PurposeChange / DisplayNameChange)
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelUpdated
```

```
api4/channel.go:updateChannelPrivacy()             [切換隱私]
    ├─ 轉為公開：PermissionConvertPrivateChannelToPublic
    ├─ 轉為私人：PermissionConvertPublicChannelToPrivate
    └─ app.UpdateChannelPrivacy(channel, user)
        ├─ store.Channel().Update(channel with new Type)
        ├─ 更新 SchemeId（如有需要）
        ├─ postSystemMessage(ConvertedPrivate / ConvertedPublic)
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelConverted
```

---

## 七、移除頻道成員

### 入口端點

```
DELETE /api/v4/channels/{channel_id}/members/{user_id}
```

### 權限檢查

| 情境 | 所需權限 |
|------|---------|
| 使用者自己離開 | 無（隱性允許） |
| 移除他人（公開頻道） | `PermissionManagePublicChannelMembers`（channel-scoped） |
| 移除他人（私人頻道） | `PermissionManagePrivateChannelMembers`（channel-scoped） |
| Group-constrained 限制 | 只有本人或 Bot 可移除 |

### 呼叫鏈

```
api4/channel.go:removeChannelMember()              [HTTP Handler]
    ↓
    ├─ 若移除他人：SessionHasPermissionToChannel(PermissionManagePublicChannelMembers
    │              或 PermissionManagePrivateChannelMembers)
    ├─ Group-constrained 檢查：若非自己且非 bot → 拒絕
    └─ app.RemoveUserFromChannel(userId, requestorId, channel)
    ↓
app/channel.go:RemoveUserFromChannel()
    └─ removeUserFromChannel()                     [內部核心函式]
        ├─ 驗證非 DM / Group（不可移除）
        ├─ 驗證 group-constrained 規則
        ├─ removeChannelMembership()               [刪除 ChannelMembers 記錄]
        │      DELETE FROM ChannelMembers
        │      WHERE ChannelId = ? AND UserId = ?
        ├─ ChannelMemberHistory.LogLeaveEvent()    [DB 寫入]
        ├─ 若為訪客且無其他頻道 → 自動移出 Team
        ├─ 清除快取：InvalidateChannelCacheForUser()
        ├─ RunMultiHook(UserHasLeftChannel)        [Plugin Hook，非同步]
        └─ 發布 WebSocket 事件：
               WebsocketEventUserRemoved          [廣播給頻道成員]
               WebsocketEventUserRemoved          [發給當事使用者]
    ↓
    ├─ 若自行離開：postLeaveChannelMessage()
    └─ 若被移除：  postRemoveFromChannelMessage()
    ↓
api4/channel.go:removeChannelMember()
    └─ 回傳 200 OK
```

---

## 八、更新成員角色

### 入口端點

```
PUT /api/v4/channels/{channel_id}/members/{user_id}/roles
```

### 呼叫鏈

```
api4/channel.go:updateChannelMemberRoles()         [HTTP Handler]
    ↓
    ├─ SessionHasPermissionToChannel(PermissionManageChannelRoles)
    └─ app.UpdateChannelMemberRoles(channelId, userId, newRoles)
    ↓
app/channel.go:UpdateChannelMemberRoles()
    └─ updateChannelMemberRolesInternal()
        ├─ GetChannelMember(channelId, userId)
        ├─ GetSchemeRolesForChannel(channelId)
        │      ├─ 若頻道有 SchemeId → 使用頻道方案角色名稱
        │      └─ 否則 → 使用 Team 方案或預設角色名稱
        ├─ 對每個 role 名稱：
        │      ├─ 若為方案角色 → 設定 SchemeUser / SchemeAdmin / SchemeGuest flag
        │      └─ 若為明確角色 → 加入 ExplicitRoles 字串
        ├─ 驗證：不可同時為 SchemeUser + SchemeGuest
        ├─ 驗證：不可變更 SchemeGuest 狀態
        ├─ store.Channel().UpdateMember(member)
        │      UPDATE ChannelMembers
        │      SET Roles=?, SchemeUser=?, SchemeAdmin=?, SchemeGuest=?
        │      WHERE ChannelId=? AND UserId=?
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelMemberUpdated
```

---

## 九、刪除與還原頻道

### 軟刪除

```
DELETE /api/v4/channels/{channel_id}

api4/channel.go:deleteChannel()
    ├─ 公開頻道：SessionHasPermissionToChannel(PermissionDeletePublicChannel)
    ├─ 私人頻道：SessionHasPermissionToChannel(PermissionDeletePrivateChannel)
    └─ app.DeleteChannel(channel, userId)
        ├─ RunMultiHook(ChannelWillBeArchived)     [Plugin Hook，可拒絕]
        ├─ store.Channel().Delete(channelId, deleteAt)
        │      UPDATE Channels SET DeleteAt = ?, UpdateAt = ? WHERE Id = ?
        │      UPDATE PublicChannels SET DeleteAt = ? WHERE Id = ?
        ├─ 軟刪除所有 Incoming / Outgoing Webhooks
        ├─ 刪除持久通知（PersistentNotification）
        ├─ cleanupChannelAccessControlPolicy()     [移除 ABAC policy]
        ├─ postSystemMessage("已封存")
        ├─ InvalidateCacheForChannel()
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelDeleted       [公開頻道廣播至 teamId scope]
                                                   [私人頻道廣播至 channelId scope]
```

### 永久刪除

```
DELETE /api/v4/channels/{channel_id}?permanent=true

app.PermanentDeleteChannel(channel)
    ├─ PermanentDeleteMembersByChannel()
    │      DELETE FROM ChannelMembers WHERE ChannelId = ?
    └─ store.Channel().PermanentDelete(channelId)
           DELETE FROM Channels WHERE Id = ?
```

### 還原

```
POST /api/v4/channels/{channel_id}/restore

api4/channel.go:restoreChannel()
    ├─ SessionHasPermissionToTeam(PermissionManageTeam)
    │  或 PermissionSysconsoleWriteUserManagementChannels
    └─ app.RestoreChannel(channel, userId)
        ├─ runGuardedChannelWillBeRestored()       [Plugin Hook 前置]
        ├─ store.Channel().Restore(channelId, updateAt)
        │      UPDATE Channels SET DeleteAt = 0, UpdateAt = ? WHERE Id = ?
        ├─ InvalidateCacheForChannel()
        ├─ postSystemMessage("已取消封存")
        └─ 發布 WebSocket 事件：
               WebsocketEventChannelRestored      [公開頻道廣播至 teamId scope]
                                                   [私人頻道廣播至 channelId scope]
```

---

## WebSocket 事件彙整

| 事件常數 | 觸發時機 |
|---------|---------|
| `WebsocketEventChannelCreated` | 頻道建立（發給建立者） |
| `WebsocketEventChannelUpdated` | 頻道屬性更新 |
| `WebsocketEventChannelConverted` | 頻道隱私切換（公開 ↔ 私人） |
| `WebsocketEventChannelDeleted` | 頻道軟刪除 |
| `WebsocketEventChannelRestored` | 頻道還原 |
| `WebsocketEventChannelSchemeUpdated` | 頻道權限方案變更 |
| `WebsocketEventChannelMemberUpdated` | 頻道成員角色更新 |
| `WebsocketEventUserAdded` | 使用者加入頻道 |
| `WebsocketEventUserRemoved` | 使用者離開 / 被移出頻道 |
| `WebsocketEventDirectAdded` | 新建 DM 頻道 |
| `WebsocketEventGroupAdded` | 新建 Group DM 頻道 |
| `WebsocketEventMultipleChannelsViewed` | 頻道標記已讀 |

---

## Plugin Hook 彙整

| Hook | 時機 | 可否拒絕 |
|------|------|---------|
| `ChannelHasBeenCreated` | 頻道建立後（非同步） | 否 |
| `ChannelWillBeArchived` | 頻道軟刪除前 | 是（回傳非空字串即拒絕） |
| `ChannelWillBeRestored` | 頻道還原前 | 否 |
| `UserHasJoinedChannel` | 使用者加入頻道後（非同步） | 否 |
| `UserHasLeftChannel` | 使用者離開頻道後（非同步） | 否 |

---

## 相關資料結構

### `model.Channel` (`server/public/model/channel.go`)

| 欄位 | 說明 |
|------|------|
| `Id` | 26 字元 ULID |
| `TeamId` | 所屬 Team（DM 為空） |
| `Type` | `O` / `P` / `D` / `G` |
| `Name` | URL slug（小寫、連字號） |
| `DisplayName` | 顯示名稱 |
| `CreatorId` | 建立者 UserId |
| `SchemeId` | 頻道層級權限方案 ID（可空） |
| `GroupConstrained` | 是否綁定 LDAP Group（可空） |
| `LastPostAt` | 最後一則訊息時間 |
| `TotalMsgCount` | 訊息總數 |
| `DeleteAt` | 軟刪除時間（0 = 未刪除） |

### `model.ChannelMember` (`server/public/model/channel_member.go`)

| 欄位 | 說明 |
|------|------|
| `ChannelId` | 頻道 ID |
| `UserId` | 使用者 ID |
| `Roles` | 明確角色字串（SchemeRoles + ExplicitRoles） |
| `SchemeGuest` | 方案訪客角色 flag |
| `SchemeUser` | 方案一般成員角色 flag |
| `SchemeAdmin` | 方案管理員角色 flag |
| `ExplicitRoles` | 額外自定義角色（非方案繼承） |
| `LastViewedAt` | 最後讀取時間 |
| `MentionCount` | 未讀 @mention 次數 |
| `NotifyProps` | 個人通知設定（desktop / push / etc） |

---

## 角色與權限

### 內建角色常數

| 常數 | 值 | 說明 |
|------|-----|------|
| `ChannelGuestRoleId` | `channel_guest` | 訪客角色 |
| `ChannelUserRoleId` | `channel_user` | 一般成員角色 |
| `ChannelAdminRoleId` | `channel_admin` | 頻道管理員角色 |

### 角色解析順序

```
ChannelMember.Roles（查詢時動態組合）
    = 方案角色（SchemeGuest / SchemeUser / SchemeAdmin）
    + 明確角色（ExplicitRoles）
```

方案角色優先序（`app/channel.go:GetSchemeRolesForChannel()`）：

```
1. 若 channel.SchemeId 存在 → 使用頻道方案的 DefaultChannelGuestRole / UserRole / AdminRole
2. 若 team.SchemeId 存在   → 使用 Team 方案的 DefaultChannelGuestRole / UserRole / AdminRole
3. 否則                   → 使用預設值 "channel_guest" / "channel_user" / "channel_admin"
```

Store 層在 `GetMembers()` 時以 LEFT JOIN 同時取得 ChannelScheme 與 TeamScheme：

```sql
SELECT ChannelMembers.*,
       TeamScheme.DefaultChannelGuestRole, TeamScheme.DefaultChannelUserRole, ...
       ChannelScheme.DefaultChannelGuestRole, ChannelScheme.DefaultChannelUserRole, ...
FROM ChannelMembers
INNER JOIN Channels ON ChannelMembers.ChannelId = Channels.Id
LEFT JOIN Schemes ChannelScheme ON Channels.SchemeId = ChannelScheme.Id
LEFT JOIN Teams ON Channels.TeamId = Teams.Id
LEFT JOIN Schemes TeamScheme ON Teams.SchemeId = TeamScheme.Id
```

### 角色更新限制

- 不可同時為 `SchemeUser` 和 `SchemeGuest`
- 不可透過此端點變更 `SchemeGuest` 狀態
- 成員必須至少擁有 `SchemeUser` 或 `SchemeGuest` 其一

---

## 相關檔案路徑

| 檔案 | 職責 |
|------|------|
| `server/channels/api4/channel.go` | 所有頻道 HTTP handlers |
| `server/channels/app/channel.go:159` | `CreateChannelWithUser()` |
| `server/channels/app/channel.go:235` | `CreateChannel()` 核心邏輯 |
| `server/channels/app/channel.go:1890` | `AddUserToChannel()` |
| `server/channels/app/channel.go:2635` | `JoinChannel()` |
| `server/channels/app/channel.go:1698` | `DeleteChannel()` 軟刪除 |
| `server/channels/app/channel.go:944` | `RestoreChannel()` 還原 |
| `server/channels/app/channel.go:1075` | `GetSchemeRolesForChannel()` 角色解析 |
| `server/channels/store/sqlstore/channel_store.go` | SQL 持久化 |
| `server/public/model/channel.go` | Channel 資料模型 |
| `server/public/model/channel_member.go` | ChannelMember 資料模型 |
