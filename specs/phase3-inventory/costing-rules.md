# Phase 3 — 成本估值與成本層規格（Inventory Costing Rules）

本文件定義 Flexora ERP **庫存管理（Inventory Management, IM）** 的  
成本估值方法、成本層（CostLayer）行為、加權平均與 FIFO 的實作規範。  
此文件為：

- 後端成本計算（CostingService）
- 庫存異動（InventoryTransactionService）
- 出庫成本（COGS）
- 成本報表（Inventory Valuation）
- 補貨與採購分析（Replenishment）

的唯一權威規格。

---

# 1. 核心原則（Executive Summary）

1. **Tenant 預設成本法：加權平均（Moving Average）**
2. **SKU 可獨立覆寫成本法（SKU-level override）**
   - 可選：AVG（預設） / FIFO
3. **每一筆入庫 → 永遠建立 CostLayer（成本層）**
   - 即使該 SKU 使用平均法（AVG）
4. **成本層管理以「SKU × Warehouse」維度為最小粒度**
5. **出庫時同步產生 COGS 成本**
   - AVG → 使用 moving average  
   - FIFO → 依層序消耗 remainingQty
6. **所有成本計算由後端進行（前端只顯示）**

---

# 2. 成本法類別（CostingMethod）

| Enum | 意義 |
|------|------|
| `COSTING_AVG` | 加權平均（Moving Average） |
| `COSTING_FIFO` | 先進先出（FIFO） |

> 若未來需要可加入：Standard Cost、LIFO、Actual Batch Cost。

---

# 3. Cost Layer（成本層）基本規範

CostLayer 必須完整記錄一次「入庫」的成本資訊：

```

CostLayer

* itemSkuId
* warehouseId
* receivedQty
* remainingQty
* unitCost
* totalCost
* sourceType
* sourceId
* createdDate

```

### 3.1 每一次入庫（GR / MO-IN / ADJ+）都要建立 CostLayer

包含：

- Purchase Receipt（PO 收貨）
- MO 完工入庫
- 調整增加（ADJ-IN）
- 調撥入（Transfer-IN），視規則決定是否保留原層/建立新層

### 3.2 成本層與程式碼對照（Derived from flexora-react）

| 程式碼位置 | 說明 |
| ---------- | ---- |
| `InventoryCostingService` | `applyReceipt`/`applyIssue`/`applyAdjustment` 會建立並更新 `CostLayer`，移除 FIFO 層時呼叫 `consumeFifoLayer`；所有 AVG 計算透過 `calculateMovingAverage`。 |
| `InventoryTransactionService` | 每筆交易會寫 `InventoryTransaction`，並在 `persistCostLayerForReceipt`、`persistCostLayerForIssue` 等 helper 中維護 CostLayer；Controller 對應 `POST /api/inventory/transactions`。 |
| `CostLayerRepository` | 提供 `findFirstByItemSkuIdAndWarehouseIdAndRemainingQtyGreaterThanOrderByCreatedDateAsc` 作 FIFO 消耗，對照 `InventoryCostingService.consumeFifoLayer`。 |

### 3.3 CostLayer 欄位補充（程式碼為最終權威）

1. `receivedQty` / `remainingQty`：FIFO 出庫後由 `InventoryCostingService.consumeFifoLayer` 扣減，若 `remainingQty = 0` 則標記 `closedDate`（對應 `CostLayer.closedDate` 欄位）。  
2. `unitCost` / `totalCost`：若 API payload 傳入 `unitCost`（PO Receipt、ADJ-IN），以該值；若未提供則由 `InventoryCostingService.calculateMovingAverage` 取 `InventoryBalance.avgUnitCost`。  
3. `sourceType` / `sourceId`：對應 `InventoryTransaction.transactionType` 與 `sourceId`，例如 `PURCHASE_ORDER_RECEIPT`、`MO_FINISHING`、`ADJUSTMENT`；若發現 mismatch 會拋出 `inventory.cost.layer_mismatch`。

以上欄位皆可在 `flexora-react/src/main/java/com/asynctide/flexora/domain/CostLayer.java` 與 `InventoryCostingService` 看到；若後續 service/ entity 有變更，以程式碼為真實來源，並在本節註記 `Updated per flexora-react/src/...`.

### 3.2 remainingQty = receivedQty （入庫後初始）

### 3.3 對於 FIFO SKU

- 出庫會按層消耗 remainingQty  
- 當 remainingQty = 0 → 此層封存

### 3.4 對於 AVG SKU

- 仍會保留 CostLayer（僅作 histórico）
- 不依 CostLayer 計算 COGS，但可用於報表  
- 出庫不會修改 CostLayer.remainingQty

---

# 4. 加權平均成本（Moving Average Cost）規則

### 4.1 平均成本公式

入庫後平均成本更新為：

```

newAvgCost = (oldQty * oldAvgCost + grQty * grUnitCost) / (oldQty + grQty)

```

套用後：

- 更新 `InventoryBalance.avgUnitCost`
- 記錄 cost history（若有 CostHistory）

### 4.2 出庫時成本（COGS）

```

COGS = transactionQty * currentAvgCost

```

- AVG SKU 出庫不會改變 avgCost  
- AVG 成本法對 FIFO 層無依賴性

---

# 5. FIFO 成本規則

對於每一個 SKU × Warehouse：

### 5.1 出庫時逐層消耗

假設需要出庫 18 單位：

| Layer | remainingQty | unitCost | 消耗 |
|-------|--------------|----------|------|
| L1 | 10 | 100 | 全部 10 |
| L2 | 5 | 110 | 全部 5 |
| L3 | 20 | 105 | 3 |
| 合計 | | | 18 |

COGS 計算：

```

COGS = Σ(consumedQty × unitCost)

```

### 5.2 消耗後更新 remainingQty

若 Layer L1 消耗完：

```

remainingQty = 0 → 封存（不可再次使用）

```

### 5.3 FIFO 需要根據 createdDate（或 LayerId）排序

排序規則：

```

createdDate ASC, id ASC

```

---

# 6. 調撥（Transfer）成本規則

調撥需分別處理：

### 6.1 Transfer OUT（來源倉）

- 對來源倉：等同出庫  
- AVG → 使用來源倉平均成本  
- FIFO → 消耗來源倉 CostLayer

### 6.2 Transfer IN（目的倉）

有兩種策略：

---

## 策略 A（建議）— 目的倉「成本維持一致」，直接建立新 CostLayer

```

目的倉 CostLayer.unitCost = 來源倉消耗的層平均成本

```

例：

來源倉出庫（FIFO）消耗：

- 5 units @ 100
- 3 units @ 110

平均消耗成本：

```

(5*100 + 3*110) / 8 = 103.75

```

目的倉入庫：  
→ 建立新 CostLayer：

```

receivedQty = 8
unitCost = 103.75

```

**好處：**  
- 目的倉成本乾淨獨立  
- 不會把來源倉的歷史 FIFO 層塞到目的倉

**大型 ERP（SAP、Oracle、Acumatica）多採用此策略**

---

## 策略 B — 逐層移轉（較複雜，預留）

- 來源倉被消耗的層按照比例複製到目的倉
- 目的倉層會分裂出小層

目前不建議採用，太複雜。

---

# 7. 調整（Adjustment）成本規則

### 7.1 調整增加（ADJ-IN）

- 等同入庫，需建立 CostLayer  
- 若提供單位成本：使用輸入的 unitCost  
- 若未提供：
  - 使用當前 AVG 成本（若 SKU=AVG）
  - 使用 0（若 SKU=FIFO）

### 7.2 調整減少（ADJ-OUT）

- AVG：以 avgCost 計算 COGS  
- FIFO：依照 FIFO 消耗層

---

# 8. 盤點（Stocktake）成本規則

差異量需分成：

- 盤盈 → ADJ-IN  
- 盤虧 → ADJ-OUT  

動作等同 Adjustment 規則。

---

# 9. 成本查詢（Valuation）與報表規則

### 9.1 AVG 視角（Moving Average Report）

- 對每 SKU × Warehouse  
- 計算：

```

qtyOnHand
avgUnitCost
inventoryValue = qtyOnHand * avgUnitCost

```

### 9.2 FIFO 視角（FIFO Valuation Report）

取所有 remainingQty > 0 的 CostLayer：

```

Σ(remainingQty × unitCost)

```

---

# 10. 成本與庫存異動一致性（Atomic Rule）

所有庫存異動需遵守 ACID 原則：

### 10.1 單次出庫（或入庫）動作：

- 建立 / 更新：
  - InventoryBalance  
  - InventoryTransaction  
  - CostLayer（若是入庫）  
  - Reservation（若需）  
- 需要在同一個 Transaction 中執行

### 10.2 若任何步驟失敗 → 全部 rollback

尤其是 FIFO 並行消耗需要保證一致。

---

# 11. API 端的成本行為

> 詳細 API 會在 `api-spec.md` 撰寫，這裡先定義行為。

### 11.1 入庫 API（GR / MO-IN）

- 使用 payload 的 unitCost（若是 PO Receipt）
- 更新 AVG
- 建立新的 CostLayer
- 建立 InventoryTransaction

### 11.2 出庫 API（SO Delivery / GI）

- 依 SKU 成本法決定成本  
- 計算 COGS  
- 建立 InventoryTransaction  
- 若 FIFO → 消耗 CostLayer.remainingQty

### 11.3 查詢成本 API

- 可提供：
  - AVG 視角的成本  
  - FIFO 視角的成本（由 CostLayer 即時計算）

---

# 12. 成本錯誤碼（與 error-codes.md 同步）

| errorKey | 說明 |
|----------|------|
| `inventory.cost.negative_qty` | 異動量不可為負（或不符交易方向） |
| `inventory.cost.no_layer_to_consume` | FIFO 無可用層可消耗 |
| `inventory.cost.layer_mismatch` | 出庫層與來源資料不一致 |
| `inventory.cost.moving_avg_zero_division` | 平均成本除以零錯誤 |
| `inventory.cost.invalid_unit_cost` | 單位成本不正確 |
| `inventory.cost.transfer_calculation_failed` | 調撥成本計算失敗 |

---

# 13. 與其他文件關聯

- `domain-model.md`（所有欄位來源）  
- `reservation-rules.md`（ATP 與預留）  
- `api-spec.md`（入庫 / 出庫 API）  
- `test-strategy.md`（成本法測試案例）  
- Phase 1 Pricing（銷售價格 vs 成本）  

---

# 14. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立庫存成本規則初稿。 |
