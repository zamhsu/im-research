# 核心資料模型

所有資料模型定義在 `server/public/model/` 目錄下。

---

## User（使用者）

**檔案**：`server/public/model/user.go`  
**資料表**：`Users`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | 26 字元 ULID，主鍵 |
| `CreateAt` | int64 | 建立時間（毫秒 unix timestamp） |
| `UpdateAt` | int64 | 最後更新時間 |
| `DeleteAt` | int64 | 軟刪除時間（0 = 未刪除） |
| `Username` | string | 登入名稱（唯一） |
| `Password` | string | bcrypt 雜湊值 |
| `AuthData` | *string | SSO 的外部 ID |
| `AuthService` | string | 認證來源（空=本機, `ldap`, `saml`, `google`...） |
| `Email` | string | 電子郵件（唯一） |
| `EmailVerified` | bool | 是否已驗證 email |
| `Nickname` | string | 暱稱 |
| `FirstName` | string | 名字 |
| `LastName` | string | 姓氏 |
| `Position` | string | 職稱 |
| `Roles` | string | 系統角色（`system_admin`, `system_user`...） |
| `LastActivityAt` | int64 | 最後活動時間 |
| `MfaActive` | bool | 是否啟用 MFA |
| `MfaSecret` | string | TOTP secret |
| `Props` | StringMap | 延伸屬性 |
| `NotifyProps` | StringMap | 通知設定（desktop/push/email 等） |
| `Timezone` | StringMap | 時區設定 |
| `Locale` | string | 語言設定（如 `zh-TW`） |
| `LastPasswordUpdate` | int64 | 最後改密碼時間 |
| `FailedAttempts` | int | 登入失敗次數 |
| `IsBot` | bool | 是否為 Bot 帳號 |

---

## Session（登入 Session）

**檔案**：`server/public/model/session.go`  
**資料表**：`Sessions`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | Session ID |
| `Token` | string | 26 字元隨機 token，客戶端存於 cookie/header |
| `CreateAt` | int64 | 建立時間 |
| `ExpiresAt` | int64 | 到期時間（0 = 永不過期） |
| `LastActivityAt` | int64 | 最後活動時間 |
| `UserId` | string | 所屬使用者 ID |
| `DeviceId` | string | 行動裝置 ID |
| `Roles` | string | Session 擁有的角色 |
| `IsOAuth` | bool | 是否為 OAuth session |
| `Props` | StringMap | 平台、瀏覽器、OS 資訊 |

---

## Team（工作區）

**檔案**：`server/public/model/team.go`  
**資料表**：`Teams`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | 26 字元 ULID |
| `Name` | string | URL slug（唯一，小寫） |
| `DisplayName` | string | 顯示名稱 |
| `Type` | string | `O`=公開, `I`=邀請制 |
| `Description` | string | 說明 |
| `Email` | string | 管理員 email |
| `CreatorId` | string | 建立者 UserId |
| `InviteId` | string | 邀請連結 ID |
| `AllowOpenInvite` | bool | 允許公開連結加入 |
| `DeleteAt` | int64 | 軟刪除時間 |

---

## Channel（頻道）

**檔案**：`server/public/model/channel.go`  
**資料表**：`Channels`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | 26 字元 ULID |
| `CreateAt` | int64 | 建立時間 |
| `UpdateAt` | int64 | 更新時間 |
| `DeleteAt` | int64 | 軟刪除時間 |
| `TeamId` | string | 所屬 Team（DM/GM 為空） |
| `Type` | ChannelType | `O`/`P`/`D`/`G` |
| `DisplayName` | string | 顯示名稱 |
| `Name` | string | URL slug（在 Team 內唯一） |
| `Header` | string | 頻道 header 說明 |
| `Purpose` | string | 頻道目的 |
| `LastPostAt` | int64 | 最後一則訊息時間 |
| `TotalMsgCount` | int64 | 訊息總數 |
| `TotalMsgCountRoot` | int64 | 根訊息（非回覆）總數 |
| `CreatorId` | string | 建立者 |
| `SchemeId` | *string | 使用的 Permission Scheme |

---

## ChannelMember（頻道成員）

**檔案**：`server/public/model/channel_member.go`  
**資料表**：`ChannelMembers`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `ChannelId` | string | 頻道 ID（複合主鍵） |
| `UserId` | string | 使用者 ID（複合主鍵） |
| `Roles` | string | 頻道角色 |
| `LastViewedAt` | int64 | 最後讀取時間 |
| `MsgCount` | int64 | 已讀訊息數 |
| `MsgCountRoot` | int64 | 已讀根訊息數 |
| `MentionCount` | int64 | 未讀 @mention 次數 |
| `MentionCountRoot` | int64 | 未讀 @mention 根訊息次數 |
| `UrgentMentionCount` | int64 | 緊急 @mention 次數 |
| `NotifyProps` | StringMap | 個人通知設定 |
| `SchemeGuest` | bool | 是否為 Guest 身份 |
| `SchemeUser` | bool | 是否為一般成員 |
| `SchemeAdmin` | bool | 是否為頻道管理員 |

---

## Post（訊息）

**檔案**：`server/public/model/post.go`  
**資料表**：`Posts`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | 26 字元 ULID |
| `CreateAt` | int64 | 建立時間（顯示用） |
| `UpdateAt` | int64 | 最後更新時間 |
| `EditAt` | int64 | 最後編輯時間（0 = 未編輯） |
| `DeleteAt` | int64 | 軟刪除時間 |
| `UserId` | string | 發文者 |
| `ChannelId` | string | 所在頻道 |
| `RootId` | string | 執行緒根訊息 ID（回覆時填入） |
| `OriginalId` | string | 原始訊息 ID（移動時保留） |
| `Message` | string | Markdown 格式訊息內容 |
| `Type` | string | 訊息類型（空=一般，`system_*`=系統訊息） |
| `Props` | StringInterface | 延伸屬性（webhook、priority、reactions 等） |
| `Hashtags` | string | 訊息中的 hashtag |
| `FileIds` | StringArray | 附件檔案 ID 列表 |
| `PendingPostId` | string | 客戶端去重 ID |
| `ReplyCount` | int64 | 執行緒回覆數 |
| `LastReplyAt` | int64 | 最後回覆時間 |
| `Participants` | []*User | 參與執行緒的使用者（非 DB 欄位） |
| `IsPinned` | bool | 是否釘選 |
| `Metadata` | *PostMetadata | 延伸 metadata（embed、reactions、files） |

---

## FileInfo（檔案資訊）

**檔案**：`server/public/model/file_info.go`  
**資料表**：`FileInfo`

| 欄位 | 型別 | 說明 |
|------|------|------|
| `Id` | string | 26 字元 ULID |
| `CreatorId` | string | 上傳者 |
| `PostId` | string | 關聯訊息（上傳後綁定） |
| `ChannelId` | string | 所在頻道 |
| `Name` | string | 原始檔名 |
| `Extension` | string | 副檔名 |
| `Size` | int64 | 檔案大小（bytes） |
| `MimeType` | string | MIME type |
| `Width` / `Height` | int | 圖片尺寸 |
| `HasPreviewImage` | bool | 是否有縮圖 |
| `Path` | string | 儲存路徑 |
| `ThumbnailPath` | string | 縮圖路徑 |

---

## ID 格式

Mattermost 的所有實體 ID 使用 **ULID**（Universally Unique Lexicographically Sortable Identifier）：

- 26 字元
- 時間排序（前 10 字元為時間戳）
- URL-safe（無需 encoding）
- 例子：`2cx7ugm8bbyetfqkxwbcfpzjzr`

---

## 時間戳記格式

所有 `CreateAt`、`UpdateAt` 等時間欄位均為：
- **Unix 毫秒時間戳**（int64）
- 例：`1719340800000` = 2024-06-25 16:00:00 UTC
