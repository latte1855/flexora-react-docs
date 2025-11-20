# Flexora ERP — 系統架構總覽（Overview）

本文件提供 Flexora ERP 的整體架構全貌，  
包含技術選型、模組邊界、資料流、前後端整合、開發流程與文件協作方式。

> 本文件為架構總入口（entry point）。  
> 詳細資料模型、模組規格、流程圖等請參考同目錄下其他文件。

---

# 1. Flexora ERP 的定位

Flexora ERP 為一套模組化、可擴充、可長期維護（10+ 年生命周期）的企業管理系統。  
核心目標：

- 以 **模組化架構** 支援不同業務領域（產品、庫存、報價、訂單、採購…）
- 保持 **可讀性、可維護性、可測試性** 的後端 Domain 分層
- 前後端 **完全分離**
- 文件與程式碼分離但同步（docs = SSOT）

---

# 2. 系統架構總覽（High-Level Architecture）

```

+---------------------------------------------------------------+
|                       Flexora Workspace                       |
|                                                               |
|  +----------------+   +-------------------+   +--------------+ |
|  |  flexora-react |   | flexora-react-ui  |   |    docs      | |
|  |  (Backend)     |   |  (Frontend)       |   | (Documentation)|
|  +----------------+   +-------------------+   +--------------+ |
|        | REST API                | UI/API Usage       ^        |
|        |                         v                    |        |
|   [Spring Boot] <------------> [React + Tailwind] <---+        |
|                                                               |
+---------------------------------------------------------------+

```

### 三大子專案（各自獨立 Git Repo）：

| Repo | 說明 |
|------|------|
| `flexora-react` | 後端，基於 JHipster 8.11.0 / Spring Boot 3 |
| `flexora-react-ui` | 正式 ERP 前端（Ecme React Admin Template） |
| `docs` | 完整文件 repo（SRS / specs / API / architecture） |

---

# 3. 後端（flexora-react）架構

後端基於 **JHipster 8.11.0**，採用 Spring Boot 3.x：

### 3.1 技術選型

- Java 17+
- Spring Boot 3.x
- Spring MVC / REST
- Spring Data JPA / Hibernate
- Spring Security（JWT）
- PostgreSQL（主要資料庫）
- MapStruct（DTO Mapper）
- JHipster Code-first API
- JUnit5 / AssertJ / Mockito / Integration Tests

---

### 3.2 後端分層架構

```

┌───────────────────────────┐
│        Web Layer          │  ← Resource / Controller
└───────────────┬───────────┘
│
┌───────────────▼───────────┐
│        Service Layer       │  ← Business Logic
│    (Application Service)   │
└───────────────┬───────────┘
│
┌───────────────▼───────────┐
│        Domain Layer        │  ← Entity / Aggregate / Rules
└───────────────┬───────────┘
│
┌───────────────▼───────────┐
│    Persistence Layer       │  ← Repository (JPA)
└────────────────────────────┘

```

---

### 3.3 API 設計原則（Code-first）

Flexora ERP 採 **Code-first API**：

- Controller 是 API 的唯一真實定義（SSOT）
- OpenAPI（`/v3/api-docs`）僅用作文件輸出
- DTO 必須使用 MapStruct 進行轉換
- 所有 API 必須提供完整中文 Javadoc

---

# 4. 前端（flexora-react-ui）架構

前端基於：

- **Ecme React Tailwind Admin Template**
- React 18+
- TypeScript
- Tailwind CSS

### 4.1 前端目錄結構（建議）

```

src/
components/          ← 共用 UI 元件
hooks/
layouts/
pages/
services/            ← API 呼叫封裝，對應後端路徑
store/               ← 狀態管理（選擇 RTK / Zustand / Query 等）
utils/
assets/

```

---

# 5. JHipster 內建前端（非正式前端）

後端專案 `flexora-react/` 內仍包含 JHipster 自動產生的 React UI。

其用途為：

1. ✔ 後端 API 功能驗證  
2. ✔ 驗證欄位/DTO/錯誤格式一致性  
3. ✔ 提供正式前端 `flexora-react-ui` 參考（表單欄位、驗證、流程）

> ⚠ 此前端不可作為正式 ERP 前端使用。  
> 正式 ERP UI = `flexora-react-ui/`

---

# 6. 系統資料流（Data Flow Overview）

```

[Front-End UI]
|
| HTTP REST API
v
[Web Layer (Controller)]
|
v
[Service Layer (業務邏輯)]
|
v
[Domain Model]
|
v
[Repository (JPA)]
|
v
[PostgreSQL]

```

---

# 7. 模組化設計（Modular Boundaries）

Flexora ERP 持續演進中，但主要模組邊界如下（依 Roadmap）：

- **Phase 0 — System Foundation**
  - Owner / Team / User / Role
  - 基礎 API / 驗證 / 錯誤格式

- **Phase 1 — Product Management / Pricing**
  - Item / SKU / UoM / PriceList

- **Phase 3 — Inventory Management**
  - Warehouse / Bin  
  - Stock / Reservations  
  - Inventory Transactions

- **Phase 4 — Quotation**
  - QuotationThread  
  - QuotationRevision  
  - Workflow / Guard Rules

- **Additional Upcoming Modules**
  - Sales Order  
  - Delivery Note  
  - Purchase Order  
  - Vendor / RFQ  
  - 抽屜式 workflow drawer  
  - ExtAttrs（自訂欄位）

---

# 8. 文件與程式碼同步策略（Docs = SSOT）

本架構採用 **文件與程式碼分離但同步** 的策略：

| 行為 | 需更新文件 |
|------|------------|
| API 變更 | `/docs/specs/` + `/docs/api-spec/` |
| Entity / DTO 變更 | `/docs/architecture/DATA_MODEL.md` |
| 業務流程變更 | `/docs/specs/<module>/` |
| UI 欄位/流程變更 | `/docs/specs/` |

文件庫（docs）為前後端與 Agent 的 **Single Source of Truth**。

---

# 9. 自動化與 Agent 介入的階段

Codex / Agent 允許自動化：

- CRUD API 產生
- UI 表單草稿產生
- 文件同步（依程式碼變更更新 specs）
- 流程圖 / ER 圖自動產生
- 驗證 API File / DTO 正確性

但 **不可**：

- 擅自新增無來源的 API 或欄位
- 修改 Domain 業務邏輯（僅能依指示）

---

# 10. 部署架構（未來擴充）

此區域視實作進度補充：

- Docker Compose（開發）
- K8s / Helm（產品）
- Redis（Cache）
- CI/CD Pipeline（GitHub Action or GitLab CI）
- Database Migration（Liquibase）

---

# 11. 結語

本文件為 Flexora ERP 系統架構的概略說明。  
詳細內容請參考：

- `DATA_MODEL.md`（資料模型）
- `/specs/`（功能規格）
- `MODULES.md`（模組邊界）
- `WORKFLOW.md`（狀態機與流程）

本文件會隨系統演進持續更新。


---
