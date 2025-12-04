# Pricing Module – UI Spec

> 目標：讓行銷/產品人員可視化管理 PriceRule、PriceList Assignment，並提供 Trace 工具。

## 介面概觀

- **規則列表 (PriceRule Hub)**：左側 Filter（狀態、通路、客群、品牌），右側列表顯示規則。  
- **動作列**：`新增規則`（Drawer/Form）、`調整優先順序`（拖拉/排序）、搜尋。  
- **Detail Panel**：顯示規則摘要、命中狀況、最近修改。  
- **子頁面**：`PriceList Assignment`、`Pricing Preview / Trace`。

## PriceRule Drawer

1. **基本資訊**：名稱、代碼、狀態、優先順序。  
2. **適用條件**：客戶群、通路、PriceList、ItemGroup、日期，支援多選。  
3. **計算方式**：固定價、百分比折扣、公式（包含欄位）。  
4. **審批**（若有）：顯示審批狀態與按鈕。  
5. **Ext Attr / 註解**。

## PriceList Assignment

- 以 Hub + List 呈現客戶群/通路與 PriceList 的 mapping。  
- Drawer 可設定優先權、有效日期；支援匯入/批次修改。  
- Detail Panel 顯示已套用的客戶 / 地區列表。

## Pricing Preview / Trace

- 輸入欄位：SKU、Qty、Customer、PriceList、Channel。  
- 顯示結果：建議價、折扣細節、命中規則列表（含優先順序）。  
- Trace Viewer：類似樹狀/表格，顯示每個規則的條件、是否命中、計算結果。

## TODO

- [ ] 插入實際 wireframe / Figma。  
- [ ] 列出欄位與驗證（特別是公式編輯器）。  
- [ ] 決定優先順序調整的 UI（拖拉 or 數字）。  
- [ ] Pricing Trace 的視覺化呈現（例如 step log）。  
- [ ] PriceList Assignment 的匯入/匯出流程。
