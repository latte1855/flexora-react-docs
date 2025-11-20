# Legacy Inventory Domain Events

本文整理 `docs/migration/flexora-react/domain/Inventory Domain Events.md` 所列的事件，並提醒使用者可直接參照 `flexora-react/src/main/java/com/asynctide/flexora/event/InventoryEventPublisher` 與 `InventoryCommandService` 的實作。

| 事件 | 說明 | 典型用途 |
| --- | --- | --- |
| `inventory.tx.posted` | 任一庫存交易（Receipt/Issue/Adjustment/Transfer/Reclassify）建立時發佈，payload 帶 txId/sku/warehouse/qty | WMS、財務、報表系統可訂閱並匯入交易 |
| `inventory.reserved` / `inventory.released` | 庫存預留建立與釋放（含履約）時發佈，payload 含 reservationId、qty、reference | CRM/Sales 可更新預留狀態、UI 重新計算 available |
| `valuation.changed` | 影響估值（收貨/出貨/調整/成本調整）時触发，payload 包含 sku/warehouse/lot | 成本報表或財務模組重新計算 current valuation |

## 實作提醒

1. 事件由 `InventoryEventPublisher` 發佈（位於 `service/events`），`InventoryCommandService` 呼叫並帶入 `InventoryTransaction` 資料。
2. 若新增事件（如 `inventory.replenishment.requested`），請同步：legacy doc → `docs/specs/phase3-inventory` → `docs/api-spec/legacy...`。
