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

## 與其他模組整合

| 模組 | 整合方式 |
| --- | --- |
| Sales Order | 從 SO 產生 Delivery、同步 shipped qty、更新 SO 狀態與 Timeline |
| Inventory | 出貨觸發 `InventoryTransaction`(ISSUE)、維護序號/批號 |
| Packaging / WMS | 若需外部包裝系統，可透過匯入包裹資訊或 Webhook 更新 tracking |
| Billing / AR | Delivery 完成後可觸發 Invoice 或寄送對帳資料 |
| Notification | 出貨/送達時發送 Email/SMS/Line，並在 Delivery Detail 顯示紀錄 |

## TODO

- [ ] 匯入流程圖（狀態、事件）與狀態圖，清楚標示 `PACKING → READY → SHIPPED → DELIVERED`。  
- [ ] 整理欄位清單與驗證（包裝尺寸、重量、追蹤號、承運商）。  
- [ ] 描述與 WMS/物流服務的資料交換（API / Webhook / EDI）。  
- [ ] 規劃多階段出貨（部分行項出貨、多包裹、合併出貨）。  
- [ ] 若需與 Billing 模組串接，補充 Invoice 流程與對帳欄位。  
- [ ] 補充通知/報表需求（PDF、追蹤連結、客戶通知模板）。  
