# Flexora ERP — 統一整合規格總覽（Summary）

此檔案概要記錄 `flexora-react/docs/Flexora ERP 規格書(整合).md` 的全域理念與常見約束，並置於 `docs/migration/flexora-react/Flexora ERP 規格書(整合).md` 供日後參考。實際行為仍以後端程式碼為最終權威（例如 `flexora-react/src/main/java/com/asynctide/flexora/domain`、`.../service`、`.../web/rest`）。

## 1. 核心原則

* **命名與精度**：所有業務單號以 `XXX-No` 形式呈現，金額/數量採 `DECIMAL(19,4)` 或 `DECIMAL(19,6)`，稅率與折扣以 `0–1` 儲存。
* **跨模組共享資料**：Customer/Item/UoM 等主檔保留在原模組，其他模組透過 ID 參照；資料擁有者由 `owner_id`（且常搭配 `tenant_id`）表達，軟刪除欄位與樂觀鎖 (`version`) 必備。
* **狀態與事件治理**：每張單據附 `*_status_def`、`*_event_def`、`*_state_transition`、`*_status_history` 表，Service 內部 Guard 控管合法轉換（例如 `QuotationWorkflow` 中的 SUBMIT/APPROVE/CANCEL）。
* **資料權限**：Row-level 權限以 `Owner` 為邊界，`User/Team/Department` 有 `MapsId` 關聯；規格書也列出 Owner/Department/Team 表設計，供 `flexora-react` 的 `com.asynctide.flexora.domain.*` 對照。

## 2. 模組關聯提醒

1. **Quotation ⇄ Sales Order**：已核准報價可轉為 `SalesOrder`（`SalesOrderQuotationLink` 代表關聯），轉單欄位/流程在 `flexora-react` service/mapper 中實作。
2. **Inventory & Cost Layer**：所有入庫產生 `InventoryTransaction` 與對應 `CostLayer`，定價與稅務計算則依 `pricing-rules.md` 的算法與 `phase4` API 規範。
3. **ExtAttr / Document Link**：提供 `ext_attr_header/value`、`document/document_link` 等通用擴充，用於 Item、Order、Quotation 等。

## 3. Next steps from the legacy doc

* 審視 `docs/migration/flexora-react/Flexora ERP 規格書(整合).md` 的各章節，把必要細節逐步拆進對應 `specs/phaseX` 檔案。
* 若未能從程式碼判定某條規則（如特殊流程），請在該模組的 `docs/specs/.../legacy` 檔中註記待確認事項供使用者決策。
