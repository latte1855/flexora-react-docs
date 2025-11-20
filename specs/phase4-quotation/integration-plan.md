# Phase 4 — 報價整合計畫（Integration Plan）

本計畫依據 `docs/migration/flexora-react/flexora-quotation-management-spec.md` 與後端實作（`QuotationThreadService`, `QuotationRevisionService`, `QuotationPricingService`, `QuotationWorkflowService`, `QuotationThreadResource` 等）逐步將 legacy 資料拆進正式 spec，並以程式碼為權威。請依照以下順序進行整合，完成各小節後在 README 或該檔末尾註記 `Derived from ...`。

## 1. 核心步驟

1. **API spec (`api-spec.md`)**  
   - 將 legacy doc 中的 `/api/quotation-threads`、`/api/quotation-revisions`、workflow action endpoints 整理成表格，包括 Request/Response DTO 項目（fields, validation）與 status code。  
   - 以 `QuotationThreadResource`, `QuotationRevisionResource`, `QuotationWorkflowResource` 作為最終參考，補上 Guard/权限（`@PreAuthorize`）與 `BadRequestAlertException` 錯誤碼。
2. **Workflow & Guard (`workflow-spec.md`)**  
   - 逐步檢查 `Reservation`/`Workflow` states;確認 SUBMIT/APPROVE/CANCEL/CLONE 等 `QuotationWorkflowService` 方法的 guard（`validateSubmitGuard` 等）與 error key（`quotation.invalid_state`, `quotation.reject_reason_required`）是否反映於 spec。  
   - 記錄 DTO fields `status`, `validUntil`, `guards` returned from `GET /workflow-info`；若 workflow 需 extra data（e.g., `availableActions`）, 直接鏡射 `QuotationWorkflowService.getWorkflowInfo`.
3. **Pricing rules (`pricing-rules.md`)**  
   - 依據 `QuotationPricingService` 追加程式碼參照與 error key（`quotation.amount_*`）。  
   - 確保 `lineSubtotal`, `taxAmount`, `totalAmount` 的 formulas 與 `calculateLineAmounts`/`applyOverallDiscount` 同步，並補上 rounding/precision note derived from code.
4. **UI guard / error codes**  
   - `ui-spec.md` 應標明 `Workflow Drawer` 需從 `GET /quotation-revisions/{id}/workflow-info` 取得 `availableActions` 與 `guards`，UI actions 必須對應 backend error keys。  
   - `error-codes.md` 需更新 `quotation.workflow.*`, `quotation.amount_*`, `quotation.active_revision_must_be_approved` 等錯誤，並標註出處。

## 2. 校對標準

- 任何字段/端點/錯誤碼的改動，均需比對 `flexora-react/src/main/java/com/asynctide/flexora/service` 及 `web/rest` 對應檔案；若 spec 與程式不一致，以程式碼為準並附註 `Updated per ...`.  
- Workflow Guard/Action 需與 `QuotationWorkflowService` 內的 `validateXxxGuard` 函數對齊，並將 guard 條件逐項列出作為 checklist。  
- Pricing formulas/rounding 以 `QuotationPricingService` 的 `BigDecimal` `HALF_UP` 為最終決定，若 front-end `UI` 需參考 `pricing-rules` 作 tooltip/guard 也應注明 dataset.

## 3. 更新與追蹤

1. 每完成一節，更新 `docs/migration/log.md` 紀錄新增檔案/修改，並附上參考 code path。  
2. 若某段 legacy note 仍無法對應至 `flexora-react` 中的 service，請在 `docs/specs/phase4-quotation/legacy-flexora-quotation-management.md` 加上 `TODO` 條目並標明需 developer review。  
3. 完成後，將 README 的 `## 6. 合併來源與程式碼參照` 更新進度並列出新增文件，避免混淆。
