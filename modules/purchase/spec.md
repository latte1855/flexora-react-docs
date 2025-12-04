# Purchase Module – Functional Spec

> 狀態：骨架（2025-12-03），請暫時參考 `docs/flexora-purchase-management-spec.md`。  
> 涵蓋 Rfq、Purchase Order、Receipt、Supplier Return 等流程，未來會搬遷至此。

## 範圍

- **Rfq / Vendor Quote**：詢價、狀態轉換、Vendor 選擇。
- **Purchase Order**：建立、審批、收貨進度、與 Sales/Inventory 的關聯。
- **Purchase Receipt**：收貨記錄、發票、庫存更新。
- **Supplier Return / Cancellation**：退貨、作廢流程。

## TODO

- [ ] 匯入 ERD 與狀態機（Rfq/Purchase Order/Purchase Receipt）。
- [ ] 整理與 Inventory、Accounting 的資料交換。
- [ ] 補上審批/權限說明。
- [ ] 若已有 workflow 圖，轉貼或引用。
