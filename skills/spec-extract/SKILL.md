---
name: spec-extract
description: PRD → FRD 轉換器。讀取任何格式的產品需求文件，透過系統性問題拆解，產出結構化的 FRD（功能需求文件）。會主動掃描專案上下文（codebase、現有 API、DB schema），不純靠訪談。問題依選定角色動態調整。
---

# spec-pipeline:spec-extract — PRD → FRD 轉換器

從 PRD（或任何需求來源）萃取出結構化 FRD，讓下一階段（`spec-pipeline:spec-deliver`）能直接產出交付規格。

## 核心原則

> **PRD 告訴你「要做什麼」，FRD 告訴你「每個系統要怎麼配合」。**
> PRD 容許模糊，FRD 不行——因為 FRD 的下一站是 RD，模糊會變成猜測。

---

## Usage

```
spec-pipeline:spec-extract                  # 互動模式，從零開始
spec-pipeline:spec-extract {prd-path}       # 讀取 PRD 文件，開始萃取
```

如果使用者指定了 PRD 檔案路徑，先確認檔案存在。如果不存在，告知使用者並詢問正確路徑。不要猜測內容。

---

## 前置：角色確認

如果從 `spec-pipeline:spec-pipeline` 進入，角色已選定。
如果獨立使用，先確認角色組合：

| Key | 角色 | 職責範圍 |
|-----|------|---------|
| `f2e` | 前端 RD (F2E) | SPA/SSR 路由、狀態管理、API 串接、渲染策略、快取 |
| `app` | App RD (Native) | 推播整合、Deep Link、離線行為、原生元件 |
| `backend` | Backend RD | API 設計、資料流、排程 Job、並行安全、認證授權 |
| `dba` | DBA | Schema 設計、目標 DB 對照、索引策略、讀寫分離 |
| `qa` | QA / 一致性檢查 | 跨章節交叉比對、遺漏偵測 — **永遠啟用** |

常見組合：Web 全端（f2e+backend+dba）、App 全端（app+backend+dba）、Full Stack（全選）、Pure API（backend+dba）。
詢問使用者選擇角色組合後再繼續。

選定角色決定：
- 要掃描 codebase 的哪些部分
- 要問哪些維度的問題
- FRD 要產出哪些章節

---

## Phase 1 流程

### Step 0：Reconnaissance — 掃描專案上下文

在問任何問題之前，先做：

1. **讀取 PRD**（使用者提供的文件或貼上的內容）
2. **掃描 codebase**，依選定角色決定掃描範圍：

#### 共通（所有角色都掃）
- 認證機制（token 管理、header 格式）
- 現有 API 模式（URL 規範、錯誤碼格式、分頁方式）
- 專案結構（mono-repo? 多模組?）

#### 有 `app` 時額外掃描
- 推播機制（FCM / APNs、現有 payload model 結構）
- URL scheme / Deep Link（iOS: Info.plist、Android: AndroidManifest）
- 通知模型（NotificationModel 或等效物）
- 離線快取策略（iOS: CoreData / UserDefaults、Android: Room / DataStore、或檔案快取）
- 登入狀態管理與安全儲存（iOS: Keychain、Android: Keystore / EncryptedSharedPreferences）

#### 有 `f2e` 時額外掃描
- 路由框架（React Router? Next.js? Vue Router?）
- 狀態管理（Redux? Zustand? Pinia?）
- SSR/CSR 策略
- API client 封裝（Axios? fetch wrapper?）
- 認證 token 存放（cookie? localStorage? memory?）

#### 有 `backend` 時額外掃描
- API 框架（Express? Spring? .NET?）
- 現有中介層（auth middleware、rate limiter）
- 排程 Job 框架（Hangfire? Bull? Cron?）
- 訊息佇列（RabbitMQ? Kafka? SQS?）

#### 有 `dba` 時額外掃描
- DB 平台（MySQL / PostgreSQL / MS-SQL / MongoDB）
- ORM（Entity Framework? Sequelize? Prisma?）
- 現有 migration 工具
- 現有 schema 慣例（命名規範、型別偏好）

3. **識別所有系統邊界**（依選定角色）：

| 角色 | 對應系統 |
|------|---------|
| `f2e` | Web Frontend（SPA / SSR） |
| `app` | App（iOS / Android） |
| `backend` | Backend API / Worker / Scheduler |
| `dba` | Database |
| 共通 | 推播平台、CDN、第三方服務、CMS |

#### Greenfield 專案處理

如果 codebase 掃描無結果（空專案或尚無原始碼），切換為「設計模式」：
1. 跳過 codebase 掃描
2. 在 Step 1 改問「你們計畫用什麼技術棧」（框架 / DB / 認證方式）
3. FRD 的「系統邊界」章節標註「技術棧：待定」或「技術棧：已決定 — {choices}」
4. 後續 Delivery Spec 中的技術選型需要更明確的理由說明（因為沒有既有慣例可依循）

### Step 0.5：判斷專案類型，調整問題深度

根據 codebase 掃描結果和 PRD 內容，判斷專案類型並調整後續問題的優先順序：

| 專案類型 | 判斷依據 | 高優先維度 | 低優先維度 |
|---------|---------|-----------|-----------|
| 面向消費者（C2C / B2C App） | 有推播、離線需求、多入口 | 網路韌性、推播、離線/快取 | — |
| 內部工具 / Admin | 後台管理介面、CRUD 為主 | 存取控制、資料流 | 離線/快取、推播 |
| SaaS / 多租戶 | 多組織、權限隔離 | 存取控制、多租戶隔離、i18n | 離線/快取 |
| API / 微服務 | 無前端、純後端 | 資料流、並行安全、可觀測性 | 渲染策略、路由 |
| 跨平台 | 同時有 Web + App | 網路韌性、路由、推播 | — |

**不需要問使用者專案類型——從 codebase 和 PRD 推斷。** 如果不確定專案類型，對所有**選定角色**的問題維度不做深度裁剪，走完整深度。（不影響角色篩選——仍只問選定角色的維度。）

### Step 1：系統性問題拆解

從 PRD 中萃取所有需要回答的問題。按以下維度拆解，**只問選定角色相關的維度**。依 Step 0.5 判斷的專案類型調整問題深度。

#### 1.1 資料流（Data Flow）— 所有角色
- 資料從哪來？（訂單系統？CMS？使用者操作？）
- 資料存在哪？（新表？現有表擴充？）
- 資料怎麼流？（事件驅動？API 呼叫？排程？）
- 資料給誰看？（全體？個人？角色差異？）

#### 1.2 觸發時機（Trigger）— 所有角色
- 什麼事件觸發什麼行為？
- 排程類：怎麼算時間？用什麼時區？
- 使用者操作類：什麼狀態下可以操作？什麼狀態下禁止？
- 系統事件類：訂單成立？取消？修改？

#### 1.3 邊界條件（Edge Cases）— 所有角色
- 重複操作怎麼辦？（冪等性）
- 並行操作怎麼辦？（race condition）
- 過期/取消怎麼辦？
- 生命週期結束後怎麼辦？（結束後行為矩陣）

#### 1.4 認證與授權（Auth）— 所有角色
- 用現有認證還是新的？
- userID 怎麼取得？（從 token 解析？client 傳入？）
- 角色差異？（Phase1 不分？Phase2 才分？）

#### 1.5 推播與通知（Push）— 僅 `app` 角色
- 用現有推播框架還是新的？
- Payload 結構怎麼設計？（沿用現有 model？）
- 推播授權狀態怎麼處理？（.notDetermined / .denied）
- 三種 App 狀態（前景/背景/Cold launch）各怎麼處理？

#### 1.6 網路錯誤與韌性（Network Resilience）— `app` 或 `f2e` 角色

這是前端/App 最容易遺漏的維度。不是「顯示錯誤訊息」就好，而是要定義每種失敗場景的 UX 策略。

**通用（app + f2e）：**
- API timeout 設多少？（建議依操作類型區分：讀取 10s、寫入 30s）
- Retry 策略？（哪些 API 可以自動 retry？幾次？間隔？指數退避？）
- 哪些 API 失敗是 silent（靜默降級），哪些要 blocking（擋住使用者）？
- Loading 狀態：skeleton screen? spinner? placeholder?
- 錯誤 UI：inline error? toast? 全頁錯誤? retry 按鈕?
- 樂觀更新 vs 悲觀更新？（先更新 UI 再等 API？還是等 API 成功才更新？）
- 連續失敗（3+ 次 API 失敗）怎麼處理？（降級模式？維護頁面？）

**僅 `app`：**
- 無網路 / 飛航模式下，哪些功能可用？（離線快取的資料？已下載的音檔？）
- 弱網路（高延遲/低頻寬）怎麼辦？（圖片降級? 延遲載入?）
- Request in-flight 時 App 進背景，怎麼處理？（iOS: URLSession background task、Android: WorkManager / foreground service）
- 網路恢復後，要不要自動重試排隊的操作？（離線 queue?）
- Token 剛好在斷線時過期 → 網路恢復後要先 refresh 還是直接 retry？

**僅 `f2e`：**
- Stale-while-revalidate 策略？（SWR / React Query 的 staleTime?）
- 錯誤邊界（Error Boundary）的粒度？（元件級? 頁面級? 全域?）
- SSR 時後端 API 掛了怎麼辦？（fallback HTML? 503?）
- WebSocket / SSE 斷線重連策略？（指數退避? 最大重試?）

#### 1.6b 離線與快取（Offline/Cache）— `app` 或 `f2e` 角色
- 離線時哪些功能可用？哪些不可用？
- 快取策略？（哪些資料快取？快取多久？快取失效機制？）
- 斷線後恢復怎麼處理？（自動重新載入? 使用者手動刷新?）

#### 1.7 路由與導航（Routing）— `app` 或 `f2e` 角色
- Deep Link / URL scheme 怎麼設計？
- 未登入時的導流？（登入後自動跳轉？暫存 context？）
- 多入口進入同一頁面？（推播、分享連結、首頁入口）

#### 1.8 前端渲染策略（Rendering）— 僅 `f2e` 角色
- SSR / CSR / ISR？
- SEO 需求？
- 首屏效能要求？
- 元件庫/設計系統沿用哪一套？

#### 1.9 DB 策略（Database）— 僅 `dba` 角色
- 目標 DB 平台？（需不需要出 DB 平台對照表？）
- 讀寫分離需求？
- 索引策略？（聚集索引選擇）
- 資料保留策略？（多久後歸檔/清除？）

#### 1.10 外部依賴（Dependencies）— 所有角色
- 哪些功能依賴外部系統？（推播平台、支付、搜尋引擎、第三方 API）
- 外部系統掛了怎麼辦？
- 有沒有 rate limit？

#### 1.11 存取控制模型（Access Control）— 所有角色
- 用什麼授權模型？（RBAC / ABAC / 自訂）
- 有沒有多租戶（multi-tenant）需求？（資料隔離策略：row-level / schema-level / DB-level）
- 角色/權限的粒度？（頁面級? 功能級? 欄位級?）
- 權限異動時，已登入 session 怎麼處理？（即時生效? 下次登入?）

#### 1.12 國際化（i18n）— 所有角色（如果有多語系需求）
- 需要支援哪些語系？
- 翻譯來源？（靜態 i18n 檔? CMS? 即時翻譯?）
- 使用者偏好語系存在哪？（profile? browser? device?）
- 有沒有 RTL（右到左）需求？
- 幣別 / 日期 / 數字格式的在地化？

#### 1.13 可觀測性（Observability）— 主要 `backend`，但所有角色適用
- 需要追蹤哪些關鍵指標？（延遲、錯誤率、吞吐量）
- 結構化 logging 的策略？（request tracing? correlation ID?）
- 告警觸發條件？（error rate > X%? latency > Yms?）
- 前端/App 端需要回報哪些 metrics？（crash report? performance timing?）

### Step 2：互動式釐清

**一次只問一個問題。** 不要一次丟出 10 個問題。

問題排序策略：
1. 先問**架構級**的（資料從哪來、存在哪、給誰）
2. 再問**行為級**的（什麼時候觸發、怎麼觸發）
3. 最後問**邊界級**的（重複/取消/過期/結束後）

問問題時，給出你的建議和理由：
```
❌ 「通知時間怎麼算？」
✅ 「通知時間建議用出發地時區（departureTimezone）統一計算。
    理由：Phase1 國內團旅客皆 Asia/Taipei，且 postAppLogin 不含裝置時區。
    Phase2 有國外團再加 per-member timezone。同意嗎？」
```

**跳過不相關的問題。** 如果沒有 `app` 角色，不要問推播相關的問題。

### Step 3：產出 FRD

確認所有問題後，產出結構化 FRD。

---

## FRD 輸出結構

```markdown
# {專案名稱} Phase{N} — 功能需求文件（FRD）

## 0. 文件資訊
- Epic: {title}
- PO: {name}
- TL: {name}
- 版本: v{X.Y.Z}
- 日期: {YYYY-MM-DD}
- 涉及角色: {f2e, backend, dba} ← 記錄選定角色

## 1. 範圍定義
### 1.1 In Scope
### 1.2 Out of Scope
### 1.3 平台範圍

## 2. 系統邊界
（僅列出選定角色對應的系統 + 各自的責任概述）

## 3. 功能需求（FR）
### FR-{N.M}: {功能名稱}
- 觸發條件
- 前置條件
- 預期行為（Happy Path）
- 異常行為（Error Cases）
- 資料來源 / 目的地

## 4. 邊界條件（EC）
### EC-{N}: {條件名稱}

## 5. 行為矩陣
### 5.1 生命週期結束後行為
### 5.2 角色行為差異

## 6. 非功能需求（依選定角色裁剪）
- 認證策略                    ← 所有角色
- 存取控制模型                ← 所有角色
- 時區策略                    ← 所有角色
- 分頁策略                    ← 所有角色
- 推播策略                    ← 僅 app
- 離線/快取策略               ← app 或 f2e
- 渲染策略（SSR/CSR）         ← 僅 f2e
- DB 平台策略                 ← 僅 dba
- 國際化策略（i18n）          ← 所有角色（如果有多語系需求）
- 可觀測性策略                ← 主要 backend，但所有角色適用

## 7. 未決事項（Open Questions）
```

---

## 品質檢查

FRD 產出後，自我檢查：

| 檢查項 | 標準 |
|--------|------|
| 每個 FR 都有觸發條件？ | 不能只說「使用者可以...」 |
| 每個 FR 都有錯誤行為？ | 至少：未授權、找不到、重複操作 |
| 邊界條件涵蓋「結束後」？ | 每個 API 行為都要定義 |
| 沒有「視情況而定」？ | 如果真的視情況，列出所有情況 |
| 認證策略明確？ | userID 來源、token 格式、header 格式 |
| 只涵蓋選定角色的範圍？ | 沒有 `app` 就不該出現推播章節 |

---

## 產出物

- 檔案：`docs/specs/YYYY-MM-DD-{topic}-frd.md`
- Commit message：`docs: add {topic} Phase{N} FRD`

完成後提示使用者：
> 「FRD 已產出在 `{path}`。這是 `spec-pipeline:spec-pipeline` 的 Phase 1 產出。確認後可以 `spec-pipeline:spec-deliver` 進入 Phase 2 產出交付規格。」
