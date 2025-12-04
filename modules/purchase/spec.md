# Purchase Module – Functional Spec

> 參考文件：`docs/flexora-purchase-management-spec.md`、`docs/flexora-purchase-management-spec.md`、Inventory 相關規格。  
> 目標：整合 Rfq → Purchase Order → Receipt/Return 的資料模型與流程。

## 範圍與資料層級

| 範疇 | 說明 |
| --- | --- |
| Rfq / Vendor Quote | `RfqResource`, `RfqVendorQuoteResource`：詢價、Vendor 回覆。 |
| Purchase Order (PO) | `PurchaseOrderResource`：主檔、行項、交期、審批。 |
| Purchase Receipt | `PurchaseReceiptResource`：收貨與庫存入帳。 |
| Supplier Return | `SupplierReturnResource`：退貨流程。 |
| Workflow / State | `PurchaseOrderStateTransitionResource`, `PurchaseReceiptStateTransitionResource` 等。 |
| Activity / ExtAttr | 記錄操作、客製欄位。 |

### 主要關聯

- PO ↔ Quote/Rfq（可由 Rfq 生成 PO）。
- PO ↔ Receipt ↔ Inventory transaction。
- PO ↔ Supplier Return。
- Workflow：事件定義 (`PurchaseOrderEventDefResource` 等)、歷史 (`StatusHistoryResource`)。

## 資料摘要

| 類別 | 欄位 (例) | 備註 |
| --- | --- | --- |
| PO 基本 | poNo, vendorId, ownerId, status (`DRAFT/IN_REVIEW/APPROVED/RECEIVING/CLOSED/CANCELLED`) | 需可連結 Rfq、Quote |
| 金額 | currency, exchangeRate, subtotal, taxTotal, grandTotal | 與 PriceList 或 Vendor 合約對應 |
| 交期 | requestedDeliveryDate, promisedDate, shippingAddress | 給倉儲與生產參考 |
| 行項 | itemSkuId, uomId, qtyOrdered, qtyReceived, unitPrice, taxCode | 可核對 Receipt |
| Receipt | receiptNo, warehouseId, receivedQty, status (`RECEIVING/POSTED`) | 對應 Inventory |

## 流程摘要

1. **Rfq → PO**：Rfq 送 Vendor → 接收 Quote → 選 Vendor → 產生 PO。
2. **PO 編輯 / 審批**：草稿 → 送審 (`openWorkflow`) → 核准/拒絕。  
3. **收貨**：建立 Purchase Receipt → 對應行項 → 更新庫存與 PO qtyReceived。  
4. **退貨 / 作廢**：若收貨有問題，透過 Supplier Return 處理。  
5. **結案**：所有行項收貨完成，或手動關閉。

## TODO

- [ ] 匯入 ERD 與流程圖（Rfq、PO、Receipt、Return）。
- [ ] 補齊權限 / 審批說明，尤其針對 PriceList / Vendor 合約。
- [ ] 描述與 Inventory、Accounting 的資料交換與異常處理。
- [ ] 針對匯入（CSV/Excel）或 API 同步需求補充說明。
