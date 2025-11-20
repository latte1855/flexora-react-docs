# Phase 3 — 庫存預留與 ATP 規則（Reservation & ATP Rules）

本節以 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md` 及後端實作（`InventoryReservationService`, `InventoryReservationResource`, `InventoryTransactionService`, `InventoryBalance`, `ReservationType`）為依據，描述 Reservation 的行為、Priority、錯誤碼與 code reference。任何時候若 spec 與程式碼不一致，以 `flexora-react` code 為最終來源，並在該段最後標記 `Updated per flexora-react/src/main/java/...`.

## 1. 預留分類與行為

| 來源 | ReservationType | 目的 | 關聯 |
| --- | --- | --- | --- |
| Sales Order | `SALES` | 為 Sales/Quotation 保留可用量 | 透過 `InventoryReservationService.reserveForSalesOrder` 建立 `InventoryReservation` 並扣 `InventoryBalance.reservedQty`。 |
| Manufacturing | `PRODUCTION` | MO 投料所需 | `reserveForMo` 會檢查 `InventoryBalance.availableQty`，必要時依 SKU 的 `allowNegativeStock` policy。 |
| Transfer | `TRANSFER` | Transfer Out 需將庫存鎖定 | `reserveForTransfer` 協調 `InventoryTransactionService.transferPending`. |

所有預留皆標記 `status`（`OPEN` / `RELEASED` / `CLOSED`）並保留 `sourceId` 與 `priority`（Priority higher value = earlier release）。

## 2. API 與 Guard（Derived from `InventoryReservationResource` / Service）

1. `POST /api/inventory/reservations`：由 `InventoryReservationResource.createReservation` 接收 `InventoryReservationDTO`，呼叫 `InventoryReservationService.reserve(...)`；若可用量不足會拋 `inventory.reservation.insufficient_stock`。
2. `PUT /api/inventory/reservations/{id}/release`：呼叫 `releaseReservation` 並減少 `InventoryBalance.reservedQty`，若 reservation 已 `CLOSED` 則回 `inventory.reservation.already_closed`。
3. `GET /api/inventory/reservations?itemSkuId=&warehouseId=`：目錄列 API，用於 ATP 查詢，service 會根據 `DataScope` 自動過濾 `owner_id`。

## 3. 錯誤碼對照（同步 `error-codes.md`）

- `inventory.reservation.insufficient_stock`：當 `InventoryBalance.availableQty < qty` 時拋出，`InventoryReservationService.ensureAvailableQty` 會檢查。
- `inventory.reservation.invalid_owner`：當 `InventoryReservation.ownerId` 不在 `DataScope`（`v_owner_visible`）中時跳出，由 `DataScopePredicateBuilder` 於 query 層攔截。
- `inventory.reservation.already_released` / `already_closed`：避免重複操作，由 ReservationService 檢查 `status`.

## 4. code references

- `flexora-react/src/main/java/com/asynctide/flexora/service/inventory/InventoryReservationService.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/web/rest/InventoryReservationResource.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/domain/InventoryReservation.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/domain/InventoryBalance.java`

## 5. 下一步

依計劃請接續補 `replenishment-rules.md`，並在 `docs/specs/phase3-inventory/error-codes.md` 確保 `inventory.reservation.*` 錯誤碼與本節一致。完工後在 `docs/migration/log.md` 記錄進度。
