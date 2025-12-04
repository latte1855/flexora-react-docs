# Purchase Module – API Spec (草稿)

| 資源 | Controller / Path | 備註 |
| --- | --- | --- |
| Rfq | `RfqResource` (`/api/rfqs`) | 標準 CRUD、Criteria |
| Rfq Vendor Quote | `RfqVendorQuoteResource` (`/api/rfq-vendor-quotes`) | 供 vendor 回覆 |
| Purchase Order | `PurchaseOrderResource` (`/api/purchase-orders`) | CRUD + Workflow |
| Purchase Receipt | `PurchaseReceiptResource` (`/api/purchase-receipts`) | 收貨 CRUD |
| Supplier Return | `SupplierReturnResource` (`/api/supplier-returns`) | 退貨流程 |
| Workflow | `PurchaseOrderStateTransitionResource`, `PurchaseOrderStatusHistoryResource`, `PurchaseOrderEventDefResource` 等 | 狀態變更 |

## 1. Purchase Order CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/purchase-orders` | 列表 / Criteria（vendorId、buyerId、status、requestedDeliveryDate…） | ✅ |
| POST | `/api/purchase-orders` | 建立 PO | ✅ |
| GET | `/api/purchase-orders/{id}` | 詳細資料 | ✅ |
| PUT/PATCH | `/api/purchase-orders/{id}` | 更新/部分更新 | ✅ |
| DELETE | `/api/purchase-orders/{id}` | 軟刪除 | ✅ |

## 2. Transaction API（建議）

類似報價整單儲存，可新增：

```
POST /api/purchase-orders/transactions
{
  "order": { ...PurchaseOrderDTO... },
  "items": [ ...PurchaseOrderItemDTO... ],
  "extAttrs": { ... },
  "rfqLink": { "rfqId": 123, "vendorId": 456 }
}
```

確保行項、交期、Ext Attr 可一次寫入，錯誤時 rollback。

## 3. Workflow

常見端點（取自現有 Resource）：

- `POST /api/purchase-orders/{id}/openWorkflow`
- `POST /api/purchase-orders/{id}/approve`
- `POST /api/purchase-orders/{id}/reject`
- `POST /api/purchase-orders/{id}/cancel`
- `GET /api/purchase-orders/{id}/status-history`

也有 `PurchaseOrderEventDefResource` 可查詢事件清單。

## 4. Rfq / Vendor Quote

- `GET /api/rfqs`、`POST /api/rfqs`
- `POST /api/rfqs/{id}/publish`（若有）  
- `RfqVendorQuoteResource`：記錄各 Vendor 報價，可由此建立 PO。

## 5. Receipt / Return

- `PurchaseReceiptResource` (`/api/purchase-receipts`)：收貨 CRUD。  
- `PurchaseReceiptStateTransitionResource`、`PurchaseReceiptStatusHistoryResource`：workflow。  
- `SupplierReturnResource`: 退貨 CRUD；`SupplierReturnStateTransitionResource`：狀態。

## 6. Lookup / Async Select

尚未有 `/lookup`，建議新增：

- `GET /api/purchase-orders/lookup?keyword=&vendorId=&status=&limit=20`
- `GET /api/rfqs/lookup?keyword=&vendorId=&status=&limit=20`

回傳 `id`, `code/no`, `vendor { id, name }`, `status`, `requestedDeliveryDate`, `grandTotal`。

## 7. 匯入 / 批次操作

若需 CSV 匯入 PO 行項或交期，可參考 `ItemPriceMaintenanceResource` 的 `bulk-upsert` 模式：  
`POST /api/purchase-orders/{id}/items:bulk-upsert` (尚未存在，需要新增)。

## 8. TODO

- [ ] 梳理 `PurchaseOrderResource` 中的自訂方法（如 `openWorkflow`, `close`, `reopen`）與錯誤碼。  
- [ ] Transaction API 的 payload/錯誤格式。  
- [ ] Lookup 端點需求。  
- [ ] 若 Receipt/Return 需要掃碼或匯入，補充對應 API。  
- [ ] 與 Inventory/Accounting 的同步端點（例如自動入庫、帳款），後續補入 spec。
