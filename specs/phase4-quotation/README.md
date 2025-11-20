# Phase 4 — 報價管理（Quotation Management）規格總覽

本文件說明 Flexora ERP 在 **Phase 4：報價管理（Quotation）** 的整體需求與範圍，  
包含報價主線（Thread）、報價版本（Revision）、明細（Line）、狀態流程（Workflow）、  
以及與商品 / 價格 / 客戶 / 銷售訂單等模組的關聯。

> 報價模組是「銷售流程的起點」，未來將與 Sales Order / Inventory 緊密串接。  
> 本文件為 Phase 4 的總入口，詳細拆分將放在同目錄其他檔案中。

---

## 1. 目標與定位

### 1.1 模組目標

1. 支援「一個客戶需求、多次報價往返」的情境：
   - 使用 **QuotationThread** 表示一次「詢價/專案」主線
   - 使用 **QuotationRevision** 表示此主線下多次調整版本
2. 清楚管理報價狀態：
   - 草稿、已送出、客戶接受、已逾期、已作廢……
3. 與商品/定價模組整合：
   - 每一行報價明細對應 `ItemSku` 與 `PriceList` / 計價邏輯
4. 為「銷售訂單（Sales Order）」提供穩定的轉單來源：
   - 支援「由已核准的報價，一鍵產生 Sales Order」的設計（詳細在 Sales 模組 spec）

### 1.2 不在本階段範圍（Out of Scope）

- 銷售訂單（Sales Order）實際實作（會在 Sales spec 定義）
- 真正的財務計價 / 稅額 / 帳款模組
- 任何列印 / 匯出 PDF 的詳細格式（可預留 hooks / API）

---

## 2. 相關文件與模組關聯

### 2.1 相關架構文件

- `/docs/architecture/OVERVIEW.md`
- `/docs/architecture/DATA_MODEL.md`
- `/docs/architecture/WORKFLOW.md`
- `/docs/architecture/MODULES.md`

### 2.2 相關模組 spec

- Phase 0：`/docs/specs/phase0-system-foundation/`
  - Numbering、Validation、Security、Error Handling
- Phase 1：`/docs/specs/phase1-product-pricing/`
  - Item / ItemSku / PriceList / ItemPrice
- Phase 3：`/docs/specs/phase3-inventory/`（未來）
- Sales Order（未來）：`/docs/specs/sales-order/`

報價模組 **依賴**：

- 使用者 / Owner / Team（Phase 0）
- 商品與定價（Phase 1）

並將被：

- 銷售訂單（Sales Order）
- 分析報表（未來）

所依賴。

---

## 3. 預計實體與核心概念（高階概覽）

詳細 Entity 欄位之後會拆到 `domain-model.md`，  
這裡先列出報價模組的**概念級物件**：

- `QuotationThread`
  - 一次客戶詢價 / 專案主線
  - 屬於某個 Owner、某個客戶
  - 可能有多個版本（Revision）
- `QuotationRevision`
  - 某一版具體報價版本
  - 含有效期限（ValidUntil）、條款、折扣等
  - 與多筆 `QuotationLine` 關聯
- `QuotationLine`
  - 單一品項的報價明細
  - 關聯到 `ItemSku` 與單價 / 數量 / 折扣 / 稅別
- `QuotationStatusDef`（或 Enum）
  - 定義報價狀態（DRAFT / SUBMITTED / APPROVED / EXPIRED / CANCELLED…）
- 狀態歷程 / Log（未來）
  - 用於稽核與 UI 時間軸展示

---

## 4. 流程與狀態（Workflow 概要）

完整 Workflow 將在 `workflow-spec.md` 中詳細描述，  
這裡做高階摘要：

### 4.1 典型生命週期（示意）

```text
DRAFT → SUBMITTED → APPROVED → (轉 Sales Order / CLOSE)
   │          │
   └──────────┴→ CANCELLED / EXPIRED
```

### 4.2 典型使用情境

1. 業務建立一個新的報價主線（Thread）
2. 建立或複製一個版本（Revision）作為草稿
3. 編輯明細、條款、有效期限
4. 送出（SUBMIT）給客戶
5. 客戶接受 → 將該 Revision 標記為 APPROVED
6. 由該 Revision 轉成 Sales Order（在未來 Sales 模組中實作）

---

## 5. 與其他模組的資料關聯（高階）

* 客戶（Customer / CRM 模組）

  * `QuotationThread` 需關聯客戶
* 商品 / 定價（Product & Pricing 模組）

  * `QuotationLine` 需關聯到 `ItemSku`
  * 單價可基於 `PriceList` 或客製價格
* Owner / Team

  * 報價歸屬特定 Owner
  * 可能關聯負責業務（User）與 Team
* Sales Order（未來）

  * 已核准報價可作為銷售訂單的來源版本

---

## 6. 文件結構規劃（本目錄）

本資料夾預計包含下列文件（可隨進度擴充）：

```text
specs/phase4-quotation/
  README.md              ← 本文件（Phase 4 總覽）
  domain-model.md        ← 報價相關 Entity / DTO / 關聯與欄位規格
  workflow-spec.md       ← 報價狀態與流程（含 Guard 規則）
  api-spec.md            ← 報價 REST API 介面行為（CRUD / 特殊操作）
  ui-spec.md             ← 前端畫面 / Workflow Drawer / 驗證規則 / 操作流程
  pricing-rules.md       ← 報價計價邏輯與 PriceList / 折扣規則（與 Phase 1 聯動）
  error-codes.md         ← 報價模組專屬錯誤碼表（對應 error-handling-spec）
  test-strategy.md       ← 測試策略：單元 / 整合 / E2E 應覆蓋的情境
```

> 一開始可以僅建立 `README.md` + 1～2 個檔案，
> 之後每當實作報價相關功能時同步補齊對應 spec。

---

## 7. 實作原則（Implementation Principles）

1. **以 Workflow 為中心設計**

   * 所有關鍵操作（SUBMIT / APPROVE / CANCEL…）必須經過後端 Guard 檢查。
2. **Thread / Revision 分離**

   * Thread 表示「客戶需求主線」
   * Revision 表示「具體報價版本」
3. **Numbering 一致**

   * `quotationNo` 由統一 Numbering Service 產生
   * 規則參照：`../phase0-system-foundation/numbering-spec.md`
4. **錯誤與驗證遵守 Phase 0 規範**

   * 所有錯誤碼需整合到 `error-handling-spec.md` / `error-codes.md`
   * Bean Validation + Service Guard 清楚分工

---

## 8. 變更控制（Change Control）

任何會影響報價模組的重大變更必須：

1. 在 `../decisions/DECISION_LOG.md` 新增記錄（必要時建立 ADR）。
2. 更新：

   * 本 `README.md`（若屬範圍 / 目標調整）
   * `domain-model.md`（若變更 Entity / 欄位）
   * `workflow-spec.md`（若變更狀態流程）
   * `api-spec.md`（若變更 API）
   * `ui-spec.md`（若變更前端行為）
3. 確保相關模組 spec（例如 Sales、Inventory）若有依賴，也一併補上註記。

---

## 9. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                     |
| ---- | ---------- | ----- | ---------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 4 報價模組規格總覽初稿。 |

## 10. 合併來源與程式碼參照

- 此目錄的 spec 以 `docs/migration/flexora-react/flexora-quotation-management-spec.md` 為主來源，並以實作為最終權威，例如 `com.asynctide.flexora.service.QuotationThreadService`, `QuotationRevisionService`, `QuotationWorkflowService`, `QuotationThreadResource`, `QuotationRevisionResource` 等。
- 若 `workflow-spec.md`、`api-spec.md`、`pricing-rules.md` 等內容與上述 service/controller 產生差異，請直接以程式碼為準並在該 spec 檔中註明 `Updated per flexora-react/src/...` 。
- 報價 UI 的 Guard 與 Workflow 提示依據可驗證 `flexora-react-ui`、`flexora-react`（JHipster）的 Drawer/Timeline 實作；必要時在 `ui-spec.md` 補充 `Derived from flexora-react/...`。
