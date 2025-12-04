# Customer Module – UI Spec

> 參考既有 Customer Workspace 原型，這裡整理新版結構與行為。

## Workspace Layout

```
┌──────── Filter Hub ────────┐┌──────── Customer Workspace ─────┐
│Preset：我的客戶、潛在、黑名單 ││操作列：快速建立客戶 / 建立客戶        │
│進階：Owner、狀態、產業、標籤 etc││View：List / Kanban(可選)            │
└────────────────────────────┘│列表 + Detail Panel               │
                             └──────────────────────────────────┘
```

- **Filter Hub**：預設列表 + 收藏條件；支援關鍵字、Owner、Team、狀態、Industry、標籤。  
- **List View**：欄位（客戶名稱、Owner、客戶群、狀態、最近活動、健康度）。  
- **Kanban（可選）**：若需要以 Pipeline 方式呈現（例如 潛在→洽談→客戶），可沿用報價 Pipeline 元件。

## Detail Panel

- **摘要卡**：客戶基本資料、Owner、付款條件、健康指標（Health Status, Last Activity）。  
- **Tabs**：摘要 / 聯絡人 / 活動 / 銷售機會 / 附件 / Ext Attr。  
- **快速動作**：新增 Contact、寫 Activity、開啟 Workflow（若有）。  
- **Workflow / Timeline**：若啟用審批或健康流程，顯示流程圖與歷程。

## Customer Drawer / Full Editor

1. **基本資料**：名稱、客戶編號、類型、Owner、狀態、標籤。  
2. **聯絡資訊**：主聯絡人、電話、Email、網站。  
3. **地址**：Billing / Shipping（與報價/訂單 Drawer 相同結構，支援自動帶入公司名稱/國家）。  
4. **財務**：付款條件、信用額度、幣別。  
5. **Ext Attr**：三欄網格輸入。  
6. **Full Editor**：左右佈局 + Footer（取消 / 儲存 / 送審）。

## 聯絡人 / Activity 整合

- Detail Panel 的聯絡人 Tab：使用 Contact 模組的列表元件（支援搜尋/新增/標星）。  
- Activity Tab：顯示行程/紀錄，支援時間軸與匯入。  
- 可在 Drawer 內直接新增 Contact（子 Drawer）。

## TODO

- [ ] 插入最新 wireframe / Figma 圖。  
- [ ] 確認 Kanban 是否需要；若不需要，移除該選項。  
- [ ] 詳列欄位驗證（稅號格式、Email 驗證、地址自動帶入）。  
- [ ] 若有 Workflow/審批，描述 UI 上的按鈕與提示。  
- [ ] 規劃匯入/匯出功能的入口（在操作列或 Drawer）。*** End Patch
