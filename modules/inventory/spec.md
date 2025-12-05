# Inventory Module – Functional Spec

> 主要參考 `[docs/specs/phase3-inventory/README.md]`、`flexora-inventory-management-spec.md`；此處整理後續規格撰寫骨架，供前後端/流程討論使用。

## 範圍與關聯

| 範疇 | 說明 | 相關模組 |
| --- | --- | --- |
| Warehouse & Bin | 倉庫主檔、儲位層級、權限 | Delivery、Purchase |
| Inventory Balance | Qty On Hand / Reserved / Available、成本 | Sales Order、Pricing |
| Inventory Transaction | Receipt、Issue、Transfer、Adjustment | Purchase、Delivery、Manufacturing |
| Reservation / Allocation | SO / Project 保留、批次釋放 | Sales Order、Quotation |
| Replenishment / Batch Job | Min/Max、建議補貨單、任務 | Purchase、Manufacturing |
| Serial / Lot Tracking | 追蹤序號 / 批號、保固資訊 | Delivery、Service |
| Stock Count | 盤點、循環盤點、差異調整 | Finance、Audit |

## 主要資料表 / 欄位摘要（草稿）

| Entity | 重要欄位 | 備註 |
| --- | --- | --- |
| `Warehouse` | code, name, type, address, manager | 權限依 Warehouse/Team 控管 |
| `Inventory` | warehouseId, skuId, qtyOnHand, qtyReserved, qtyAvailable, cost | Snapshot 表 |
| `InventoryTransaction` | type, referenceType/Id, qty, cost, lot/serial, createdBy | Receipt/Issue/Transfer/Adjust |
| `InventoryReservation` | salesOrderId, lineId, qty, dueDate, status | 釋放/轉移規則 |
| `StockCount` | type, warehouseId, status, scheduledDate | 子表 `StockCountLine` |
| `ReplenishmentTask` | skuId, warehouseId, minQty, maxQty, status | 產出 Purchase/Replenish 單 |

## 核心流程

1. **入庫 / 收貨**：Purchase Receipt → `InventoryTransaction` (RECEIPT) → 更新 Inventory Balance；若有質檢流程，可先進入 `QC` 狀態再入帳。  
2. **出貨 / Issue**：Delivery → `InventoryTransaction` (ISSUE) → Reserved → On Hand；記錄 serial/lot。  
3. **保留 / Allocation**：Sales Order 建立 → 依策略鎖定 qtyReserved；取消或逾期時釋放並通知。  
4. **調整 / Transfer**：手動或批次調整，支援倉庫間/儲位間轉移。  
5. **盤點**：建立 Stock Count → 匯入盤點結果 → 差異轉為 Adjustment Transaction。  
6. **補貨建議**：每日排程計算 On Hand + 在途 + 保留，若低於 Min 則產生 Replenishment Task，對應 Purchase / Manufacturing。  
7. **成本 / FIFO**：可選 FIFO / Average，Transaction 記錄成本來源以供會計對帳；若與 Finance 整合需定義帳務 API。  

## TODO / 待補內容

- [ ] 詳細欄位定義（含 UoM、成本、序號）與驗證規則。  
- [ ] Workflow / 狀態機需求（Stock Count、Replenishment）。  
- [ ] 與 Billing / Finance 的對帳邏輯（GL Posting）。  
- [ ] API 介面整理（`InventoryResource`, `InventoryTransactionResource`, `StockCountResource`, `ReservationResource` 等）。  
- [ ] UI 對應（Workspace、Drawer、盤點工具、補貨任務）。  
- [ ] 匯入工具與權限規則。  
- [ ] 通知 / 事件：補貨警示、保留即將到期提醒、盤點任務提醒。  
- [ ] 若需與外部 WMS 互通，定義資料交換格式與同步策略。  

後續若展開開發，請依此骨架擴充 spec、ui-spec、api-spec，並同步更新 `implementation-plan.md`。  
