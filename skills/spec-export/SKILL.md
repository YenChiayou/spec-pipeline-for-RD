---
name: spec-export
description: Delivery Spec 格式轉換器。將 Markdown Delivery Spec 轉換為 HTML（可列印 / 正式歸檔）或其他格式，保留所有技術內容（mermaid 圖、code blocks、表格、API 契約）。
---

# spec-pipeline:spec-export — Delivery Spec 格式轉換器

把 Delivery Spec 轉換成其他格式（歸檔、列印、分享給不用 HackMD 的人）。

## 核心原則

> **格式變，內容不變。** 轉換過程不修改任何技術內容，只做結構映射。

---

## Usage

```
spec-pipeline:spec-export                              # 自動找最新的 Delivery Spec，預設轉 HTML
spec-pipeline:spec-export {spec-path}                  # 指定來源文件
spec-pipeline:spec-export {spec-path} --format html    # 明確指定目標格式
```

如果未指定 Delivery Spec 路徑，在 `docs/specs/` 目錄中以 `*-system-flows.md` pattern 搜尋。如果有多個，取檔案修改時間最新者。如果找不到，詢問使用者指定路徑。

如果使用者指定了路徑，先確認檔案存在。如果不存在，告知使用者並詢問正確路徑。

---

## 支援的轉換

| 來源 | 目標 | 用途 |
|------|------|------|
| Markdown → HTML | 正式文件歸檔、列印、Artifact 預覽 | **預設** |
| Markdown → PDF | 正式文件歸檔 | 透過 Artifact 列印 |

---

## Markdown → HTML 轉換規則

### 結構映射

| Markdown | HTML |
|----------|------|
| `# ` | `<h1>` |
| `## ` | `<h2>` |
| `### ` | `<h3>` |
| `#### ` | `<h4>` |
| `> ` blockquote | `<div class="note">` |
| emoji 圖例列表 | `<div class="system-legend">` chips |

### 系統圖例

```
Markdown emoji → HTML chips:
  🔵 Web Frontend            → chip-frontend
  🟢 App（iOS + Android）   → chip-app
  🟣 Backend API / DB       → chip-backend
  🔴 推播平台（FCM / APNs） → chip-push
  🟡 外部服務 / CMS         → chip-content
```

### Mermaid 圖

```
Markdown: ```mermaid
           ...
           ```

HTML: <pre class="mermaid">...</pre>
```

Mermaid 圖使用 `<pre class="mermaid">`。如果用 Artifact 發佈，不需要額外 script（Artifact 原生支援）。如果產出獨立 HTML 檔案供離線檢視，需要 inline mermaid.min.js。

```
（以下為獨立 HTML 才需要的 script 範例）
<script src="...mermaid.min.js"></script>
```

### Code Blocks

```
Markdown: ```sql / ```json / ``` (plain)

HTML: <div class="api-block">...</div>
```

- 特殊字元轉 HTML entities（`<` → `&lt;`, `>` → `&gt;`, `&` → `&amp;`）
- `**text**` 在 code block 內 → `<strong>text</strong>`

### Markdown Table → HTML Table

```
Markdown: | ... | ... |
          |-----|-----|
          | ... | ... |

HTML: <table>
  <thead><tr><th>...</th></tr></thead>
  <tbody><tr><td>...</td></tr></tbody>
</table>
```

### HTML 輸出要求

- 自包含（所有 CSS inline）
- 支援 light/dark theme
- 響應式（max-width: 960px）
- 可選：用 Artifact 發佈預覽（使用者需要時）

---

## 品質檢查

轉換後自我檢查：

| 檢查項 | 方法 |
|--------|------|
| Mermaid 圖數量一致？ | 數 Markdown 的 ` ```mermaid ` vs HTML 的 `<pre class="mermaid">` |
| API 契約數量一致？ | 數 C-0 ~ C-N 的標題數 |
| DB 表數量一致？ | 數 E-1 ~ E-N 的標題數 |
| A 章節責任矩陣表格完整？ | 抽查表格欄數和分組標題數 |
| D 章節數量一致？ | 數 D-1 ~ D-9 的標題數（僅比對存在的 D 章節） |
| 表格欄數一致？ | 抽查幾張表的欄位數 |

---

## 產出物

- 檔案：`docs/specs/{topic}-system-flows.html`
- Commit message：`docs: export {topic} delivery spec to HTML`

完成後提示使用者：
> 「HTML 版已產出在 `{path}`。可以用 Artifact 預覽或直接開啟瀏覽器檢視。」
