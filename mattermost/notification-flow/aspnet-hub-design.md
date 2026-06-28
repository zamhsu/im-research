# ASP.NET Core 精簡即時通訊服務 — Hub 實作設計

## 設計原則對應

| Mattermost（Go） | ASP.NET Core（C#） |
|---|---|
| `goroutine` | `Task` + `BackgroundService` |
| `go channel` | `System.Threading.Channels.Channel<T>` |
| `Hub struct` | `HubService : BackgroundService` |
| `hubConnectionIndex` | `ConcurrentDictionary` 多層索引 |
| `WebConn` | `WebSocketConnection` class |
| `NumCPU` 個 Hub | 多個 `HubShard`，consistent hashing 分流 |
| `ClusterInterface` | `IClusterBus`（NATS / Redis 實作） |

---

## 整體架構

```
客戶端 WebSocket 連線
        ↓
WebSocketMiddleware
        ↓
  WebSocketConnection（每條連線一個物件）
  ├─ send Channel<IWebSocketMessage>（buffer 256）
  ├─ deadQueue（最近 128 筆，斷線重連用）
  └─ ReadPump / WritePump（各一個 Task）
        ↓
  HubShard（每個 CPU core 一個，BackgroundService）
  ├─ connectionIndex（userId/channelId → connections）
  ├─ broadcast Channel<WebSocketEvent>（buffer 4096）
  └─ 處理：Register / Unregister / Broadcast / InvalidateUser
        ↓
  HubRouter（consistent hash，決定走哪個 HubShard）
        ↓
  [叢集] IClusterBus.Publish()
         └─ 其他節點收到後各自走 HubRouter → HubShard
```

---

## 一、WebSocketConnection

每條 WebSocket 連線對應一個物件，持有自己的收送佇列。

```csharp
public sealed class WebSocketConnection : IAsyncDisposable
{
    public string ConnectionId { get; } = Guid.NewGuid().ToString("N");
    public string UserId { get; init; } = string.Empty;
    public string[] ChannelIds { get; set; } = [];
    public bool IsActive { get; set; } = true;
    public long LastActivityAt { get; set; } = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
    public long Sequence { get; private set; }

    // 對應 Mattermost 的 webConn.send channel（buffer 256）
    private readonly Channel<IWebSocketMessage> _send =
        Channel.CreateBounded<IWebSocketMessage>(new BoundedChannelOptions(256)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true
        });

    // 斷線重連用：保存最近 128 筆（circular buffer）
    private readonly IWebSocketMessage?[] _deadQueue = new IWebSocketMessage?[128];
    private int _deadQueuePointer;

    private readonly WebSocket _socket;
    private readonly CancellationTokenSource _cts = new();

    public WebSocketConnection(WebSocket socket, string userId)
    {
        _socket = socket;
        UserId = userId;
    }

    // 非阻塞寫入，失敗代表佇列滿 → 關閉連線
    public bool TryEnqueue(IWebSocketMessage message)
        => _send.Writer.TryWrite(message);

    public ChannelReader<IWebSocketMessage> Reader => _send.Reader;

    public void WriteToDeadQueue(IWebSocketMessage message)
    {
        _deadQueue[_deadQueuePointer % 128] = message;
        _deadQueuePointer++;
    }

    public long NextSequence() => Interlocked.Increment(ref Sequence);

    public async ValueTask DisposeAsync()
    {
        await _cts.CancelAsync();
        _send.Writer.TryComplete();
        await _socket.CloseAsync(WebSocketCloseStatus.NormalClosure, null, CancellationToken.None);
    }
}
```

### ReadPump / WritePump

每條連線啟動兩個長駐 Task（對應 Mattermost 的 goroutine）：

```csharp
public static class WebSocketPumps
{
    // 從 WebSocket 讀取客戶端訊息
    public static async Task RunReadPump(
        WebSocketConnection conn,
        IClientMessageRouter router,
        CancellationToken ct)
    {
        var buffer = new byte[4096];
        while (!ct.IsCancellationRequested)
        {
            var result = await conn.Socket.ReceiveAsync(buffer, ct);
            if (result.MessageType == WebSocketMessageType.Close) break;

            var msg = ClientMessage.Parse(buffer.AsSpan(0, result.Count));
            await router.RouteAsync(conn, msg, ct);
            conn.LastActivityAt = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        }
    }

    // 從 send channel 讀取，寫入 WebSocket frame
    public static async Task RunWritePump(
        WebSocketConnection conn,
        CancellationToken ct)
    {
        await foreach (var message in conn.Reader.ReadAllAsync(ct))
        {
            var seq = conn.NextSequence();
            var frame = message.Serialize(seq);
            await conn.Socket.SendAsync(frame, WebSocketMessageType.Text, true, ct);
            conn.WriteToDeadQueue(message);
        }
    }
}
```

---

## 二、HubShard（對應 Mattermost 的 Hub）

每個 CPU core 一個 `HubShard`，各自管理自己負責的連線。

```csharp
public sealed class HubShard : BackgroundService
{
    public int Index { get; init; }

    // broadcast channel，buffer 4096
    private readonly Channel<WebSocketEvent> _broadcast =
        Channel.CreateBounded<WebSocketEvent>(new BoundedChannelOptions(4096)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true
        });

    private readonly Channel<HubCommand> _commands =
        Channel.CreateUnbounded<HubCommand>(new UnboundedChannelOptions
        {
            SingleReader = true
        });

    private readonly ConnectionIndex _index = new();
    private readonly IEnumerable<IBroadcastHook> _hooks;
    private readonly ILogger<HubShard> _logger;

    public HubShard(IEnumerable<IBroadcastHook> hooks, ILogger<HubShard> logger)
    {
        _hooks = hooks;
        _logger = logger;
    }

    // --- 對外 API（非同步，送入 channel）---

    public ValueTask RegisterAsync(WebSocketConnection conn)
        => _commands.Writer.WriteAsync(new RegisterCommand(conn));

    public ValueTask UnregisterAsync(WebSocketConnection conn)
        => _commands.Writer.WriteAsync(new UnregisterCommand(conn));

    public ValueTask BroadcastAsync(WebSocketEvent evt)
        => _broadcast.Writer.WriteAsync(evt);

    public ValueTask InvalidateUserAsync(string userId)
        => _commands.Writer.WriteAsync(new InvalidateUserCommand(userId));

    public ValueTask UpdateActivityAsync(string userId, string token, long activityAt)
        => _commands.Writer.WriteAsync(new ActivityCommand(userId, token, activityAt));

    // --- 主迴圈 ---

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _logger.LogInformation("HubShard {Index} starting", Index);

        // 同時監聽 commands 和 broadcast 兩個 channel
        var commandTask = ProcessCommandsAsync(ct);
        var broadcastTask = ProcessBroadcastsAsync(ct);
        await Task.WhenAll(commandTask, broadcastTask);
    }

    private async Task ProcessCommandsAsync(CancellationToken ct)
    {
        await foreach (var cmd in _commands.Reader.ReadAllAsync(ct))
        {
            switch (cmd)
            {
                case RegisterCommand r:
                    _index.Add(r.Connection);
                    break;

                case UnregisterCommand u:
                    u.Connection.IsActive = false;
                    _index.Remove(u.Connection);
                    break;

                case InvalidateUserCommand inv:
                    foreach (var c in _index.ForUser(inv.UserId))
                        c.InvalidateSessionCache();
                    break;

                case ActivityCommand act:
                    foreach (var c in _index.ForUser(act.UserId))
                        if (c.SessionToken == act.Token)
                            c.LastActivityAt = act.ActivityAt;
                    break;
            }
        }
    }

    private async Task ProcessBroadcastsAsync(CancellationToken ct)
    {
        // 控制並行數（對應 Mattermost 的 hubSemaphore）
        var sem = new SemaphoreSlim(Environment.ProcessorCount * 4);

        await foreach (var evt in _broadcast.Reader.ReadAllAsync(ct))
        {
            // 預先序列化 JSON，所有連線共用一份
            var precomputed = evt.ToJson();
            var targetConns = ResolveTargets(evt);

            await sem.WaitAsync(ct);
            _ = Task.Run(async () =>
            {
                try
                {
                    foreach (var conn in targetConns)
                    {
                        if (!conn.ShouldReceive(evt)) continue;

                        // 執行個人化 broadcast hooks
                        var final = await RunHooksAsync(evt, conn, precomputed);
                        if (!conn.TryEnqueue(final))
                        {
                            _logger.LogWarning(
                                "HubShard {Index}: send queue full, closing conn {ConnId}",
                                Index, conn.ConnectionId);
                            await conn.DisposeAsync();
                            _index.Remove(conn);
                        }
                    }
                }
                finally { sem.Release(); }
            }, ct);
        }
    }

    private IEnumerable<WebSocketConnection> ResolveTargets(WebSocketEvent evt)
    {
        if (evt.Broadcast.ConnectionId is { } connId)
            return _index.ForConnection(connId) is { } c ? [c] : [];

        if (evt.Broadcast.UserId is { } userId)
            return _index.ForUser(userId);

        if (evt.Broadcast.ChannelId is { } channelId)
            return _index.ForChannel(channelId);

        return _index.All();
    }

    private async ValueTask<IWebSocketMessage> RunHooksAsync(
        WebSocketEvent evt, WebSocketConnection conn, string precomputed)
    {
        var ctx = new BroadcastHookContext(evt, conn, precomputed);
        foreach (var hook in _hooks)
            await hook.ExecuteAsync(ctx);
        return ctx.Result;
    }
}
```

---

## 三、ConnectionIndex（連線索引）

```csharp
public sealed class ConnectionIndex
{
    private readonly Dictionary<string, HashSet<WebSocketConnection>> _byUser = new();
    private readonly Dictionary<string, HashSet<WebSocketConnection>> _byChannel = new();
    private readonly Dictionary<string, WebSocketConnection> _byConnectionId = new();

    public void Add(WebSocketConnection conn)
    {
        _byConnectionId[conn.ConnectionId] = conn;

        if (!_byUser.TryGetValue(conn.UserId, out var userSet))
            _byUser[conn.UserId] = userSet = new HashSet<WebSocketConnection>();
        userSet.Add(conn);

        foreach (var channelId in conn.ChannelIds)
        {
            if (!_byChannel.TryGetValue(channelId, out var chSet))
                _byChannel[channelId] = chSet = new HashSet<WebSocketConnection>();
            chSet.Add(conn);
        }
    }

    public void Remove(WebSocketConnection conn)
    {
        _byConnectionId.Remove(conn.ConnectionId);
        _byUser.GetValueOrDefault(conn.UserId)?.Remove(conn);
        foreach (var channelId in conn.ChannelIds)
            _byChannel.GetValueOrDefault(channelId)?.Remove(conn);
    }

    public IEnumerable<WebSocketConnection> ForUser(string userId)
        => _byUser.GetValueOrDefault(userId) ?? [];

    public IEnumerable<WebSocketConnection> ForChannel(string channelId)
        => _byChannel.GetValueOrDefault(channelId) ?? [];

    public WebSocketConnection? ForConnection(string connectionId)
        => _byConnectionId.GetValueOrDefault(connectionId);

    public IEnumerable<WebSocketConnection> All()
        => _byConnectionId.Values;
}
```

---

## 四、HubRouter（Consistent Hashing 分流）

```csharp
public sealed class HubRouter
{
    private readonly HubShard[] _shards;

    public HubRouter(IEnumerable<HubShard> shards)
        => _shards = shards.ToArray();

    public HubShard GetShardForUser(string userId)
    {
        var hash = (uint)HashCode.Combine(userId);
        return _shards[hash % (uint)_shards.Length];
    }

    public async ValueTask RegisterAsync(WebSocketConnection conn)
        => await GetShardForUser(conn.UserId).RegisterAsync(conn);

    public async ValueTask UnregisterAsync(WebSocketConnection conn)
        => await GetShardForUser(conn.UserId).UnregisterAsync(conn);

    // Broadcast 可能需要送給多個 shard（channel 廣播時無法確定 user 在哪個 shard）
    public async ValueTask BroadcastAsync(WebSocketEvent evt)
    {
        if (evt.Broadcast.UserId is { } userId)
        {
            await GetShardForUser(userId).BroadcastAsync(evt);
        }
        else
        {
            // Channel 廣播或全廣播：送給所有 shard
            var tasks = _shards.Select(s => s.BroadcastAsync(evt).AsTask());
            await Task.WhenAll(tasks);
        }
    }
}
```

---

## 五、叢集支援：IClusterBus

單節點只用 HubRouter，叢集部署在 `Publish()` 時額外廣播給其他節點。

### 介面定義

```csharp
public interface IClusterBus
{
    // 發佈事件到叢集（其他節點）
    Task PublishAsync(WebSocketEvent evt, CancellationToken ct = default);

    // 訂閱來自其他節點的事件
    Task SubscribeAsync(Func<WebSocketEvent, Task> handler, CancellationToken ct = default);
}
```

### 發佈流程

```csharp
public sealed class EventPublisher(HubRouter router, IClusterBus clusterBus)
{
    public async Task PublishAsync(WebSocketEvent evt, CancellationToken ct = default)
    {
        // Phase 1：投遞給本節點的連線
        await router.BroadcastAsync(evt);

        // Phase 2：廣播給其他節點
        await clusterBus.PublishAsync(evt, ct);
    }
}
```

### NATS 實作

```csharp
public sealed class NatsClusterBus(IConnection nats) : IClusterBus
{
    private const string SubjectPrefix = "chat.cluster.event";

    public Task PublishAsync(WebSocketEvent evt, CancellationToken ct = default)
    {
        var subject = $"{SubjectPrefix}.{evt.EventType}";
        var data = JsonSerializer.SerializeToUtf8Bytes(evt);

        // 重要事件（posted、edited）用 JetStream 保證送達
        // 其他事件（status、activity）用 Core NATS（best effort）
        if (evt.IsReliable)
            return nats.RequestAsync(subject, data, ct).AsTask();
        else
            return Task.FromResult(nats.Publish(subject, data));
    }

    public async Task SubscribeAsync(Func<WebSocketEvent, Task> handler, CancellationToken ct = default)
    {
        var sub = nats.SubscribeAsync($"{SubjectPrefix}.>", async (_, args) =>
        {
            var evt = JsonSerializer.Deserialize<WebSocketEvent>(args.Message.Data);
            if (evt is not null)
                await handler(evt);
        });

        await Task.Delay(Timeout.Infinite, ct);
        sub.Unsubscribe();
    }
}
```

### Redis Pub/Sub 實作（替代方案）

```csharp
public sealed class RedisClusterBus(IConnectionMultiplexer redis) : IClusterBus
{
    private const string Channel = "chat:cluster:events";
    private readonly ISubscriber _sub = redis.GetSubscriber();

    public Task PublishAsync(WebSocketEvent evt, CancellationToken ct = default)
    {
        var data = JsonSerializer.Serialize(evt);
        return _sub.PublishAsync(Channel, data).AsTask();
    }

    public async Task SubscribeAsync(Func<WebSocketEvent, Task> handler, CancellationToken ct = default)
    {
        await _sub.SubscribeAsync(Channel, async (_, value) =>
        {
            var evt = JsonSerializer.Deserialize<WebSocketEvent>(value);
            if (evt is not null)
                await handler(evt);
        });

        await Task.Delay(Timeout.Infinite, ct);
    }
}
```

---

## 六、WebSocket Middleware 接入點

```csharp
public class WebSocketMiddleware(RequestDelegate next, HubRouter router, EventPublisher publisher)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        if (!ctx.WebSockets.IsWebSocketRequest)
        {
            await next(ctx);
            return;
        }

        var userId = ctx.User.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new UnauthorizedAccessException();

        var socket = await ctx.WebSockets.AcceptWebSocketAsync();
        var conn = new WebSocketConnection(socket, userId);
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ctx.RequestAborted);

        await router.RegisterAsync(conn);
        try
        {
            // 送出 hello 事件
            conn.TryEnqueue(new HelloMessage(conn.ConnectionId));

            // 同時跑 ReadPump 和 WritePump
            await Task.WhenAll(
                WebSocketPumps.RunReadPump(conn, /* router */ null!, cts.Token),
                WebSocketPumps.RunWritePump(conn, cts.Token)
            );
        }
        finally
        {
            await router.UnregisterAsync(conn);
            await conn.DisposeAsync();
        }
    }
}
```

---

## 七、Dependency Injection 設定

```csharp
var builder = WebApplication.CreateBuilder(args);

// 建立 NumCPU 個 HubShard
var shardCount = Environment.ProcessorCount;
for (var i = 0; i < shardCount; i++)
{
    builder.Services.AddSingleton<HubShard>(sp => new HubShard(
        sp.GetRequiredService<IEnumerable<IBroadcastHook>>(),
        sp.GetRequiredService<ILogger<HubShard>>()
    ) { Index = i });
}

// HubRouter 注入所有 HubShard
builder.Services.AddSingleton<HubRouter>();
builder.Services.AddSingleton<EventPublisher>();

// Broadcast Hooks
builder.Services.AddSingleton<IBroadcastHook, MentionsBroadcastHook>();
builder.Services.AddSingleton<IBroadcastHook, FollowersBroadcastHook>();

// 叢集 Bus（擇一）
builder.Services.AddSingleton<IClusterBus, NatsClusterBus>();
// builder.Services.AddSingleton<IClusterBus, RedisClusterBus>();

// 把每個 HubShard 當 BackgroundService 跑
builder.Services.AddHostedService(sp =>
    sp.GetServices<HubShard>().First(s => s.Index == 0));
// ... 依此類推（或用工廠方式批次登記）

var app = builder.Build();
app.UseWebSockets();
app.UseMiddleware<WebSocketMiddleware>();
app.Run();
```

---

## 八、與 Mattermost Hub 的對比

| 特性 | Mattermost（Go） | 本設計（ASP.NET Core C#） |
|---|---|---|
| 並行模型 | goroutine + go channel | Task + `Channel<T>` |
| Hub 數量 | `runtime.NumCPU()` | `Environment.ProcessorCount` |
| 連線路由 | `hash(userID) % NumCPU` | `HashCode.Combine(userId) % shards.Length` |
| 廣播佇列 | buffered channel 4096 | `BoundedChannel<T>` capacity 4096 |
| 送訊佇列 | buffered channel 256 | `BoundedChannel<T>` capacity 256 |
| Dead queue | `[128]*WebSocketEvent` 環形 | `IWebSocketMessage?[128]` 環形 |
| JSON 序列化 | `PrecomputeJSON()` 一次 | `ToJson()` 預先計算，多連線共用 |
| Broadcast hooks | `runBroadcastHooks()` | `IBroadcastHook` 介面，DI 注入 |
| 叢集 | 企業版 Gossip 實作 | `IClusterBus`（NATS / Redis） |
| 線程安全 | Hub goroutine 是單一 select 迴圈 | ConnectionIndex 在單一 reader Task 內操作，不需要鎖 |

---

## 九、不建議使用 SignalR 的原因

ASP.NET Core 內建的 SignalR 雖然也有 Hub 概念，但用途不同：

| | SignalR Hub | 本設計 |
|---|---|---|
| 抽象層級 | 高（RPC 風格，自動序列化） | 低（直接控制 WebSocket frame） |
| 連線過濾 | Groups（粗粒度） | 自訂 broadcast hooks（細粒度，如 ABAC） |
| Dead queue | 無 | 有（支援斷線重連恢復） |
| 序列號 | 無 | 有（保證訊息順序） |
| 叢集 backplane | Redis（官方支援） | 可選 NATS 或 Redis |
| 適用場景 | 快速開發、標準場景 | 需要精細控制的即時通訊服務 |

如果只是 prototype 或功能需求簡單，SignalR + Redis backplane 是更快的選擇。若要完整複製 Mattermost 的 dead queue、broadcast hook、per-connection filtering，則用本設計的自訂 WebSocket 方案。
