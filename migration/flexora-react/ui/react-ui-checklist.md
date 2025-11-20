# React UI Checklist — Quotation Module

## Layout & Navigation
- [ ] 左側 menu / top bar 一致，List 頁可切換至 Pipeline 視圖（toggle state 需記錄）。
- [ ] List view 需提供快速 filter（Subject/Stage/Opportunity/Organization/Total/Assigned To）並對應 backend criteria。
- [ ] Pagination 需包含 rows per page + jump to page (input) + Prev/Next，數值與 `X-Total-Count` 一致。
- [ ] Pipeline view：columns (DRAFT, SUBMITTED, APPROVED, EXPIRED, CANCELLED)，支援拖放與 workflow actions；拖動後必須更新 list/pipeline data。

## Detail Drawer
- [ ] List / pipeline 點擊 row/card 時，右側 Drawer 打開；初始 Table 保持全寬（方便年長使用者閱讀）。
- [ ] Drawer Tabs：Summary / Lines / Workflow / Attachments；每 tab 與 API 對應 (`QuotationRevisionDTO`, `/workflow-info`, `DocumentResource`...）。
- [ ] Summary card 顯示 ThreadNo, Customer, Owner, Status, ValidUntil, TotalAmount，並標註 `workflow-info` 的 guard hints。
- [ ] Associations card：Contacts、Sales Orders、Deliveries、Documents，最多顯示前 2~3 筆並提供 “View all” link。

## Workflow / Guard
- [ ] Drawer 與 pipeline card actions 需根據 `GET /quotation-revisions/{id}/workflow-info` 的 `availableActions` + `guards` 決定是否顯示按鈕、提示。
- [ ] Error key mapping：`quotation.invalid_state`, `quotation.customer_required`, `quotation.no_lines`, `quotation.invalid_valid_until`, `quotation.amount_mismatch`, `quotation.pricelist_no_price_found`.
- [ ] Action buttons (`Submit`, `Approve`, `Cancel`, `Clone`, `Set Active`) 需呼叫 `/api/quotation-revisions/{id}/...` endpoints，成功後刷新 list/pipeline；失敗時顯示對應 error key (toast/dialog)。

## Pricing / Lines
- [ ] Lines tab 進行編輯時，遵守 `pricing-rules`：line discount -> overall discount -> tax；保存時呼叫 `QuotationPricingService`（REST update）並顯示計算後結果。
- [ ] 若 PricingService 回 `quotation.amount_mismatch`，UI 顯示錯誤並提示使用者 re-check lines。

## Attachments
- [ ] Attachments tab 透過 Phase 0 Document API 讀取/上傳 (`DocumentResource`)，需檢查 `owner` / `DataScope` 權限，並與 detail drawer 區塊對齊。
