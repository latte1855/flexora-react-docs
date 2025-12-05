# Delivery Module – UI Spec

> 介面與報價 / Sales Order 保持一致，方便使用者快速切換。

## Workspace

```
┌──────── Filter Hub ────────┐┌──────── Delivery Workspace ───────┐
│Preset：待打包 / 待出貨 / 已出貨 ││操作列：快速建立 / 建立出貨單          │
│SO / Warehouse / Status     ││View：List / Pipeline (可選)       │
└────────────────────────────┘│列表 + Detail Panel                │
                             └────────────────────────────────────┘
```

- **Filter Hub**：提供預設條件與進階篩選（SalesOrder、Warehouse、狀態、日期、承運商）。  
- **視圖**：列表為預設；若需要以狀態卡片顯示（Packing → Ready → Shipped）則啟用 Pipeline。  
- **操作列**：黑底 `建立出貨單`（滿版）、次要 `快速建立` Drawer。

## Detail Panel / Drawer

1. **Detail Panel**：顯示 Delivery No、Sales Order、Warehouse、狀態、地址、Tracking。  
   - Tabs：摘要 / 行項 / Package / Workflow / 附件 / 關聯記錄。  
   - Workflow Buttons：Packing → Ready → Ship → Deliver / Cancel。
2. **Drawer（快速建立／編輯）**：
   - 基本欄位：Sales Order、Warehouse、出貨日期、承運商、追蹤資訊。  
   - 行項：選擇 Sales Order 行項與出貨數量，顯示已出/剩餘。  
   - Package 子卡：新增包裹、指定重量/尺寸、序號。  
   - Ext Attr：三欄網格輸入（例如溫控、特殊標籤）。

## Package / Serial UI

- 在 Detail Panel 的 Package Tab 顯示包裹列表，每筆可展開查看內容。  
- 連結「新增包裹」→ 開啟子 Drawer：輸入箱號、重量、序號掃描紀錄。  
- 支援「複製上一箱」與匯入 CSV（若有 WMS 資料）。  
- Serial Tab：清單顯示 SKU + Serial/Lot，支援掃碼與批量貼上。

## 與 Sales Order / Warehouse

- 從 Sales Order Detail 直接開啟「建立出貨單」→ 跳至 Delivery Drawer 並帶入行項。  
- Warehouse 模組可透過 Async Select 選擇 Delivery；Detail Panel 顯示倉庫任務狀態。

## TODO

- [ ] 插入實際 wireframe / Figma 圖（Workspace、Package Drawer、Trace/Workflow）。  
- [ ] 列出所有欄位與驗證（尤其重量、追蹤號碼、序號、收件地址）。  
- [ ] Pipeline 是否需要 → 與 PM 確認，若保留需描述拖拉限制。  
- [ ] 匯入/掃碼流程的 UI（可在 Package Drawer 補註）。  
- [ ] 如果需顯示運費/票據等資訊，另建卡片。  
- [ ] 通知提示：Detail Panel 顯示「已寄送 Email / SMS / Line」，歷史紀錄在 Timeline。  
