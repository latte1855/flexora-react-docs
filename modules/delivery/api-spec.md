# Delivery Module – API Spec (骨架)

| 資源 | Controller / Path | 備註 |
| --- | --- | --- |
| Delivery Note | `DeliveryNoteResource` (`/api/delivery-notes`) | 標準 CRUD |
| Delivery Items | `DeliveryNoteItemResource`, `DeliveryNoteItemSerialResource`, `DeliveryNoteItemTaxResource` | 行項／序號 |
| Workflow | `DeliveryNoteStateTransitionResource`, `DeliveryNoteStatusHistoryResource`, `DeliveryNoteEventDefResource` | 狀態轉換 |
| Package | `DeliveryNotePackageResource` | 包裝資訊 |
| Link | `DeliveryNoteSalesOrderLinkResource` | SO 與 Delivery 關聯 |

## TODO

- [ ] 梳理常用 API（Create from SO, Post Shipment, Cancel）。
- [ ] 若需要 bulk 打包或掃碼功能，補充 API / payload。
- [ ] Async Select / Lookup（供倉儲系統用）。
