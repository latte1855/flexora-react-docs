# Phase 3 — 庫存整合計畫（Integration Plan）

本計畫依據 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md` 與後端實作（例如 `InventoryTransactionService`, `InventoryCostingService`, `InventoryReservationService`, `CostLayerRepository`, `InventorySettingService` 等）逐步將 legacy 資料拆成正式 spec，並以程式碼為最終依據。請依下列順序與標準完成每個子主題：

## 1. 核心模組拆分順序

1. **Costing & CostLayer (`costing-rules.md`)**
   - 將 legacy 文件中的 `CostLayer` 表格列出所有欄位（`receivedQty`, `remainingQty`, `unitCost`, `totalCost`, `sourceType`, `layerId`）、狀態（`remainingQty=0` 封存）與變更 API。
   - 對照 `InventoryCostingService` 的 `calculateMovingAverage`, `consumeFifoLayer` 邏輯，補充公式與錯誤碼（`inventory.cost.*`）。
2. **Workflow & InventoryTransaction (`workflow-spec.md`)**
   - 根據 `InventoryTransactionService` `postReceipt`, `postIssue`, `confirmTransfer` 等方法整理轉換規則與 Guard。
   - 說明每個 `InventoryTxType` 對應的狀態與 CostLayer 行為，並於 spec 中標註 `Derived from flexora-react/src/...`.
3. **Reservation & ATP (`reservation-rules.md`)**
   - 從 legacy 文檔提取預留類型（Sales/Production/Transfer）、保留規則、Priority，對照 `InventoryReservationService`。
   - 補充 data scope / owner filter 於 reservation API（`InventoryReservationResource`）。
4. **Replenishment (`replenishment-rules.md`)**
   - 根據 `ReplenishmentRequest`、`MO`/`PO` 建議邏輯補充計算公式與欄位。
   - 提供 `InventorySetting` 參數（`defaultValuationMethod`）如何影響 `replenishment`。

## 2. 校對標準

- 每次從 legacy doc 拷貝段落後，務必加上 `Derived from flexora-react/docs/...` 註記，以便追蹤。
- 比對程式碼（service + repository + controller）來確認欄位、API 路徑、Error Key，一旦發現差異就以 code 為權威，並在 spec 加註 `Updated per flexora-react/src/...`.
- 所有 `enum` 必須與資料模型同步，例如 `InventoryTxType`, `ReplenishAction`, `ValuationMethod`。

## 3. 記錄與更新

1. 每完成一節，在 `docs/migration/log.md` 新增條目說明整合內容與參考程式碼路徑。
2. 若遇不確定項（例如某段流程未在程式中出現），保留 `TODO` 並寫入 `legacy-flexora-inventory.md` 供使用者決策。
3. 完成整合後，更新 `docs/specs/phase3-inventory/README.md` 說明此節已依程式碼同步，並列出新增檔案。
