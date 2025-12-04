# Delivery Module – API Spec (草稿)

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| Delivery Note | `DeliveryNoteResource` (`/api/delivery-notes`) | CRUD + Criteria |
| Delivery Items | `DeliveryNoteItemResource`, `DeliveryNoteItemSerialResource`, `DeliveryNoteItemTaxResource` | 行項/序號/稅別 |
| Package | `DeliveryNotePackageResource` | 包裝資訊 |
| Workflow | `DeliveryNoteStateTransitionResource`, `DeliveryNoteStatusHistoryResource`, `DeliveryNoteEventDefResource` | 狀態轉換 / 歷史 |
| Link | `DeliveryNoteSalesOrderLinkResource` | SO ↔ Delivery |

## 1. Delivery Note CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/delivery-notes` | 列表，支援 Criteria（salesOrderId, status, warehouseId, shippingDate...） | ✅ |
| POST | `/api/delivery-notes` | 建立 Delivery Note | ✅ |
| GET | `/api/delivery-notes/{id}` | 詳細資料 | ✅ |
| PUT/PATCH | `/api/delivery-notes/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/delivery-notes/{id}` | 軟刪除 | ✅ |

## 2. Transaction / 整單儲存（建議）

可仿照報價整單儲存，新增：

```
POST /api/delivery-notes/transactions
{
  "delivery": { ...DeliveryNoteDTO... },
  "items": [ ...DeliveryNoteItemDTO... ],
  "packages": [ ...DeliveryNotePackageDTO... ],
  "serials": [ ...DeliveryNoteItemSerialDTO... ],
  "links": { "salesOrderId": 1001 }
}
```

一次寫入 Delivery + items + packages，並處理 rollback。

## 3. Workflow

可用事件（取自 `DeliveryNoteEventDef`）：`PACK`, `READY`, `SHIP`, `CONFIRM_DELIVERY`, `CANCEL` 等。  
常用端點：

- `POST /api/delivery-notes/{id}/openWorkflow`（建議）  
- `POST /api/delivery-notes/{id}/ship` → 轉 SHIPPED  
- `POST /api/delivery-notes/{id}/cancel`
- `GET /api/delivery-notes/{id}/status-history`

（實際方法需檢查 Controller，若尚未有 `ship` 端點則需新增。）

## 4. Package / Serial

- `DeliveryNotePackageResource`：`GET/POST/PUT/DELETE /api/delivery-note-packages`。  
- `DeliveryNoteItemSerialResource`：記錄序號/批號。  
- 若需 bulk 匯入序號，建議新增 `POST /api/delivery-notes/{id}/serials:bulk-upsert` 等端點。

## 5. Lookup

尚未提供 `/lookup`，建議加入：

```
GET /api/delivery-notes/lookup?keyword=DN-2025&salesOrderId=&status=&limit=20
```

回傳 `id`, `deliveryNo`, `salesOrder { id, soNo }`, `status`, `shippingDate`, `trackingNo`.

## 6. Create-from-SO / 快速生成

- 若 UI 需要從 Sales Order 直接產生 Delivery，可新增 `/api/delivery-notes/from-sales-order` 之類端點，或在 Transaction API 中接收 `salesOrderId`，後端自動帶入行項。

## 7. TODO

- [ ] 確認現有 Controller 是否已有 `ship`, `confirmDelivery` 等操作，若無需新增。  
- [ ] 定義 Transaction API payload 與錯誤處理。  
- [ ] Bulk 操作（序號匯入、包裝匯入）需求。  
- [ ] 與 WMS/外部物流的同步接口（若有）。  
- [ ] 若 Delivery 會產生 Invoice/成本，需記錄對應 API。
