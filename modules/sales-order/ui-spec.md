# Sales Order Module – UI Spec

> UI/UX 原則：完全沿用報價 Workspace 的構圖與互動，降低使用者切換成本。

## Workspace 佈局

```
┌──────── Filter Hub ────────┐┌───────── SO Workspace ───────┐
│Preset：                    ││操作列：快速建立 / 建立訂單      │
│- 我的草稿                   ││View：List / Pipeline*        │
│- 待審核                     ││列表 + Detail Panel           │
│- 本月出貨                   ││                              │
│- 自訂收藏                   │└────────────────────────────┘
└────────────────────────────┘
```

- **Filter Hub**：提供 Preset + 進階篩選（關鍵字、客戶、Owner、狀態、交期區間、PriceList）。
- **List View**：欄位包含 SO 編號、客戶、狀態、交期、金額、Owner。Row 點擊展開右側 Detail。
- **Pipeline View**（可選）：若要追蹤 `DRAFT → IN_REVIEW → APPROVED → FULFILLED`，可與報價的 Pipeline 元件共用；若未啟用可隱藏。

## Detail / Drawer

1. **Detail Panel（右側）**
   - 顯示主要資訊卡：客戶、Owner、金額、交期、PriceList、付款條件。
   - Workflow Buttons：送審、退回、取消、查看Workflow圖。
   - Tabs：摘要 / 行項 / Workflow / 附件 / 關聯記錄（與 Quote Detail 一致）。
2. **SO Drawer（Quick Create / Quick Edit）**
   - 三欄網格：客戶、Owner、付款條件、交期、PriceList、備註。
   - 行項區塊（表格 + Drawer）與報價行項元件共用，支援 SKU Lookup、折扣、稅別。
3. **Full Editor（建立訂單）**
   - 版面與報價整單編輯一致：左側主資訊卡 + 行項卡 + Ext Attr；右側顯示 Workflow/TL。
   - Footer：取消 / 儲存 / 送審。

## 行項與關聯作業

- 可從 Quote 複製行項（在 Drawer/Detail 中提供「複製自報價」按鈕）。
- 行項表格欄位：SKU、描述、Qty、UoM、單價、折扣、稅別、金額。
- 若需要同步出貨狀態，在 Detail Panel 的行項卡顯示 Delivery Note/Receipt。

## Workflow / Timeline UI

- Detail Panel 中加入「查看流程圖」按鈕（與報價一致）。
- Timeline Tab：顯示 `submit/approve/reject/cancel` 紀錄，支援版本篩選（僅此 SO）。

## TODO

- [ ] 置入實際 wireframe / 圖示（可與設計協作）。
- [ ] 明細欄位與驗證規則（含 PriceList/稅別連動）。
- [ ] Pipeline View 是否真的需要 → 待 PM 確認。
- [ ] 與 Delivery / Invoice 的連結（Detail Panel 中顯示快捷連結）。
