# Flexora ERP — Phase 3 庫存核心 API 實作說明（不產生 OpenAPI）

> 版本：2025-10-30（Asia/Taipei）  
> 範圍：Warehouse / Bin / StockByBin / InventoryReservation / InventoryTransaction / CostLayer / Replenishment / Valuation  
> 基底：JHipster v8.11.0（Spring Boot 3.x, Java 21, PostgreSQL, Redis）

---

## 0) Commit 建議序（一步一步可直推）

1. **feat(im): 倉別/倉位 CRUD 與查詢條件**
   - Controller：`WarehouseResource`, `BinResource`
   - 功能：基礎 CRUD + 條件查詢（`warehouseNo/name/enableBinTracking`；Bin 依倉別、binType）
2. **feat(im): 即時庫存查詢 API**
   - `StockAvailabilityResource.getBySkuAndWarehouse`
   - 來源：讀 `StockByBin` 聚合（可視需求 fallback 到 `InventoryTransaction` 彙算）。
3. **feat(im): 庫存預留/釋放 API（含併發保護、冪等標頭）**
   - `InventoryReservationResource.create/release/bulkRelease`
   - 服務層同交易更新 `StockByBin.reserved/available`。
4. **feat(im): 庫存交易過帳 API（收/發/調整；AVG/FIFO 快照）**
   - `InventoryPostingResource.postReceipt/postIssue/postAdjust`
   - 建立 `InventoryTransaction`、必要時寫 `CostLayer`、同步 `StockByBin`。
5. **feat(im): 估值查詢 API（SKU 層）**
   - `ValuationQueryResource.getSkuValuation`：回傳 AVG/FIFO 估值與層摘要。
6. **test(im): 端到端測試矩陣（MockMvc + Testcontainers）**
   - 併發預留、AVG 入出庫、FIFO 跨層耗用、QUARANTINE → PASS。

> 以上每步附對應 Migration（索引/唯一/檢核）與 Redis 快取（`@Cacheable`）微調。

---

## 1) 倉別與儲位（Warehouse / Bin）

### 1.1 路由

- `GET /api/warehouses`：條件查詢（`no`, `name`, `enabled`, `enableBinTracking`）
- `POST /api/warehouses` / `PUT /api/warehouses/{id}` / `PATCH ...` / `DELETE ...`
- `GET /api/bins`：條件查詢（`warehouseId`, `binCode`, `binType`, `enabled`）
- `POST /api/bins` / `PUT /api/bins/{id}` / `DELETE ...`

> 權限：套用 `@Filter(name="ownerFilter")` 的倉別過濾；Bin 查詢必須附 `warehouseId` 或具備倉別可視權。

### 1.2 驗證與商規

- `Warehouse.allowNegative=false` 為預設；若設 `true` 需管理員權限＋稽核。
- `enableBinTracking=true` 時，交易 API 允許指定 `binId`，否則應為 `null`。
- Partial unique：`unique(warehouse_no) WHERE deleted=false`；Bin `(warehouse_id, bin_code)` 唯一。

---

## 2) 即時庫存（Stock Availability）

### 2.1 路由

- `GET /api/stock/available?skuId={id}&warehouseId={id}&binId?=&lotNo?=&serialNo?=`

### 2.2 回應格式

```json
{
  "skuId": 1,
  "warehouseId": 10,
  "binId": null,
  "onHand": "12.000000",
  "reserved": "2.000000",
  "available": "10.000000",
  "status": "AVAILABLE",
  "lastCost": "125.5000",
  "lotNo": null,
  "serialNo": null,
  "asOf": "2025-10-30T06:12:13Z"
}
```

### 2.3 實作要點（Service）

- 以 `(skuId, warehouseId, binId?, lot?, serial?)` 聚合 **`StockByBin`**。
- 若找不到紀錄，回 `available=0`。
- **一致性**：`available = onHand - reserved`，服務層在任何寫入後同步回寫。

---

## 3) 庫存預留（InventoryReservation）

### 3.1 路由

- `POST /api/inventory/reservations`：建立預留（冪等建議：`Idempotency-Key`）
- `POST /api/inventory/reservations/{id}/release`：釋放單筆預留（回沖 reserved）
- `POST /api/inventory/reservations/release`：批次釋放（body: ids[] 或 reference 條件）
- `GET /api/inventory/reservations`：條件查詢（`skuId, warehouseId, reservationType, referenceType, referenceId, activeOnly`）

### 3.2 請求/回應

**建立預留**

```json
{
  "skuId": 1,
  "warehouseId": 10,
  "binId": null,
  "qty": "3.000000",
  "reservationType": "SALES",
  "referenceType": "SALES_ORDER",
  "referenceId": 5001,
  "expiresAt": "2025-11-05T00:00:00Z"
}
```

**回應**

```json
{
  "id": 90001,
  "reserved": "3.000000",
  "availableAfter": "7.000000"
}
```

### 3.3 交易規則（同一個 `@Transactional`）

1. 驗證：`qty > 0`；`binId` 僅在倉位啟用時允許。
2. 鎖定：以 `(skuId, warehouseId)` 範圍 **Select ... for update** 或樂觀鎖重試（建議 3 次）。
3. 檢核：`available >= qty`（或允許負庫覆寫）。
4. 寫入 `InventoryReservation`（`fulfilled=false, cancelled=false`）。
5. 更新 `StockByBin.reserved += qty`、`available = onHand - reserved`。
6. 建立 `InventoryTransaction`（`txType=RESERVATION`, `qty=0` 或僅做審計不動數量；**建議記錄**）。

### 3.4 釋放規則

- 成功釋放：`reservation.cancelled=true` 或完成出貨/領料將 `fulfilled=true`。
- 同步回沖 `StockByBin.reserved`，重算 `available`。
- Idempotent：重複釋放直接回 200（狀態已釋放）。

---

## 4) 庫存交易過帳（Inventory Posting）

> 統一入口：`/api/inventory/transactions`（依場景亦提供語義化端點）。

### 4.1 路由

- `POST /api/inventory/transactions/receipt`（收貨/入庫）
- `POST /api/inventory/transactions/issue`（出庫/領料/出貨）
- `POST /api/inventory/transactions/adjust`（數量調整；高風險需審批）

> Header：`Idempotency-Key`（避免重試重複過帳）。

### 4.2 通用請求欄位

```json
{
  "skuId": 1,
  "warehouseId": 10,
  "binId": null,
  "qty": "5.000000", // 入為正、出為負（issue API 允許傳正數，服務轉為負）
  "unitCost": "120.0000", // 入庫必填；出庫由估值決定
  "referenceType": "PO",
  "referenceId": 70001,
  "note": "GRN-70001"
}
```

### 4.3 過帳規則

- 共同：建立 `InventoryTransaction`（寫入 `stockBefore/After`、`valuationMethodUsed`、`valuationResolvedFrom`）。
- **Receipt**：
  - `qty > 0`，更新 `StockByBin.onHand += qty`；
  - AVG：更新 SKU `averageCost`；FIFO：寫 `CostLayer(qtyReceived, unitCost)`；兩者皆寫層（保留報表）。
  - IQC 若需要：`StockByBin.stockStatus=QUARANTINE`（不入 `available`）。
- **Issue**：
  - `qty < 0`，更新 `StockByBin.onHand += qty`；
  - AVG：`unitCost = currentAvg`；FIFO：依層耗用（最舊優先），寫回 `fifoLayersJson` 或明細表；
  - 禁負庫時阻擋；允許負庫時記錄稽核。
- **Adjust**：
  - 數量方向由 `qty` 符號決定；
  - **成本**：必要時另建 `COST_ADJUST`（不動數量只動單價）。

> 任何寫入後：`reserved` 不變；`available = onHand - reserved` 即時計算寫回。

---

## 5) 估值查詢（Valuation）

### 5.1 路由

- `GET /api/inventory/valuation/by-sku?skuId=&warehouseId=&lotNo?=`

### 5.2 回應

```json
{
  "lastCost": "125.50",
  "averageCost": "118.73"
}
```

> `lastCost` 取最新成本層單價；`averageCost` 取尚未耗盡層（`closed=false`）之 qtyRemaining 加權平均。若查無成本層，兩欄位皆為 `null`。

---

## 6) Controller 與 Service 介面（關鍵方法）

> 下列只列出**新加**或**調整**的方法簽名與要點；CRUD 由 JHipster 產生的 Resource 照常保留。

```java
@RestController
@RequestMapping("/api/stock")
public class StockAvailabilityResource {

  @GetMapping("/available")
  public ResponseEntity<StockAvailabilityDTO> getAvailable(
    @RequestParam Long skuId,
    @RequestParam Long warehouseId,
    @RequestParam(required = false) Long binId,
    @RequestParam(required = false) String lotNo,
    @RequestParam(required = false) String serialNo
  ) {
    /* ... */
  }
}

```

```java
@RestController
@RequestMapping("/api/inventory/reservations")
public class InventoryReservationResource {

  @PostMapping
  public ResponseEntity<ReservationCreateResultDTO> create(
    @RequestBody ReservationCreateDTO dto,
    @RequestHeader(name = "Idempotency-Key", required = false) String idemKey
  ) {
    /* ... */
  }

  @PostMapping("/{id}/release")
  public ResponseEntity<Void> release(@PathVariable Long id, @RequestHeader(name = "Idempotency-Key", required = false) String idemKey) {
    /* ... */
  }
}

```

```java
@RestController
@RequestMapping("/api/inventory/transactions")
public class InventoryPostingResource {

  @PostMapping("/receipt")
  public ResponseEntity<InventoryTransactionDTO> postReceipt(
    @RequestBody ReceiptPostDTO dto,
    @RequestHeader(name = "Idempotency-Key", required = false) String idemKey
  ) {
    /* ... */
  }

  @PostMapping("/issue")
  public ResponseEntity<InventoryTransactionDTO> postIssue(
    @RequestBody IssuePostDTO dto,
    @RequestHeader(name = "Idempotency-Key", required = false) String idemKey
  ) {
    /* ... */
  }

  @PostMapping("/adjust")
  public ResponseEntity<InventoryTransactionDTO> postAdjust(
    @RequestBody AdjustPostDTO dto,
    @RequestHeader(name = "Idempotency-Key", required = false) String idemKey
  ) {
    /* ... */
  }
}

```

```java
@RestController
@RequestMapping("/api/inventory/valuation")
public class ValuationQueryResource {

  @GetMapping("/by-sku")
  public ResponseEntity<SkuValuationDTO> getSkuValuation(
    @RequestParam Long skuId,
    @RequestParam Long warehouseId,
    @RequestParam(required = false) String lotNo
  ) {
    /* ... */
  }
}

```

### 6.1 Service 要點（偽碼）

**預留建立**

```java
@Transactional
public ReservationCreateResultDTO reserve(ReservationCreateDTO dto) {
  lockSkuWarehouse(dto.getSkuId(), dto.getWarehouseId());
  StockByBin s = loadOrInit(dto);
  BigDecimal available = s.getOnHand().subtract(s.getReserved());
  if (!allowNegative(dto) && available.compareTo(dto.getQty()) < 0) throw new BadRequest("insufficient");
  InventoryReservation r = saveReservation(dto);
  s.setReserved(s.getReserved().add(dto.getQty()));
  s.setAvailable(s.getOnHand().subtract(s.getReserved()));
  stockRepo.save(s);
  txRepo.save(auditReservationTx(r));
  return map(r, s.getAvailable());
}

```

**Receipt（含 AVG+FIFO）**

```java
@Transactional
public InventoryTransactionDTO postReceipt(ReceiptPostDTO dto) {
  resolveValuation(dto); // method + source snapshot
  lockSkuWarehouse(dto.getSkuId(), dto.getWarehouseId());
  StockByBin s = findStock(dto);
  s.setOnHand(s.getOnHand().add(dto.getQty()));
  s.setAvailable(s.getOnHand().subtract(s.getReserved()));
  stockRepo.save(s);
  // AVG
  updateAverageCostIfNeeded(dto);
  // CostLayer
  costLayerRepo.save(newLayer(dto));
  return map(txRepo.save(newReceiptTx(dto, s)));
}

```

**Issue（FIFO 耗用）**

```java
@Transactional
public InventoryTransactionDTO postIssue(IssuePostDTO dto) {
  resolveValuation(dto); // AVG or FIFO
  lockSkuWarehouse(dto.getSkuId(), dto.getWarehouseId());
  if (!allowNegative(dto)) ensureEnoughOnHand(dto);
  FifoConsumeResult fifo = fifoService.consume(dto); // 收集耗用層與金額
  applyOnHandDelta(dto.getQty().negate());
  return map(txRepo.save(newIssueTx(dto, fifo)));
}

```

---

## 7) 錯誤碼與訊息（範例）

| HTTP | code                       | 說明                                  |
| ---: | -------------------------- | ------------------------------------- |
|  400 | `im.insufficient`          | 可用量不足                            |
|  400 | `im.bin.disabled`          | 倉位未啟用或不存在                    |
|  409 | `im.idempotent.conflict`   | Idempotency-Key 重複但 payload 不一致 |
|  422 | `im.valuation.unsupported` | 不支援的估值法                        |

---

## 8) 交易一致性與併發

- 所有指令型 API 仍維持單一 `@Transactional` 邊界；`InventoryCommandService` 以 `withStockLock(...)` 封裝悲觀鎖＋最少三次重試，並加入指數退避（25ms 起跳）。
- 庫存寫入一律走 `StockByBinRepository.lockOneForUpdate`，確保 `SKU×Warehouse×Bin×Status` 維度唯一。
- 冪等：所有寫入端點透過 `@IdempotentEndpoint` 切面讀取 `Idempotency-Key`，自動計算 payload hash、鎖定 `(endpoint, method, key)`，成功後留下 `responseStatus/Body`，並在重送時回放舊結果；payload 不一致則拋 `im.idempotent.conflict`。
- Cache：`stock.available.point`、`stock.available.warehouse`、`valuation.sku` 以 Redis/JCache 快取查詢，任何寫入後透過 `InventoryCacheService` 精準失效相關 key。

---

## 9) 測試計畫（MockMvc + Testcontainers）

| 類別              | 內容                                                                                          | 實作                                                                                                                                |
| ----------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 即時可用量        | 收貨→預留→釋放，確認 `available = onHand - reserved`                                          | `InventoryPostingResourceIT`                                                                                                        |
| AVG / FIFO        | 多層入庫＋出庫驗證 AVG 金額與 `fifoLayersJson`（整合測試 `InventoryCommandServiceIT` 已覆蓋） | 既有 `InventoryCommandServiceIT`                                                                                                    |
| 負庫阻擋          | `allowNegative=false` 倉別，出庫超過 onHand 觸發 403                                          | `InventoryPostingResourceIT.issueShouldRespectAllowNegativeFlag`                                                                    |
| QUARANTINE → PASS | 收貨至 QUARANTINE，再呼叫 `/api/inventory/transactions/reclassify`，available 從 0 → 正確數   | `InventoryPostingResourceIT.reclassifyShouldMoveQuarantineStock`                                                                    |
| Idempotency       | `Receipt`、`Reservation` 具相同 `Idempotency-Key` 時，僅第一次產生交易並重播舊結果            | `InventoryPostingResourceIT.receiptShouldBeIdempotent`、`InventoryReservationCommandResourceIT.createReservationShouldBeIdempotent` |

MockMvc 測試皆搭配 Testcontainers PostgreSQL / Redis（`@IntegrationTest`＋`@AutoConfigureMockMvc`），確保與正式環境一致。

---

## 10) 參考 DTO（精簡）

```java
public record ReservationCreateDTO(
  Long skuId,
  Long warehouseId,
  Long binId,
  BigDecimal qty,
  ReservationType reservationType,
  InventoryReferenceType referenceType,
  Long referenceId,
  Instant expiresAt
) {}

public record ReceiptPostDTO(
  Long skuId,
  Long warehouseId,
  Long binId,
  BigDecimal qty,
  BigDecimal unitCost,
  InventoryReferenceType referenceType,
  Long referenceId,
  String note
) {}

public record IssuePostDTO(
  Long skuId,
  Long warehouseId,
  Long binId,
  BigDecimal qty,
  InventoryReferenceType referenceType,
  Long referenceId,
  String note
) {}

public record AdjustPostDTO(Long skuId, Long warehouseId, Long binId, BigDecimal qty, BigDecimal unitCost, String note) {}

```

---

## 11) Liquibase 提示（索引/唯一）

- `stock_by_bin (sku_id, warehouse_id, bin_id)` 索引。
- `bin unique (warehouse_id, bin_code) WHERE deleted=false`。
- `warehouse unique (warehouse_no) WHERE deleted=false`。
- `cost_layer (sku_id, warehouse_id, created_at)`、`(closed, created_at)` 索引。
- 新增 `inventory_transaction.tx_type` 索引用於快取/事件訂閱，加速 `inventory.tx.posted` 監聽。

---

## 12) 事件與快取

- 事件：`inventory.reserved`（建立預留）、`inventory.released`（釋放 / 履約）、`inventory.tx.posted`（所有調帳交易）、`valuation.changed`（收/發/調整影響估值）。
- 所有事件由 `InventoryEventPublisher` 發佈，payload 與觸發時機詳見《docs/domain/Inventory Domain Events.md》。
- 快取：`StockAvailabilityService` 與 `ValuationService` 以 Redis/JCache 提供短暫快取，對應 key 分別為 `stock.available.point`、`stock.available.warehouse`、`valuation.sku`；任一寫入 API 完成後由 `InventoryCacheService` 精準 `@CacheEvict`。

---

## 13) 範例 Curl（片段）

```bash
# 查可用量
curl -s "http://localhost:8080/api/stock/available?skuId=1&warehouseId=10"

# 建立預留（冪等）
curl -X POST http://localhost:8080/api/inventory/reservations \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: so-5001-l1' \
  -d '{"skuId":1,"warehouseId":10,"qty":"3.000000","reservationType":"SALES","referenceType":"SALES_ORDER","referenceId":5001}'

# 收貨
curl -X POST http://localhost:8080/api/inventory/transactions/receipt \
  -H 'Content-Type: application/json' -H 'Idempotency-Key: grn-70001' \
  -d '{"skuId":1,"warehouseId":10,"qty":"5.000000","unitCost":"120.0000","referenceType":"PO","referenceId":70001}'
```

---

> **備註**：未納入 OpenAPI 輸出；若後續需要，建議以 Springdoc 直接從 Controller 註解產生 YAML。
