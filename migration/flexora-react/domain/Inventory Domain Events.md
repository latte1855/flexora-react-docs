# Flexora Inventory Domain Events

> 版本：2025-11-XX  
> 目的：列出已實作之庫存領域事件（事件名稱、觸發時機、payload），供他模組訂閱或擴充。

| 事件代碼              | 觸發時機 / API                                               | Payload 主要欄位                                                                                                         | 用途 / 訂閱建議                         |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| `inventory.tx.posted` | 任一庫存交易建立（receipt/issue/adjust/transfer/reclassify） | `txId`, `txType`, `skuId`, `warehouseId`, `binId`, `qty`, `occurredAt`                                                   | 供 WMS / 財務模組同步交易、推播審批流程 |
| `inventory.reserved`  | `POST /api/inventory/reservations` 成功                      | `reservationId`, `skuId`, `warehouseId`, `binId`, `qty`, `reservationType`, `referenceType`, `referenceId`, `reservedAt` | CRM / SO 追蹤預留資源；異常提醒         |
| `inventory.released`  | 單筆釋放（含履約）或批次釋放完成                             | `reservationId`, `skuId`, `warehouseId`, `binId`, `qty`, `reason`, `fulfilled(boolean)`, `occurredAt`                    | 回補 SO 狀態、重算可用量或釋放預留鏡像  |
| `valuation.changed`   | 收貨、出庫、調整、成本調整影響估值                           | `skuId`, `warehouseId`, `lotNo`, `occurredAt`                                                                            | 成本報表或財務模組重新載入估值快取      |

## 實作細節

- 事件統一由 `InventoryEventPublisher` 發佈，並於 `InventoryCommandService` 內呼叫。
- Payload 以 Spring ApplicationEvent 形式傳遞，可直接在他模組以 `@EventListener` 訂閱。
- 為了便於 cross-module 重播，可搭配 `inventory.tx.posted` 取得完整交易資訊，再依需求回查 `InventoryTransaction` 表。

> 後續模組若新增新事件（例如 `inventory.replenishment.requested`），請同步更新此檔並在 `docs/api/phase3_im_api.md` / `Flexora ERP API 規格` 內註記。
