# Phase 3 — 補貨建議與 Replenishment 規則

本節以 legacy document (`docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md`) 以及 `InventoryReplenishmentService` / `ReplenishmentRequest` / `InventoryPolicy` 的實作為主，描述 `ReplenishmentRequest` 的產生邏輯、參數與 API，並以程式碼為最終依據。

## 1. 主要資料模型

| 資料表 | 核心欄位 | 說明 |
| --- | --- | --- |
| `inventory_policy` | `item_sku_id`, `warehouse_id`, `safety_stock_qty`, `reorder_point_qty`, `max_stock_qty`, `lead_time_days` | 定義每個 SKU × 倉庫的策略。 |
| `replenishment_request` | `item_sku_id`, `warehouse_id`, `suggested_qty`, `reason`, `status` | 系統計算出的補貨建議，`status` 由 `PENDING`、`CONFIRMED`、`PO_CREATED` 等 state 組成。 |

以上欄位可在 `flexora-react/src/main/java/com/asynctide/flexora/domain/InventoryPolicy.java` 和 `ReplenishmentRequest.java` 找到，並被 `InventoryReplenishmentService` 讀取。

## 2. 補貨邏輯（Implementation-aligned）

1. `InventoryReplenishmentService.calculateReplenishment()` 會依據：
   * `InventoryBalance.availableQty` （搭配 `InventoryBalanceService`）  
   * `InventoryPolicy.safety_stock_qty` / `reorder_point_qty`  
   * `OnOrder` / `Reserved` 數值  
   * `InventorySetting.defaultLeadTimeDays` 或 `InventoryPolicy.lead_time_days`
2. 產出 `ReplenishmentRequest` 並寫入 `ReplenishmentRequestRepository`; 每筆記錄含 `reason`（如「低於安全庫存」）與 `suggested_qty`。
3. 若 `ItemSku.allowNegativeStock` 為 true，Service 仍會提示並可產生 `ReplenishmentRequest` 但不會強制 prevent `availableQty` < 0。

## 3. API 與參數

| Endpoint | 描述 | 備註 |
| --- | --- | --- |
| `POST /api/replenishment/requests/calculate` | 重新計算補貨建議（可傳 `itemSkuId`, `warehouseId`, `force=true`） | 呼叫 `InventoryReplenishmentService.calculateReplenishment`. |
| `GET /api/replenishment/requests` | 查詢 pending/confirmed 建議 | 支援 `status`, `itemSkuId`, `warehouseId` 過濾。 |
| `PUT /api/replenishment/requests/{id}/confirm` | 將建議標記 `CONFIRMED` 並可觸發 PO/MO | 具體轉單邏輯交由 Sales/Purchase 模組。 |

以上 API 在 `InventoryReplenishmentResource`（`web/rest`）內實作，會自動觸發 `DataScope` 過濾。

## 4. 參考服務與 Code Link

- `flexora-react/src/main/java/com/asynctide/flexora/service/inventory/InventoryReplenishmentService.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/web/rest/InventoryReplenishmentResource.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/domain/ReplenishmentRequest.java`

## 5. 進度紀錄

依循 integration plan，補完 Phase 3 的 costing/workflow/reservation/replenishment 四個子章節後，請在 README、新增 `Derived from ...` 註記並更新 `docs/migration/log.md`，完成第一波整合後再進行 Phase 4。
