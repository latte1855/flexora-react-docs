# Phase 3 — 庫存流程與狀態（Workflow Specification）

本文件根據 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md` 以及後端實作整理，作為 `InventoryTransaction`/`CostLayer`/`Reservation` 的流程權威。若本 spec 與實作不一致，請以以下程式碼為最終依據，並在章節末補註 `Updated per flexora-react/src/main/java/...`。

## 1. 主要流程與角色

- **Transaction**：入庫/出庫/調整/調撥對應 `InventoryTransactionType`。  
- **Cost Layer**：每筆入庫（GR/MO/ADJ）建立 `CostLayer`，FIFO 出庫依 `InventoryCostingService.consumeFifoLayer` 逐層消耗。  
- **Reservation**：由 `InventoryReservationService.reserve`/`release` 管理 ATP，`InventoryReservationResource` 提供 REST API。  
- **Replenishment**：根據 `InventorySetting` 參數計算 reorder quantity，並產生 `ReplenishmentRequest`（與 PO/MO 建議整合）。

## 2. 主要 Transition（Derived from code）

| Transaction Type | Description | Guard & Service |
| --- | --- | --- |
| `RECEIPT` | PO 收貨 / MO 完工 / ADJ-IN | `InventoryTransactionService.postReceipt` 呼叫 `InventoryCostingService.applyReceipt`，建立 `CostLayer` + update `InventoryBalance`；若 `unitCost` 缺失會拋 `inventory.cost.invalid_unit_cost`。 |
| `ISSUE` | SO 出貨 / GI / MO 投料 | `InventoryTransactionService.postIssue` 先檢查 available，再用 `InventoryCostingService.consumeFifoLayer`; FIFO SKU 若無剩餘會拋 `inventory.cost.no_layer_to_consume`。 |
| `ADJUSTMENT` | ADJ-IN / ADJ-OUT | `InventoryTransactionService.processAdjustment` 匯入 `unitCost`（若有），並同步更新 CostLayer（AVG SKU 僅調整 `InventoryBalance.avgUnitCost`）。 |
| `TRANSFER_OUT/TRANSFER_IN` | 調撥 | 依 `InventoryCostingService.transferCost` 移轉貨值，轉出側會產生 issue、轉入側建立 receipt。 |
| `RESERVATION` / `RELEASE_RESERVATION` | 預留 / 釋放 | `InventoryReservationService` 管理 `InventoryReservation`，並更新 `InventoryBalance.reservedQty`。 |

每個 transition 都會記錄 `InventoryTransaction.status`（`PENDING`→`COMPLETED`）並寫入 `InventoryStatusHistory`（若有）。

## 3. Guard & Error Codes

1. `inventory.cost.negative_qty`：`InventoryTransactionService` 遇到負數 qty 時拋出（`validateQuantity`）。  
2. `inventory.cost.layer_mismatch`：FIFO layer 與 transaction source 不一致時由 `InventoryCostingService.consumeFifoLayer` 拋出。  
3. `inventory.reservation.*`：`InventoryReservationService` 會拋 `inventory.reservation.insufficient_stock`、`inventory.reservation.invalid_owner`，需同步 `error-codes.md` 與 `reservation-rules.md`。  
4. `inventory.replenishment.failed`（預留）：`ReplenishmentService` 在 `calculateReplenishment` 遇資料不足時會回報。

## 4. Code References

- `InventoryTransactionService`（`src/.../service/InventoryTransactionService.java`）  
- `InventoryCostingService`（`src/.../service/inventory/InventoryCostingService.java`）  
- `InventoryReservationService`（`src/.../service/inventory/InventoryReservationService.java`）  
- `InventoryTransactionResource`（`web/rest/InventoryTransactionResource.java`）  
- `InventoryReservationResource`（`web/rest/InventoryReservationResource.java`）

## 5. 下一步

依據 integration plan，接下來要在 `costing-rules.md`、`reservation-rules.md`、`replenishment-rules.md` 補齊欄位與公式，並在 `error-codes.md` 整理對應錯誤碼；所有表格需以上述 service/class 作為最終核對標準。
