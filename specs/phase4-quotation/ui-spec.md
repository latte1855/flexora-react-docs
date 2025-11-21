# Phase 4 — Quotation UI 規格

> 根據現有 `flexora-react-ui` + Ecme layout 和本 repo 的 wireframe（`docs/ui/wireframes/quotation-detail-wireframe.md`）整理，後續 UI 實作需遵循此規格：包括 List/Pipeline 兩種視圖、Detail Pane、Workflow Drawer 與 Guard 提示等。

## 1. Overview

- **Layout**：左側 menu bar（Ecme 基礎）、右上 Top Bar、右下主內容（List 或 Pipeline + Detail Pane）。  
- **List View**：左邊 Lists panel（快速 filter）、右邊 Quotes table (可搜尋多欄 filter, pagination, jump to page)。  
- **Pipeline View**：按板顯示 DRAFT/SUBMITTED/APPROVED/... columns，卡片可拖放以呼叫 workflow action。  
- **Detail Pane**：選取某筆後右側顯示 Summary + Workflow Guard + Associations。  
- **UI 與後端 API**：列表、filter、actions 需直接對應 `docs/specs/phase4-quotation/api-spec.md` 裡的 endpoints (QuotationThread/Revision/Workflow)。

## 2. List View（第一頁）

1. **Lists Panel 左欄**  
   - 搜尋欄 (`Search for List`)、My List (Open Quotes / Rejected Quotes)、Shared List、Tags (VIP, 重要) 等。  
   - 可對資料表 `saved filters` 進 CRUD；UI spec 需指明 `private` vs `shared` 等欄位。
2. **Quotes Panel 右欄**  
   - Header: `Quotes` + `+ Add` 按鈕。  
   - Filter toolbar：Subject, Quote Stage, Opportunity, Organization, Total, Assigned To（對應 `QuotationThreadCriteria`）。  
   - Table columns：Subject / Quote Stage / Opportunity / Organization / Total / Assigned To（按 `api-spec` 取資料）。  
   - Pagination：提供 rows per page selector、range indicator (`1-4 of 4`)、Prev/Next； 추가 `Jump to page [input] Go` 功能，數值與 `X-Total-Count` header 對應。  
   - 點 Row 後右側 detail 會更新（若 detail pane 展現為 Drawer，則在右側開 Drawer）。

## 3. Pipeline View

- List View 頂部提供 Toggle (`List`, `Pipeline`)，切換時重載畫面 (REST API 仍為 `GET /api/quotation-threads`, 但 UI 需 aggregate by status)。  
- Pipeline columns：DRAFT, SUBMITTED, APPROVED, EXPIRED, CANCELLED（對應 workflow spec）；可加 `REJECTED` 視 UI 規劃。  
- 每張卡片顯示：ThreadNo, Customer, Owner, Amount, validUntil, Status badge, Actions (submit/approve/cancel/clone/set-active).  
- 拖放：`n` -> target column；成功後呼叫對應 workflow API (`/submit`, `/approve`, ...)，並刷新 column 資料；失敗時顯示 errorKey mapping (`quotation.invalid_state` 等)。
- 在 pipeline 仍可選卡片打開 detail pane (Summary/Workflow/Associations) 以檢視更多資料或執行 `Workflow Drawer`.

## 4. Detail Pane / Drawer

> 依 `docs/ui/wireframes/quotation-detail-wireframe.md` 版本 4 設計。

- Tabs：Summary / Lines / Workflow / Attachments。  
- Summary Tab：顯示 ThreadNo, Customer, Owner, Status badge, Valid Until, Total Amount, margin, Workflow guard card（`guards` from `/workflow-info` → `requiresValidUntil`, `hasNoLines`...），Action Buttons (SUBMIT/CANCEL/CLONE).  
- Associations cards：Contacts, Related Sales Orders, Delivery Notes, Documents；透過對應 API（`SalesOrderResource`, `DeliveryNoteResource`, `DocumentResource`）依 `sourceQuotationThreadId` or `revisionId` 篩選，並顯示 summary data (status, date, amount, link).  
- Lines Tab：呈現 QuotationLine table，支援 inline edit / reorder (若版本已鎖，顯示 read-only).  
- Workflow Tab：顯示 timeline (status history) + available actions (buttons, reason/textarea).  
- Attachments Tab：列出 Document cards + Upload button.

## 5. Workflow Drawer & Guard

- Drawer 內需顯示 `availableActions`（SUBMIT/CANCEL/CLONE/APPROVE/REJECT/EXPIRE/SET_ACTIVE）以及 `guards` flag（`requiresValidUntil`, `hasNoLines`, `validUntilExpired`...），UI 以 `quotation-ui-redesign-plan` 內的 guard card 為範例。  
- 呼叫 `POST /api/quotation-revisions/{id}/workflow-info` 取得 newest guard; actions 對應 `POST /{id}/submit` etc.

## 6. Designer/Developer Checklist（Reminder）

1. List/Pipeline filter mapping, columns, actions; guard toast mapping for `quotation.*` error key.
2. Detail Pane layout & associations; ensure everything has data source and error handling.
3. Workflow Drawer integration; fetch guard info per revision, map actions to backend.
4. Attachments + Documents list to align with Phase 0 Document spec.

## 7. React 實作參考

- 主要畫面實作於 `flexora-react-ui/src/views/sales/QuotesWorkspace.tsx`，負責 List/Pipeline/Drawer layout 與篩選邏輯。
- 路由設定：`/sales/quotes` 在 `flexora-react-ui/src/configs/routes/sales.routes.ts` 指向上述組件。
