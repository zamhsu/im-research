# WebSocket 事件目錄

## 訊息結構

所有 WebSocket 事件使用 JSON 格式傳輸，外層 envelope 固定如下：

```json
{
  "event": "<事件名稱>",
  "data": { ... },
  "broadcast": {
    "omit_users":         { "<userId>": true },
    "user_id":            "<userId 或空>",
    "channel_id":         "<channelId 或空>",
    "team_id":            "<teamId 或空>",
    "connection_id":      "<connectionId 或空>",
    "omit_connection_id": "<connectionId 或空>",
    "contains_sanitized_data": false,
    "contains_sensitive_data": false
  },
  "seq": 42
}
```

### broadcast 廣播範圍說明

| broadcast 欄位有值 | 廣播對象 |
|------------------|---------|
| `channel_id` 有值 | 該頻道所有連線成員 |
| `team_id` 有值 | 該團隊所有連線成員 |
| `user_id` 有值 | 指定使用者的所有連線 |
| `connection_id` 有值 | 單一 WebSocket 連線 |
| 全部為空 | 全伺服器廣播 |
| `omit_users` 有值 | 排除特定使用者 |

---

## 一、連線事件

### `hello`
客戶端建立連線後，Server 立即送出的握手訊息。

```json
{
  "event": "hello",
  "data": {
    "server_version": "9.10.0.1.1",
    "connection_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 0
}
```

---

## 二、訊息（Post）事件

### `posted`
新訊息建立，廣播給頻道所有成員。

```json
{
  "event": "posted",
  "data": {
    "channel_type":         "O",
    "channel_display_name": "General",
    "channel_name":         "town-square",
    "sender_name":          "@alice",
    "team_id":              "<teamId>",
    "set_online":           true,
    "post": "{\"id\":\"<postId>\",\"channel_id\":\"<channelId>\",\"user_id\":\"<userId>\",\"message\":\"Hello!\",\"type\":\"\",\"root_id\":\"\",\"create_at\":1700000000000,\"update_at\":1700000000000,\"delete_at\":0,\"props\":{},\"file_ids\":[],\"is_pinned\":false}"
  },
  "broadcast": {
    "channel_id": "<channelId>",
    "omit_users": { "<senderId>": true }
  },
  "seq": 1
}
```

> `post` 欄位值為 JSON 字串（二次序列化），客戶端需再次 `JSON.parse()`。

### `post_edited`
訊息內容更新（含釘選/取消釘選）。

```json
{
  "event": "post_edited",
  "data": {
    "post": "{\"id\":\"<postId>\",\"message\":\"Hello (edited)!\",\"edit_at\":1700000001000, ...}"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 2
}
```

### `post_deleted`
訊息刪除。

```json
{
  "event": "post_deleted",
  "data": {
    "post":      "{\"id\":\"<postId>\",\"delete_at\":1700000002000, ...}",
    "delete_by": "<deletedByUserId>"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 3
}
```

> `delete_by` 只在管理員刪除他人訊息時存在。

### `ephemeral_message`
暫時訊息，只送給指定使用者，不存 DB。

```json
{
  "event": "ephemeral_message",
  "data": {
    "post": "{\"id\":\"\",\"channel_id\":\"<channelId>\",\"user_id\":\"<targetUserId>\",\"message\":\"Only you can see this.\",\"type\":\"ephemeral\", ...}"
  },
  "broadcast": { "user_id": "<targetUserId>" },
  "seq": 4
}
```

### `post_unread`
標記訊息為未讀。

```json
{
  "event": "post_unread",
  "data": {
    "team_id":           "<teamId>",
    "channel_id":        "<channelId>",
    "last_viewed_at":    1699999990000,
    "mention_count":     1,
    "mention_count_root": 1,
    "msg_count":         10,
    "msg_count_root":    10,
    "urgent_mention_count": 0
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 5
}
```

### `post_revealed`
Burn-on-read 訊息被使用者閱覽。

```json
{
  "event": "post_revealed",
  "data": {
    "post":       "{...}",
    "recipients": ["<userId1>", "<userId2>"]
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 6
}
```

### `post_burned`
Burn-on-read 訊息已過期銷毀。

```json
{
  "event": "post_burned",
  "data": {
    "post_id": "<postId>"
  },
  "broadcast": { "user_id": "<authorUserId>" },
  "seq": 7
}
```

### `burn_on_read_all_revealed`
所有收件人皆已閱覽 burn-on-read 訊息。

```json
{
  "event": "burn_on_read_all_revealed",
  "data": {
    "post_id":         "<postId>",
    "sender_expire_at": 1700003600000
  },
  "broadcast": { "user_id": "<authorUserId>" },
  "seq": 8
}
```

### `post_acknowledgement_added`
使用者確認（acknowledge）訊息。

```json
{
  "event": "post_acknowledgement_added",
  "data": {
    "acknowledgement": {
      "user_id":          "<userId>",
      "post_id":          "<postId>",
      "acknowledged_at":  1700000010000
    }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 9
}
```

### `post_acknowledgement_removed`
使用者移除確認。

```json
{
  "event": "post_acknowledgement_removed",
  "data": {
    "acknowledgement": {
      "user_id": "<userId>",
      "post_id": "<postId>",
      "acknowledged_at": 0
    }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 10
}
```

---

## 三、執行緒（Thread）事件

### `thread_updated`
Thread 有新回覆或狀態變更。

```json
{
  "event": "thread_updated",
  "data": {
    "thread_id":   "<rootPostId>",
    "state":       "following",
    "reply_count": 5,
    "channel_id":  "<channelId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 11
}
```

### `thread_follow_changed`
使用者追蹤/取消追蹤 Thread。

```json
{
  "event": "thread_follow_changed",
  "data": {
    "thread_id":   "<rootPostId>",
    "state":       "following",
    "reply_count": 5
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 12
}
```

### `thread_read_changed`
Thread 已讀狀態更新。

```json
{
  "event": "thread_read_changed",
  "data": {
    "thread_id":               "<rootPostId>",
    "timestamp":               1700000005000,
    "unread_mentions":         0,
    "unread_replies":          0,
    "previous_unread_mentions": 2,
    "previous_unread_replies":  3,
    "channel_id":              "<channelId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 13
}
```

---

## 四、Reaction 事件

### `reaction_added`

```json
{
  "event": "reaction_added",
  "data": {
    "reaction": "{\"user_id\":\"<userId>\",\"post_id\":\"<postId>\",\"emoji_name\":\"thumbsup\",\"create_at\":1700000020000}"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 14
}
```

### `reaction_removed`

```json
{
  "event": "reaction_removed",
  "data": {
    "reaction": "{\"user_id\":\"<userId>\",\"post_id\":\"<postId>\",\"emoji_name\":\"thumbsup\",\"create_at\":1700000020000}"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 15
}
```

---

## 五、頻道事件

### `channel_created`

```json
{
  "event": "channel_created",
  "data": {
    "channel_id": "<channelId>",
    "team_id":    "<teamId>"
  },
  "broadcast": { "user_id": "<creatorUserId>" },
  "seq": 16
}
```

### `channel_updated`

```json
{
  "event": "channel_updated",
  "data": {
    "channel": "{\"id\":\"<channelId>\",\"display_name\":\"New Name\",\"header\":\"...\",\"purpose\":\"...\", ...}"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 17
}
```

### `channel_deleted`

```json
{
  "event": "channel_deleted",
  "data": {
    "channel_id": "<channelId>",
    "delete_at":  1700000030000
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 18
}
```

> 私人頻道改用 `channel_id` 廣播（不用 `team_id`）。

### `channel_restored`

```json
{
  "event": "channel_restored",
  "data": {
    "channel_id": "<channelId>"
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 19
}
```

### `channel_converted`
頻道隱私切換（公開 ↔ 私人）。

```json
{
  "event": "channel_converted",
  "data": {
    "channel_id": "<channelId>"
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 20
}
```

### `channel_scheme_updated`

```json
{
  "event": "channel_scheme_updated",
  "data": {},
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 21
}
```

### `user_added`（加入頻道）

```json
{
  "event": "user_added",
  "data": {
    "user_id": "<addedUserId>",
    "team_id": "<teamId>"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 22
}
```

### `user_removed`（移出頻道）

廣播給頻道其他成員：
```json
{
  "event": "user_removed",
  "data": {
    "user_id":    "<removedUserId>",
    "remover_id": "<removerUserId>"
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 23
}
```

同時送給被移除使用者：
```json
{
  "event": "user_removed",
  "data": {
    "user_id":    "<removedUserId>",
    "remover_id": "<removerUserId>"
  },
  "broadcast": { "user_id": "<removedUserId>" },
  "seq": 24
}
```

### `channel_member_updated`

```json
{
  "event": "channel_member_updated",
  "data": {
    "channelMember": "{\"channel_id\":\"<channelId>\",\"user_id\":\"<userId>\",\"roles\":\"channel_user channel_admin\",\"scheme_admin\":true, ...}"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 25
}
```

### `direct_added`
新 DM 頻道建立。

```json
{
  "event": "direct_added",
  "data": {
    "teammate_id": "<otherUserId>"
  },
  "broadcast": { "channel_id": "<dmChannelId>" },
  "seq": 26
}
```

### `group_added`
新 Group DM 頻道建立。

```json
{
  "event": "group_added",
  "data": {
    "teammate_ids": ["<userId1>", "<userId2>"]
  },
  "broadcast": { "channel_id": "<groupChannelId>" },
  "seq": 27
}
```

### `multiple_channels_viewed`

```json
{
  "event": "multiple_channels_viewed",
  "data": {
    "channel_times": {
      "<channelId1>": 1700000050000,
      "<channelId2>": 1700000051000
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 28
}
```

### `channel_bookmark_created`

```json
{
  "event": "channel_bookmark_created",
  "data": {
    "bookmark": {
      "id":          "<bookmarkId>",
      "channel_id":  "<channelId>",
      "owner_id":    "<userId>",
      "display_name":"Useful Link",
      "link_url":    "https://example.com",
      "type":        "link",
      "emoji":       ":bookmark:",
      "sort_order":  0,
      "create_at":   1700000060000,
      "update_at":   1700000060000,
      "delete_at":   0
    }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 29
}
```

### `channel_bookmark_updated`

```json
{
  "event": "channel_bookmark_updated",
  "data": {
    "bookmarks": {
      "updated": { "id": "<bookmarkId>", "display_name": "Updated Name", ... },
      "deleted": null
    }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 30
}
```

### `channel_bookmark_deleted`

```json
{
  "event": "channel_bookmark_deleted",
  "data": {
    "bookmark": { "id": "<bookmarkId>", "delete_at": 1700000070000, ... }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 31
}
```

### `channel_bookmark_sorted`

```json
{
  "event": "channel_bookmark_sorted",
  "data": {
    "bookmarks": [
      { "id": "<bookmarkId1>", "sort_order": 0, ... },
      { "id": "<bookmarkId2>", "sort_order": 1, ... }
    ]
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 32
}
```

### `channel_join_request_created`

```json
{
  "event": "channel_join_request_created",
  "data": {
    "channel_id": "<channelId>",
    "request": {
      "id":         "<requestId>",
      "channel_id": "<channelId>",
      "user_id":    "<requestingUserId>",
      "create_at":  1700000080000,
      "status":     "pending"
    }
  },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 33
}
```

### `channel_join_request_updated`

```json
{
  "event": "channel_join_request_updated",
  "data": {
    "channel_id": "<channelId>",
    "request": {
      "id":       "<requestId>",
      "status":   "approved",
      "reviewer_user_id": "<reviewerUserId>"
    }
  },
  "broadcast": { "user_id": "<requestingUserId>" },
  "seq": 34
}
```

---

## 六、Sidebar / 分類事件

### `sidebar_category_created`

```json
{
  "event": "sidebar_category_created",
  "data": {
    "category_id": "<categoryId>",
    "team_id":     "<teamId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 35
}
```

### `sidebar_category_updated`

```json
{
  "event": "sidebar_category_updated",
  "data": {
    "updatedCategories": [
      {
        "id":          "<categoryId>",
        "user_id":     "<userId>",
        "team_id":     "<teamId>",
        "type":        "custom",
        "display_name":"My Category",
        "muted":       false,
        "collapsed":   false,
        "channels":    ["<channelId1>", "<channelId2>"]
      }
    ]
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 36
}
```

### `sidebar_category_deleted`

```json
{
  "event": "sidebar_category_deleted",
  "data": {
    "category_id": "<categoryId>",
    "team_id":     "<teamId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 37
}
```

### `sidebar_category_order_updated`

```json
{
  "event": "sidebar_category_order_updated",
  "data": {
    "order":   ["<categoryId1>", "<categoryId2>", "<categoryId3>"],
    "team_id": "<teamId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 38
}
```

---

## 七、團隊事件

### `added_to_team`

```json
{
  "event": "added_to_team",
  "data": {
    "team_id": "<teamId>",
    "user_id": "<addedUserId>"
  },
  "broadcast": { "user_id": "<addedUserId>" },
  "seq": 39
}
```

### `leave_team`

廣播給團隊其他成員：
```json
{
  "event": "leave_team",
  "data": {
    "team_id": "<teamId>",
    "user_id": "<leavingUserId>"
  },
  "broadcast": {
    "team_id":    "<teamId>",
    "omit_users": { "<leavingUserId>": true }
  },
  "seq": 40
}
```

### `update_team`

```json
{
  "event": "update_team",
  "data": {
    "team": "{\"id\":\"<teamId>\",\"display_name\":\"My Team\",\"type\":\"O\", ...}"
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 41
}
```

### `delete_team`

```json
{
  "event": "delete_team",
  "data": {
    "team": "{\"id\":\"<teamId>\",\"delete_at\":1700000100000, ...}"
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 42
}
```

### `restore_team`

```json
{
  "event": "restore_team",
  "data": {
    "team": "{\"id\":\"<teamId>\",\"delete_at\":0, ...}"
  },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 43
}
```

### `update_team_scheme`

```json
{
  "event": "update_team_scheme",
  "data": {},
  "broadcast": { "team_id": "<teamId>" },
  "seq": 44
}
```

### `memberrole_updated`
成員在 Team 的角色變更。

```json
{
  "event": "memberrole_updated",
  "data": {
    "member": "{\"team_id\":\"<teamId>\",\"user_id\":\"<userId>\",\"roles\":\"team_user team_admin\",\"scheme_admin\":true, ...}"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 45
}
```

---

## 八、使用者事件

### `new_user`

```json
{
  "event": "new_user",
  "data": {
    "user_id": "<newUserId>"
  },
  "broadcast": {},
  "seq": 46
}
```

### `user_updated`

```json
{
  "event": "user_updated",
  "data": {
    "user": {
      "id":           "<userId>",
      "username":     "alice",
      "first_name":   "Alice",
      "last_name":    "Smith",
      "nickname":     "",
      "email":        "",
      "position":     "Engineer",
      "roles":        "system_user",
      "locale":       "en",
      "timezone":     { "automaticTimezone": "Asia/Taipei", "useAutomaticTimezone": "true" },
      "last_picture_update": 1700000110000
    }
  },
  "broadcast": {},
  "seq": 47
}
```

> 非系統管理員收到的 `user` 物件會隱藏 email 等敏感欄位（`contains_sanitized_data: true`）。

### `user_role_updated`

```json
{
  "event": "user_role_updated",
  "data": {
    "user_id": "<userId>",
    "roles":   "system_user system_admin"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 48
}
```

### `user_activation_status_change`

```json
{
  "event": "user_activation_status_change",
  "data": {},
  "broadcast": {},
  "seq": 49
}
```

### `guests_deactivated`

```json
{
  "event": "guests_deactivated",
  "data": {},
  "broadcast": {},
  "seq": 50
}
```

---

## 九、輸入狀態 & 在線狀態

### `typing`

```json
{
  "event": "typing",
  "data": {
    "user_id":   "<typingUserId>",
    "parent_id": "<rootPostId 或空>"
  },
  "broadcast": {
    "channel_id": "<channelId>",
    "omit_users": { "<typingUserId>": true }
  },
  "seq": 51
}
```

### `status_change`

```json
{
  "event": "status_change",
  "data": {
    "status":  "online",
    "user_id": "<userId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 52
}
```

> `status` 可為 `"online"` / `"away"` / `"offline"` / `"dnd"`。

---

## 十、通知 & 偏好設定事件

### `preferences_changed`

```json
{
  "event": "preferences_changed",
  "data": {
    "preferences": "[{\"user_id\":\"<userId>\",\"category\":\"display_settings\",\"name\":\"message_display\",\"value\":\"compact\"}]"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 53
}
```

### `preferences_deleted`

```json
{
  "event": "preferences_deleted",
  "data": {
    "preferences": "[{\"user_id\":\"<userId>\",\"category\":\"<category>\",\"name\":\"<name>\"}]"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 54
}
```

### `persistent_notification_triggered`
Priority 訊息的持續通知。

```json
{
  "event": "persistent_notification_triggered",
  "data": {
    "post":         "{...}",
    "channel_type": "O"
  },
  "broadcast": { "user_id": "<targetUserId>" },
  "seq": 55
}
```

---

## 十一、Draft 草稿事件

### `draft_created`

```json
{
  "event": "draft_created",
  "data": {
    "draft": {
      "id":          "<draftId>",
      "channel_id":  "<channelId>",
      "root_id":     "",
      "user_id":     "<userId>",
      "message":     "Work in progress...",
      "props":       {},
      "file_ids":    [],
      "create_at":   1700000120000,
      "update_at":   1700000120000,
      "delete_at":   0
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 56
}
```

### `draft_updated`

```json
{
  "event": "draft_updated",
  "data": {
    "draft": { "id": "<draftId>", "message": "Updated draft...", "update_at": 1700000130000, ... }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 57
}
```

### `draft_deleted`

```json
{
  "event": "draft_deleted",
  "data": {
    "draft": { "id": "<draftId>", "delete_at": 1700000140000, ... }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 58
}
```

---

## 十二、排程訊息事件

### `scheduled_post_created`

```json
{
  "event": "scheduled_post_created",
  "data": {
    "scheduledPost": {
      "id":            "<scheduledPostId>",
      "channel_id":    "<channelId>",
      "user_id":       "<userId>",
      "message":       "Reminder: meeting at 3pm",
      "scheduled_at":  1700090000000,
      "create_at":     1700000150000,
      "update_at":     1700000150000
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 59
}
```

### `scheduled_post_updated`

```json
{
  "event": "scheduled_post_updated",
  "data": {
    "scheduledPost": { "id": "<scheduledPostId>", "scheduled_at": 1700093600000, ... }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 60
}
```

### `scheduled_post_deleted`

```json
{
  "event": "scheduled_post_deleted",
  "data": {
    "scheduledPost": { "id": "<scheduledPostId>", ... }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 61
}
```

---

## 十三、Emoji 事件

### `emoji_added`

```json
{
  "event": "emoji_added",
  "data": {
    "emoji": {
      "id":          "<emojiId>",
      "name":        "partycorgi",
      "creator_id":  "<userId>",
      "create_at":   1700000160000,
      "delete_at":   0
    }
  },
  "broadcast": {},
  "seq": 62
}
```

---

## 十四、角色 & 權限事件

### `role_updated`

```json
{
  "event": "role_updated",
  "data": {
    "role": {
      "id":          "<roleId>",
      "name":        "channel_admin",
      "display_name":"Channel Admin Role",
      "permissions": ["create_post", "edit_post", "delete_post", "manage_public_channel_members"],
      "scheme_managed": true
    }
  },
  "broadcast": {},
  "seq": 63
}
```

---

## 十五、設定 & 授權事件

### `config_changed`

```json
{
  "event": "config_changed",
  "data": {
    "config": {
      "ServiceSettings": { "SiteURL": "https://mattermost.example.com", ... },
      "TeamSettings":    { "MaxUsersPerTeam": 50, ... }
    }
  },
  "broadcast": {},
  "seq": 64
}
```

> 只包含客戶端允許看到的欄位（敏感設定已過濾）。

### `license_changed`

```json
{
  "event": "license_changed",
  "data": {
    "license": {
      "Id":         "<licenseId>",
      "IssuedAt":   1700000000000,
      "ExpiresAt":  1731628800000,
      "Features":   { "Elasticsearch": true, "LDAP": true, ... }
    }
  },
  "broadcast": {},
  "seq": 65
}
```

---

## 十六、Plugin 事件

### `plugin_enabled`

```json
{
  "event": "plugin_enabled",
  "data": {
    "manifest": {
      "id":          "com.mattermost.calls",
      "name":        "Calls",
      "version":     "0.26.0",
      "min_server_version": "7.6.0"
    }
  },
  "broadcast": {},
  "seq": 66
}
```

### `plugin_disabled`

```json
{
  "event": "plugin_disabled",
  "data": {
    "manifest": { "id": "com.mattermost.calls", "name": "Calls", ... }
  },
  "broadcast": {},
  "seq": 67
}
```

### `plugin_statuses_changed`

```json
{
  "event": "plugin_statuses_changed",
  "data": {},
  "broadcast": {},
  "seq": 68
}
```

---

## 十七、互動 UI 事件

### `open_dialog`
打開 Interactive Dialog（斜線指令或按鈕觸發）。

```json
{
  "event": "open_dialog",
  "data": {
    "dialog": {
      "trigger_id":   "<triggerId>",
      "url":          "https://example.com/dialog/submit",
      "dialog": {
        "callback_id": "my-dialog",
        "title":       "Create Issue",
        "elements": [
          { "type": "text", "name": "title", "display_name": "Title", "optional": false },
          { "type": "textarea", "name": "description", "display_name": "Description" }
        ],
        "submit_label": "Submit",
        "notify_on_cancel": true
      }
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 69
}
```

### `show_toast`

```json
{
  "event": "show_toast",
  "data": {
    "title":    "Action completed",
    "message":  "Your request has been processed.",
    "url":      "/team/channel",
    "duration": 5000
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 70
}
```

---

## 十八、LDAP Group 事件

### `received_group`

```json
{
  "event": "received_group",
  "data": {
    "group": {
      "id":            "<groupId>",
      "name":          "engineering",
      "display_name":  "Engineering",
      "source":        "ldap",
      "remote_id":     "cn=engineering,ou=groups,dc=example,dc=com",
      "create_at":     1700000200000,
      "delete_at":     0,
      "has_syncables": true
    }
  },
  "broadcast": {},
  "seq": 71
}
```

### `received_group_associated_to_team`

```json
{
  "event": "received_group_associated_to_team",
  "data": { "group_id": "<groupId>" },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 72
}
```

### `received_group_not_associated_to_team`

```json
{
  "event": "received_group_not_associated_to_team",
  "data": { "group_id": "<groupId>" },
  "broadcast": { "team_id": "<teamId>" },
  "seq": 73
}
```

### `received_group_associated_to_channel`

```json
{
  "event": "received_group_associated_to_channel",
  "data": { "group_id": "<groupId>" },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 74
}
```

### `received_group_not_associated_to_channel`

```json
{
  "event": "received_group_not_associated_to_channel",
  "data": { "group_id": "<groupId>" },
  "broadcast": { "channel_id": "<channelId>" },
  "seq": 75
}
```

### `group_member_add`

```json
{
  "event": "group_member_add",
  "data": {
    "groupMember": {
      "group_id":  "<groupId>",
      "user_id":   "<userId>",
      "create_at": 1700000210000,
      "delete_at": 0
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 76
}
```

### `group_member_deleted`

```json
{
  "event": "group_member_deleted",
  "data": {
    "groupMember": {
      "group_id":  "<groupId>",
      "user_id":   "<userId>",
      "delete_at": 1700000220000
    }
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 77
}
```

---

## 十九、自訂個人屬性（CPA）事件

### `custom_profile_attributes_field_created`

```json
{
  "event": "custom_profile_attributes_field_created",
  "data": {
    "object_type":    "user",
    "property_field": {
      "id":          "<fieldId>",
      "name":        "Department",
      "type":        "text",
      "attrs": { "sort_order": 0, "visibility": "when_set" }
    }
  },
  "broadcast": {},
  "seq": 78
}
```

### `custom_profile_attributes_field_updated`

```json
{
  "event": "custom_profile_attributes_field_updated",
  "data": {
    "object_type":    "user",
    "property_field": { "id": "<fieldId>", "name": "Department (updated)", ... }
  },
  "broadcast": {},
  "seq": 79
}
```

### `custom_profile_attributes_field_deleted`

```json
{
  "event": "custom_profile_attributes_field_deleted",
  "data": {
    "object_type": "user",
    "field_id":    "<fieldId>"
  },
  "broadcast": {},
  "seq": 80
}
```

### `custom_profile_attributes_values_updated`

```json
{
  "event": "custom_profile_attributes_values_updated",
  "data": {
    "target_id":       "<userId>",
    "property_values": "{\"<fieldId1>\":\"Engineering\",\"<fieldId2>\":\"Taiwan\"}"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 81
}
```

---

## 二十、Cloud & 其他事件

### `cloud_subscription_changed`

```json
{
  "event": "cloud_subscription_changed",
  "data": {
    "subscription": {
      "product_id":   "<productId>",
      "seats":        50,
      "status":       "active",
      "trial_end_at": 0
    }
  },
  "broadcast": {},
  "seq": 82
}
```

### `recap_updated`

```json
{
  "event": "recap_updated",
  "data": {
    "recap_id": "<recapId>"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 83
}
```

### `file_upload_rejected`

```json
{
  "event": "file_upload_rejected",
  "data": {
    "filename": "malware.exe",
    "reason":   "File failed virus scan"
  },
  "broadcast": { "user_id": "<userId>" },
  "seq": 84
}
```

---

## 附錄：事件常數對照表

| 常數名稱 | 事件字串值 | 廣播範圍 |
|--------|-----------|---------|
| `WebsocketEventHello` | `hello` | user |
| `WebsocketEventPosted` | `posted` | channel |
| `WebsocketEventPostEdited` | `post_edited` | channel |
| `WebsocketEventPostDeleted` | `post_deleted` | channel |
| `WebsocketEventEphemeralMessage` | `ephemeral_message` | user |
| `WebsocketEventPostUnread` | `post_unread` | user |
| `WebsocketEventPostRevealed` | `post_revealed` | user |
| `WebsocketEventPostBurned` | `post_burned` | user |
| `WebsocketEventBurnOnReadAllRevealed` | `burn_on_read_all_revealed` | user |
| `WebsocketEventAcknowledgementAdded` | `post_acknowledgement_added` | channel |
| `WebsocketEventAcknowledgementRemoved` | `post_acknowledgement_removed` | channel |
| `WebsocketEventThreadUpdated` | `thread_updated` | user |
| `WebsocketEventThreadFollowChanged` | `thread_follow_changed` | user |
| `WebsocketEventThreadReadChanged` | `thread_read_changed` | user |
| `WebsocketEventReactionAdded` | `reaction_added` | channel |
| `WebsocketEventReactionRemoved` | `reaction_removed` | channel |
| `WebsocketEventChannelCreated` | `channel_created` | user |
| `WebsocketEventChannelUpdated` | `channel_updated` | channel |
| `WebsocketEventChannelDeleted` | `channel_deleted` | team/channel |
| `WebsocketEventChannelRestored` | `channel_restored` | team/channel |
| `WebsocketEventChannelConverted` | `channel_converted` | team |
| `WebsocketEventChannelSchemeUpdated` | `channel_scheme_updated` | channel |
| `WebsocketEventUserAdded` | `user_added` | channel |
| `WebsocketEventUserRemoved` | `user_removed` | channel + user |
| `WebsocketEventChannelMemberUpdated` | `channel_member_updated` | user |
| `WebsocketEventDirectAdded` | `direct_added` | channel |
| `WebsocketEventGroupAdded` | `group_added` | channel |
| `WebsocketEventMultipleChannelsViewed` | `multiple_channels_viewed` | user |
| `WebsocketEventChannelBookmarkCreated` | `channel_bookmark_created` | channel |
| `WebsocketEventChannelBookmarkUpdated` | `channel_bookmark_updated` | channel |
| `WebsocketEventChannelBookmarkDeleted` | `channel_bookmark_deleted` | channel |
| `WebsocketEventChannelBookmarkSorted` | `channel_bookmark_sorted` | channel |
| `WebsocketEventChannelJoinRequestCreated` | `channel_join_request_created` | channel |
| `WebsocketEventChannelJoinRequestUpdated` | `channel_join_request_updated` | user |
| `WebsocketEventSidebarCategoryCreated` | `sidebar_category_created` | user |
| `WebsocketEventSidebarCategoryUpdated` | `sidebar_category_updated` | user |
| `WebsocketEventSidebarCategoryDeleted` | `sidebar_category_deleted` | user |
| `WebsocketEventSidebarCategoryOrderUpdated` | `sidebar_category_order_updated` | user |
| `WebsocketEventAddedToTeam` | `added_to_team` | user |
| `WebsocketEventLeaveTeam` | `leave_team` | team (omit user) |
| `WebsocketEventUpdateTeam` | `update_team` | team |
| `WebsocketEventDeleteTeam` | `delete_team` | team |
| `WebsocketEventRestoreTeam` | `restore_team` | team |
| `WebsocketEventUpdateTeamScheme` | `update_team_scheme` | team |
| `WebsocketEventMemberroleUpdated` | `memberrole_updated` | user |
| `WebsocketEventNewUser` | `new_user` | global |
| `WebsocketEventUserUpdated` | `user_updated` | global |
| `WebsocketEventUserRoleUpdated` | `user_role_updated` | user |
| `WebsocketEventUserActivationStatusChange` | `user_activation_status_change` | global |
| `WebsocketEventGuestsDeactivated` | `guests_deactivated` | global |
| `WebsocketEventTyping` | `typing` | channel (omit user) |
| `WebsocketEventStatusChange` | `status_change` | user |
| `WebsocketEventPreferencesChanged` | `preferences_changed` | user |
| `WebsocketEventPreferencesDeleted` | `preferences_deleted` | user |
| `WebsocketEventPersistentNotificationTriggered` | `persistent_notification_triggered` | user |
| `WebsocketEventDraftCreated` | `draft_created` | user |
| `WebsocketEventDraftUpdated` | `draft_updated` | user |
| `WebsocketEventDraftDeleted` | `draft_deleted` | user |
| `WebsocketScheduledPostCreated` | `scheduled_post_created` | user |
| `WebsocketScheduledPostUpdated` | `scheduled_post_updated` | user |
| `WebsocketScheduledPostDeleted` | `scheduled_post_deleted` | user |
| `WebsocketEventEmojiAdded` | `emoji_added` | global |
| `WebsocketEventRoleUpdated` | `role_updated` | global |
| `WebsocketEventConfigChanged` | `config_changed` | global |
| `WebsocketEventLicenseChanged` | `license_changed` | global |
| `WebsocketEventPluginEnabled` | `plugin_enabled` | global |
| `WebsocketEventPluginDisabled` | `plugin_disabled` | global |
| `WebsocketEventPluginStatusesChanged` | `plugin_statuses_changed` | global |
| `WebsocketEventOpenDialog` | `open_dialog` | user |
| `WebsocketEventShowToast` | `show_toast` | user |
| `WebsocketEventReceivedGroup` | `received_group` | global |
| `WebsocketEventReceivedGroupAssociatedToTeam` | `received_group_associated_to_team` | team |
| `WebsocketEventReceivedGroupNotAssociatedToTeam` | `received_group_not_associated_to_team` | team |
| `WebsocketEventReceivedGroupAssociatedToChannel` | `received_group_associated_to_channel` | channel |
| `WebsocketEventReceivedGroupNotAssociatedToChannel` | `received_group_not_associated_to_channel` | channel |
| `WebsocketEventGroupMemberAdd` | `group_member_add` | user |
| `WebsocketEventGroupMemberDelete` | `group_member_deleted` | user |
| `WebsocketEventCPAFieldCreated` | `custom_profile_attributes_field_created` | global |
| `WebsocketEventCPAFieldUpdated` | `custom_profile_attributes_field_updated` | global |
| `WebsocketEventCPAFieldDeleted` | `custom_profile_attributes_field_deleted` | global |
| `WebsocketEventCPAValuesUpdated` | `custom_profile_attributes_values_updated` | user |
| `WebsocketEventCloudSubscriptionChanged` | `cloud_subscription_changed` | global |
| `WebsocketEventRecapUpdated` | `recap_updated` | user |
| `WebsocketEventFileUploadRejected` | `file_upload_rejected` | user |
