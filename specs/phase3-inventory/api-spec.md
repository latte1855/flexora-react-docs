# Phase 3 — 庫存管理 API 規格（Inventory API Specification）

本文件定義 Flexora ERP **庫存管理（Inventory Management, IM）** 的 REST API 介面，  
涵蓋：

- 倉庫 / 倉位查詢
- 庫存餘量查詢（InventoryBalance）
- 庫存交易（InventoryTransaction）與調整 / 調撥
- 成本層查詢（CostLayer）與估值
- 預留（InventoryReservation）查詢與操作端點（由 SO/MO 呼叫）
- 補貨建議（ReplenishmentRequest）計算與查詢

> 相關規格請參考：  
> - `domain-model.md`  
> - `costing-rules.md`  
> - `reservation-rules.md`  
> - `replenishment-rules.md`  

---

## 0. 共用原則

### 0.1 Base Path

所有 API 以 `/api` 為前綴，例如：

```text
/api/warehouses
/api/inventory-balances
/api/inventory-transactions
/api/inventory-reservations
/api/replenishment-requests
/api/inventory/cost-layers
/api/inventory/valuation
```

### 0.2 分頁與過濾

* 列表 API 一律支援：

  * `page`（0-based）
  * `size`
  * `sort=field,asc|desc`
* 若採 JHipster Criteria 風格：

  * `itemSkuId.equals=...`
  * `warehouseId.equals=...`
  * `transactionType.in=GR,GI`
  * `createdDate.greaterThanOrEqual=...`

### 0.3 安全與權限

* 一律需登入（JWT）
* 權限建議：

  * `ROLE_INVENTORY_USER`：基本查詢
  * `ROLE_INVENTORY_MANAGER`：調整 / 調撥 / 補貨計算
  * `ROLE_ADMIN`：全部

---

## 1. Warehouse / WarehouseBin API（倉庫 / 倉位）

### 1.1 列出倉庫列表

**`GET /api/warehouses`**

用於前端下拉與管理畫面。

#### Query 參數（可選）

* `active.equals=true|false`
* `type.equals=FINISHED_GOODS,RAW_MATERIAL,IN_TRANSIT...`

#### Response（200 OK）

```json
[
  {
    "id": 1,
    "warehouseCode": "FG-TW",
    "warehouseName": "成品倉（台灣）",
    "type": "FINISHED_GOODS",
    "isVirtual": false,
    "active": true,
    "remark": "總公司成品倉"
  }
]
```

---

### 1.2 取得單一倉庫

**`GET /api/warehouses/{id}`**

回傳單筆 Warehouse DTO。

---

### 1.3 倉位列表（若有啟用 Bin）

**`GET /api/warehouse-bins`**

支援依倉庫過濾：

* `warehouseId.equals=1`

---

## 2. Inventory Balance API（庫存餘量查詢）

### 2.1 列出庫存餘量（依 SKU × Warehouse）

**`GET /api/inventory-balances`**

用途：

* 庫存總覽
* SKU / 倉庫查詢畫面
* 報價 / SO 等模組查詢現況

#### Query 參數（建議）

| 參數                             | 說明                           |
| ------------------------------ | ---------------------------- |
| `itemSkuId.equals`             | 指定某 SKU                      |
| `warehouseId.equals`           | 指定倉庫                         |
| `itemSkuCode.contains`         | 依 SKU code 模糊搜尋（可透過 join 實作） |
| `availableQty.lessThanOrEqual` | 找出快缺貨的 SKU                   |
| `safetyStockBelow.equals=true` | 查詢低於安全庫存者（可用自訂 flag）         |

#### Response（200 OK）

```json
[
  {
    "id": 1001,
    "itemSkuId": 101,
    "itemSkuCode": "SKU-001",
    "itemSkuName": "工業機台 A",
    "warehouseId": 1,
    "warehouseCode": "FG-TW",
    "onHandQty": 50.0,
    "reservedQty": 20.0,
    "availableQty": 30.0,
    "onOrderQty": 40.0,
    "safetyStockQty": 25.0,
    "reorderPointQty": 40.0,
    "maxStockQty": 200.0
  }
]
```

> `availableQty` 可由後端計算後填入 DTO，不一定要存在 DB。

---

### 2.2 取得單一庫存餘量

**`GET /api/inventory-balances/{id}`**

或依組合查詢：

**`GET /api/inventory-balances/by-sku-and-warehouse`**

Query 參數：

* `itemSkuId`
* `warehouseId`

---

### 2.3 查詢某 SKU 的多倉庫分布

**`GET /api/inventory-balances/by-sku/{itemSkuId}`**

回傳所有 Warehouse 的分布，便於 UI 顯示「各倉庫存」。

---

## 3. Inventory Transaction API（庫存交易 / 異動）

> 多數交易由其他模組（PO / DN / MO）觸發，但仍需要：
>
> * 查詢歷史交易
> * 提供「純庫存」的調整 / 調撥 API

### 3.1 查詢庫存交易列表

**`GET /api/inventory-transactions`**

#### Query 參數建議

| 參數                               | 說明                                                    |
| -------------------------------- | ----------------------------------------------------- |
| `itemSkuId.equals`               | 指定 SKU                                                |
| `warehouseId.equals`             | 指定倉庫                                                  |
| `transactionType.in`             | GR, GI, TR_IN, TR_OUT, ADJ_IN, ADJ_OUT, MO_IN, MO_OUT |
| `sourceType.equals`              | PO, SO, DN, MO, ADJ…                                  |
| `sourceId.equals`                | 單據 ID                                                 |
| `createdDate.greaterThanOrEqual` | 起始日期                                                  |
| `createdDate.lessThanOrEqual`    | 結束日期                                                  |

#### Response（200 OK）

```json
[
  {
    "id": 5001,
    "itemSkuId": 101,
    "warehouseId": 1,
    "transactionType": "GR",
    "sourceType": "PO_RECEIPT",
    "sourceId": 9001,
    "quantity": 10.0,
    "beforeQty": 40.0,
    "afterQty": 50.0,
    "unitCost": 50000.0,
    "totalCost": 500000.0,
    "layerId": 3001,
    "createdDate": "2025-11-19T09:00:00Z",
    "createdBy": "buyer1"
  }
]
```

---

### 3.2 手動調整庫存（Adjustment）

**`POST /api/inventory-transactions/adjustment`**

用於盤點 / 修正庫存，不透過 PO / DN。

#### Request Body 範例

```json
{
  "itemSkuId": 101,
  "warehouseId": 1,
  "quantity": -5.0,
  "reason": "盤點差異",
  "adjustmentType": "STOCKTAKE",     // or CORRECTION
  "unitCostOverride": 48000          // optional, for ADJ_IN
}
```

規則：

* `quantity > 0` → ADJ_IN（增加庫存，需建立 CostLayer）
* `quantity < 0` → ADJ_OUT（減少庫存）

錯誤碼：

* `inventory.cost.negative_qty`（方向不符合）
* `inventory.cost.invalid_unit_cost`

#### Response（201 Created）

回傳建立的 InventoryTransaction + 更新後庫存摘要。

---

### 3.3 調撥（Warehouse Transfer）

**`POST /api/inventory-transactions/transfer`**

用於來源倉 → 目的倉的庫存調撥。

#### Request Body 範例

```json
{
  "itemSkuId": 101,
  "sourceWarehouseId": 1,
  "targetWarehouseId": 2,
  "quantity": 8.0,
  "reason": "補足分公司庫存"
}
```

行為：

* 在來源倉建立 `TR_OUT` 交易
* 在目的倉建立 `TR_IN` 交易
* 成本邏輯依 `costing-rules.md`（策略 A：目的倉建立新 CostLayer，成本 = 來源倉平均消耗成本）

錯誤碼：

* `inventory.cost.no_layer_to_consume`（FIFO 下來源倉無庫可用）
* `inventory.transaction.invalid_warehouse`（source == target）

---

## 4. Cost Layer & Valuation API（成本層與估值）

### 4.1 查詢 CostLayer 列表

**`GET /api/inventory/cost-layers`**

#### Query 參數

* `itemSkuId.equals`
* `warehouseId.equals`
* `remainingQty.greaterThan`（僅看尚有餘量的層）
* `sourceType.equals` / `sourceId.equals`

#### Response 範例

```json
[
  {
    "id": 3001,
    "itemSkuId": 101,
    "warehouseId": 1,
    "receivedQty": 10.0,
    "remainingQty": 3.0,
    "unitCost": 50000.0,
    "totalCost": 500000.0,
    "sourceType": "PO_RECEIPT",
    "sourceId": 9001,
    "createdDate": "2025-11-01T08:00:00Z"
  }
]
```

---

### 4.2 即時庫存估值（Valuation）

**`GET /api/inventory/valuation`**

輸出依 SKU × Warehouse 的成本估值，可指定方法：

#### Query 參數

* `method=AVG|FIFO`
* `itemSkuId.equals`
* `warehouseId.equals`

#### Response 範例

```json
[
  {
    "itemSkuId": 101,
    "warehouseId": 1,
    "qtyOnHand": 50.0,
    "valuationMethod": "AVG",
    "unitCost": 48000.0,
    "inventoryValue": 2400000.0
  }
]
```

---

## 5. Inventory Reservation API（預留）

> 實際建立 / 解除 Reservation 多由 Sales / MO 模組呼叫，
> IM 模組提供「預留 Service + REST API」給其他模組使用。

### 5.1 查詢預留列表

**`GET /api/inventory-reservations`**

#### Query 參數（範例）

* `itemSkuId.equals`
* `warehouseId.equals`
* `sourceType.equals=SO_LINE,MO_COMPONENT`
* `sourceId.equals`
* `status.equals=OPEN`

#### Response 範例

```json
[
  {
    "id": 7001,
    "itemSkuId": 101,
    "warehouseId": 1,
    "sourceType": "SO_LINE",
    "sourceId": 12001,
    "reservedQty": 10.0,
    "reservationType": "HARD",
    "status": "OPEN",
    "createdDate": "2025-11-19T08:10:00Z",
    "releasedDate": null
  }
]
```

---

### 5.2 由 SO 建立 / 調整預留

> Endpoint 名稱可依實作調整，下列為建議接口。
> 實際上 Service 會讀取 SO line 的數量，算差量後更新 Reservation。

#### 5.2.1 建立或同步預留（SO）

**`POST /api/inventory-reservations/so-line/{soLineId}/sync`**

行為：

* 讀取對應 `SalesOrderLine`：

  * itemSkuId
  * warehouseId（或預設發貨倉）
  * orderedQty
* 計算與既有 Reservation 差量，新增/更新/刪減 Reservation
* 驗證 `availableQty` 是否足夠

錯誤碼：

* `inventory.reservation.insufficient_available`
* `inventory.reservation.warehouse_required`

#### Request Body（可選）

```json
{
  "reservationType": "HARD"
}
```

#### Response（200 OK）

回傳對應 Reservation DTO（或多筆）。

---

#### 5.2.2 解除預留（SO 取消）

**`POST /api/inventory-reservations/so-line/{soLineId}/release`**

行為：

* 找出該 SO line 的所有 OPEN Reservation
* 將其 status 改為 `RELEASED`，reservedQty 不再計入 `InventoryBalance.reservedQty`

---

### 5.3 由 MO 建立 / 調整預留

類似 SO，提供：

* `POST /api/inventory-reservations/mo-component/{moComponentId}/sync`
* `POST /api/inventory-reservations/mo-component/{moComponentId}/release`

---

### 5.4 全系統重算預留（Rebuild，管理工具）

**`POST /api/inventory-reservations/rebuild`**

用途：

* 修復資料或導入後第一次重建

行為：

1. 將所有現有 Reservation 標記為 `CLOSED` 或 `RELEASED`
2. 掃描所有「有效 SO / MO」重新建立 Reservation
3. 重算每個 SKU × Warehouse 的 `reservedQty`

> 僅允許 `ROLE_INVENTORY_MANAGER` 或 `ROLE_ADMIN` 呼叫。

---

## 6. Replenishment API（補貨建議）

### 6.1 執行補貨計算

**`POST /api/inventory/replenishments/run`**

可由排程或人為觸發。

#### Request Body（可選）

```json
{
  "warehouseId": 1,
  "itemSkuIds": [101, 102],
  "dryRun": true
}
```

* `warehouseId`：若省略則跑所有倉
* `itemSkuIds`：若省略則跑所有 SKU
* `dryRun = true`：只回傳建議，不寫入 DB
* `dryRun = false`：真正建立 `ReplenishmentRequest` 記錄

#### Response（200 OK）

```json
[
  {
    "itemSkuId": 101,
    "warehouseId": 1,
    "onHandQty": 50.0,
    "reservedQty": 40.0,
    "availableQty": 10.0,
    "onOrderQty": 0.0,
    "safetyStockQty": 20.0,
    "reorderPointQty": 30.0,
    "maxStockQty": 200.0,
    "suggestedQty": 150.0,
    "reason": "BELOW_SAFETY",
    "reorderStrategy": "FIXED"
  }
]
```

---

### 6.2 查詢補貨建議列表

**`GET /api/replenishment-requests`**

#### Query 參數

* `status.equals=OPEN,CONFIRMED,CANCELLED`
* `warehouseId.equals`
* `itemSkuId.equals`

#### Response 範例

```json
[
  {
    "id": 8001,
    "itemSkuId": 101,
    "warehouseId": 1,
    "suggestedQty": 150.0,
    "reason": "BELOW_SAFETY",
    "status": "OPEN",
    "createdDate": "2025-11-19T10:00:00Z"
  }
]
```

---

### 6.3 確認 / 取消補貨建議

**`POST /api/replenishment-requests/{id}/confirm`**

* 將狀態改為 `CONFIRMED`
* 可在後續流程轉為 PR / MO

**`POST /api/replenishment-requests/{id}/cancel`**

* 將狀態改為 `CANCELLED`

> 是否在此 API 直接產生 PR / MO，由採購 / 製造模組 spec 決定。

---

## 7. 錯誤處理與錯誤碼

詳細請參考：

* `../phase0-system-foundation/error-handling-spec.md`
* `error-codes.md`（IM 模組專屬）

本文件涉及的關鍵錯誤碼包含（但不限於）：

* `inventory.cost.*`

  * `inventory.cost.no_layer_to_consume`
  * `inventory.cost.invalid_unit_cost`
* `inventory.reservation.*`

  * `inventory.reservation.insufficient_available`
  * `inventory.reservation.invalid_status`
* `inventory.replenishment.*`

  * `inventory.replenishment.invalid_policy`
  * `inventory.replenishment.insufficient_data`

API 返回格式需遵守全域錯誤格式（帶 `errorKey`、`message`、`fieldErrors`）。

---

## 8. 安全與權限（Security）

建議權限區分如下：

| 角色                       | 權限摘要                                   |
| ------------------------ | -------------------------------------- |
| `ROLE_INVENTORY_USER`    | 查詢倉庫、庫存餘量、交易、成本層                       |
| `ROLE_INVENTORY_MANAGER` | 以上全部 + 調整 / 調撥 / Reservation 重建 / 補貨計算 |
| `ROLE_ADMIN`             | 全部操作（含限定管理工具）                          |

Controller 上的 `@PreAuthorize` 範例（概念）：

```java
@PreAuthorize("hasAnyAuthority('ROLE_INVENTORY_USER', 'ROLE_INVENTORY_MANAGER', 'ROLE_ADMIN')")
@GetMapping("/inventory-balances")
public ResponseEntity<List<InventoryBalanceDTO>> getAllBalances(...) { ... }

@PreAuthorize("hasAnyAuthority('ROLE_INVENTORY_MANAGER', 'ROLE_ADMIN')")
@PostMapping("/inventory-transactions/adjustment")
public ResponseEntity<InventoryTransactionDTO> adjust(...) { ... }
```

---

## 9. 文件同步規則

任何以下變更，都必須更新本 `api-spec.md`：

* 新增 / 修改 IM 模組 API 路徑
* 變更 Request / Response DTO 結構
* 修改與 Reservation / Costing / Replenishment 有關的錯誤碼

並同步更新：

* `domain-model.md`
* `costing-rules.md`
* `reservation-rules.md`
* `replenishment-rules.md`
* `error-codes.md`
* `decisions/DECISION_LOG.md`（若屬重要設計決策）

---

## 10. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明               |
| ---- | ---------- | ----- | ---------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立庫存管理 API 規格初稿。 |

