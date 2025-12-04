# Delivery Module – Functional Spec

> 參考：`docs/flexora-deliverynote-management-spec.md`、Inventory / Sales Order 模組。  
> 目標：統一定義 Delivery Note / 出貨流程與資料，供 UI/後端/串接引用。

## 範圍與關聯

| 範疇 | 說明 |
| --- | --- |
| Delivery Note | `DeliveryNote` 主檔、行項、狀態、追蹤資料 |
| Package / Serial | `DeliveryNotePackage`, `DeliveryNoteItemSerial` |
| Workflow | `DeliveryNoteStateTransitionResource`, `DeliveryNoteStatusHistoryResource`, `DeliveryNoteEventDefResource` |
| Link | `DeliveryNoteSalesOrderLink`：SO ↔ Delivery，亦可連至 Invoice |
| Inventory | 出貨扣庫存、批號/序號紀錄 (`InventoryTransactionResource`) |

## 資料摘要

| 類別 | 欄位示例 | 備註 |
| --- | --- | --- |
| 基本欄位 | deliveryNo, salesOrderId, warehouseId, shippingAddress, shippingMethod, trackingNo | 可連結物流公司 |
| 狀態 | `DRAFT / PACKING / READY_TO_SHIP / SHIPPED / DELIVERED / CANCELLED` | Workflow 需可自訂 |
| 行項 | skuId, description, qtyToShip, qtyShipped, uomId, taxCode | 來源為 Sales Order Item |
| Package | packageNo, weight, dimension, contents | 可能對接外部 WMS |
| Serial / Batch | serialNumber, lotNumber | 針對需追蹤的 SKU |

## 流程摘要

1. **建立 Delivery Note**：由 Sales Order 產生或手動建立；初始為 DRAFT。
2. **打包**：設定 Package、序號、重量；狀態轉為 PACKING → READY_TO_SHIP。
3. **出貨**：呼叫 `ship` / 更新 trackingNo，狀態變 SHIPPED；通知客戶。
4. **送達 / 結案**：可手動/自動更新為 DELIVERED。
5. **取消 / 退貨**：若需作廢或退貨，更新狀態並回寫 Sales Order / Inventory。

## TODO

- [ ] 匯入流程圖（狀態、事件）。  
- [ ] 整理欄位清單與驗證（含包裝/重量/追蹤資料）。  
- [ ] 描述與 WMS/物流服務的資料交換（API / EDI）。  
- [ ] 規劃多階段出貨（部分出貨、多包裹）。  
- [ ] 若需與 Billing 模組串接，補充 Invoice 流程描述。
