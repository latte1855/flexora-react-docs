# Phase 4 — Quotation UI 規格

> 根據現有 `flexora-react-ui` + Ecme layout 和本 repo 的 wireframe（`docs/ui/wireframes/quotation-detail-wireframe.md`）整理，後續 UI 實作需遵循此規格：包括 List/Pipeline 兩種視圖、Detail Pane、Workflow Drawer 與 Guard 提示等。

## 1. Overview

- **Layout**：左側 menu bar（Ecme 基礎）、右上 Top Bar、右下主內容（List 或 Pipeline + Detail Pane）。選取某筆報價後，右側會展開 Detail Pane（Summary + Tabs），不再以全螢幕 Drawer 遮住工具列；左側列表仍可操作/切換。  
- **List View**：左邊 Lists panel（快速 filter）、右邊 Quotes table (可搜尋多欄 filter, pagination, jump to page)。  
- **Pipeline View**：按板顯示 DRAFT/SUBMITTED/APPROVED/... columns，卡片可拖放以呼叫 workflow action。  
- **Detail Pane**：選取某筆後右側顯示 Summary + Workflow Guard + Associations。  
- **UI 與後端 API**：列表、filter、actions 需直接對應 `docs/specs/phase4-quotation/api-spec.md` 裡的 endpoints (QuotationThread/Revision/Workflow)。
- **流程圖快捷**：標題區提供「流程圖」按鈕，點擊後依 DB `quotation_state_transition` 畫出 from→event→to 的樹狀圖（使用共用 WorkflowDiagram 元件，未來模組可套用），`isDefault=true` 的狀態以綠色起點標示，`isClosed=true` 則以紅色終點表示，協助業務快速理解完整流程；Modal 右上提供 Legend，說明「起點 / 終點 / 目前狀態」三種節點顏色，方便業務快速判讀。

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
- Pipeline columns：依 `quotation_status_def` 定義動態呈現（現階段至少 DRAFT / IN_APPROVAL / SENT / APPROVED / WON / LOST / REJECTED / EXPIRED / CANCELLED），切換/增減狀態時只需更新資料表。  
- 每張卡片顯示：ThreadNo, Customer, Owner, Amount, validUntil, Status badge, Actions (submit/approve/cancel/clone/set-active).  
- 拖放：`n` -> target column；成功後呼叫對應 workflow API (`/submit`, `/approve`, ...)，並刷新 column 資料；失敗時顯示 errorKey mapping (`quotation.invalid_state` 等)。
- 在 pipeline 仍可選卡片打開 detail pane (Summary/Workflow/Associations) 以檢視更多資料或執行 `Workflow Drawer`.

## 4. Detail Pane / Drawer

> 依 `docs/ui/wireframes/quotation-detail-wireframe.md` 版本 4 設計。

- Tabs：Summary / Lines / Workflow / Attachments；桌機版面採雙欄設計，左欄固定展示 Summary / Guard / Actions（約 360px 寬），右欄為完整工作區 Tabs，可獨立捲動並擴充高度。Detail Pane 直接嵌入 Quotation Workspace，頂部提供上一筆／下一筆／新增報價／關閉等捷徑。  
- Summary Tab：顯示 ThreadNo, Customer, Owner, Status badge, Valid Until, Total Amount, margin, Workflow guard card（`guards` from `/workflow-info` → `requiresValidUntil`, `hasNoLines`...），Action Buttons (SUBMIT/CANCEL/CLONE)；另包含「價目表」卡片，顯示目前版本價目表名稱、通路、幣別（僅作資訊提供，無推薦更新按鈕）。  
- Associations cards：Contacts, Related Sales Orders, Delivery Notes, Documents；透過對應 API（`SalesOrderResource`, `DeliveryNoteResource`, `DocumentResource`）依 `sourceQuotationThreadId` or `revisionId` 篩選，並顯示 summary data (status, date, amount, link).  
- Lines Tab：呈現 QuotationLine table，支援 inline edit / reorder (若版本已鎖，顯示 read-only).  
- Workflow Tab：顯示 timeline (status history) + available actions (buttons, reason/textarea)，以目前單據歷史為準；資料來源為 `GET /api/quotation-status-histories?sort=changedAt,desc&amp;quotationId.equals={threadId}`，可由 UI Segment 切換「全部歷程 / 僅此版本」，切換時多帶 `revisionId.equals={selectedRevisionId}`，並提供版本下拉讓使用者在不切換 Detail Revision 的情況下檢視其他版本歷程；Timeline 需高亮目前狀態並支援事件篩選（以 eventCode 為單位），上方流程圖負責呈現完整 transition tree，兩者相互補充。  
- Attachments Tab：列出 Document cards + Upload button，下載按鈕呼叫 `GET /api/documents/{id}/download` 並於新視窗開啟串流；需提供拖曳上傳 Drop Zone 與 PDF/圖片預覽（Modal 內顯示 iframe / 圖片），便於使用者快速檢視附件內容。  

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

## 8. Quick Create Drawer 規格補充

- Drawer 分為 **基本資料 / 明細行 / 預覽** 三步，送出 `QuotationPreviewRequestDTO`；Phase 4 起請求必須帶 `priceListId`，前端需提供價目表下拉（列表取自 `/api/price-lists?deleted.equals=false&sort=priceListName,asc`）。  
- 選擇客戶後需呼叫 `/api/pricing/price-lists/applicable` 取得依 Assignment 優先序排序之推薦價目表，UI 以 ⭐ 標記並優先列出，若使用者尚未手動選擇價目表則自動套用第一筆建議並同步幣別/通路。  
- Owner 改為 **Async Select**，呼叫 `/api/owners/lookup?keyword=` 取得權限過濾後的清單；若當前 Thread Owner 不在預設 options，需自動補上並設定為預設值，但仍禁止在 revise/edit 模式修改。  
- 選取價目表 + SKU 後，UI 會以 `/api/item-prices?priceListId.equals={id}&skuId.equals={sku}&sort=minQty,asc&size=1` 取得預設 UOM/Tax Code，寫入行項並以 badge 提示「來自價目表預設」，同時鎖定 UOM、稅別欄位避免覆寫。若價目表不存在該 SKU，就維持欄位可編輯。  
- UOM / 稅別皆為 **必填**，只要選了 SKU 就必須提供這兩個欄位；若價目表缺少預設值，UI 會在行卡上顯示「缺單位/稅別」標籤並於欄位下方以紅字提示，同時禁止進入預覽步驟，避免送出會被後端拒絕的 payload。  
- 當價目表缺少某 SKU 價格時，行卡與 Step3 Preview 需顯示醒目提醒與缺價清單，並引導「請到價目維護補價（功能尚未開放時由後台補資料）」以阻擋送出。  
- Step2、Step3 均提供「前往價目維護」按鈕，導向 `/inventory/price-books?priceListId={selected}`（可帶 skuId），讓使用者快速切換到價目維護頁面補資料。  
- `canPreview` 需同時檢查客戶、Owner、價目表與至少一筆行項；錯誤訊息也需同步更新。  
- Step 3 的 Preview 卡在行項總結前需加一張「使用中的價目表」資訊卡，僅展示目前套用的價目表名稱 / 通路 / 幣別與回到 Step 1 的按鈕，提供使用者確認來源資訊（不在此變更價目表）。  
- 行項仍可輸入手動單價 (`manualUnitPrices`)，欄位值會放進 `properties.extAttrs.manualUnitPrices`，供後續 Phase 5 正式欄位化。
