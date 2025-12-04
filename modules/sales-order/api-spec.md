# Sales Order Module – API Spec (草稿)

> Controller 參考：`SalesOrderResource`, `SalesOrderItemResource`, `SalesOrderStateTransitionResource`, `SalesOrderQuotationLinkResource`…  
> 目的：整理既有端點與缺口，便於前後端拉齊。

## 1. Sales Order CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/sales-orders` | 列表 / Criteria（customer、owner、status、交期…） | ✅ |
| POST | `/api/sales-orders` | 建立 SO | ✅ |
| GET | `/api/sales-orders/{id}` | 詳細資料 | ✅ |
| PUT/PATCH | `/api/sales-orders/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/sales-orders/{id}` | 軟刪除 | ✅ |

> Criteria 物件可用 `SalesOrderCriteria`，支援 `threadId`, `quotationRevisionId`, `requestedDeliveryDate` 等欄位。

## 2. Transaction / 整單儲存（建議新增）

目前 API 需分多次呼叫（先建立 SO，再建立 Items）。若要仿照報價整單儲存，可新增：

```
POST /api/sales-orders/transactions
{
  "order": { ...SalesOrderDTO... },
  "items": [ ...SalesOrderItemDTO... ],
  "extAttrs": { ... },
  "quotationLink": { "threadId": 1001, "revisionId": 2002 }
}
```

回傳：新 SO + link 資訊。Service 需確保 Items 寫入失敗時 rollback。

## 3. Workflow / State Transition

既有資源：
- `POST /api/sales-orders/{id}/openWorkflow`（送審）
- `POST /api/sales-orders/{id}/approve` / `/reject` / `/cancel`
- `GET /api/sales-orders/{id}/status-history`（`SalesOrderStatusHistoryResource`）
- `GET /api/sales-orders/{id}/state-transitions`（可取得下一步事件）

> UI 的「查看流程圖」可沿用 workflow component；若需要 Graph API，可再討論。

## 4. Sales Order Items

`/api/sales-order-items` 提供 CRUD；建議配合 Transaction API 一併處理，或至少在 UI 中批次提交。

## 5. Lookup / Async Select

尚未提供 `/lookup` 端點。建議新增:

```
GET /api/sales-orders/lookup?keyword=SO-2025&customerId=&status=&limit=20
Response: [{ id, soNo, customer { id, name }, status, grandTotal, requestedDeliveryDate }]
```

供其它模組（出貨、客服）快速查詢 SO。

## 6. SO ↔ Quotation / Delivery Link

- `SalesOrderQuotationLinkResource`：建立 / 刪除 Link。  
- `DeliveryNoteSalesOrderLinkResource`：出貨與 SO 的對應。  
在 API Spec 中提醒前端需依 scenario 呼叫對應資源（例如從 Quote 建立 SO 後要 call link）。

## 7. TODO / 缺口

- [ ] 梳理 `SalesOrderResource` 自訂端點（`openWorkflow` 等）詳細參數與錯誤碼。
- [ ] 決定是否實作 Transaction API（SO + Items + Ext Attr）。
- [ ] Lookup API 需實作（參考 Contact/Owner）。
- [ ] 若需要匯入 / 批次更新交期，定義對應端點與 CSV 格式。
