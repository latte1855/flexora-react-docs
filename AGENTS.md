# AGENTS.md — Flexora ERP Workspace 指南  
（給 Codex / AI Agent / Automation Agent）

> 📌 Workspace 根目錄：`flexora/`  
> 本文件說明：在 Flexora ERP 的前後端與文件共存環境中，  
> AI Agent 應如何分析檔案、撰寫程式、產生文件、同步更新規格。

---

# 0. Workspace 結構（請務必遵守）

請假設 VS Code / Codex 的工作目錄為：

```text
flexora/
  docs/                ← 文件專案（本 repo）
  flexora-react/       ← ERP 後端（Java / JHipster）
  flexora-react-ui/    ← ERP 正式前端（React / Ecme Template）
```

三個資料夾 **都是獨立的 Git Repo**。
請勿在錯誤的 repo 進行 commit。

---

# 1. Agent 操作基本原則（最重要）

1. **不得跨 repo 混寫檔案**

   * 修改後端 → 只能 commit 在 `flexora-react`
   * 修改正式前端 → 只能 commit 在 `flexora-react-ui`
   * 修改文件 → 只能 commit 在 `docs`

2. **程式碼行為變更必須同步更新文件**
   當你新增、調整或優化以下內容時：

   * REST API（新增路徑、參數、DTO、錯誤格式…）
   * Entity / DTO / Domain 模型欄位
   * 業務流程（報價、訂單、庫存、請購…）
   * 前端 UI 表單欄位、驗證、事件流程

   必須同步更新：

   * `/docs/specs/`（模組功能規格）
   * `/docs/api-spec/`（必要時）
   * `/docs/architecture/`（若屬架構或流程更新）

3. **Javadoc / TSDoc 一律使用繁體中文說明**

   * Java 類、方法、欄位 → 必須撰寫中文 Javadoc
   * 前端 TS/TSX → 重要邏輯需有中文註解

4. **不可未經確認擅自調整架構或 public API**
   若改動以下內容，需寫進 Decision Log 或 ADR：

   * API 格式 / 命名規則
   * 錯誤回傳格式
   * 核心資料模型欄位
   * 狀態機流程（報價、訂單、庫存等）

5. **所有功能規格以 `/docs/specs/` 為 Single Source of Truth（SSOT）**
   若程式碼與文件不一致：

   * 以程式碼為準
   * 再自動或協助修正文件

---

# 2. 文件（docs/）目錄使用規範

```text
docs/
  README.md
  AGENTS.md
  srs/
  api-spec/
  architecture/
  decisions/
  guides/
  specs/
  samples/
  misc/
```

### 2.1 新文件放置原則

| 類型                | 放置目錄                                      |
| ----------------- | ----------------------------------------- |
| 新功能規格             | `specs/flexora-erp/.../`                  |
| API 介面 / 快照       | `api-spec/flexora-erp/`                   |
| 模組 / Domain 邏輯變更  | `architecture/flexora-erp/`               |
| 資料表調整             | `architecture/flexora-erp/DATA_MODEL.md`  |
| 重要決策（API、架構、資料模型） | `decisions/`                              |
| UI 操作規格           | `specs/` 或 `guides/frontend-dev-guide.md` |
| 範例資料              | `samples/`                                |
| 無法判斷分類            | `misc/`（並提醒使用者分類）                         |

---

# 3. 後端（flexora-react）操作守則

Flexora ERP 後端使用：

* JHipster 8.11.0
* Spring Boot 3.x
* Java 17+
* Code-first API（Controller 為真正權威）

Agent 在後端編輯時：

1. 若修改 REST Controller：

   * 同步更新 `/docs/specs/` 與必要的 OpenAPI 快照
2. 若修改 Entity / DTO：

   * 同步更新 `/docs/architecture/DATA_MODEL.md`
3. 若新增 Service / Domain 行為：

   * 補上繁中 Javadoc
   * 更新對應模組的規格（報價、庫存、銷售訂單…）
4. 嚴禁：

   * 靜默移除欄位
   * 靜默變更 API 路徑或 response 格式
   * 未更新文檔的模型/流程變更

---

# 4. 前端（flexora-react-ui）操作守則

（正式 ERP 前端）

正式前端使用：

* Ecme React Admin Template
* React 18+
* TypeScript
* Tailwind CSS

Agent 在前端編輯時：

1. UI 表單欄位 / 驗證規則變更 → 更新 `/docs/specs/`
2. 新增 Page / Component → 加入繁體中文註解
3. 新 API 呼叫 → 檢查後端是否已有端點
4. 避免引入不必要的 UI 套件（若需引入，需在文件註明原因）

---

## 4.1 JHipster 內建 React UI 的用途（非常重要）

ERP 後端（flexora-react）中仍保留 JHipster 自動產生的內建 React UI。
此 UI **並非正式產品前端**，用途如下：

1. **後端開發測試使用**

   * 讓後端工程師快速驗證 API、CRUD、驗證規則是否正確
   * 可由 Codex / Agent 自動產生頁面來驗證後端功能

2. **API 正確性驗證**

   * 用於確認 DTO、回傳格式、錯誤處理是否符合規格
   * 確保正式前端能順利串接 API

3. **提供正式前端（flexora-react-ui）做參考與複製**

   * 正式前端可參考 JHipster 內建頁面的欄位、流程、欄位驗證
   * 避免 API 使用錯誤
   * 在新功能未來不及時，可短暫參考內建 UI 的範例實作

> ⚠️ 注意：
> 內建 React UI **只能做後端驗證與前端參考**，
> 正式 ERP UI 一律在 `flexora-react-ui/` 中開發。

---

# 5. 文件更新同步的黃金準則（Golden Rules）

每次後端 / 前端變更後 → 逐一檢查：

### ✔ 是否影響前端 UI？

→ 若是，更新 `/docs/specs/`。（表單欄位、驗證、流程）

### ✔ 是否影響後端 API？

→ 若是，更新 `/docs/specs/` 與 `/docs/api-spec/`

### ✔ 是否影響資料模型？

→ 若是，更新 `/docs/architecture/DATA_MODEL.md`

### ✔ 是否屬重大設計決策？

→ 更新 `/docs/decisions/DECISION_LOG.md` 或新增 ADR

---

# 6. 錯誤處理守則

若 Agent 發現：

### ◼ 文件與程式碼不一致

→ 以程式碼為準
→ 並協助修正文件

### ◼ 規格缺漏

→ 建立 Draft（草稿）
→ 標記「Pending Human Review」

### ◼ API 或流程格式錯誤

→ 提醒使用者並提出可行修正

---

# 7. 安全界線（Do / Don't）

### ✔ DO（允許）

* 自動補註解（Javadoc/TSDoc）
* 同步更新 spec / API 文件
* 產生流程圖/ER 圖/Mermaid 草稿
* 梳理文件結構、修正文檔錯誤
* 自動比對程式與文件差異並提示

### ❌ DON'T（禁止）

* 擅自創造 API、欄位、流程
* 修改核心業務流程而不通知使用者
* 靜默變更資料模型
* 刪除文件無備註

---

# 8. 結語

本文件確保 Flexora ERP 的所有 AI 工具與 Agent
都能在多 repo 環境下正確工作，
並維持前後端與文件之間的一致性。

若遇不確定情況，請保守處理並提示使用者。

