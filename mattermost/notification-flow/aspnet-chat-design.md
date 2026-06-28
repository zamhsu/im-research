# 簡易即時通訊服務設計（ASP.NET Core）

參考 Mattermost 架構，實作四個核心功能：
- 加入 / 離開公開 Team
- 加入 / 離開公開 Channel
- Channel 文字聊天
- 使用者一對一聊天（Direct Message）

---

## 技術選擇

| 層面 | 選擇 | 說明 |
|---|---|---|
| Framework | ASP.NET Core 8 | |
| 即時通訊 | **ASP.NET Core SignalR** | 內建 Hub、自動重連、支援 Redis backplane |
| REST API | Minimal API / Controller | 非即時操作（加入 team、查歷史訊息等）走 REST |
| 資料庫 | PostgreSQL + EF Core | 參考 Mattermost 使用 PostgreSQL |
| 快取 | Redis | Session、線上狀態、叢集 backplane |
| 認證 | JWT Bearer | stateless，適合 WebSocket 升級時傳 token |

> **為什麼用 SignalR 而不是裸 WebSocket？**
> 這是精簡版，SignalR 已內建連線管理、Groups（對應 Mattermost 的 channel 廣播）、自動重連、Redis backplane（叢集支援）。省去自刻 Hub、ConnectionIndex 的時間，適合先把功能做完。若之後需要 dead queue、broadcast hook、per-connection filtering，再考慮換成自訂 WebSocket Hub。

---

## 資料模型

### 核心 Entity

```
User
├─ Id (Guid)
├─ Username (string)
└─ DisplayName (string)

Team
├─ Id (Guid)
├─ Name (string)
├─ DisplayName (string)
└─ IsPublic (bool)

TeamMember
├─ TeamId (Guid) → Team
├─ UserId (Guid) → User
└─ JoinedAt (DateTimeOffset)

Channel
├─ Id (Guid)
├─ TeamId (Guid) → Team        ← null 表示 DM channel
├─ Name (string)
├─ DisplayName (string)
├─ Type (enum: Public / Direct)
└─ IsPublic (bool)

ChannelMember
├─ ChannelId (Guid) → Channel
├─ UserId (Guid) → User
└─ LastReadAt (DateTimeOffset)  ← 用於計算未讀數

Post
├─ Id (Guid)
├─ ChannelId (Guid) → Channel
├─ UserId (Guid) → User
├─ Message (string)
├─ CreatedAt (DateTimeOffset)
└─ UpdatedAt (DateTimeOffset?)
```

### DM Channel 的設計

Mattermost 將 DM 也視為一種 Channel（Type = Direct），由兩個 ChannelMember 組成。這樣 Post 的結構統一，查歷史訊息的邏輯也一致。

```
建立 DM：
1. 建立 Channel（Type = Direct, TeamId = null）
2. 建立兩筆 ChannelMember（userA, userB）
3. 之後雙方都透過這個 channelId 發訊息
```

---

## REST API 設計

### 認證

```
POST /api/auth/login
     Body: { username, password }
     Response: { token: "jwt..." }
```

JWT 帶在 `Authorization: Bearer <token>` header，WebSocket 連線時帶在 query string：`?access_token=<token>`

---

### Team

```
GET    /api/teams                        列出所有公開 team
POST   /api/teams/{teamId}/members       加入 team
DELETE /api/teams/{teamId}/members/me    離開 team
GET    /api/teams/{teamId}/channels      列出該 team 的公開 channel
```

---

### Channel

```
GET    /api/channels/{channelId}                    取得 channel 資訊
POST   /api/teams/{teamId}/channels/{channelId}/members    加入 channel
DELETE /api/teams/{teamId}/channels/{channelId}/members/me 離開 channel
GET    /api/channels/{channelId}/posts?before=<postId>&limit=50  取得歷史訊息（分頁）
```

---

### Direct Message

```
POST /api/direct-messages
     Body: { targetUserId: "..." }
     Response: { channelId: "..." }   ← 建立或取得既有的 DM channel
```

---

### Post（送訊息走 WebSocket，REST 只提供歷史查詢）

```
GET /api/channels/{channelId}/posts?before=<postId>&limit=50
```

---

## WebSocket（SignalR Hub）設計

### 連線

```
wss://host/hubs/chat?access_token=<jwt>
```

連線後 Server 自動將這條連線加入該 user 所有已加入 channel 的 SignalR Group。

---

### Client → Server 方法（Invoke）

| 方法 | 參數 | 說明 |
|---|---|---|
| `SendMessage` | `channelId, message` | 在 channel / DM 發訊息 |
| `MarkRead` | `channelId, postId` | 標記已讀 |

---

### Server → Client 事件（On）

| 事件名稱 | Payload | 觸發時機 |
|---|---|---|
| `PostCreated` | `{ postId, channelId, userId, message, createdAt }` | 有新訊息 |
| `UserJoinedChannel` | `{ channelId, userId }` | 有人加入 channel |
| `UserLeftChannel` | `{ channelId, userId }` | 有人離開 channel |
| `UserOnline` | `{ userId }` | 有人上線 |
| `UserOffline` | `{ userId }` | 有人離線 |

---

## 核心流程

### 1. 加入 Team

```
Client                   REST API              DB
  │                          │                  │
  │  POST /teams/{id}/members│                  │
  │─────────────────────────>│                  │
  │                          │  INSERT TeamMember
  │                          │─────────────────>│
  │                          │  SELECT public channels in team
  │                          │─────────────────>│
  │                          │  INSERT ChannelMember (預設加入 Town Square)
  │                          │─────────────────>│
  │<─────────────────────────│                  │
  │  200 OK                  │                  │
```

> Mattermost 加入 team 時會自動加入 Town Square（預設 channel）。精簡版可以保留這個行為。

---

### 2. 加入 Channel

```
Client                   REST API           DB              SignalR Hub
  │                          │               │                   │
  │  POST /channels/{id}/members             │                   │
  │─────────────────────────>│               │                   │
  │                          │  INSERT ChannelMember
  │                          │──────────────>│                   │
  │                          │               │                   │
  │                          │  AddToGroupAsync(connId, channelId)
  │                          │──────────────────────────────────>│
  │                          │               │                   │
  │                          │  Clients.Group(channelId).UserJoinedChannel(...)
  │                          │──────────────────────────────────>│
  │                          │               │         廣播給 channel 所有成員
  │<─────────────────────────│               │                   │
  │  200 OK                  │               │                   │
```

---

### 3. 發送 Channel 訊息

```
Client               SignalR Hub          DB             Group Members
  │                       │               │                   │
  │  Invoke SendMessage    │               │                   │
  │  (channelId, message)  │               │                   │
  │──────────────────────>│               │                   │
  │                       │  驗證 user 是 ChannelMember
  │                       │──────────────>│                   │
  │                       │  INSERT Post  │                   │
  │                       │──────────────>│                   │
  │                       │               │                   │
  │                       │  Clients.Group(channelId)         │
  │                       │  .PostCreated(post)               │
  │                       │──────────────────────────────────>│
  │<──────────────────────│               │          每個在線成員收到 PostCreated
  │  (自己也收到同一則事件)  │               │
```

---

### 4. 一對一聊天（DM）

```
Client A              REST API          DB           SignalR Hub      Client B
  │                       │              │                │               │
  │  POST /direct-messages│              │                │               │
  │  { targetUserId: B }  │              │                │               │
  │──────────────────────>│              │                │               │
  │                       │  找是否已有 A+B 的 DM channel   │               │
  │                       │─────────────>│                │               │
  │                       │              │                │               │
  │                       │  （無）INSERT Channel(Type=Direct)             │
  │                       │  INSERT ChannelMember(A), ChannelMember(B)    │
  │                       │─────────────>│                │               │
  │                       │              │  AddToGroupAsync(A.connId, dmChannelId)
  │                       │              │  AddToGroupAsync(B.connId, dmChannelId)
  │                       │──────────────────────────────>│               │
  │<──────────────────────│              │                │               │
  │  { channelId }        │              │                │               │
  │                       │              │                │               │
  │  Invoke SendMessage   │              │                │               │
  │  (dmChannelId, "Hi")  │              │                │               │
  │──────────────────────────────────────────────────────>│               │
  │                       │              │  INSERT Post   │               │
  │                       │              │<───────────────│               │
  │                       │              │                │  PostCreated  │
  │<──────────────────────────────────────────────────────│──────────────>│
  │  PostCreated          │              │                │               │
```

---

### 5. 離線偵測與上線狀態

SignalR 提供 `OnConnectedAsync` / `OnDisconnectedAsync`，用來維護線上狀態：

```
連線建立：
  OnConnectedAsync()
  └─ Redis SET user:{userId}:status "online" EX 300
  └─ Clients.All.UserOnline(userId)   （或只廣播給相關 user）

連線中斷：
  OnDisconnectedAsync()
  └─ 確認該 user 沒有其他連線（Redis DECR user:{userId}:conn_count）
  └─ 若 conn_count == 0：
      Redis SET user:{userId}:status "offline"
      Clients.All.UserOffline(userId)
```

> 對應 Mattermost 的 Hub unregister → 詢問 Cluster → QueueSetStatusOffline 流程。

---

## 叢集支援（Scale Out）

單機用預設 in-memory SignalR。要水平擴展只需加一行：

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("redis:6379", opts =>
    {
        opts.Configuration.ChannelPrefix = RedisChannel.Literal("chat");
    });
```

Redis backplane 會自動把 `Clients.Group(channelId).PostCreated(...)` 廣播到所有節點上屬於這個 Group 的連線，不需要自己實作 ClusterInterface。

---

## 專案結構建議

```
ChatService/
├─ Api/
│   ├─ AuthController.cs
│   ├─ TeamsController.cs
│   ├─ ChannelsController.cs
│   └─ DirectMessagesController.cs
├─ Hubs/
│   └─ ChatHub.cs              ← SignalR Hub，處理 SendMessage / MarkRead
├─ Services/
│   ├─ TeamService.cs
│   ├─ ChannelService.cs
│   ├─ PostService.cs
│   └─ PresenceService.cs      ← 線上狀態（Redis）
├─ Data/
│   ├─ ChatDbContext.cs
│   └─ Entities/
│       ├─ User.cs
│       ├─ Team.cs
│       ├─ TeamMember.cs
│       ├─ Channel.cs
│       ├─ ChannelMember.cs
│       └─ Post.cs
└─ Program.cs
```

---

## ChatHub 骨架

```csharp
[Authorize]
public class ChatHub : Hub
{
    private readonly IChannelService _channelService;
    private readonly IPostService _postService;
    private readonly IPresenceService _presence;

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;

        // 把這條連線加入該 user 所有已加入 channel 的 Group
        var channelIds = await _channelService.GetChannelIdsForUserAsync(userId);
        foreach (var channelId in channelIds)
            await Groups.AddToGroupAsync(Context.ConnectionId, channelId);

        await _presence.SetOnlineAsync(userId);
        await Clients.Others.SendAsync("UserOnline", userId);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier!;
        var isOffline = await _presence.DecrementAndCheckOfflineAsync(userId);

        if (isOffline)
            await Clients.Others.SendAsync("UserOffline", userId);

        await base.OnDisconnectedAsync(exception);
    }

    public async Task SendMessage(string channelId, string message)
    {
        var userId = Context.UserIdentifier!;

        // 驗證 user 是 channel member
        if (!await _channelService.IsMemberAsync(channelId, userId))
        {
            Context.Abort();
            return;
        }

        var post = await _postService.CreateAsync(channelId, userId, message);

        // 廣播給 channel 所有在線成員
        await Clients.Group(channelId).SendAsync("PostCreated", new
        {
            post.Id,
            post.ChannelId,
            post.UserId,
            post.Message,
            post.CreatedAt
        });
    }

    public async Task MarkRead(string channelId, string postId)
    {
        var userId = Context.UserIdentifier!;
        await _channelService.UpdateLastReadAsync(channelId, userId, postId);
    }
}
```

---

## 與 Mattermost 的對應關係

| Mattermost | 本設計 |
|---|---|
| Hub + hubConnectionIndex | SignalR Hub + SignalR Groups |
| WebConn.send channel | SignalR 內部佇列 |
| ClusterInterface + Gossip | Redis backplane（`AddStackExchangeRedis`） |
| `Publish(WebSocketEvent)` | `Clients.Group(channelId).SendAsync(...)` |
| WebsocketEventPosted | `PostCreated` 事件 |
| DM channel（Type=Direct） | Channel（Type=Direct），同一張表 |
| `QueueSetStatusOffline` | `OnDisconnectedAsync` → Redis |
| `ShouldSendEvent` | SignalR Groups 隱含的成員過濾 |

---

## 後續可擴充的方向

| 功能 | 做法 |
|---|---|
| 訊息已讀回執 | ChannelMember.LastReadAt + 返回未讀數 API |
| 訊息編輯 / 刪除 | UPDATE Post + 廣播 `PostUpdated` / `PostDeleted` 事件 |
| 檔案上傳 | REST API 上傳到 S3 / MinIO，Post.Message 存 URL |
| Thread 回覆 | Post 加 `RootId` 欄位（對應 Mattermost 的 RootId） |
| Push Notification | 整合 APNs / FCM，`OnDisconnectedAsync` 時改送 push |
| 自訂 WebSocket Hub | 需要 dead queue、broadcast hook 時替換 SignalR |
