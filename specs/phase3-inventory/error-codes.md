# Phase 3 — 錯誤碼（Inventory Error Codes）

本文件列出庫存模組（Inventory）的錯誤碼及對應說明，並同步對照 `InventoryCostingService`、`InventoryReservationService`、`InventoryTransactionService` 的 guard 實作。

| errorKey | 說明 | 對應 service |
|----------|------|------------|
| `inventory.cost.negative_qty` | 當 transaction qty 為負或 direction 不符 | `InventoryTransactionService.validateQuantity` |
| `inventory.cost.no_layer_to_consume` | FIFO SKU 無可消耗的 CostLayer | `InventoryCostingService.consumeFifoLayer` |
| `inventory.cost.layer_mismatch` | CostLayer source 不匹配 transaction | `InventoryCostingService.consumeFifoLayer` |
| `inventory.cost.invalid_unit_cost` | 入庫/調整時未提供 unitCost 且無預設值 | `InventoryCostingService.applyReceipt` |
| `inventory.reservation.insufficient_stock` | 預留時 available qty 不足 | `InventoryReservationService.ensureAvailableQty` |
| `inventory.reservation.invalid_owner` | 資料域內 owner 不可見 | `DataScopePredicateBuilder` / `InventoryReservationService` |
| `inventory.reservation.already_released` | 釋放的 reservation 已處理 | `InventoryReservationService.releaseReservation` |
| `inventory.reservation.already_closed` | 試圖操作已 `CLOSED` 的預留 | `InventoryReservationService` |

> 若新增 error 補 `flexora-react/src/...` 中的 guard 並同步更新此列表與 `docs/specs/error-handling-spec.md`。
