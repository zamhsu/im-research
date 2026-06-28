# 企業 IM 研究待辦清單

本文整理從 Mattermost 原始碼中尚未研究、但對打造企業級 IM 服務至關重要的面向。

---

## 身分與存取管理

| 主題 | 研究方向 |
|------|----------|
| **RBAC 權限系統** | `server/channels/app/permissions*.go` — roles、schemes、channel_member.roles 的整個模型 |
| **LDAP / SAML / SSO** | `server/channels/app/ldap*.go`, `saml*.go` — 企業認證整合的 interface 設計 |
| **MFA** | `server/channels/app/mfa.go` — TOTP 整合方式 |

---

## 擴展性與基礎設施

| 主題 | 研究方向 |
|------|----------|
| **叢集 / HA** | `server/channels/app/cluster*.go`, `platform/services/cluster/` — 多節點如何同步事件 |
| **Cache 層** | `server/channels/store/storetest/` + Redis adapter — 讀取快取策略 |
| **Rate Limiting** | `server/channels/api4/middleware.go` 裡的 throttling 實作 |
| **背景工作排程** | `server/channels/jobs/` — 定時任務（email digest、清理、合規匯出）的 worker 架構 |

---

## 檔案與媒體

| 主題 | 研究方向 |
|------|----------|
| **檔案儲存抽象** | `server/platform/services/filestore/` — S3 / 本地 / MinIO 的 interface 設計 |
| **上傳 / 下載流程** | `server/channels/app/file*.go` — chunk upload、縮圖、安全驗證 |

---

## 通知系統（部分已有，需補充）

| 主題 | 研究方向 |
|------|----------|
| **Mobile Push Proxy** | Mattermost 用自己的 push proxy 轉發 APNS/FCM，`server/channels/app/notification_push.go` |
| **Email 通知** | `server/channels/app/email*.go` — batching、template、digest 邏輯 |

---

## 搜尋

| 主題 | 研究方向 |
|------|----------|
| **搜尋 interface** | `server/channels/app/search*.go` + `server/platform/services/searchengine/` — Elasticsearch / Bleve 抽象層 |
| **全文搜尋設計** | 如何 index message、file、user |

---

## 整合與外掛系統

| 主題 | 研究方向 |
|------|----------|
| **Plugin 架構** | `server/channels/app/plugin*.go` — hashicorp/go-plugin 的 RPC 隔離設計 |
| **Webhook / Slash Command** | `server/channels/app/webhooks*.go`, `slash_commands*.go` |
| **Bot 帳號機制** | `model/bot.go` — bot vs 一般 user 的差異 |

---

## 合規與審計

| 主題 | 研究方向 |
|------|----------|
| **Audit Log** | `server/channels/audit/` — 每個操作如何記錄 actor/event/metadata |
| **Data Retention** | `server/channels/jobs/data_retention/` — 自動刪除訊息的策略執行 |
| **Compliance Export** | `server/channels/jobs/compliance_export/` — eDiscovery 格式匯出 |
| **E2E Encryption** | Mattermost **沒有** server-side E2EE（用戶端有實驗性），這個設計決策本身值得研究 |

---

## 維運與設定管理

| 主題 | 研究方向 |
|------|----------|
| **Config 系統** | `server/channels/app/config*.go` — 環境變數、DB stored config、hot reload 機制 |
| **Schema Migration** | `server/db/migrations/` — morph 工具的版本遷移策略 |
| **Metrics** | `server/platform/services/telemetry/` + Prometheus exporter — 哪些指標暴露、如何設計 |

---

## 優先建議

若要從零建企業 IM，以下四個是最有槓桿的研究優先順序：

1. **RBAC 權限系統** — 影響所有 API 的設計（已完成，見 `13-rbac-permission-system.md`）
2. **叢集同步機制** — 決定你能否水平擴展
3. **Plugin/Webhook 架構** — 企業整合生態的基礎
4. **合規 Audit Log** — 企業採購最常見的硬性需求

---

## 進度追蹤

| 主題 | 狀態 |
|------|------|
| RBAC 權限系統 | ✅ 完成（`13-rbac-permission-system.md`） |
| LDAP / SAML / SSO | ⬜ 待研究 |
| MFA | ⬜ 待研究 |
| 叢集 / HA | ⬜ 待研究 |
| Cache 層 | ⬜ 待研究 |
| Rate Limiting | ⬜ 待研究 |
| 背景工作排程 | ⬜ 待研究 |
| 檔案儲存抽象 | ⬜ 待研究 |
| 上傳 / 下載流程 | ⬜ 待研究 |
| Mobile Push Proxy | ⬜ 待研究 |
| Email 通知 | ⬜ 待研究 |
| 搜尋 interface | ⬜ 待研究 |
| Plugin 架構 | ⬜ 待研究 |
| Webhook / Slash Command | ⬜ 待研究 |
| Bot 帳號機制 | ⬜ 待研究 |
| Audit Log | ⬜ 待研究 |
| Data Retention | ⬜ 待研究 |
| Compliance Export | ⬜ 待研究 |
| E2EE 設計決策 | ⬜ 待研究 |
| Config 系統 | ⬜ 待研究 |
| Schema Migration | ⬜ 待研究 |
| Metrics | ⬜ 待研究 |
