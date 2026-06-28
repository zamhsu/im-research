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

| 常數名稱 | 事件字串值 | 廣播範圍 | 觸發時機 |
|--------|-----------|---------|---------|
| `WebsocketEventHello` | `hello` | user | 客戶端建立 WebSocket 連線後，Server 握手回應 |
| `WebsocketEventPosted` | `posted` | channel | 新訊息建立（`POST /posts`）時 |
| `WebsocketEventPostEdited` | `post_edited` | channel | 訊息內容被修改，或釘選/取消釘選時 |
| `WebsocketEventPostDeleted` | `post_deleted` | channel | 訊息被刪除（軟刪除，`delete_at` 設值）時 |
| `WebsocketEventEphemeralMessage` | `ephemeral_message` | user | 系統或插件發送不存 DB 的臨時訊息給特定使用者時 |
| `WebsocketEventPostUnread` | `post_unread` | user | 使用者手動將頻道標記為未讀時 |
| `WebsocketEventPostRevealed` | `post_revealed` | user | Burn-on-read 訊息被收件人首次閱覽時 |
| `WebsocketEventPostBurned` | `post_burned` | user | Burn-on-read 訊息超過有效期自動銷毀時 |
| `WebsocketEventBurnOnReadAllRevealed` | `burn_on_read_all_revealed` | user | 所有收件人皆已閱覽 burn-on-read 訊息時 |
| `WebsocketEventAcknowledgementAdded` | `post_acknowledgement_added` | channel | 使用者對 Priority 訊息按下確認（Acknowledge）時 |
| `WebsocketEventAcknowledgementRemoved` | `post_acknowledgement_removed` | channel | 使用者移除對訊息的確認時 |
| `WebsocketEventThreadUpdated` | `thread_updated` | user | Thread 有新回覆，或回覆被編輯/刪除時 |
| `WebsocketEventThreadFollowChanged` | `thread_follow_changed` | user | 使用者追蹤或取消追蹤 Thread 時 |
| `WebsocketEventThreadReadChanged` | `thread_read_changed` | user | 使用者標記 Thread 已讀（更新 `last_viewed_at`）時 |
| `WebsocketEventReactionAdded` | `reaction_added` | channel | 使用者對訊息新增 emoji reaction 時 |
| `WebsocketEventReactionRemoved` | `reaction_removed` | channel | 使用者移除訊息的 emoji reaction 時 |
| `WebsocketEventChannelCreated` | `channel_created` | user | 公開或私人頻道建立時（通知建立者） |
| `WebsocketEventChannelUpdated` | `channel_updated` | channel | 頻道名稱、header 或 purpose 更新時 |
| `WebsocketEventChannelDeleted` | `channel_deleted` | team/channel | 頻道被封存（軟刪除）時 |
| `WebsocketEventChannelRestored` | `channel_restored` | team/channel | 已封存頻道被還原時 |
| `WebsocketEventChannelConverted` | `channel_converted` | team | 頻道隱私從公開切換為私人（或反向）時 |
| `WebsocketEventChannelSchemeUpdated` | `channel_scheme_updated` | channel | 頻道套用的 Permission Scheme 變更時 |
| `WebsocketEventUserAdded` | `user_added` | channel | 使用者被加入頻道時 |
| `WebsocketEventUserRemoved` | `user_removed` | channel + user | 使用者自行離開或被移出頻道時（雙重廣播） |
| `WebsocketEventChannelMemberUpdated` | `channel_member_updated` | user | 頻道成員角色或通知偏好設定更新時 |
| `WebsocketEventDirectAdded` | `direct_added` | channel | 兩人之間首次建立 DM 頻道時 |
| `WebsocketEventGroupAdded` | `group_added` | channel | 新的 Group DM（多人私聊）頻道建立時 |
| `WebsocketEventMultipleChannelsViewed` | `multiple_channels_viewed` | user | 使用者批次更新多個頻道的已讀時間戳時 |
| `WebsocketEventChannelBookmarkCreated` | `channel_bookmark_created` | channel | 頻道書籤新增時 |
| `WebsocketEventChannelBookmarkUpdated` | `channel_bookmark_updated` | channel | 頻道書籤內容更新時 |
| `WebsocketEventChannelBookmarkDeleted` | `channel_bookmark_deleted` | channel | 頻道書籤刪除時 |
| `WebsocketEventChannelBookmarkSorted` | `channel_bookmark_sorted` | channel | 頻道書籤拖曳重新排序時 |
| `WebsocketEventChannelJoinRequestCreated` | `channel_join_request_created` | channel | 使用者申請加入受限或私人頻道時 |
| `WebsocketEventChannelJoinRequestUpdated` | `channel_join_request_updated` | user | 加入申請被核准或拒絕時（通知申請者） |
| `WebsocketEventSidebarCategoryCreated` | `sidebar_category_created` | user | 使用者建立 Sidebar 自訂分類時 |
| `WebsocketEventSidebarCategoryUpdated` | `sidebar_category_updated` | user | Sidebar 分類名稱、靜音/收合狀態或所含頻道更新時 |
| `WebsocketEventSidebarCategoryDeleted` | `sidebar_category_deleted` | user | Sidebar 自訂分類刪除時 |
| `WebsocketEventSidebarCategoryOrderUpdated` | `sidebar_category_order_updated` | user | Sidebar 分類拖曳重新排列順序時 |
| `WebsocketEventAddedToTeam` | `added_to_team` | user | 使用者被加入團隊時（通知被加入者） |
| `WebsocketEventLeaveTeam` | `leave_team` | team (omit user) | 使用者自行離開或被移出團隊時（廣播給其他成員） |
| `WebsocketEventUpdateTeam` | `update_team` | team | 團隊名稱、描述或設定更新時 |
| `WebsocketEventDeleteTeam` | `delete_team` | team | 團隊被刪除（軟刪除）時 |
| `WebsocketEventRestoreTeam` | `restore_team` | team | 已刪除團隊被還原時 |
| `WebsocketEventUpdateTeamScheme` | `update_team_scheme` | team | 團隊套用的 Permission Scheme 變更時 |
| `WebsocketEventMemberroleUpdated` | `memberrole_updated` | user | 使用者在團隊中的角色（如升為 team_admin）變更時 |
| `WebsocketEventNewUser` | `new_user` | global | 新使用者帳號在系統中建立時 |
| `WebsocketEventUserUpdated` | `user_updated` | global | 使用者個人資料更新時（名稱、頭像、職稱等） |
| `WebsocketEventUserRoleUpdated` | `user_role_updated` | user | 使用者系統角色變更時（如升為 system_admin） |
| `WebsocketEventUserActivationStatusChange` | `user_activation_status_change` | global | 使用者帳號被啟用或停用時 |
| `WebsocketEventGuestsDeactivated` | `guests_deactivated` | global | 系統全體訪客帳號被批次停用時 |
| `WebsocketEventTyping` | `typing` | channel (omit user) | 使用者在頻道輸入框開始輸入文字時（節流發送） |
| `WebsocketEventStatusChange` | `status_change` | user | 使用者在線狀態切換時（online/away/offline/dnd） |
| `WebsocketEventPreferencesChanged` | `preferences_changed` | user | 使用者更新個人偏好設定時 |
| `WebsocketEventPreferencesDeleted` | `preferences_deleted` | user | 使用者刪除個人偏好設定項目時 |
| `WebsocketEventPersistentNotificationTriggered` | `persistent_notification_triggered` | user | Priority 訊息重複通知排程觸發時 |
| `WebsocketEventDraftCreated` | `draft_created` | user | 使用者建立新草稿時（跨裝置同步） |
| `WebsocketEventDraftUpdated` | `draft_updated` | user | 使用者更新草稿內容時 |
| `WebsocketEventDraftDeleted` | `draft_deleted` | user | 草稿被刪除（含訊息發送後自動清除）時 |
| `WebsocketScheduledPostCreated` | `scheduled_post_created` | user | 使用者建立排程訊息時 |
| `WebsocketScheduledPostUpdated` | `scheduled_post_updated` | user | 排程訊息的內容或排程時間被修改時 |
| `WebsocketScheduledPostDeleted` | `scheduled_post_deleted` | user | 排程訊息被取消，或到期執行後自動移除時 |
| `WebsocketEventEmojiAdded` | `emoji_added` | global | 管理員或使用者新增自訂 Emoji 時 |
| `WebsocketEventRoleUpdated` | `role_updated` | global | 系統角色的權限清單被管理員修改時 |
| `WebsocketEventConfigChanged` | `config_changed` | global | 管理員在後台修改系統設定後儲存時 |
| `WebsocketEventLicenseChanged` | `license_changed` | global | 系統授權憑證更換或移除時 |
| `WebsocketEventPluginEnabled` | `plugin_enabled` | global | 插件被管理員啟用時 |
| `WebsocketEventPluginDisabled` | `plugin_disabled` | global | 插件被管理員停用時 |
| `WebsocketEventPluginStatusesChanged` | `plugin_statuses_changed` | global | 插件狀態批次變更（如伺服器重啟後重新掃描）時 |
| `WebsocketEventOpenDialog` | `open_dialog` | user | 斜線指令或互動按鈕觸發 Interactive Dialog 開啟時 |
| `WebsocketEventShowToast` | `show_toast` | user | 系統或插件推送 Toast 通知給特定使用者時 |
| `WebsocketEventReceivedGroup` | `received_group` | global | LDAP 群組同步後新增或更新群組資訊時 |
| `WebsocketEventReceivedGroupAssociatedToTeam` | `received_group_associated_to_team` | team | LDAP 群組與團隊建立綁定關係時 |
| `WebsocketEventReceivedGroupNotAssociatedToTeam` | `received_group_not_associated_to_team` | team | LDAP 群組與團隊解除綁定關係時 |
| `WebsocketEventReceivedGroupAssociatedToChannel` | `received_group_associated_to_channel` | channel | LDAP 群組與頻道建立綁定關係時 |
| `WebsocketEventReceivedGroupNotAssociatedToChannel` | `received_group_not_associated_to_channel` | channel | LDAP 群組與頻道解除綁定關係時 |
| `WebsocketEventGroupMemberAdd` | `group_member_add` | user | LDAP 同步後使用者被加入群組時 |
| `WebsocketEventGroupMemberDelete` | `group_member_deleted` | user | LDAP 同步後使用者被移出群組時 |
| `WebsocketEventCPAFieldCreated` | `custom_profile_attributes_field_created` | global | 管理員新增自訂個人屬性欄位定義時 |
| `WebsocketEventCPAFieldUpdated` | `custom_profile_attributes_field_updated` | global | 管理員修改自訂個人屬性欄位定義時 |
| `WebsocketEventCPAFieldDeleted` | `custom_profile_attributes_field_deleted` | global | 管理員刪除自訂個人屬性欄位定義時 |
| `WebsocketEventCPAValuesUpdated` | `custom_profile_attributes_values_updated` | user | 使用者儲存自訂個人屬性欄位的值時 |
| `WebsocketEventCloudSubscriptionChanged` | `cloud_subscription_changed` | global | Cloud 訂閱方案或授權狀態變更時 |
| `WebsocketEventRecapUpdated` | `recap_updated` | user | AI 頻道摘要（Recap）內容產生或更新完成時 |
| `WebsocketEventFileUploadRejected` | `file_upload_rejected` | user | 檔案上傳被病毒掃描或大小限制拒絕時 |
