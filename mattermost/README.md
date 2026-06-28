# Mattermost 原始碼分析

本目錄包含對 Mattermost 後端原始碼的架構與功能流程分析。

## 目錄

| 文件 | 說明 |
|------|------|
| [01-architecture-overview.md](01-architecture-overview.md) | 整體架構與分層設計 |
| [02-authentication-flow.md](02-authentication-flow.md) | 使用者認證流程 |
| [03-messaging-flow.md](03-messaging-flow.md) | 訊息發送流程 |
| [04-channel-flow.md](04-channel-flow.md) | 頻道操作流程 |
| [05-websocket-flow.md](05-websocket-flow.md) | WebSocket 即時通訊流程 |
| [06-database-layer.md](06-database-layer.md) | 資料庫層設計 |
| [07-server-startup.md](07-server-startup.md) | 伺服器啟動流程 |
| [08-data-models.md](08-data-models.md) | 核心資料模型 |
| [10-team-flow.md](10-team-flow.md) | 團隊（Team）操作流程 |
| [11-websocket-events-catalog.md](11-websocket-events-catalog.md) | 所有 WebSocket 事件目錄（含 JSON 資料格式） |
| [12-member-list-visibility.md](12-member-list-visibility.md) | Team / Channel 成員清單顯示規則 |

## 程式碼庫結構

```
mattermost/
├── server/                  # 後端 Go 服務
│   ├── channels/
│   │   ├── api4/            # HTTP API 處理層
│   │   ├── app/             # 應用程式業務邏輯層
│   │   ├── store/           # 資料存取層（介面 + SQL 實作）
│   │   ├── wsapi/           # WebSocket API 處理
│   │   └── web/             # 靜態檔案與 HTTP 伺服器
│   ├── cmd/mattermost/      # 程式入口點
│   ├── public/model/        # 資料模型定義
│   └── platform/            # 跨服務基礎設施
└── webapp/                  # 前端 React 應用
```

## 核心架構原則

- **三層架構**：API → App → Store，職責分離清晰
- **介面驅動的 Store 層**：方便測試與切換資料庫
- **WebSocket Hub 廣播**：即時事件推送機制
- **主從分離**：Master DB 寫入、Replica DB 讀取
