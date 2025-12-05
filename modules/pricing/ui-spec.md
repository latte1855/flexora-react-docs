# Pricing Module – UI Spec

> 目標：讓行銷/產品人員可視化管理 PriceRule、PriceList Assignment，並提供 Trace 工具。

## 介面概觀

- **規則列表 (PriceRule Hub)**：左側 Filter（狀態、通路、客群、品牌），右側列表顯示規則。Filter 採 chips + 進階面板，與報價 Hub 一致。  
- **動作列**：`新增規則`（Drawer/Form）、`調整優先順序`（拖拉/排序）、搜尋。  
- **Detail Panel**：顯示規則摘要、命中狀況、最近修改。  
- **子頁面**：`PriceList Assignment`、`Pricing Preview / Trace`。
- **Wireframe**：  
  ```
  [Filter Hub 280px] | [Rule List] | [Detail]
  操作列固定於上方 (新增規則 / 調整優先)
  規則列點擊 → Detail Panel 展開 → Trace Drawer
  ```

## PriceRule Drawer

1. **基本資訊**：名稱、代碼、狀態、優先順序。  
2. **適用條件**：客戶群、通路、PriceList、ItemGroup、日期，支援多選與 `AND/OR`。  
3. **計算方式**：固定價、百分比折扣、公式（包含欄位）。  
   - 公式編輯器提供語法提示與驗證（例如 `price * 0.95`）。  
   - UI：Monaco-like code editor + 資料欄位清單；錯誤時在下方顯示 `line/column` 與 i18n 訊息。  
   - 欄位清單包含：`basePrice`, `unitCost`, `qty`, `customerLevel`, `currencyRate`，點選即可插入程式碼。  
   - 支援 Snippet（固定折扣、成本加成），並顯示最近測試結果。  
   - ASCII：
     ```
     ┌──────────── Formula Editor ─────────────┐
     │ Fields  │ Snippets │  編輯器           │
     │ [basePrice]        │ basePrice * 0.95 │
     │ [qty]              │                  │
     │ ...                │                  │
     │-----------------------------------------│
     │ Error: line 1 col 10 unknown token      │
     └─────────────────────────────────────────┘
     ```
4. **審批**（若有）：顯示審批狀態與按鈕。  
5. **Ext Attr / 註解**。
6. **Trace 預覽**：輸入 SKU + Qty + 客戶後，顯示僅此規則計算結果，方便驗證。
7. **匯入/匯出**：Drawer Footer 提供「下載範本」連結；Import 面板列出必填欄位與錯誤訊息。

## PriceList Assignment

- 以 Hub + List 呈現客戶群/通路與 PriceList 的 mapping。  
- Drawer 可設定優先權、有效日期；支援匯入/批次修改。  
- Detail Panel 顯示已套用的客戶 / 地區列表。

## Pricing Preview / Trace

- 輸入欄位：SKU、Qty、Customer、PriceList、Channel。  
- 顯示結果：建議價、折扣細節、命中規則列表（含優先順序）。  
- Trace Viewer：類似樹狀/表格，顯示每個規則的條件、是否命中、計算結果。高亮目前命中的規則，並提供展開/摺疊。  
- Viewer 需提供「顯示原始 JSON」按鈕，方便開發調試。
- ASCII 佈局：
  ```
  ┌───────────── Trace Viewer ─────────────┐
  │ Input Summary   │ Suggestions          │
  │----------------------------------------│
  │ Rule#1  [命中✓]  條件...   折扣 -20    │
  │   └── Calc: basePrice * 0.8            │
  │ Rule#2  [未命中] 條件...               │
  └────────────────────────────────────────┘
  ```
- Wireframe：`https://figma.com/file/TBD/pricing-trace?node-id=trace-viewer`。
- 支援「儲存 Trace」與「分享 Link」，可供報價/產品人員檢視。  
- 當試算失敗時，在右側顯示錯誤資訊（如缺價目表、規則衝突）。

## TODO

- [ ] 插入實際 wireframe / Figma，含 Pricing Hub、Trace Viewer。  
- [ ] 列出欄位與驗證（特別是公式編輯器、進階條件）。  
- [ ] 決定優先順序調整的 UI（拖拉 or 數字）。  
- [ ] Pricing Trace 的視覺化呈現（例如 step log）。  
- [ ] PriceList Assignment 的匯入/匯出流程。
- [ ] 研究與報價 Drawer 的 Trace 彈窗是否共用。  
