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
- **ASCII Wireframe**
  ```
  [Filter 260px] | [Delivery List] | [Detail/Workflow]
  操作列：快速建立 / 建立出貨單
  列表欄位：Delivery No / SO / Warehouse / Status / ShipDate
  Pipeline (若啟用)：Packing → Ready → Shipped
  ```

### Pipeline 規則（若啟用）

| 狀態 | 代碼 | 可拖曳至 | 條件 | 提示 |
| --- | --- | --- | --- | --- |
| 打包中 | PACKING | READY | 已建立至少一個包裹並指定重量 | 缺少包裹則彈窗 |
| 待出貨 | READY | SHIPPED | 承運商、追蹤號碼已填、出貨日期>=今天 | 若庫存未扣減顯示警示 |
| 已出貨 | SHIPPED | DELIVERED | 收到物流事件或手動確認 | 必須輸入實際送達日期 |
| 已送達 | DELIVERED | CLOSED | 所有行項完成/退款結清 | 僅管理員可結案 |
| 已取消 | CANCELLED | — | 只能透過 Workflow 按鈕 | 需輸入理由 |

## Detail Panel / Drawer

1. **Detail Panel**：顯示 Delivery No、Sales Order、Warehouse、狀態、地址、Tracking。  
   - Tabs：摘要 / 行項 / Package / Workflow / 附件 / 關聯記錄。  
   - Workflow Buttons：Packing → Ready → Ship → Deliver / Cancel。
2. **Drawer（快速建立／編輯）**：
   - 基本欄位：Sales Order、Warehouse、出貨日期、承運商、追蹤資訊。  
   - 行項：選擇 Sales Order 行項與出貨數量，顯示已出/剩餘。  
   - Package 子卡：新增包裹、指定重量/尺寸、序號。  
   - Ext Attr：三欄網格輸入（例如溫控、特殊標籤）。

### 欄位驗證

| 區塊 | 欄位 | 規則 |
| --- | --- | --- |
| 基本 | salesOrderId | 必填；SO 狀態需為 APPROVED |
| 基本 | warehouseId | 必填；若倉庫無可出貨權限顯示錯誤 |
| 基本 | shippingDate | 不得早於今天；若空白顯示警示 |
| 承運商 | carrierId / trackingNo | `READY → SHIPPED` 前必填；tracking 格式依承運商顯示遮罩 |
| 行項 | qtyShipped | `<= qtyRemain`；若輸入 IME 需防止 NaN |
| Package | weight | 必須 >0，支援公斤/磅轉換 |
| Serial | serials | 數量需與 qty 對應，掃碼時不可重複 |

## Package / Serial UI

- 在 Detail Panel 的 Package Tab 顯示包裹列表，每筆可展開查看內容。  
- 連結「新增包裹」→ 開啟子 Drawer：輸入箱號、重量、序號掃描紀錄。  
- 支援「複製上一箱」與匯入 CSV（若有 WMS 資料）。  
- Serial Tab：清單顯示 SKU + Serial/Lot，支援掃碼與批量貼上。ASCII：
  ```
  ┌──── Packages ────┬──── Serial List ─────┐
  │Pkg001  2.5kg     │SKU   Serial          │
  │  • SKU A  Qty 2  │P-100 SN001           │
  │Pkg002  1.2kg     │P-100 SN002           │
  └──────────────────┴──────────────────────┘
  ```

### 匯入 / 掃碼

- 行項與包裹支援 CSV 匯入，欄位：`packageNo,carrierCode,trackingNo,skuNo,qty,weight,serials`。  
- 匯入流程：上傳 → 預覽 → 顯示錯誤列（例如 `delivery.import.invalidTracking`）→ 套用。  
- 掃碼模式：側邊顯示掃描進度、允許切換單件/批次模式，序號重複時即時警示。

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
