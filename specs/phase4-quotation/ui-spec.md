# Phase 4 — Quotation UI 規格

## 0. 進度總結（2025-12-03）

| 狀態 | 項目 | 備註 |
| --- | --- | --- |
| ✅ | Quote Editor 全畫面 overlay、3 欄表單、地址/ExtAttr 卡片與 `/api/quotations/transactions` 整批儲存整合 | QuoteEditorPage 已能建立/編輯 thread + revision + items，並透過 Preview 卡顯示金額摘要。 |
| ✅ | Quick Create Drawer 3-step Flow（客戶/Owner/價目表/行項/預覽）、附件拖曳、流程圖按鈕、Workflow Timeline Segments（僅 ALL / Current Revision） | Drawer 與 Quote Editor 一樣改走 `/api/quotations/transactions`，完成草稿 / 建立新版本 / 編輯皆走同一 transaction，並沿用 `focusThreadAfterChange` 回到 Detail。 |
| ✅ | Attachments Tab、Workflow Diagram 起訖節點上色、Timeline highlight 當前狀態、Owner/Contact Async Select 共用元件 | 附件下載改 `/api/documents/{id}/download` 並支援拖曳 + PDF 預覽。 |
| ✅ | Workflow Drawer / Pipeline 導向一致化 | Workflow 事件仍以 `triggerEvent` API 為主（保留後端 guard/事件設計），並記錄在此：之後若客戶驗收要求整單 transaction，再評估改走 `/transactions`；目前僅補上備註與 TODO。 |
| ✅ | Workflow Timeline Segments（ALL / Selected Revision / Revision Filter chips）、事件分類高亮、Timeline 與流程圖互動一致 | Workflow Tab 已提供 THREAD/CURRENT/REVISION Segment、版本下拉、事件 chips 與清除按鈕，CURRENT 會顯示當前版本資訊並高亮目前狀態；若 chip 篩選無結果會顯示提示並可一鍵清除。 |
| ⏳ | List / Pipeline 的 preset/filter 架構 + 後端 enum：需要 ownerScope preset、狀態/日期/客戶等進階篩選、chips/清除互動、URL 同步 | 已建立 `QuotePresets` + advanced filter panel、chips 可個別清除並加上「視圖」提示，且已支援 URL query 同步；仍待 Saved Filters / 自訂條件收藏。 |
| ⚠️ | Hub 篩選 URL 同步仍會在選取 preset 時觸發多次 API（VIP→ALL 反覆） | `QuotesWorkspace` 的 state/URL 雙向同步尚未完全穩定，需調整 guard 邏輯；詳見 TODO。 |
| ⏳ | 行項表格式編輯 / 匯入、整批儲存後自動重新計價、Preview 行卡顯示稅別/折扣與 PDF 下載 | 目前行項仍為卡片式，未提供批次匯入或 inline row edit；Preview 區僅顯示總額，行明細仍為 placeholder。 |
| ⏳ | Quick Filter / Pipeline Guard 文件補齊 | 需在本文與 `ui-implementation-plan` 標註 TODO 與完成項目。 |

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
   - **已完成**：List/Pipeline 已導入 `QuotationPresetEnum`（ALL/MY_OPEN/REJECTED/VIP/IMPORTANT）與進階篩選 panel（關鍵字、客戶、Owner、狀態、日期），並提供 chips + 單顆清除、立即套用。  
   - **TODO**：提供 Saved Filters / ownerScope 標註與分享鏈結，讓常用條件可以命名儲存。

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
- Workflow Tab：顯示 timeline (status history) + available actions (buttons, reason/textarea)，以目前單據歷史為準；資料來源為 `GET /api/quotation-status-histories?sort=changedAt,desc&amp;quotationId.equals={threadId}`，可由 UI Segment 切換「全部歷程 / 僅此版本」，切換時多帶 `revisionId.equals={selectedRevisionId}`，並提供版本下拉讓使用者在不切換 Detail Revision 的情況下檢視其他版本歷程；Timeline 需高亮目前狀態並支援事件篩選（以 eventCode 為單位），上方流程圖負責呈現完整 transition tree，兩者相互補充。高亮狀態只對 `history.toStatusCode` 與目前 revision 的 `statusCode` 全大寫後比對時相同的那筆有效，該紀錄以邊框/淡彩背景與陰影凸顯舒緩視覺，讓使用者在 Timeline 與流程圖中都能快速辨識目前狀態。  
- Attachments Tab：列出 Document cards + Upload button，下載按鈕呼叫 `GET /api/documents/{id}/download` 並於新視窗開啟串流；需提供拖曳上傳 Drop Zone 與 PDF/圖片預覽（Modal 內顯示 iframe / 圖片），便於使用者快速檢視附件內容。拖曳區旁需持續顯示「拖曳檔案或按上傳按鈕以上傳」提示，若上傳失敗則由 Drop Zone 顯示後端 `error.*` message（例如 `error.unsupportedMediaType` / `error.forbidden`），並附上紅色警示 icon，以避免錯誤訊息重複出現在其他位置。  

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
- **完成**：Quick Create Drawer 與 Quote Editor 皆改走 `/api/quotations/transactions`，並在成功後透過 `focusThreadAfterChange` 回到 Detail。後續若需要支援批次匯入或更多欄位，只需擴充相同 payload。

## 9. Quote Editor Page（全畫面編輯）

- **Layout**：採 fixed overlay 滿版顯示，頂部保留標題與取消/儲存按鈕；主要表單以 3 欄網格排列（客戶 + 主旨、Owner/聯絡人/匯率、付款日期組、通路/幣別/備註），避免 drawer 模式造成的空白。通路、幣別、折扣型態統一使用與 Quick Create 共用的 `ChannelSelect`、`CurrencySelect`、`DiscountTypeSelect` 元件與 option，保持互動一致。
- **價目表卡片**：位於表單與預覽之間，Field label 右側統一提供「開啟價目維護 / 更新推薦 / 重新整理」三個動作按鈕；僅能在選擇客戶後啟用，且會顯示推薦來源或缺少推薦的提示。
- **預覽卡**：追蹤行項數量、總金額、未稅/稅額，視覺與 Quick Create Step3 相同，後續會串接 `/api/quotations/preview` 以即時計算。
- **行項區**：支援 async SKU 下拉（同 Quick Create），並沿用 `DISCOUNT_TYPE_OPTIONS` 以確保折扣欄位提示一致。
- **價目預設**：選取 SKU 後即呼叫 `/api/item-prices` 查詢選定價目表的 UOM / 稅別；若找得到則鎖定欄位並自動套用，若找不到則在行卡顯示缺價提示並阻擋預覽/儲存，行為與 Quick Create Drawer 相同。
- **地址矩陣**：帳單/寄送地址以 2 欄卡片放在行項之後，每張卡片提供 10 個欄位（收件人、公司、電話、國家/州/城市/區/郵遞區號、地址行1/2），placeholder 與 Quick Create 一致，輸入 line1 會同步覆寫 fullAddress。
- **擴充欄位**：`ExtAttrPanel` 預設以 `columns={3}` 呈現，配合 3 欄網格觀感。
- **共用元件**：Quick Create 與 Quote Editor 共用 Channel/Currency/Discount 下拉及聯絡人 Async select，確保 refresh 行為與提示文字相同；若需新增其他下拉（付款條件、SKU 折扣型態等）也優先放在 `/src/components/shared/quote/QuoteSelects.tsx` 內擴充，避免雙邊定義不一致。
