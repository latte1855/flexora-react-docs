# Inventory Module – API Spec

> 依 Phase 3 草案重新整理，列出已存在 vs. 需補強的 REST/API 需求，後續可據此撰寫 OpenAPI 或 Controller。

## 1. Inventory Balance

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `GET /api/inventories` | 查詢 SKU 在各倉庫的 Qty (OnHand/Reserved/Available) | 部分實作於 `InventoryResource` | 需支援 filter（warehouseId、skuId、lot、bin）與 pagination |
| `GET /api/inventories/{id}` | 取得單筆庫存 | 需確認 entity | |
| `GET /api/inventories/lookup` | Async Select（Delivery/SO 使用） | TODO | Response：`skuId, skuName, warehouseId, qtyAvailable` |
| `GET /api/inventories/{id}/ledger` | 交易明細（Receipt/Issue/Transfer） | TODO | 可 reuse `InventoryTransactionResource` |

## 2. Inventory Transaction

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `POST /api/inventory-transactions` | 建立調整/轉移/Issue/Receipt | 有舊版草稿 | 建議以 `type + payload` 統一；回傳 traceNo |
| `PUT /api/inventory-transactions/{id}` | 編輯 | TODO | 視需求限制 |
| `GET /api/inventory-transactions` | 查詢交易 | 有 | 需新增 filter 與匯出 |
| `POST /api/inventory-transactions/transfer` | 倉庫/儲位轉移 | TODO | 支援批次、多 SKU |

## 3. Reservation / Allocation

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `GET /api/inventory-reservations` | 查詢 Reservation | TODO | filter: salesOrderId, projectId, status, dueDate |
| `POST /api/inventory-reservations` | 建立保留 | TODO | 與 Sales Order 對應 |
| `POST /api/inventory-reservations/{id}/release` | 釋放/調整 | TODO | 需紀錄歷史 |

## 4. Stock Count / Adjustment

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `POST /api/stock-counts` | 建立盤點任務 | TODO | 包含任務明細、負責人 |
| `POST /api/stock-counts/{id}/import` | 匯入盤點結果 | TODO | CSV/Excel |
| `POST /api/stock-counts/{id}/submit` | 送審 / 產生 Adjustment | TODO | 需對接 Workflow |

## 5. Replenishment / Task

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `GET /api/replenishment-tasks` | 查詢補貨任務 | TODO | filter: warehouseId, status |
| `POST /api/replenishment-tasks` | 建立/更新建議 | TODO | 排程產生 |
| `POST /api/replenishment-tasks/{id}/convert` | 轉為 Purchase Order / Transfer | TODO | 需與 Purchase 整合 |

## 6. Serial / Lot Management

| API | 說明 | 現況 | 備註 |
| --- | --- | --- | --- |
| `GET /api/inventory-serials` | 查詢序號/批號 | TODO | 支援篩選 |
| `POST /api/inventory-serials/import` | 匯入序號 | TODO | 供 Receipt/Adjustment 使用 |

## 7. Integration Endpoints

- **Delivery**：`POST /api/deliveries/transactions` 內嵌 Inventory Issue；Inventory 模組需提供同步接口或 event。  
- **Purchase**：Receipt/Return 透過 `InventoryTransaction` 產生入庫/退貨。  
- **Manufacturing**：BOM Issue / Receipt → 需復用 `inventory-transactions`。  
- **Reporting**：提供 `GET /api/inventories/export` / `GET /api/inventory-transactions/export`。

## TODO

- [ ] 將既有 Phase3 Controller 與新需求對照，整理實際 route。  
- [ ] 決定是否整合為 Transaction API（類似 Quotation Transaction）以降低多次呼叫。  
- [ ] 設計共用 DTO（`InventoryBalanceDTO`, `InventoryTransactionDTO`, `ReservationDTO`）。  
- [ ] 權限：依 Warehouse/Team 控管，並提供審計日誌。  
- [ ] 定義錯誤碼（例如 `inventory.insufficientQty`, `inventory.serialMismatch`, `inventory.reservationExpired`）。  
- [ ] 若需通知/事件（補貨警示、保留到期），需定義 Webhook 或內部事件。  
- [ ] 匯入/匯出（盤點、序號、調整）所需的 API 與批次狀態查詢。  
