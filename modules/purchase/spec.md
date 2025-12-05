# Purchase Module – Functional Spec

> 參考：`docs/flexora-purchase-management-spec.md`、Inventory / Supplier 模組。  
> 目標：整合 Rfq → Vendor Quote → Purchase Order → Receipt / Return / Invoice 的資料模型、流程與權限。

## 範圍與資料層級

| 範疇 | 說明 | 主要 API / Entity |
| --- | --- | --- |
| Rfq / Vendor Quote | 需求彙整、Vendor 報價、比較決策 | `RfqResource`, `RfqVendorQuoteResource` |
| Purchase Order (PO) | 主檔、行項、交期、審批、附件 | `PurchaseOrderResource`, `PurchaseOrderItemResource` |
| Purchase Receipt / GRN | 收貨、入庫、品質檢驗、差異處理 | `PurchaseReceiptResource`, `PurchaseReceiptItemResource` |
| Supplier Return | 供應商退貨、借出歸還 | `SupplierReturnResource` |
| Payment / Invoice Link | 與 AP Invoice、財務對帳資料 | `PurchasePaymentResource` (若有) |
| Workflow / State | 事件、狀態、歷史紀錄 | `PurchaseOrderStateTransitionResource`, `PurchaseReceiptStateTransitionResource` |
| Activity / Ext Attr | 操作紀錄、客製欄位 | ActivityLog, ExtAttr 表 |

## 主要關聯

- RFQ 可轉換為 Vendor Quote，再轉為 Purchase Order。  
- PO 與 Receipt、Supplier Return 構成 1:N 關係；Receipt 與 `InventoryTransaction` 連動。  
- PO 與 Invoice / Payment 具一或多對多關係；需記錄對帳狀態。  
- Workflow 需要事件 (`submit`, `approve`, `cancel`, `receive`, `close`)，並在 History 中保留軌跡。

## 資料摘要（草稿）

| 類別 | 欄位 (例) | 備註 |
| --- | --- | --- |
| PO 基本 | poNo, vendorId, ownerId, requesterId, status (`DRAFT/IN_REVIEW/APPROVED/RECEIVING/CLOSED/CANCELLED`), vendorContactId | 支援多幣別、合約 |
| 金額 | currency, exchangeRate, subtotal, taxTotal, discount, grandTotal | 可引用 PriceList / Vendor 合約價格 |
| 交期 / 地址 | requestedDeliveryDate, promisedDate, shippingAddress, billingAddress, incoterm | 提供倉儲/物流參考 |
| 行項 | itemSkuId, description, uomId, qtyOrdered, qtyReceived, unitPrice, taxCode, projectCode | 與 Receipt 行項核對 |
| Receipt | receiptNo, warehouseId, receivedQty, inspectedQty, status (`RECEIVING/POSTED/RETURNED`), attachments | 可能觸發 Inventory / QA |
| Return | returnNo, reason, qty, referenceReceiptId | 退貨時更新庫存與 PO qtyReturned |
| 權限 / Ext Attr | owner/team、可自訂欄位（合約編號、付款條件等） | 需審批的欄位標註 |

## 核心流程

1. **Rfq 建立** → 指派供應商 → 收集 Vendor Quote → 選擇供應商與條件。  
2. **Purchase Order 草稿** → 填寫行項、交期、金額 → 送審。  
3. **審批**：依角色／金額門檻核准；拒絕退回草稿。  
4. **收貨 / Receipt**：依 PO 行項建立收貨單，輸入實收數量、檢驗結果、倉庫；產生 Inventory Transaction。  
5. **差異 / 退貨**：若有短少或品質問題，建立 Supplier Return；更新 PO / Inventory。  
6. **發票 / 付款**：與 AP 模組連動（可在 PO 內顯示付款狀態）。  
7. **結案**：所有行項收貨完成或手動關閉；保留 Workflow/History 在 Timeline。  

## 欄位與驗證（示例）

| 類別 | 驗證重點 |
| --- | --- |
| Vendor / Buyer | vendorId 必填、buyerId 需在授權清單內；付款條件、幣別需與 Vendor 合約吻合 |
| 行項 | `qtyOrdered > 0`、`uom` 與 SKU 相容、`qtyReceived <= qtyOrdered`、稅別/PriceList 需匹配 |
| 交期 / 地址 | `requestedDeliveryDate >= orderDate`; 多交期行需逐行檢查；地址沿用 AddressSnapshot 結構 |
| Workflow 欄位 | 審批時僅允許指定欄位可編輯，並在 Timeline 顯示原因 |
| Receipt | 需輸入倉庫、收貨人；若序號/批號必填，未提供不得過帳 |

## 匯入 / 批次流程

- **Rfq / PO 匯入**：支援 CSV/Excel；欄位包含 `vendorCode, skuNo, qty, unitPrice, deliveryDate` 等。匯入前提供預覽，錯誤行需註明原因。  
- **Receipt 匯入／掃碼**：可上傳 CSV 或以掃碼方式輸入實收數量、批號、序號；支援即時驗證。  
- **匯出**：列表可匯出 PO/Rfq/Receipt，方便供應商下載或內部對帳。  
- **批次送審 / 批次核准**：在 Workspace 多選後可一次送審或核准（需權限）。  

## TODO

- [ ] 匯入 ERD 與流程圖（RFQ、PO、Receipt、Return、Invoice）。  
- [ ] 補齊權限 / 審批說明：金額門檻、Owner/Team、敏感欄位遮罩。  
- [ ] 與 Inventory / Accounting 的資料交換、異常處理、同步策略。  
- [ ] 匯入/同步（CSV/Excel、ERP）規格與錯誤處理。  
- [ ] UI 與 API 的欄位清單、驗證規則、Workflow 事件參數。  
- [ ] 若需 PriceList / Contract 整合，補充關聯與查價邏輯。  
