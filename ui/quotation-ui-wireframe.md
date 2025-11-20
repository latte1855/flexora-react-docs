# Flexora ERP — 報價模組 React UI Wireframe（Draft）

> 版本：2025-11-13  
> 依據：JHipster 8.11（React 18 + Redux Toolkit + React Router 7 + Bootstrap 5）  
> 參考系統：vtiger CRM、nexterp（以 pipeline 中心 + drawer driven 編輯為靈感）  
> ⚠️ 原 JHipster Admin/Entity CRUD 體驗需完整保留，此 wireframe 為「加值層」：在現有 route 與 component tree 上擴充（不任意移除原元件），僅在確認安全或 UX 套件優勢後才評估替換。  
> 🎨 Theme：沿用 Bootswatch Materia（Bootstrap 5 皮膚），新增元件須使用 `var(--bs-*)` 色票或 Materia SCSS token，自訂色僅能在 theme token 基礎上延伸。  
> 📁 目錄規劃：UI 文件集中於 `docs/ui/`（詳見 `docs/ui/README.md`），本檔與 `react-ui-checklist.md` 為報價模組 UI 的主要參考。

## 1. UI 設計原則

- **App Shell 常駐**：Top App Bar（搜尋、快速建立、通知）、左側 Module Rail（CRM / Sales / Purchase / IM / MFG / Docs / System）。
- **雙模式清單**：Table 與 Pipeline（Kanban）在同頁切換，行動與桌機皆保持一致操作手勢。
- **Drawer First**：維持 JHipster CRUD 主畫面的路由結構，但主要互動（新增、編輯、Workflow、轉單）以 Drawer / Side Panel 呈現，避免整頁跳轉。
- **Context Tabs**：詳細頁以 Tab/Section 區分「概要、行項、價格、Workflow、附件 / ExtAttr」。
- **可組態的 Toolbar**：依權限顯示批次動作（送出、轉單、鎖定、刪除），與 vtiger 的 Mass Action 類似。
- **可延伸 Component Library**：沿用 `app/shared/components`，新增 `QuotationBoard`, `RevisionTimeline`, `ExtAttrPanel`, `AttachmentGrid` 等可於 Sales Order 重用的元件。

## 2. 全域導航 / Menu Tree（草案）

| 層級        | 模組           | 子選單（僅列出與 Phase 4/5 相關）                              |
| ----------- | -------------- | -------------------------------------------------------------- |
| Global      | Dashboard      | 即時 KPI（Sales Pipeline、Open Orders）                        |
| Module Rail | CRM            | Leads、Opportunities（Phase 5.1 後續）                         |
|             | Sales          | **Quotations**、Sales Orders、Delivery Notes、Customer Returns |
|             | Purchase       | RFQ、PO、Receipts、Supplier Returns                            |
|             | Inventory (IM) | Stock, Reservations, Adjustments                               |
|             | Manufacturing  | MO、Routing、Work Centers                                      |
|             | Documents      | Library、Templates                                             |
|             | System         | Numbering Rules、ExtAttr 定義、Workflow Designer               |

> Menu Trigger：左上角漢堡鍵 → Module Rail（類似 nexterp），Rail 可縮入為 icon-only；子選單採「工作台 / 清單 / 報表」分類。

## 3. Wireframe — Quotations

### 3.1 Quotations Hub（List + Pipeline 切換）

```
+--------------------------------------------------------------------------------+
| Top App Bar: [☰] Flexora | 全域搜尋 | 快速建立(+) | 通知🔔 | 使用者頭像              |
|--------------------------------------------------------------------------------|
| Module Rail | 上層 Breadcrumb: Sales / Quotations | Filter Chip Row            |
| (icons)     | [Pipeline ☐] [Table ☑]  [Owner ▼] [Stage ▼] [Date Range ▼]       |
|-------------+------------------------------------------------------------------|
|             | Action Toolbar: (New) (Send) (Approve) (Convert to SO) (More…)   |
|             |                                                                  |
|             | ┌───────────────Table Mode─────────────────────────────────────┐ |
|             | | Checkbox | Quote # | Customer | Stage | Total | Owner | ...  | |
|             | | [ ] QTN-2025-001  | 華星 | Draft | $45k | Shelly | ...      | |
|             | | [ ] QTN-2025-002  | 鴻裕 | Sent  | $12k | Allen  | ...      | |
|             | └──────────────────────────────────────────────────────────────┘ |
|             |                                                                  |
|             | Pipeline 切換：Draft / Sent / Approved / Won / Lost / Expired    |
|             | ┌──── Draft ────┐  ┌──── Sent ─────┐ ...                          |
|             | | Card:         |  | Card ...      |                               |
|             | | 客戶+金額     |  | 支援拖拉 Stage|                               |
|             | └───────────────┘  └──────────────┘                               |
|--------------------------------------------------------------------------------|
```

### 3.2 Quotation Detail + Revision Timeline

```
+--------------------------------------------------------------------------------+
| Breadcrumb: Sales > Quotations > QTN-2025-001                                  |
| Header: [Status Pill] QTN-2025-001  | Customer | Grand Total | CTA Buttons     |
| CTA: (Edit) (Send) (Approve) (Convert) (More Actions ▼)                         |
|--------------------------------------------------------------------------------|
| Tabs: Overview | Line Items | Pricing | Workflow | Attachments | Ext. Fields   |
|--------------------------------------------------------------------------------|
| Overview Tab (Two-column layout)                                               |
|  - Primary Info (customer, owner, validity, channel)                           |
|  - Address Snapshot (Billing / Shipping cards)                                 |
|  - Summary cards (Subtotal, Discount, Tax, Grand Total)                        |
|--------------------------------------------------------------------------------|
| Right Side Drawer (sticky)                                                     |
|  - Revision Timeline (cards: Rev.1 Draft, Rev.2 Sent, ...)                     |
|  - Status History timeline                                                     |
|  - Linked Sales Orders list (per SOT-7)                                        |
|--------------------------------------------------------------------------------|
| Workflow Tab                                                                   |
|  - Event buttons grid                                                          |
|  - Trigger form (reason/channel/attachments)                                   |
|  - Activity log                                                                |
--------------------------------------------------------------------------------+
```

### 3.3 Drawer：Create / Edit Quotation

```
┌──────────────────────────── Drawer (60% width) ──────────────────────────────┐
│ Header: 新增報價單  | Stepper: 基本資料 → 行項 → 確認                           │
│ Body:                                                                          │
│   Section 基本資料：Customer selector、Currency、有效日期、Numbering (auto)     │
│   Section 行項：DataGrid (SKU, 描述, 單價, 折扣, 稅碼) + Add Row / Import       │
│   Section 客製欄位：ExtAttrPanel（動態欄位 + 驗證）                              │
│   Section 附件：AttachmentGrid（可拖拉上傳 / 選取 DocumentLink）                 │
│ Footer: (取消) (儲存草稿) (送出並通知客戶)                                      │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 Workflow / Convert Drawer

- Workflow Drawer：依事件顯示需填欄位（channel/reason/validUntil）→ 預覽差異 → Submit。
- Convert Drawer（Quotation → Sales Order）：顯示可勾選 line items、剩餘可轉量、付款條件與新的 Sales Order 起始狀態；成功後顯示 Sales Order 連結（支援多單）。
- 佈局靈感：nexterp 的 Side Panel + vtiger 的 Process Wizard。

## 4. 互動流程（摘要）

1. **快速建立**：從 Top App Bar 的 `+` 按鈕直接開 Drawer（帶入預設 owner/number）。
2. **Pipeline 操作**：拖拉卡片改變 Stage → 彈出 Workflow quick-form（補 channel/reason）；失敗則卡片退回。
3. **Revision 管理**：在 Detail 頁右側 Timeline 選擇 Revision → Panel 顯示 Diff，提供「複製成新版本」按鈕。
4. **客製欄位/附件**：共用 `ExtAttrPanel`/`AttachmentGrid`，也供 Sales Order 模組使用。
5. **轉 Sales Order**：Drawer 中選擇項目 + 指定 fulfillment 策略 → 呼叫 `/to-sales-order` API，成功後顯示新單據卡片及下一步 CTA（SUBMIT/CONFIRM）。

## 5. 建議任務拆解

1. **UI 基礎建置**：調整 `app/layout`（App Shell, Module Rail, Quick Create 菜單）、新增 Navigation config。
2. **Quotation Hub**：實作 Table + Pipeline 切換、快速篩選、批次動作（Redux slice + RTK Query）。
3. **Detail & Drawer 元件**：建立 `QuotationDetailPage`, `QuotationDrawer`, `RevisionTimeline`.
4. **Workflow 與轉單體驗**：整合後端 `/events/{code}`、`/to-sales-order` API，支援 Validation 提示。
5. **共用組件**：`ExtAttrPanel`, `AttachmentGrid`, `AddressCard`，後續 Sales Order 亦可套用。
6. **Storybook / 視覺稿（可選）**：為核心元件撰寫 stories，確認樣式。
7. **文件**：於 `docs/flexora-quotation-management-spec.md` 補 UI 操作章節；AGENTS.MD 記錄前端 UI 原則。

## 6. 待確認事項

- 欄位配置：建議維持與後端 DTO 一致（資料不缺漏），但 UI 顯示採「核心 + 詳細」雙層：概要區呈現 90% 常用欄位，其餘欄位放入 `Advanced` accordion 或 Detail Tab，以避免一次塞滿畫面；新增欄位時只需在詳細區補上即可。
- Pipeline 欄位：沿用 `quotation_status_def` 既有代碼（Draft / Sent / Approved / Won / Lost / Rejected / Expired / Cancelled）。顏色預設對應 Materia 主題：`Draft=secondary`、`Sent=primary`、`Approved=success`、`Won=teal`、`Lost=danger`、`Rejected=warning`、`Expired=purple`、`Cancelled=dark`。若未來要自訂色，可於 `status_def.metadata.color` 設定 HEX，UI 再 fallback 到上述預設。
- Quick Actions：Toolbar 固定包含（新增、送出、批准、轉單、匯出 CSV、軟刪除/恢復）。Phase 4.1 擴充項（Excel 行項匯入、共用連結、複製 thread）預留在「更多」下拉中，初期以 disabled 狀態顯示並標記 TODO，避免再次調整 Toolbar 佈局。
- 權限模型：列表依後端 Criteria（含 owner/data scope）提供 Filter，不在前端自行過濾；詳情頁與 Workflow 按鈕須根據 API 回傳的 `canEdit/canTriggerEvent` 或 `ownerId == currentOwner` 決定是否顯示；Pipeline 拖拉若因 DataScope 被拒絕需立即 toast 並將卡片移回原欄。前端 guard 規則需寫入 React Checklist，並與後端 DataScope 實作保持一致。

> 本文件作為 wireframe 初稿，後續可轉成 Figma 或 Storybook 原型供 UI/UX 討論。
