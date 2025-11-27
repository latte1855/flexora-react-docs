# Phase 4 — 報價模組 API 規格（API Specification）

本文件以 `flexora-react/src/main/java/com/asynctide/flexora/web/rest/QuotationThreadResource.java`, `QuotationRevisionResource.java`, `QuotationWorkflowResource.java` 及對應 `service`/`dto` 為最終來源，列出主要端點與 guard 參數定義。若 API 與程式碼有不同，以程式碼為權威並於此註記 `Updated per ...`.

## 1. Core Endpoints

| Path | Method | Description | Reference |
| --- | --- | --- | --- |
| `GET /api/quotation-threads` | GET | 取得 Thread 列表，支援 `Criteria` 過濾（owner/customer/threadNo/status） | `QuotationThreadResource.getAllQuotationThreads`, `QuotationThreadCriteria` |
| `POST /api/quotation-threads` | POST | 建立 Thread + 初始 Revision | `QuotationThreadResource.createQuotationThread`, `QuotationThreadService.createThread` |
| `GET /api/quotation-revisions/{id}` | GET | 取得 Revision 詳細（含 lines） | `QuotationRevisionResource.getQuotationRevision`, `QuotationRevisionMapper` |
| `PUT /api/quotation-revisions/{id}` | PUT | 更新 Revision（價格/有效期限/lines） | `QuotationRevisionResource.updateQuotationRevision`, `QuotationPricingService` |
| `POST /api/quotation-revisions/{id}/workflow-info` | GET | Workflow Drawer 初始資料 | `QuotationWorkflowResource.getWorkflowInfo`, `QuotationWorkflowService.getWorkflowInfo` |
| `POST /api/quotation-revisions/{id}/submit` | POST | SUBMIT action | `QuotationWorkflowResource.submitQuotation`, `QuotationWorkflowService.submitQuotation` |
| `POST /api/quotation-revisions/{id}/approve` | POST | APPROVE action | `QuotationWorkflowResource.approve` |
| `POST /api/quotation-revisions/{id}/cancel` | POST | CANCEL action | `QuotationWorkflowResource.cancel` |
| `POST /api/quotation-revisions/{id}/clone` | POST | Clone Revision | `QuotationWorkflowResource.cloneQuotation`, `QuotationWorkflowService.cloneQuotation` |
| `POST /api/quotation-revisions/{id}/set-active` | POST | 設定 active Revision | `QuotationWorkflowResource.setActiveRevision` |
| `GET /api/owners/lookup` | GET | Quick Create Owner 搜尋（Async Select，含權限過濾） | `OwnerResource.lookupOwners`, `OwnerService.lookup` |

## 2. DTO / Validation Notes

* `QuotationThreadDTO` / `QuotationRevisionDTO`` fields (status, validUntil, currencyCode, totalAmount)` defined in `service/dto`.
* Bean Validation annotations in DTO ensure `@NotNull` on `customerId`, `validUntil` etc.; service guard (e.g., `validateSubmitGuard`) double-checks semantics.
* `QuotationPreviewRequestDTO` 自 Phase 4.2 起新增 `priceListId` 欄位：當前端傳入時，`PricingService.preview()` 會鎖定該價目表並在後續 `createDraft`/`createNextRevision`/`updateRevision` 中用於儲存 `QuotationRevision.priceList`。若未指定則維持原有的 Assignment 選價流程。
* Quick Create Drawer 的 Owner 欄位改走 `GET /api/owners/lookup?keyword=`，此 endpoint 會依 `VOwnerVisible` 過濾可見 owner，預設上限 50 筆供 Async Select 使用。

## 3. Guard Error Codes

| errorKey | Trigger | Reference |
| --- | --- | --- |
| `quotation.invalid_state` | guard rejects incorrect status transitions | `QuotationWorkflowService.validateXxxGuard` |
| `quotation.customer_required` | missing customer in Submit | `validateSubmitGuard` |
| `quotation.no_lines` | lines empty / invalid | `validateSubmitGuard` |
| `quotation.reject_reason_required` | reject without reason | `QuotationWorkflowService.reject` |
| `quotation.amount_mismatch` | pricing mismatch before submit | `QuotationPricingService.verifyAmounts` |
| `quotation.pricelist_no_price_found` | no price entry for SKU | `ItemPriceService.getPrice` |

## 4. Workflow Info Payload

```json
{
  "revisionId": 123,
  "status": "DRAFT",
  "availableActions": ["SUBMIT", "CANCEL"],
  "guards": {
    "requiresCustomer": true,
    "requiresValidUntil": true,
    "requiresAtLeastOneLine": true
  }
}
```

Payload is built in `QuotationWorkflowService.getWorkflowInfo`; UI should use `availableActions` to enable buttons and `guards` to surface warnings.

## 5. Version & Change Log

| Version | Date | Notes |
| --- | --- | --- |
| v0.1 | 2025-11-19 | Initial API spec from legacy document. |
| v0.2 | 2025-11-XX | Added code alignment notes per `flexora-react` services. |
