# Purchase Module – UI Spec

> UI/UX 原則：與報價、Sales Order 保持一致，採用 Hub + List + Detail + Drawer 的模式。

## Workspace

```
┌──────── Filter Hub ────────┐┌───────── PO Workspace ─────────┐
│Preset: 草稿 / 待審 / 逾期     ││操作列：快速建立 / 建立 PO          │
│Vendor、Buyer、狀態、交期等   ││Tab：PO / Rfq / Receipt          │
└────────────────────────────┘│列表 + Detail Panel               │
                             └──────────────────────────────────┘
```

- **Filter Hub**：提供預設條件與進階篩選（Vendor、Buyer、狀態、交期、物料）。  
- **主區域**：  
  - Tab 切換 `Purchase Order` / `Rfq` / `Receipt`（可依權重調整）。  
  - 操作列：`快速建立 PO`（Drawer）與黑底 `建立 PO`（滿版）。  
  - 視圖：列表為主，Pipeline 視圖視需求保留。
- **Detail Panel**：點擊列表行後在右側展開，顯示 PO 摘要、Workflow 按鈕、附件、Rfq/Receipt 連結。
- **Pipeline 互動**：若 PM 核可保留 Pipeline，欄位為 `DRAFT → RFQ → IN_APPROVAL → APPROVED → RECEIVING → CLOSED`。拖曳時需檢查是否允許跳階，並在 Hub 進階篩選顯示目前 owner。若不採 Pipeline，需在 README 標註「僅 List」。

## Purchase Order – Drawer / Full Editor

1. **基本資訊卡**：Vendor、Buyer、幣別、付款條件、PriceList、預計交期。  
2. **行項表格**：SKU、描述、Qty、單價、交期、已收數量、稅別。可從 Rfq/Quote 複製。  
3. **交期排程**：若單一行有多個交期，在 Drawer 內開分頁編輯。  
4. **Workflow**：送審、退回、取消、查看流程圖（與報價 UI 共用）。  
5. **Ext Attr**：三欄網格輸入（如採購專案編號等）。  
6. **Full Editor**：左右佈局與報價整單編輯一致，下方固定顯示「取消 / 儲存 / 送審」。欄位驗證與 Drawer 相同，但需附加整張表的錯誤提示（如 PriceList 缺少、交期未填）。

### 欄位驗證（示例）

| 欄位 | 規則 | 提示 |
| --- | --- | --- |
| Vendor | 必填；若關聯關閉則顯示 `vendor.inactive` | 紅框 + i18n |
| 付款條件 / 幣別 | 與 Vendor 預設不一致時顯示提示 | Toast + 文案 |
| 交期 | 不得早於今天；可輸入多段交期 | Drawer 錯誤訊息 |
| 行項 Qty | 必須 > 0；允許輸入小數（依 UoM） | inline error |
| 單價 | 必須 >= 0；若 PriceList 提供建議價，差異超過閾值顯示警告 | inline badge |

## Rfq / Vendor Quote

- Rfq Workspace：類似 PO 列表，但顯示詢價狀態、Vendor、回覆期限。  
- Rfq Drawer：輸入需求描述、Vendor 列表、附件；Vendor Quote 子表格顯示各供應商的報價。  
- 操作：從 Rfq 轉 PO → 帶入選定 Vendor 的報價條件。

## Purchase Receipt / Supplier Return

- Receipt Drawer：顯示倉庫、收貨人、行項（收貨數量、批號、序號）；可掃碼或匯入。  
- Return Drawer：選擇原 PO/Receipt 行項、輸入退貨數量與理由。  
- Detail Panel 中顯示與 PO 的連結，以及 workflow 狀態（`RECEIVING/POSTED`）。

### 收貨工作區 / 任務視圖

```
┌──── Receipt Hub ────┐┌──── Receipt List ───┐
│Preset: 待收 / 收貨中 ││列表欄位：Receipt No │
│Warehouse、Buyer etc││PO、倉庫、狀態、收貨者│
└────────────────────┘└─────────────────────┘
```

- 每筆收貨可展開行項，顯示「應收 / 實收 / 差異」。  
- 提供「批次過帳」「列印收貨單」「匯入掃描結果」等操作。  
- Supplier Return 可共用同一視圖，僅切換 Tab 顯示退貨單。

### 進階操作 / 批次

- **匯入**：在行項表格上方提供 `匯入 CSV`，欄位包含 SKU、交期、Qty、單價。匯入後可預覽差異。  
- 匯入格式範例：`skuNo,qty,uomCode,unitPrice,deliveryDate,taxCode`，匯入 API 回傳錯誤列清單。
- **批次審批**：多選 PO 後，顯示「批次送審 / 批次核准」按鈕（需權限）。  
- **提醒 / 通知**：Detail Panel 提供「通知 Vendor」「寄送 PO」等快捷；可整合活動紀錄。  
- **歷史 / 備註**：Detail 中加入 Timeline 卡，列出 Workflow 和手動備註。

## TODO

- [ ] 置入實際 wireframe / Figma 截圖。  
- [ ] 補欄位清單與驗證（尤其價目表、稅別、交期）。  
- [ ] Pipeline 視圖是否必要 → 與 PM 討論後更新，若保留需補拖拉/狀態規則。  
- [ ] 匯入（CSV/Excel）與批次操作在 UI 的流程。  
- [ ] Workflow Timeline（送審、核准、收貨）需與 Detail Panel 整合。
- [ ] Supplier Return 需要額外提醒/通知欄位，與倉儲/財務串接。  
