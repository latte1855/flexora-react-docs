# Quotation UI 重構計畫

`flexora-react-ui` 現在實作為 Ecme + vtiger-style 的左右三段版面：左邊 menu bar，右邊上下分區（右上功能列、右下主內容）。在主內容中左側 list 目前預計放快速查詢條件，右側為查詢結果清單，而編輯畫面尚未有成熟規劃。依照 Phase 3 & Phase 4 目前已完成的 spec（workflow、api、pricing、error code）以及 `flexora-react-ui` 的總體架構，以下為重構方向與步驟：

## 1. 新的主畫面架構（List + Summary + Associations）

1. 保留現有三段架構，左 menu bar 不變；右下主內容改為 `List + Detail` 兩欄：
   - **右下左欄（List + Filters）**：快速查詢仍在此，可依 `api-spec` 提供的 Criteria（customerId, ownerId, status, validUntil range）設計 filter UI；提供狀態 badge 與 `activeRevisionStatus` 改變視覺。
   - **右下右欄（Detail Panel）**：點擊清單某筆後載入該 Revision 的詳細資料，包括 summary（threadNo, status, validUntil, totalAmount, approvals）與多個關聯區塊（Contacts, Sales Orders, Delivery Notes, Linked Documents...）。

2. Summary 區域：
   - 顯示 thread/ revision、customer、owner、validUntil、status badge、totalAmount、Gross/Net 提示。
   - 顯示簡易工作流狀態（SUBMIT, APPROVE, CANCEL）與 `availableActions`（由 `workflow-info` API 提供）；可呈現 guard 提示（例如缺 validUntil、無行，根據 `guards` Relay）。

3. 關聯資料區塊：
   - 每塊呈現一個 `Card`：Contacts, Related Sales Orders, Deliveries, Documents, Replenishment Requests。
   - 透過 `sales-order`、`delivery-note` API 取得資料，依 `sourceQuotationThreadId`/`revisionId` 篩選。
   - 可展開顯示核心欄位（例如 Sales Order 狀態、Delivery note shipment Date、Document Type/Name）。

## 2. 編輯畫面方向（待補充）

1. 目前編輯畫面以 Ecme 表單為主，可先暫定 `Drawer`/`Modal` 形式編輯 `QuotationRevision`。
2. 建議以 `Summary`+`Lines`+`Workflow drawer` 的順序排列，Workflow Drawer 按鈕直接呼叫 `POST /workflow` endpoint； guard 提示（`guards` flags）可直接呈現在 Drawer 中。
3. 後續可根據 `pricing-rules` 攻略、`error-codes` 的 guard 與 `phase4` UI existing mockups，設計 Tabs（Summary、Lines、Workflow、Attachments）。

## 3. 同步與驗證

1. 所有 UI 改動需對應 `api-spec`、`workflow-spec`、`pricing-rules`，尤其 `workflow-info` 的 `availableActions`/`guards` 與 `pricing` 的 `lineSubtotal` 是否同步。
2. `quotation-ui-wireframe.md` 目前的 wireframe 可以重寫成三欄 `List + Summary + Associations`，並將 guard toast、 workflow drawer 等列入 `react-ui-checklist.md`。
3. 建議在 `docs/migration/log.md` 記錄此次 UI 設計原則與程式碼依據（`flexora-react-ui` components + backend APIs）。

## 4. 後續步驟

1. 建立 `QuotationDetailCard` 九宮格：Summary / Guards / Pricing / Associations。
2. 將 associations data mapping 範例加入 `docs/specs/phase4-quotation/ui-spec.md`（例如 sales order list 需要顯示 status badge、delivery link）。
3. 依據 guard/API 結果調整 `react-ui-checklist`，保持前後端一致。

### 4.1 下一步建議

1. **Detail View Wireframe**（List 點選 → Summary+Associations Pane）：新增在 `docs/ui/wireframes/quotation-detail-wireframe.md` 的 List/Pipeline 基礎上，快速補 `Details` layout，包含 Summary cards, Associations (Contacts, Sales Orders, Deliveries, Documents)。
2. **UI Spec 更新**：將上述 wireframe 描述轉寫進 `docs/specs/phase4-quotation/ui-spec.md`（新增 section `3.x Detail Pane`, `4.x Pipeline`).  
3. **`react-ui-checklist.md`**：加入 Workflow drawer / guard / pipeline toggle 必須檢查的項目（API endpoints, error key mapping）。
4. 待 Detail wireframe 決定後，可決定是否需要 HTML prototype；目前以輕量的文字 wireframe 方式維持 PRD/UX 一致性。
5. **Drawer 方案**：為了避免右側固定區塊被 list 擠壓（使用者偏年長，字體需放大），可改採 Drawer 展現 detail；預設列表佔滿寬度，點 row 時右側滑出 Drawer 顯示 Summary/Associations，關閉即可恢復 Table 寬度。
