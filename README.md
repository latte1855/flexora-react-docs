# Flexora ERP 文件專案

本專案為 **Flexora ERP** 的集中式文件庫，  
負責管理 ERP 的 **需求、規格、架構、流程、API、決策紀錄、與開發指南**。

> 📌 **程式碼不在此 repo 中**  
> ERP Backend：`../flexora-react`  
> ERP Frontend：`../flexora-react-ui`  
> 本 repo 僅保存文件（Markdown）。

---

# 1. 文件目錄

本文件庫依照 ERP 的典型開發流程分成數個主要區域：

```text
docs/
  README.md        ← 本文件
  AGENTS.md        ← 給 Codex / AI Agent 的 Workspace 使用說明

  srs/             ← 系統需求規格（System Requirement Specification）
  api-spec/        ← API / OpenAPI / Postman 相關文件
  architecture/    ← 系統架構、資料模型、流程、設計決策
  decisions/       ← ADR / Decision Log
  guides/          ← 開發 / 作業 / 部署指南
  specs/           ← ERP 各模組功能規格（Phase-based）
  modules/         ← 依模組整理的統一入口（逐步搬遷中）
  samples/         ← 範例資料（CSV / Excel / JSON）
  misc/            ← 其他未分類資訊（術語、TODO 等）
```

## 1.1 `srs/` — 系統需求規格

描述 Flexora ERP 的整體背景、目標、業務流程、非功能性需求（NFR），
包含：

* ERP 全域需求（登入、角色、權限、審計、通知…）
* 全域資料模型原則
* 全域 API 與錯誤格式規格
* 非功能性需求（效能 SLA、安全性、可觀察性等）

> **建議：** 每次重大版本（Phase）都應更新一份差異。

---

## 1.2 `api-spec/` — API 規格

由後端 JHipster Controller 所使用的 **Code-first API** 產生：

* `openapi/`：從 `/v3/api-docs` 匯出的 OpenAPI YAML
* `postman/`：Postman / Insomnia API collection
* `errors.md`：全域錯誤碼、BadRequestAlert、Validation 回傳格式

> **提醒：** OpenAPI 不是權威，只是快照。
> 權威來源是：`flexora-react/src/main/java/.../web/rest/*Resource.java`

---

## 1.3 `architecture/` — 系統架構

用於記錄 Flexora ERP 的技術與邏輯架構，包含：

* OVERVIEW（整體架構圖）
* MODULES（ERP 模組邊界與相依）
* DATA_MODEL（全域 ER 圖、Domain 物件規則）
* WORKFLOW（報價、訂單、庫存等流程）
* FRONTEND_STRUCTURE（前端頁面與路由規範）
* BACKEND_STRUCTURE（後端分層設計、Service & Domain 原則）

---

## 1.4 `decisions/` — 決策紀錄（ADR / Decision Log）

記錄專案的重要技術/架構決策，例如：

* 為何採用 JHipster 8.11.0？
* 為何前端改用 Ecme React Template？
* 為何採 Code-first 而非 API-first？
* 為何資料模型採用 X 原則？
* 重大流程（如報價、庫存）的核心決策

架構類決策應使用 ADR 格式，例如：

```
ADR-0001-use-code-first-api.md
ADR-0002-domain-separation-strategy.md
```

---

## 1.5 `guides/` — 指南（Guide）

包含：

* Backend 開發指南（分層、命名、DTO、測試）
* Frontend 開發指南（檔案結構、狀態管理、API 呼叫規範）
* Local 開發環境設置
* Release / Deploy 流程

---

## 1.6 `specs/` — ERP 功能規格（Phase-based）

每個 ERP 模組都應在此有獨立目錄，例如：

```text
specs/
  phase0-system-foundation/
  phase1-product-pricing/
  phase3-inventory/
  phase4-quotation/
  sales-order/
  purchase-order/
```

內容通常包含：

* 背景與目標
* 業務流程（Flow 或狀態機）
* API 行為說明
* UI 規格（表單層級）
* 資料表欄位規格
* 錯誤與防呆

---

## 1.7 `samples/` — 範例資料

存放：

* CSV / Excel 模板（導入/匯出）
* JSON Data Fixtures（方便測試）
* 圖片/流程圖原始檔（若必要）

---

## 1.8 `misc/` — 其他

例如：

* GLOSSARY（術語表）
* TODO / Backlog
* 版本演進紀錄

---

## 1.9 `modules/` — 模組入口（新）

為了減少文件分散，每個模組會在 `modules/<module>/` 建立 README、spec、ui-spec、api-spec、implementation-plan 等統一入口。  
目前尚在搬遷階段，文件內容仍連回舊的 `specs/phase*` 或 `ui/`，但往後查詢特定模組時可先從這裡進入。  
模組索引請參考 [`modules/README.md`](modules/README.md)。 

---

# 2. 專案關聯（三個 repo）

```text
flexora/
  docs/               ←（本文件 repo）
  flexora-react/      ← JHipster Backend（Java）
  flexora-react-ui/   ← Ecme React Template Frontend（React）
```

這三個專案共同組成 Flexora ERP。

* 若 **後端行為變更** → 應同步更新 `docs/specs/` 與 `docs/api-spec/`
* 若 **前端行為變更** → 應更新 `docs/specs/` 或 `docs/guides/frontend-dev-guide.md`
* 若 **架構與資料模型變更** → 應更新 `docs/architecture/`

---

# 3. 文件編寫規範（非常重要）

### 3.1 Markdown 為標準格式

所有文件均以 `.md` 編寫。

### 3.2 使用繁體中文

包含：

* SRS
* 規格文件
* 架構說明
* Javadoc / TSDoc（允許混中英）

### 3.3 圖表可使用（建議）

* Mermaid
* PlantUML
* Excalidraw（放圖檔即可）

### 3.4 所有重要變更**必須回寫文件**

特別是：

* API 自動產生的 controller 改動
* Domain 資料模型調整
* UI 欄位變更
* Workflow 行為變更

---

# 4. 如何與後端 / 前端協作？

### 後端改動 → 請更新：

* `specs/` 相關模組規格
* `api-spec/`（必要時）
* `architecture/DATA_MODEL.md`

### 前端改動 → 請更新：

* `specs/` 的 UI 與操作規格
* `guides/frontend-dev-guide.md`

### 架構/技術決策 → 請更新：

* `decisions/DECISION_LOG.md`
* `architecture/` 內的相關文件

---

# 5. 貢獻規則

1. 每個變更請盡量 **一項主題、一個 commit**。
2. Commit message 盡量簡潔並使用英文，例如：

   * `docs(specs): update quotation validation rules`
   * `docs(architecture): add inventory stock flow`
3. 新增文件前，請先確認是否已存在對應分類。

---

# 6. 聯絡方式（暫定）

* 文件維護者：Jimmy Liu
* 負責工具：Codex / GPT / Docs Automation Pipeline（未來）

---

# 7. 結語

Flexora ERP 的核心精神是 **同步、可追蹤、可維護、模組化**。
本文件庫的目的，就是讓前後端與所有模組能在同一份真實來源（Single Source of Truth）上協作。

若發現文件與程式碼不一致，請優先以程式碼為準，並盡快於文件中修正。
