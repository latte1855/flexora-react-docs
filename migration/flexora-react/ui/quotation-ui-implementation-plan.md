# 報價模組前端實作藍圖（Quotation UI Implementation Plan）

> 版本：2025-11-13  
> 依據：`docs/ui/quotation-ui-wireframe.md`、`docs/ui/react-ui-checklist.md`、Phase 4 Spec。  
> 目的：將報價模組的新 UI（Table + Pipeline + Drawer 等）拆解為具體畫面、資料流程與任務項目，便於逐步開發與追蹤。

---

## 1. 畫面清單與重點

| 編號  | 畫面 / 元件                         | 主要內容                                                                  | 依賴 / 備註                                         |
| ----- | ----------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------- |
| UI-01 | **Quotation Hub – List View**       | DataTable + filter chips（Owner、狀態、日期區間），批次動作 Toolbar。     | 需沿用 JHipster List；新增 pipeline 切換按鈕。      |
| UI-02 | **Quotation Hub – Pipeline View**   | Kanban 欄（Draft/Sent/...），卡片顯示客戶/金額/有效日，支援拖拉觸發事件。 | 與 Workflow API 整合；顏色依 Materia mapping。      |
| UI-03 | **Detail Page + Revision Timeline** | Header summary + Tabs（Overview/Line Items/Pricing/Workflow/Attachments） | 右側 Sticky Timeline + SalesOrder link 列表。       |
| UI-04 | **Quotation Drawer（Create/Edit）** | Drawer Wizard：基本資料 → 行項 → 客製欄位/附件；支援 auto numbering。     | 行項表格 + ExtAttrPanel + AttachmentGrid 共用元件。 |
| UI-05 | **Workflow Drawer / Quick Action**  | 事件表單（channel/reason/note），送出、批准、客戶接受、取消等。           | 守衛與 Workflow Service 對齊；Pipeline 拖拉共用。   |
| UI-06 | **Convert to Sales Order Drawer**   | 勾選行項 + 餘量提示 + Sales Order 起始狀態，顯示結果 SO 清單。            | SOT-7 API；需顯示自動 SUBMIT/CONFIRM 結果。         |
| UI-07 | **ExtAttr / 附件 / 地址共用元件**   | AddressSnapshotCard、ExtAttrPanel、AttachmentGrid，可在 Quote/SO 共用。   | 機制記錄於 `docs/ui/components/README.md`。         |
| UI-08 | **Quick Create / App Shell 更新**   | Top App Bar 的「快速建立報價」按鈕、左側 Module Rail 新增 Sales 區塊。    | 不可影響 JHipster 原有導覽。                        |

---

## 2. 資料流程（概述）

1. **列表/篩選**：透過 `GET /api/quotations`（Criteria API）取得資料；Pipeline 模式可呼叫 `?status.in=` 與分頁參數。Owner 篩選邏輯：
   - 「全部」：傳 `ownerScope=MINE`，由後端依 `v_owner_visible` 決定使用者可看 owner 清單（Admin 自動全開）。
   - 「我的報價」：傳 `ownerScope=SELF`，後端透過 `SecurityUtils.getCurrentUserId()` 強制匹配 `owner_id = userId`，不需也不允許前端帶入其他 id。
2. **Pipeline 拖拉**：拖曳卡片時呼叫 `POST /api/quotations/{id}/events/{eventCode}`；若失敗，UI 回滾並顯示錯誤（BadRequestAlert message）。
   - **Quick Create Drawer Flow**：使用者在 Drawer 填寫基本資料 → 呼叫 `POST /api/quotations/preview` 做行項/金額檢核 → 同一份 payload 再呼叫 `POST /api/quotations` 建立草稿。Drawer 需帶入行項、付款條件、Billing/Shipping Address Snapshot，並允許後續 Phase 5.1 擴充 ExtAttr/附件。
   - 行項表格會即時呼叫 `GET /api/item-skus`（同時支援 `skuNo.contains` 與 `skuName.contains`）載入可銷售 SKU，並自動帶入預設 UoM / 稅別。選項顯示 SKU 編碼、名稱與 UoM，確保 Sales 能在 Drawer 內完成 SKU 選擇。
   - Drawer 的單價欄位目前僅供覆寫需求預留：輸入的值會被序列化為 `properties.extAttrs.manualUnitPrices`（陣列：`[{ lineIndex, skuId, unitPrice }]`），後端仍依 Pricing 結果計算，待 Phase 5.1 正式支援 `unitPriceOverride` 後再改用真正欄位。
   - 行項視圖下方提供 Summary 卡片，隨輸入即時計算總筆數 / 已就緒行數 / 合計數量 / 手動單價筆數，方便銷售快速自檢；每列也會以 Badge 呈現「待選 SKU / 待填數量 / 手動單價 / 就緒」狀態，搭配紅框提示，讓使用者在第二步就能看懂哪一行尚未完成。
   - **2025-11-21 更新**：快速建立 Drawer 的 ExtAttr 區塊改為呼叫 `/api/quotation-revision-ext-attr-defs` 取得尚未刪除的欄位定義，依 `dataType/requiredAttr` 動態渲染（文字 / 數字 / 日期 / 布林下拉），並於送出前檢查必填欄位；payload 統一序列化為 `properties.extAttrs`，確保資料庫調整後前端可立即繼承。
   - **2025-11-21 新增**：Detail Drawer 增加「建立新版本」按鈕，Quick Create Drawer 會帶入 Thread/Revision/Line Items/ExtAttr 既有資料並呼叫 `POST /api/quotations/{threadId}/revisions` 產生下一版。
   - **2025-11-21 API**：新增 `PUT /api/quotations/{threadId}/revisions/{revisionId}`，重新以 Pricing 結果覆寫指定 revision（僅草稿），後續 Drawer Edit 流程可直接串此端點。
   - **2025-11-22 UI**：Detail Drawer 於 DRAFT 狀態顯示「編輯草稿」鈕，Quick Create Drawer 加入 Edit 模式並串接新 API，可直接覆寫既有修訂版。
   - **2025-11-22 新增**：Quick Create Drawer 明細新增 UoM、稅別與「行折扣型態 / 值」欄位，透過 `/api/uoms`、`/api/tax-codes` 取得選項，並在 `QuotationPreviewRequest.items` 帶入 `uomId`、`taxCode`、`discountType`、`discountValue`。後端 `QuotationPreviewItemDTO` / `QuotationService` 亦同步擴充，預覽與 Revision 落地可保留行折扣資訊並套用於 `QuotationCalculationService`。
   - **2025-11-22 Attachments/Related**：Detail Drawer 的 Attachments Tab 已串接 `/api/document-links`、`/api/documents/upload` 與 `POST /api/quotations/{threadId}/documents`，可上傳/下載並區分 Thread 與 Revision 附件；Related Tab 串接 `/api/quotations/{threadId}/links` 顯示 Sales Order 連結摘要。
   - **Pipeline 性能**：每個欄位預設只渲染 30 筆卡片，使用者可按「載入更多」逐段展開（狀態變更或重新整理時會重置），避免一次性載入上百筆造成 DOM lag。
   - Convert Drawer 會顯示選取行的稅額摘要與幣別資訊，並提供匯率提示，讓轉單前能快速檢視稅負分布。
   - 在支援 `IntersectionObserver` 的瀏覽器會自動偵測欄位捲動並載入下一批（Load More 按鈕仍保留為後援），減少手動點擊。
3. **Drawer 建立/更新**：使用 `POST /api/quotations`/`PUT /api/quotations/{id}`；行項透過 nested DTO 傳遞；`properties.extAttrs` 由 ExtAttrPanel 統一管理。
4. **Workflow Drawer**：根據事件需求動態顯示欄位（Channel/Reason/ValidUntil）；送出前先跑前端檢核，再呼叫事件 API。
   - Timeline / Pipeline 的 Action menu（例如 Retry / 重新發送）會預先帶入事件代碼與既有的理由/備註，開啟 Workflow Drawer 後即可微調再送出，避免人工重複輸入。
   - 若需要快速補發，Timeline action menu 現在也能直接向事件 API 重送，並於成功後重新載入資料。
5. **轉 Sales Order**：呼叫 `POST /api/quotations/{threadId}/revisions/{revId}:to-sales-order`，回傳 `SalesOrderDTO` + Link 資訊；完成後 Hub 需要重新整理 Pipeline 卡片狀態。
6. **共用元件**：ExtAttrPanel 讀取 `ExtAttrDef` 列表、維護 JSON；AttachmentGrid 透過 Document Service 取得 `documentId` 後再連結；AddressCard 使用 AddressSnapshot VO。
7. **ExtAttr 評估**：Drawer 於提交前會呼叫 `/api/quotation-revision-ext-attr-defs` 取得 `ExtAttr` 定義（code/dataType/required），前端將填寫結果序列化為 `properties.extAttrs` JSON，並透過 `ExtAttrPanel` 共用元件管理輸入；詳細頁同樣使用 `ExtAttrPanel` 呈現，不再重複排版。
8. **單價覆寫（TODO）**：目前 Drawer 的單價欄位僅暫存於 `properties.extAttrs.manualUnitPrices`。待 Phase 5.1 擴充 `QuotationPreviewRequestDTO` / `QuotationPreviewItemDTO` 及 `quotation_item` 欄位後，再改為正式的 `unitPriceOverride`/`pricingMode` 欄位並於 preview/create 時帶入。
9. **Hub 篩選 / 分頁 / URL 同步**：`GET /api/quotation-threads` 已支援 `currentRevisionStatusCode.*`、`currentRevisionValidUntil.*` 以及 `ownerScope` 參數；「全部」固定傳 `ownerScope=MINE`（取可見 Owner 清單）、「我的報價」傳 `ownerScope=SELF`（強制 owner=本人）。List View 以 server-side 分頁載入並將條件同步到 URL query（回上一頁時仍保留）；Pipeline 模式沿用相同條件一次抓 500 筆資料並依狀態分欄。Detail 頁面與 Hub 之間透過 `sessionStorage.quotationHubRefreshToken` 交換刷新訊號，確保在詳情頁觸發 Workflow/轉單後回到 Hub 時會自動重新查詢。
10. **Workflow Drawer UX**：Channel 改為下拉選單，若守衛要求 `channel/reason/validUntil` 會動態顯示欄位；`validUntil` 過期時會停用送出並顯示提示。Quick Create Drawer 中改以「取消 / 上一步 / 下一步」呈現，便於流程回饋。

> 詳細欄位與流程說明仍以 Phase 4 Spec 為準；本藍圖主要聚焦前端如何串接與呈現。

---

## 3. 任務拆解（建議工單）

| ID        | 任務內容                                                                                                                                                                                                                                                                                                                                                              | 說明 / 驗收點                                                                                                                                                                                                                                                  | 相關文件                 |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| QTN-UI-01 | 建立 `docs/ui` 目錄與 React checklist（✅已完成）                                                                                                                                                                                                                                                                                                                     | 基礎規範文件；本表為後續任務依據。                                                                                                                                                                                                                             | README / wireframe       |
| QTN-UI-02 | App Shell / Navigation 調整                                                                                                                                                                                                                                                                                                                                           | Module Rail 加入 Sales 區塊、Quick Create 報價按鈕；不影響既有 JHipster 入口。（✅ 已完成）                                                                                                                                                                    | wireframe 3.1、App Shell |
| QTN-UI-03 | Quotation Hub List View （Table + Filter + Toolbar）（✅ 2025-11-15 初版完成）                                                                                                                                                                                                                                                                                        | 完成 DataTable、批次動作工具列、filter chips、Pipeline 切換 tab；支援多選＋交給 Workflow。                                                                                                                                                                     | UI-01                    |
| QTN-UI-04 | Pipeline View + Drag & Drop（✅ 2025-11-15 初版完成）                                                                                                                                                                                                                                                                                                                 | 依狀態分欄、卡片顯示資訊；拖拉串接 Workflow API（SEND/APPROVE/REJECT…），即時提示成功/失敗。                                                                                                                                                                   | UI-02                    |
| QTN-UI-05 | Detail Page + Revision Timeline（進行中：2025-11-16 已完成摘要卡 / AddressCard / AttachmentList / ExtAttrPanel / Timeline / Quick Actions + 轉單 CTA；2025-11-17 新增 Overview / Items / Pricing / Workflow / Attachments Tabs、SalesOrder Links 卡片、Pricing 多幣別切換 + 稅額 tooltip + Discount History、Workflow 搜尋/匯出/Load More + 行動選單、附件下載/預覽） | 新增自訂詳細頁（/sales/quotations/:id）：摘要卡、地址、SalesOrder Link、狀態時間軸、Workflow Drawer 整合。                                                                                                                                                     | UI-03                    |
| QTN-UI-06 | Quotation Drawer（Create/Edit Wizard｜進行中：2025-11-16 新增步驟驗證、摘要區、Next Guard、共用 ExtAttrPanel）                                                                                                                                                                                                                                                        | Drawer Form（分 Step）、行項編輯表格、ExtAttr/附件區塊整合、號碼自動提示。                                                                                                                                                                                     | UI-04                    |
| QTN-UI-07 | Workflow Drawer / Quick Actions（✅ 2025-11-15 初版完成）                                                                                                                                                                                                                                                                                                             | 事件觸發對話框（SEND/APPROVE 等）、Pipeline 拖拉共用驗證；Drawer 透過 `GET /api/quotations/{threadId}/events/options` 動態載入事件與 Guard 提示，再呼叫事件 API；Quick Create Drawer 亦在此任務中串接 `preview → createDraft` 流程（含品項、付款條件、地址）。 | UI-05                    |
| QTN-UI-08 | Convert to Sales Order Drawer（進行中：2025-11-16 新增 Drawer + 行項勾選/預留策略設定 + 剩餘可轉量/摘要；2025-11-17 補 Selection Summary 容量/無上限顯示）                                                                                                                                                                                                            | 勾選行項、顯示剩餘可轉量與結果 SO 連結。                                                                                                                                                                                                                       | UI-06                    |
| QTN-UI-09 | 共用元件封裝（AddressSnapshotCard、ExtAttrPanel、AttachmentGrid）                                                                                                                                                                                                                                                                                                     | 元件放在 `app/shared/components`，具 Storybook / 測試；報價與訂單共用。（TODO：補 Storybook + Jest Snapshot 測試）                                                                                                                                             | UI-07                    |
| QTN-UI-10 | React Store / API hooks                                                                                                                                                                                                                                                                                                                                               | 整理 RTK Query service：threads、revisions、workflow events、customization API。                                                                                                                                                                               | 2. 資料流程              |
| QTN-UI-11 | 體驗 / 安全檢查                                                                                                                                                                                                                                                                                                                                                       | 對照 `react-ui-checklist.md`（theme、DataScope guard、Numbering、文檔同步）。                                                                                                                                                                                  | Checklist                |
| QTN-UI-12 | 文件更新                                                                                                                                                                                                                                                                                                                                                              | 完成各階段後更新 wireframe、ui README、phase spec，於 PR 中敘述。                                                                                                                                                                                              | README / spec            |

> 可依實際進度拆成多個 sprint；每個任務建議附 PR/Issue 連結並引用本表 ID。

---

## 4. 後續展望 / Phase 4.1

- **Excel 行項匯入**：需設計模板上傳流程，預計整合 Document Service。
- **共用連結 / 分享 PDF**：待 Document 模組完成產生報價 PDF，於 Quick Actions 預留入口。
- **Pipeline 自訂顏色**：未來可允許在 `quotation_status_def.metadata.color` 設定 HEX，UI 讀取後覆蓋 Materia 預設。
- **Dashboard Widget**：在 Sales 模組首頁加入報價 KPI 摘要（Draft/Sent/Approved 總額、逾期提醒）。

---

### TODO（測試 / Storybook）

- AddressSnapshotCard / ExtAttrPanel / AttachmentList Storybook 範例與 Jest Snapshot（QTN-UI-09）
- 共用 Drawer（Quick Create / Convert / Workflow）互動測試案例（待 QTN-UI-06 / 08 穩定後補上）

> 本藍圖可視為前端待辦清單；若任務或畫面 Scope 有變更，請同步更新此文件與 `docs/ui/README.md`，並在 AGENTS / Phase spec 中做好引用。\*\*\*
