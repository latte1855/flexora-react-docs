# Phase 3 — 庫存管理 Domain Model（Inventory Domain Model）

本文件定義 **Flexora ERP 庫存管理（IM）模組** 中所有核心實體（Entity）  
與其欄位、關聯、生命週期行為。  
此文件為：

- 後端開發（JPA / Liquibase）
- 前端型別（TypeScript Interface）
- Codex / Agent 自動產生 DTO / 測試
- API Spec、Workflow Spec 的來源

的唯一欄位來源。

---

# 1. 整體架構總覽（高階）

庫存管理的主軸由六大核心實體構成：

```

Warehouse
WarehouseBin (optional)
InventoryBalance (StockByWarehouse / StockByBin)
InventoryReservation
InventoryTransaction
CostLayer
InventoryPolicy (Safety Stock / Reorder Point)
ReplenishmentRequest (建議補貨單)

```

以下逐一定義。

---

# 2. Warehouse（倉庫）

倉庫為庫存的主要維度。

## 2.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| warehouseCode | String(20) | ✓ | 倉庫代碼（唯一） |
| warehouseName | String(100) | ✓ | 倉庫名稱 |
| type | Enum | ✓ | 倉庫類型（成品倉、原料倉、退貨倉、在途倉） |
| isVirtual | Boolean | ✓ | 是否為虛擬倉（如在途倉） |
| active | Boolean | ✓ | 是否啟用 |
| remark | String(255) |  | 備註 |

---

# 3. WarehouseBin（倉位，WMS 預留）

## 3.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| warehouseId | Long | ✓ | 所屬倉庫 |
| binCode | String(50) | ✓ | 倉位代碼（唯一） |
| binName | String(100) | ✓ | 倉位名稱 |
| active | Boolean | ✓ | 是否啟用 |

> 若未啟用 WMS，本實體可不使用。

---

# 4. InventoryBalance（庫存餘量）

對應「`ItemSku × Warehouse`」的庫存現況，是 UI 顯示庫存的主要資料來源。

## 4.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| onHandQty | Decimal(18,3) | ✓ | 現有庫存（已收貨且未出庫） |
| reservedQty | Decimal(18,3) | ✓ | 已預留給訂單或製令 |
| availableQty | Decimal(18,3) | ✓ | 可承諾庫存（計算欄位，不直接存 DB） |
| onOrderQty | Decimal(18,3) | ✓ | 在途採購量（PO 未收貨） |
| safetyStockQty | Decimal(18,3) |  | 安全庫存（源自 InventoryPolicy） |
| reorderPointQty | Decimal(18,3) |  | 補貨點 |
| maxStockQty | Decimal(18,3) |  | 最大庫存量 |

- `availableQty = onHandQty - reservedQty`
- 若有 Batch / 序號、Bin 功能，改由 `InventoryBalanceByBin` 管理。

---

# 5. InventoryReservation（庫存預留）

對應各需求來源（Sales Order / MO），預留庫存。

## 5.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| sourceType | Enum | ✓ | 來源（SO、MO、其他） |
| sourceId | Long | ✓ | 對應來源單據 ID |
| reservedQty | Decimal(18,3) | ✓ | 預留數量 |
| reservationType | Enum | ✓ | HARD / SOFT |
| status | Enum | ✓ | OPEN / RELEASED / CLOSED |
| createdDate | datetime | ✓ | 建立時間 |
| releasedDate | datetime |  | 解除預留時間 |

---

# 6. InventoryTransaction（庫存交易）

每一筆進出庫都會產生一筆交易紀錄。  
交易紀錄是系統歷史的重要來源。

## 6.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| transactionType | Enum | ✓ | GR（入庫）、GI（出庫）、TR-IN、TR-OUT、ADJ、MO-IN、MO-OUT… |
| sourceType | Enum | ✓ | 來源單據種類（PO, PRC, SO, DN, MO…） |
| sourceId | Long | ✓ | 來源單據 ID |
| quantity | Decimal(18,3) | ✓ | 正數為入庫、負數為出庫 |
| beforeQty | Decimal(18,3) | ✓ | 異動前庫存量 |
| afterQty | Decimal(18,3) | ✓ | 異動後庫存量 |
| unitCost | Decimal(18,4) | ✓ | 單位成本（由 CostLayer 決定） |
| totalCost | Decimal(18,4) | ✓ | 成本金額 |
| layerId | Long |  | 若使用 FIFO，對應消耗的 CostLayer |

---

# 7. CostLayer（成本層，成本法核心）

每一筆「入庫」都會建立一筆 CostLayer。

## 7.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| receivedQty | Decimal(18,3) | ✓ | 此層入庫量 |
| remainingQty | Decimal(18,3) | ✓ | 剩餘可用量（FIFO 用） |
| unitCost | Decimal(18,4) | ✓ | 單位成本 |
| totalCost | Decimal(18,4) | ✓ | 層總成本 |
| sourceType | Enum | ✓ | PO, MO, ADJ… |
| sourceId | Long | ✓ | 單據 ID |
| createdDate | datetime | ✓ | 入庫時間 |

### 7.2 成本法邏輯（摘要）

- AVG：
  - 異動前後動態更新：  
    `movingAvgCost = (oldQty * oldCost + GR_qty * GR_cost) / (oldQty + GR_qty)`
- FIFO：
  - 出庫時按照層順序消耗 `remainingQty`
  - 儲存 Transaction 上的 `layerId`

> 詳細公式會在 `costing-rules.md` 撰寫。

---

# 8. InventoryPolicy（庫存策略）

每 SKU / Warehouse 可設定庫存政策。

## 8.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| safetyStockQty | Decimal(18,3) | ✓ | 安全庫存 |
| reorderPointQty | Decimal(18,3) | ✓ | 補貨點 |
| maxStockQty | Decimal(18,3) | ✓ | 最大庫存 |
| leadTimeDays | Integer | ✓ | 補貨交期 |
| reorderStrategy | Enum | ✓ | FIXED / EOQ / LAST_X_DAYS / CUSTOM |

---

# 9. ReplenishmentRequest（補貨建議）

系統運算後產生補貨建議，後續可轉為 PR 或 PO。

## 9.1 欄位定義

| 欄位 | 型別 | 必填 | 說明 |
|------|-------|------|------|
| id | Long | ✓ | PK |
| itemSkuId | Long | ✓ | SKU |
| warehouseId | Long | ✓ | 倉庫 |
| suggestedQty | Decimal(18,3) | ✓ | 建議補貨量 |
| reason | String(255) | ✓ | 產生原因（例如：低於安全庫存） |
| status | Enum | ✓ | OPEN / CONFIRMED / CANCELLED |
| createdDate | datetime | ✓ | 建議時間 |

---

# 10. 主要關聯圖（ASCII UML）

```

Warehouse 1 --- * WarehouseBin

Warehouse 1 --- * InventoryBalance --- * ItemSku

Warehouse 1 --- * InventoryReservation --- 1 (sourceType,sourceId)

Warehouse 1 --- * InventoryTransaction --- * ItemSku
|
└---- (optional) CostLayer (for FIFO)

ItemSku 1 --- * CostLayer

ItemSku 1 --- * InventoryPolicy (per warehouse)

ItemSku 1 --- * ReplenishmentRequest

```

---

# 11. 與其他模組連動

| 模組 | 關聯 |
|------|------|
| Phase 1 Pricing | 成本與毛利分析 |
| Phase 4 Quotation | 查詢庫存（僅可參考，不扣庫） |
| Sales Order | 建立 Reservation |
| Delivery Note | 出庫 Transaction |
| Purchase Order / Receipt | 入庫 Transaction + 成本層 |
| Manufacturing（未來） | 投料 GI、完工 GR |

---

# 12. 待定 / 可能擴充（預留）

- 序號管理（Serial Number）
- 批號管理（Batch/Lot）
- 保質期（Expiration）
- 多維度庫存模型（Size/Color/Location）

若導入需同步更新 Domain 與 API Spec。

---

# 13. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立庫存 Domain Model 初稿。 |
