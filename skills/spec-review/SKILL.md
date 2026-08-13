---
name: spec-review
description: Delivery Spec 多角色審查。依選定角色 fan-out subagent（F2E / Backend / DBA / App + QA 永遠啟用），平行審查，合併為分級報告（Blocking / Ambiguous / Suggestion）。可獨立重跑。
---

# spec-pipeline:spec-review — 多角色 Delivery Spec 審查

用 subagent 平行審查 Delivery Spec，產出合併報告。只啟動選定角色 + QA。

## 核心原則

> **每個角色只看自己會實作的部分，用自己的專業判斷。**
> F2E 不會去審 DB schema，DBA 不會去審 UI 行為。
> 但 QA 會交叉比對所有角色的產出，找出矛盾。

---

## Usage

```
spec-pipeline:spec-review                          # 自動找最新的 Delivery Spec
spec-pipeline:spec-review {spec-path}              # 指定要審查的文件
spec-pipeline:spec-review round {N} {spec-path}    # 指定第 N 輪審查
```

如果未指定 Delivery Spec 路徑，在 `docs/specs/` 目錄中以 `*-system-flows.md` pattern 搜尋。如果有多個，取檔案修改時間最新者。如果找不到，詢問使用者指定路徑。

如果使用者指定了路徑，先確認檔案存在。如果不存在，告知使用者並詢問正確路徑。

---

## 審查流程

### Step 0：準備

1. 讀取 Delivery Spec（Markdown）
2. 讀取 FRD（如果存在，作為 review 的參照基準）
3. 讀取 codebase 上下文
4. 確認選定角色。來源優先順序：(1) 從 `spec-pipeline:spec-pipeline` 傳入，(2) 從 FRD 的「涉及角色」欄位，(3) 從 Delivery Spec 的章節結構推斷（有 D-2 → app，有 D-2b → f2e，有完整 DDL → dba），(4) 直接詢問使用者
5. 決定 round number

### Step 1：Fan-out Subagent

用 Agent tool 平行啟動 **選定角色 + QA** 的 subagent。

```
範例（App 全端）：同時啟動 4 個 Agent
  Agent 1: App RD Reviewer      ← 因為選了 app
  Agent 2: Backend RD Reviewer   ← 因為選了 backend
  Agent 3: DBA Reviewer          ← 因為選了 dba
  Agent 4: QA / Consistency      ← 永遠啟用

範例（Web 全端）：同時啟動 4 個 Agent
  Agent 1: F2E Reviewer          ← 因為選了 f2e
  Agent 2: Backend RD Reviewer   ← 因為選了 backend
  Agent 3: DBA Reviewer          ← 因為選了 dba
  Agent 4: QA / Consistency      ← 永遠啟用

範例（Pure API）：同時啟動 3 個 Agent
  Agent 1: Backend RD Reviewer   ← 因為選了 backend
  Agent 2: DBA Reviewer          ← 因為選了 dba
  Agent 3: QA / Consistency      ← 永遠啟用
```

**每個 subagent 的 prompt 結構：**

```
你是 {角色名稱}，正在審查一份 Delivery Spec。

讀取文件：{spec-path}
參照 FRD：{frd-path}（如果有）

你的審查維度：
{該角色的審查清單}

產出格式：
在你的最終回覆中，以 JSON 陣列格式輸出所有 findings。主 agent 會從你的回覆文字中解析 JSON。每個 finding 包含：
- section: 章節編號（如 "C-2", "E-3", "D-5"）
- severity: "blocking" | "ambiguous" | "suggestion"
- title: 一行摘要
- detail: 具體問題描述
- recommendation: 建議修正方式
```

**Subagent 失敗處理：**
如果任一 subagent 失敗或逾時，在合併報告中標註該角色的審查為 ⚠️ Incomplete，列出失敗原因，並詢問使用者是否要重新啟動該角色的審查。不要因為一個角色失敗就放棄整個 review。

---

## 各角色審查清單

### 角色：F2E Reviewer（僅 `f2e` 選定時啟用）

1. **API 可串接性**
   - 每支 API 的 request/response 是否完整到可以直接寫 TypeScript interface？
   - Header 格式、認證方式是否與現有 API client 一致？
   - 分頁 cursor 的編碼格式明確嗎？前端怎麼持有和傳遞？

2. **狀態管理**
   - 哪些資料需要全域狀態？哪些是區域狀態？
   - 樂觀更新的操作有沒有定義 rollback 行為？
   - 列表的排序、篩選邏輯是前端做還是 API 回傳？

3. **網路錯誤處理**
   - 每支 API 的 timeout / retry / 降級行為有定義嗎？（D-5 章節）
   - 哪些 API 失敗是 silent，哪些是 blocking？
   - Loading 狀態定義了嗎？（skeleton / spinner / placeholder）
   - 錯誤 UI 的粒度明確嗎？（inline / toast / 全頁）
   - SWR / stale-while-revalidate 策略有定義嗎？
   - SSR 時 API 失敗的 fallback 行為定義了嗎？
   - 錯誤邊界（Error Boundary）的粒度定義了嗎？

4. **路由與導航**
   - 前端路由規則完整嗎？（含 query parameter 設計）
   - 未登入 → 登入 → 自動跳回的流程定義了嗎？
   - 多入口（書籤/分享連結/導航）進同一頁面的行為一致嗎？

5. **渲染策略**
   - SSR / CSR 的分界明確嗎？
   - 首屏資料載入策略？（SSR 預取？CSR lazy load？）
   - SEO 相關的 meta tag 需求？

6. **UI 行為完整性**
   - 生命週期結束後，哪些元件 disabled / hidden？
   - Empty state 有定義嗎？（列表空、搜尋無結果）
   - 分頁載入的 UX（infinite scroll? load more button?）

7. **存取控制（D-7，如適用）**
   - 哪些路由/元件需要權限 guard？定義了嗎？
   - 無權限時的 UI 行為（redirect / 隱藏 / 提示）明確嗎？

8. **國際化（D-8，如適用）**
   - i18n 框架選型和 locale 偵測策略明確嗎？
   - RTL 佈局有考慮嗎？（如果支援 RTL 語系）

9. **可觀測性（D-9）**
   - 前端 performance timing / error tracking 策略有定義嗎？

### 角色：App RD Reviewer（僅 `app` 選定時啟用）

1. **API 可呼叫性**
   - 每支 API 的 request/response 是否完整到可以直接寫 Model？（iOS: Codable、Android: data class / POJO）
   - Header 格式是否與現有 API 一致？（讀 codebase 確認）
   - 有沒有漏掉的 error code 需要 App 處理？

2. **推播整合**
   - Payload 結構是否與現有 NotificationModel 相容？
   - 新增欄位是否用 optional 保持向下相容？
   - 三種 App 狀態（前景/背景/Cold launch）都有處理嗎？
   - URL scheme 是否正確？（讀 Info.plist / AndroidManifest 確認）

3. **網路錯誤處理**
   - 每支 API 的 timeout / retry / 降級行為有定義嗎？（D-5 章節）
   - 哪些 API 失敗是 silent，哪些是 blocking？
   - 離線時可用的功能有明確列出嗎？
   - 弱網路的降級行為有定義嗎？（圖片降級? 延遲載入?）
   - Request in-flight 時 App 進背景的處理定義了嗎？
   - 離線→上線 的恢復策略定義了嗎？（自動重試佇列?）
   - Token 過期 + 離線的邊界情況有處理嗎？
   - Loading 狀態定義了嗎？（skeleton / spinner）
   - 樂觀更新的操作有沒有定義失敗時的 rollback？

4. **Deep Link / 路由**
   - URL scheme 是否與現有一致？
   - 邀請連結的 Universal Link 處理流程完整嗎？
   - 登入前/後的 token 暫存策略是否明確？（iOS: UserDefaults / Keychain、Android: SharedPreferences / Keystore、或 Memory）

5. **Local vs Remote Action**
   - 哪些操作是離線可用的（Local Action）？
   - 哪些必須連線（Remote Action）？
   - 離線時 Remote Action 的 UI 行為？（隱藏? 灰化? 提示?）

6. **UI 行為完整性**
   - 生命週期結束後，哪些按鈕隱藏？哪些灰化？哪些正常？
   - 分頁載入的 UX（cursor 機制、loading state、empty state）
   - 已讀標記的觸發時機是否明確？

7. **存取控制（D-7，如適用）**
   - 無權限功能的 UI 行為定義了嗎？（隱藏 vs 灰化 vs 提示）

8. **國際化（D-8，如適用）**
   - 裝置語系處理和 string bundle 管理策略明確嗎？

9. **可觀測性（D-9）**
   - crash report 和 performance timing 策略有定義嗎？

### 角色：Backend RD Reviewer（僅 `backend` 選定時啟用）

1. **API 設計一致性**
   - RESTful 規範（HTTP method、status code 語義）
   - 錯誤碼格式統一？（error field 名稱、message 有無）
   - 422 vs 403 的使用是否正確？（業務規則 vs 權限）

2. **資料流完整性**
   - 每支 API 的資料來源明確嗎？（從哪張表取？JOIN 哪些表？）
   - 寫入操作的原子性有保障嗎？（transaction scope）
   - 排程 Job 的觸發條件和失敗處理完整嗎？

3. **並行安全**
   - Token 驗證有 race condition 防護嗎？（atomic UPDATE）
   - 排程 Job 多實例防重複嗎？（ROWLOCK + READPAST 或等效方案）
   - 冪等操作的衝突處理明確嗎？（UPSERT）

4. **效能考量**
   - 查詢有走索引嗎？（UNION ALL vs OR）
   - 分頁用 cursor 不是 offset？
   - 有 N+1 查詢風險嗎？

5. **認證與授權**
   - userID 來源一致？（全從 token 解析）
   - 跨資源存取有防護嗎？（NOT_MEMBER check）
   - Admin API 用不同認證機制？

6. **錯誤處理與回傳**
   - 前端/App 需要的每種錯誤都有對應的 error code？
   - error response 格式一致嗎？（每支 API 都有 error field？有些有 message 有些沒有？）
   - Rate limit 的 429 response 有帶 Retry-After header 嗎？

7. **存取控制（D-7，如適用）**
   - API 層的權限檢查邏輯明確嗎？（middleware / service layer）
   - 多租戶資料隔離有防護嗎？（如適用）

8. **國際化（D-8，如適用）**
   - API 有處理 Accept-Language header 嗎？
   - 多語內容的儲存和回傳策略明確嗎？

9. **可觀測性（D-9）**
   - 關鍵業務流程有 structured logging 嗎？（correlation ID, request tracing）
   - 告警規則和觸發條件定義了嗎？

### 角色：DBA Reviewer（僅 `dba` 選定時啟用）

1. **Schema 設計**
   - PK 選擇是否合理？（PK 型別對索引效能的影響，依目標 DB 判斷）
   - FK 兩端型別一致嗎？
   - 索引涵蓋主要查詢路徑嗎？
   - 有沒有冗餘索引或遺漏索引？

2. **目標 DB 相容性**
   - 對照表完整嗎？有沒有遺漏的轉換項目？
   - Trigger / 自動更新機制的語法正確嗎？（遞迴保護）
   - UNIQUE INDEX + NULL 行為處理了嗎？（各 DB 行為不同）
   - JSON 欄位有適當的約束嗎？（MS-SQL: ISJSON CHECK、PostgreSQL: jsonb 型別、MySQL: JSON 型別）

3. **效能與擴展**
   - 主鍵索引策略是否適合目標 DB？（自增 vs UUID vs 自訂）
   - 大表的分頁查詢效能（keyset pagination 是否有適當索引支撐）
   - 讀寫分離建議合理嗎？（如果有讀寫分離架構）
   - 寫入密集的表是否有索引碎片化風險？

4. **資料完整性**
   - 列舉欄位有 CHECK 約束嗎？
   - 必填欄位都 NOT NULL 嗎？
   - DEFAULT 值合理嗎？
   - 時間欄位精度一致嗎？

5. **存取控制（D-7，如適用）**
   - 多租戶的 tenant_id 欄位設計合理嗎？
   - Row-level security 有定義嗎？（如果需要 DB 層隔離）

6. **國際化（D-8，如適用）**
   - 多語內容的 DB 設計策略明確嗎？（分表 / JSON / 欄位後綴）

### 角色：QA / Consistency Checker（永遠啟用）

1. **API ↔ DB 一致性**
   - API response 的每個欄位都能從 DB schema 取得嗎？
   - API 的 computed field 計算邏輯和 DB 資料匹配嗎？
   - API 寫入的欄位和 DB NOT NULL 約束匹配嗎？

2. **FRD ↔ Delivery Spec 一致性**
   - FRD 的每個 FR 都有對應的 API 嗎？
   - FRD 的邊界條件都有對應的錯誤碼嗎？
   - FRD 的行為矩陣和 D-4 結束後行為矩陣一致嗎？

3. **內部一致性**
   - 時序圖中的 API 呼叫和 C 章節的 API 定義一致嗎？
   - A 章節矩陣中的「輸出」和 C 章節的 API 路徑一致嗎？
   - 推播 payload 的欄位和 App/Web 端路由邏輯一致嗎？

4. **網路錯誤一致性**（`app` 或 `f2e` 選定時）
   - D-5 網路策略矩陣涵蓋所有 C 章節的 API 嗎？
   - 標為「可 retry」的 API，其操作是否真的冪等？（retry 非冪等的 POST 會出事）
   - 標為「樂觀更新」的操作，有定義 rollback 嗎？
   - 標為「離線可用」的功能，其依賴的資料有快取策略嗎？

5. **遺漏偵測**
   - 有沒有 API 定義了但時序圖沒出現的？
   - 有沒有 DB 表定義了但沒有 API 操作的？
   - 有沒有邊界條件（取消、過期、重複、網路錯誤）沒有處理的？
   - 有沒有 Phase1 標為「不做」但 API 裡又出現的？
   - FRD 提到多租戶/權限需求，但 Delivery Spec 沒有 D-7 存取控制章節？
   - FRD 提到多語系需求，但 Delivery Spec 沒有 D-8 i18n 章節？
   - FRD 提到可觀測性需求，但 Delivery Spec 沒有 D-9 可觀測性章節？
   - 有沒有 API 缺少可觀測性考量（關鍵業務流程的 logging/metrics）？

---

### Step 2：合併報告

等所有 subagent 回傳後：

1. **去重**：不同角色可能發現同一個問題，合併後標註「多角色交叉確認」
2. **分級排序**：Blocking → Ambiguous → Suggestion
3. **編號**：每個 finding 給 ID（如 R3-B-01 = Round 3, Blocking, #01）
4. **章節歸屬**：標示 finding 屬於哪個章節

### Step 3：產出 Review Report

**格式：Markdown**（HackMD 相容）

報告結構：
```
# {專案名稱} — Review Report（Round {N}）
## 審查角色：{列出本次啟用的角色}

## 摘要
- 🔴 Blocking: {n} 項
- 🟡 Ambiguous: {n} 項
- 🟢 Suggestion: {n} 項

## Blocking
{按章節分組列出}

## Ambiguous
{按章節分組列出}

## Suggestion
{按章節分組列出}

## 各角色原始報告
（可折疊）
```

每個 finding 的格式：
```
### {ID} [{章節}] {標題}
**嚴重度：** {Blocking / Ambiguous / Suggestion}
**發現者：** {角色}
**問題：** {具體描述}
**建議：** {修正建議}
```

---

## 後續動作

Report 產出後，詢問使用者：

> 「Review Report 已產出，共 {n} Blocking、{n} Ambiguous、{n} Suggestion。
> 
> 選項：
> 1. 逐項檢視 Blocking，先修 Blocking
> 2. 全部檢視，再決定修哪些
> 3. 直接開始修所有 Blocking + Ambiguous
> 4. 匯出報告，稍後處理」

如果使用者要修：
- 按章節順序修改 Delivery Spec
- 修完後 commit
- 詢問是否要重新 review

---

## 產出物

- 檔案：`docs/specs/{topic}-review-round{N}.md`
- Artifact：可選，使用者需要時發佈預覽
- Commit message：`docs: add {topic} review report round {N} ({n} findings)`
