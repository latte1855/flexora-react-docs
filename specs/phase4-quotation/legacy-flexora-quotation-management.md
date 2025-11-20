# Flexora 報價模組 v2.0 規格（Legacy Reference）

本文件摘錄 `docs/migration/flexora-react/flexora-quotation-management-spec.md` 中的重要章節，並指向在 `docs/specs/phase4-quotation/` 相關分檔完成的資料（舉例：`pricing-rules.md`、`workflow-spec.md`、`ui-spec.md`）。實作仍以 `flexora-react/src/main/java/com/asynctide/flexora/*` 系列檔案為最終依據；若不確定時請優先參考程式碼並補註 `TODO`。

## 1. 檔案來源與概觀

- **來源**：`flexora-react/docs/flexora-quotation-management-spec.md`（已完整備份在 `docs/migration/flexora-react/`）。
- **版次**：v2.0（2025-10-01），主要補充通用欄位、狀態機、ExtAttr、稅計算。
- **關聯程式**：
  * Domain：`com.asynctide.flexora.domain.QuotationThread`, `QuotationRevision`, `QuotationStatusDef`
  * Service / Workflow Guard：`com.asynctide.flexora.service.QuotationRevisionService`（SUBMIT/APPROVE/CANCEL 等方法）
  * REST：`com.asynctide.flexora.web.rest.QuotationThreadResource`, `QuotationRevisionResource`

## 2. 關鍵規格摘錄

| 子領域 | 主要內容 |
| --- | --- |
| **通用欄位** | `created_by/created_at`, `last_modified_*`, `deleted/*`, `version` 與 `owner_id` 為所有表標準欄位；唯一鍵採 partial unique (`WHERE deleted=false`)。 |
| **數值精度** | 金額 `DECIMAL(19,4)`，單價/折扣運算 `DECIMAL(19,6)`，稅率 `DECIMAL(7,6)`（以 `0..1` 表示）。 |
| **Enum 定義** | `discount_type`（NONE/AMOUNT/RATE）、`quotation_status_def`（DRAFT/SENT/APPROVED/...）、`quotation_event_def`（send/approve/reject/expire/cancel）、`quotation_action`（SUBMIT/CLONE/SET_ACTIVE）。 |
| **狀態機** | 以表頭 `quotation_status_def` + 事件表 + `quotation_state_transition` 記錄合法轉換；所有 Guard 放在 Service，基於 `BadRequestAlertException` 拋出 `quotation.invalid_state`、`quotation.no_lines` 等錯誤碼。 |
| **稅計算** | 先行折扣再整體折扣，含稅/未稅分流；判定 `taxIncluded` 後用公式 `lineTotal = netUnitPrice * quantity` 與 `taxAmount = lineTotal × (taxRate/(1+taxRate))`。 |

## 3. 參考與驗證

1. **資料表結構**：譯自原始文件的 `quotation_thread`、`quotation_revision`、`quotation_item` 表格；詳細欄位請以後端 Entity 為準（例如 `flexora-react/src/main/java/com/asynctide/flexora/domain/QuotationThread.java`）。
2. **Workflow Guard**：此 legacy spec 內定義的 `status/event`/`guard` 表格已部分覆蓋於 `workflow-spec.md`；新增 Guard、錯誤碼或狀態需同步更新 `docs/specs/phase4-quotation/error-codes.md` 與 `phase4` workflow.
3. **ExtAttr + Document Link**：延伸欄位（例如 `ext_attr_header/value`、`document/document_link`）在 `docs/architecture/DATA_MODEL.md` 與 `phase4` `domain-model` 單獨記錄，屬於通用擴充。

## 4. TODO / open questions

- 若資料表新增欄位（`quotation_revision_address`、`quotation_item_tax` 等），請先確認 `flexora-react` 的 Liquibase migration 或 JPA Entity 是否已實作，再更新本檔的摘要。
- 若 Guard 需同步多家 `errorKey`（例如 `quotation.workflow.*`），請同時補齊 `docs/specs/phase4-quotation/error-codes.md` 與 `docs/specs/phase0-system-foundation/error-handling-spec.md`。
