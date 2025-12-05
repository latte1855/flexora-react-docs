# Inventory Module – UI Spec

> 以 Phase 3 UI 草稿為基礎，重新整理為 Hub + List + Detail + Drawer 風格。仍需在 Figma/Wireframe 中補畫面，這裡先列結構需求。

## Workspace

```
┌──────── Filter Hub ─────────┐┌──────── Inventory Workspace ───────┐
│Preset：全部 / 低庫存 / 已保留   ││操作列：調整 / 轉移 / 新增盤點任務      │
│Warehouse、Bin、SKU、Tag      ││Tabs：Balance / Reservation / Task │
└─────────────────────────────┘│列表 + Detail Panel                │
                              └────────────────────────────────────┘
```

- **Filter Hub**：預設條件（低於安全庫存、已保留、異常），進階篩選（倉庫、儲位、SKU、Lot/Serial、Owner）。搜尋支援 IME，必須按「套用」才送出 API，避免多次呼叫。
- **主視圖**：
  - `Balance` Tab：顯示 SKU + Warehouse + Qty On Hand / Reserved / Available。  
  - `Reservation` Tab：列出被 Sales Order / Project 保留的紀錄，可釋放或轉移。  
  - `Task` Tab：補貨任務、盤點任務。  
- **操作列**：`建立調整單`（Drawer）、`轉移`、`盤點`、`補貨建議`。

## Detail Panel

- **摘要卡**：SKU、Warehouse、On Hand、Reserved、Available、Cost。  
- **Tabs**：
  1. **Ledger**：依日期顯示交易紀錄（Receipt/Issue/Transfer/Adjustment），支援匯出。  
  2. **Reservation**：列出 Reservation 來源（SO、Project），可釋放。  
  3. **Lot / Serial**：顯示批號/序號與到期日。  
  4. **Tasks**：補貨/盤點任務狀態。  
  5. **Workflow / Timeline**（若盤點或補貨有審批）。  
  6. **附件 / 備註**。

## Drawer / 子 Drawer

1. **Adjustment Drawer**：倉庫、SKU、調整數量、原因、附件。支援多行項與欄位驗證（Qty >0、原因必填）。  
2. **Transfer Drawer**：來源/目的倉庫或儲位、行項、序號選取。  
3. **Reservation Drawer**：建立或修改保留條件（來源單據、數量、到期日）。  
4. **Stock Count Drawer**：盤點任務設定、套用儲位與負責人，並可匯入盤點結果。  
5. **Replenishment Task Drawer**：顯示建議補貨數量、來源 Purchase / Manufacturing。

## Dashboard / Report

- 補貨監控卡片：列出需補貨 SKU，提供「產生採購單」快捷。  
- 保留概覽：顯示即將到期的保留、釋放提示。  
- 倉庫容量 / Turnover 報表（可連結至 BI）。

## 通知 / 報表

- 補貨監控卡片：列出需補貨 SKU，提供「產生採購單」快捷。  
- 保留概覽：顯示即將到期的保留、釋放提示。  
- 倉庫容量 / Turnover 報表（可連結至 BI）。  
- 盤點報表：可匯出差異、提供 PDF。  
- 通知介面需顯示訊息紀錄（系統提醒、Webhook 回饋）。

## TODO

- [ ] 置入實際 wireframe（Balance 列表、Drawer、Detail Panel），含行距與欄寬。  
- [ ] 補欄位與驗證（序號輸入、重量、儲位格式、日期）。  
- [ ] 探討 Pipeline 是否適用於任務（如補貨/盤點），若取消需在 README 備註。  
- [ ] 定義匯入/掃碼流程 UI（盤點、序號、調整）。  
- [ ] 與 Delivery / Purchase / Manufacturing 的跳轉或子 Drawer 行為。  
- [ ] 審批或通知的 UI（例如補貨審批、盤點審核）。  
