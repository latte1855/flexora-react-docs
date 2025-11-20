# 庫存管理（Inventory Management）Legacy 規格參考

摘錄自 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md`，本檔提供高階流程與資料模型的快速回顧，實作仍以 `flexora-react/src/main/java/com/asynctide/flexora/domain/Inventory*`、`service/InventoryTransactionService`、`web/rest/InventoryResource` 等程式碼為準。

## 1. 高階決策要點

| 項目 | 內容 |
| --- | --- |
| **成本法** | Tenant 預設 `MOVING_AVERAGE`，SKU 可覆寫 `FIFO`。每次入庫都產生 `CostLayer`，報表可從 FIFO Layer 或 AVG 計算出庫成本。 |
| **多倉隔離** | 所有庫存與成本以 `ItemSku × Warehouse` 為最小維度，跨倉調撥需重新計算成本。 |
| **數值精度** | 數量 `DECIMAL(19,6)`，金額與成本 `DECIMAL(19,4)`，稅率/折扣 `DECIMAL(7,6)`。與 `phase3` domain model 定義一致。 |
| **流程** | 含 Sales Order Confirm（預留、ReplenishmentRequest）、Partial/Backorder、MO 完工、IQC 轉 AVAILABLE、盤點調整。所有動作均以 `InventoryTransaction` 分錄為唯一權威。 |
| **資料權限** | 支援 Owner/Tenant、Row-level 權限；`InventorySetting` 表處理 tenant-wide 設定（例如 `IM.DEFAULT_VALUATION_METHOD`），與 `flexora-react/domain/InventorySetting.java` 對照。 |

## 2. Enum／狀態摘要

| 枚舉 | 常見值 | 備註 |
| --- | --- | --- |
| `StockStatus` | AVAILABLE / RESERVED / QUARANTINE / HOLD | 透過 `InventoryBalance` 與 `Reservation` 判斷 |
| `ReservationType` | SALES / PRODUCTION / TRANSFER | 預留用途，以 `InventoryReservation` 表區分 |
| `ReplenishAction` | PO / MO / SUGGESTION_ONLY | ReplenishmentRequest 的建議動作 |
| `MoStatus` | DRAFT / PLANNED / RELEASED / IN_PROGRESS / COMPLETED / CANCELLED | 由 MO 制程控制 |
| `InventoryTxType` | RECEIPT / ISSUE / ADJUSTMENT / TRANSFER_IN/OUT / RESERVATION / RELEASE_RESERVATION / COST_ADJUST | 對應 `InventoryTransaction` 的分類 |
| `ValuationMethod` | MOVING_AVERAGE / FIFO / STANDARD | `STANDARD` 預留未實作，`InventorySetting` 控制默認值 |

## 3. 資料模型精要

* `InventoryTransaction`：記錄每筆入出庫與調整，連動 `InventoryBalance`、`CostLayer`、`Reservation`。Field 比對 `flexora-react/domain/InventoryTransaction.java`。
* `CostLayer`：每筆 GR/MO-in/ADJ-in 皆建立一筆，`remainingQty` 作用於 FIFO 出庫；`unitCost` 以 API payload 或 `MovingAverage` 計算。
* `InventorySetting`：Tenant 層設定（Key/Value），可儲存預設估值法、允許負庫等。預留 `effectiveFrom/To` 供 rolling 政策。

## 4. 下一步與待確認

- 請確認 `docs/specs/phase3-inventory/costing-rules.md` 與本 legacy doc 的公式是否一致；若發現差異，應修改 spec 並補充 error code（例如 `inventory.cost.moving_avg_zero_division`）。
- 若需要細節（例如 MO 工作流、IQC Gate），可逐章引用 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md` 並在 `phase3` spec 加註 link。
