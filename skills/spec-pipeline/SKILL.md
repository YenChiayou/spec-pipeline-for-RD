---
name: spec-pipeline
description: PRD → FRD → Delivery Spec → Review → Export 的完整交付管線。從產品需求文件出發，經過互動式釐清、結構化交付規格、多角色審查、格式輸出，產出可直接交給 RD 開工的交付文件。每個階段都有人類 checkpoint。支援可選角色組合：F2E / Backend / DBA / App(Native)。
---

# spec-pipeline:spec-pipeline — PRD-to-Delivery 交付管線

把 PRD 轉換成**所有相關 RD 可直接開工的交付文件**，經過結構化審查。

## 設計哲學

這不是全自動 pipeline。每個階段需要人類 checkpoint——因為最有價值的決策發生在階段之間，不是階段之內。

```
PRD ──→ [spec-pipeline:spec-extract] ──→ FRD ──→ [spec-pipeline:spec-deliver] ──→ Delivery Spec
                 ↑ 人類確認                          ↑ 人類確認
                                                                       │
         Export ←── [spec-pipeline:spec-export] ←── Review ←── [spec-pipeline:spec-review]
                                       ↑ 人類挑戰
```

## Usage

```
spec-pipeline:spec-pipeline                     # 從零開始，互動引導
spec-pipeline:spec-pipeline {prd-path}          # 讀取既有 PRD，進入 Phase 1
spec-pipeline:spec-pipeline resume              # 從上次停下的階段繼續
spec-pipeline:spec-pipeline phase {1|2|3|4}     # 跳到指定階段（需要前一階段的產出物）
```

---

## Step 0：選擇角色組合（Pipeline 入口）

在進入任何 Phase 之前，必須先確認本次專案涉及哪些角色。這會影響後續所有 Phase 的行為。

### 角色池

| Key | 角色 | 職責範圍 | 適用專案 |
|-----|------|---------|---------|
| `f2e` | 前端 RD (F2E) | SPA/SSR 路由、狀態管理、API 串接、渲染策略、快取 | Web 前端專案 |
| `app` | App RD (Native) | 推播整合、Deep Link、Local/Remote Action、離線、原生元件 | iOS / Android 原生 |
| `backend` | Backend RD | API 設計、資料流、排程 Job、並行安全、認證授權 | 幾乎所有專案 |
| `dba` | DBA | Schema 設計、目標 DB 對照、索引策略、讀寫分離 | 有 DB 的專案 |
| `qa` | QA / 一致性檢查 | 跨章節交叉比對、FRD↔Spec 一致性、遺漏偵測 | **永遠啟用，不可關閉** |

### 常見組合預設

詢問使用者選擇，提供常見預設 + 自訂選項：

| 預設 | 角色組合 | 典型專案 |
|------|---------|---------|
| Web 全端 | `f2e` + `backend` + `dba` | 後台系統、SaaS |
| App 全端 | `app` + `backend` + `dba` | Native App 功能 |
| Full Stack | `f2e` + `app` + `backend` + `dba` | 跨平台功能 |
| Pure API | `backend` + `dba` | 微服務、API Gateway |
| 自訂 | 使用者自選 | 特殊需求 |

**選擇結果會傳遞給後續每個 Phase skill。** 具體影響：

| Phase | 角色如何影響 |
|-------|------------|
| spec-extract | 問的問題依角色調整（有 `app` 才問推播/Deep Link，有 `f2e` 才問 SSR/CSR） |
| spec-deliver | 系統責任矩陣的系統欄位、時序圖的 participant、D 章節範圍 |
| spec-review | fan-out 的 subagent 數量和 prompt（只啟動選中的角色 + QA） |
| spec-export | 不受影響（格式轉換是角色無關的） |

---

## Pipeline 階段

| Phase | Skill | 輸入 | 輸出 | 判斷密度 |
|-------|-------|------|------|---------|
| 1. Extract | `spec-pipeline:spec-extract` | PRD（任意格式） | FRD markdown | 高 — 需要人類決策 |
| 2. Deliver | `spec-pipeline:spec-deliver` | FRD markdown | Delivery Spec（Markdown，HackMD 相容） | 中 — 結構固定，內容依 FRD |
| 3. Review | `spec-pipeline:spec-review` | Delivery Spec | Review Report（Markdown） | 低 — subagent fan-out |
| 4. Export | `spec-pipeline:spec-export` | Delivery Spec | 其他格式（HTML / 重新排版） | 純機械 |

---

## 執行流程

### Step 1：理解現況

1. 檢查 `docs/specs/` 目錄，看是否已有產出物
2. 如果有 FRD → 問使用者要從 Phase 2 繼續還是重新開始
3. 如果有 Delivery Spec → 問使用者要 review 還是 export
4. 確認專案的技術棧（讀 CLAUDE.md、package.json、Podfile、Gemfile 等）
5. 如果 Step 0 還沒做，先做角色選擇

### Step 2：Phase 1 — Extract（PRD → FRD）

如果使用者指定了 PRD 檔案路徑，先確認檔案存在。如果不存在，告知使用者並詢問正確路徑。

呼叫 `spec-pipeline:spec-extract`，傳入選定的角色組合。

完成後，請使用者確認 FRD：
> 「FRD 已產出在 `{path}`。請檢查後告訴我是否要進入 Phase 2（產出交付規格），或有需要修改的地方。」

**不要自動進入 Phase 2。** 等使用者確認。

如果使用者要修改 FRD：
1. 釐清要修改哪些章節
2. 就地編輯 FRD
3. 修改完成後重新提示 checkpoint 選項
如果修改幅度過大（例如範圍重新定義），建議從 Phase 1 重新執行。

### Step 3：Phase 2 — Deliver（FRD → Delivery Spec）

呼叫 `spec-pipeline:spec-deliver`，傳入選定的角色組合。

完成後，請使用者確認：
> 「Delivery Spec 已產出在 `{path}`。要進入 Review 階段嗎？」

### Step 4：Phase 3 — Review（多角色審查）

呼叫 `spec-pipeline:spec-review`，傳入選定的角色組合。

Review 會 fan-out N 個 subagent（選中的角色 + QA）平行審查，產出合併報告。

完成後：
> 「Review Report 已產出，共 {n} 個 Blocking、{n} 個 Ambiguous、{n} 個 Suggestion。要逐項檢視嗎？」

如果使用者要修，就地修改 Delivery Spec，不需要重跑整個 pipeline。
如果使用者想重新 review 修改後的版本，再次呼叫 `spec-pipeline:spec-review`。

### Step 5：Phase 4 — Export（格式轉換）

呼叫 `spec-pipeline:spec-export`。

詢問目標格式（預設 HTML，用於歸檔/列印）。如果使用者需要預覽，可用 Artifact 發佈。

---

## 檔案慣例

所有產出物放在 `docs/specs/` 下：

```
docs/specs/
  YYYY-MM-DD-{topic}-frd.md              # Phase 1 產出
  {topic}-system-flows.md                # Phase 2 產出（Markdown，HackMD 相容）
  {topic}-review-round{N}.md             # Phase 3 產出
  {topic}-system-flows.html              # Phase 4 產出（optional，需要時才跑）

```

## 狀態追蹤

Pipeline 狀態寫入 `docs/specs/.pipeline-state.json`，結構：

```json
{
  "roles": ["f2e", "backend", "dba"],
  "currentPhase": 2,
  "completedPhases": [1],
  "frdPath": "docs/specs/2026-08-13-topic-frd.md",
  "specPath": null,
  "reviewPath": null
}
```

每完成一個 phase 就更新此檔案。使用者可以 `spec-pipeline:spec-pipeline resume` 時讀取此檔案繼續。
