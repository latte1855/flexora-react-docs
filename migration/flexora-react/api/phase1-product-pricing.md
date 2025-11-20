# Phase 1 — 商品與定價主檔（Product & Pricing）落地規劃

> 位置：`docs/api/phase1-product-pricing.md`  
> 對齊：`docs/api/MODULES.md` 的 Phase 1（非狀態機階段）  
> 更新：2025-10-15（Asia/Taipei）

本文件是 **模組開發順序的下一步** 之詳細作業指引，明確交付「商品/價格主檔」能力；**狀態機與事件層**屬跨模組基礎，將在 **Phase 4（Quotation）** 首次用到時同步落地，避免前置過早造成複雜度。

---

## 1. 範圍（Scope）

- 單位與換算：`Uom`, `UomConversion`
- 品牌與屬性：`Brand`, `ItemAttribute`, `ItemAttributeValue`
- 產品主檔與群組：`Item`, `ItemGroup`
- SKU 與條碼：`ItemSku`, `ItemBarcode`, `SkuAttribute`
- 價目表與價格：`PriceList`, `ItemPrice`
- ExtAttr 定義：`ItemExtAttrDef`, `ItemSkuExtAttrDef`
- 共用應用服務：`PricingService`（查價，不落地）

> CRUD 由 JHipster 既有 Resource 產生；本階段補上 **規則、查詢 API、快取、驗收測試**。

---

## 2. Domain 規則與資料約束

### 2.1 共同原則

- **Partial Unique + 軟刪**：所有 code/number 唯一鍵皆以 `WHERE deleted=false`。
- **時間區間**：`effective_from <= at < effective_to`（右開區間）。
- **單位換算**：以 **基準單位**（Base UOM）作為唯一換算樞紐，禁止 A↔B 直接互相定義循環。
- **精度**：金額 `DECIMAL(19,4)`；數量 `DECIMAL(19,6)`；折扣/稅率 `DECIMAL(9,6)`。

### 2.2 UoM

- `uom.code` partial unique；`uom_conversion` 以 `(from_uom, to_uom)` 唯一。
- 需提供 **「到基準單位」的有向圖** 檢查，避免循環。

### 2.3 Item / SKU / Barcode

- `item.code`, `item_sku.sku_code` partial unique。
- `item_barcode` 唯一；一個 SKU 可多條碼、條碼唯一對應一個 SKU。
- SKU 為一切交易單位；Item 僅為商品家族與預設屬性容器。

### 2.4 PriceList / ItemPrice

- `price_list.code` partial unique；屬性：幣別、含稅/未稅、客群（可留空）。
- `item_price` 條件：`(price_list_id, sku_id, tier_min_qty, effective_from, effective_to)` 上不可重疊。
- 折扣模式：支援 `AMOUNT`（固定額）與 `RATE`（比例）。
- 價格回傳一律 **未稅** 與 **計算後單價**；是否含稅交由後續稅引擎。

---

## 3. API 設計

### 3.1 查價（查詢，不落地）

```
GET /api/pricing/price?skuId={id}&priceListId={id}&qty={n}&at={iso8601}
```

**Query**

- `skuId` (Long, 必填)
- `priceListId` (Long, 必填)
- `qty` (Decimal, 預設 1)
- `at` (Instant, 預設 now UTC)

**Response**

```json
{
  "skuId": 1001,
  "priceListId": 2001,
  "matchedRuleId": 3001,
  "uom": "EA",
  "basePrice": 120.0,
  "discountType": "RATE",
  "discountValue": 0.05,
  "netUnitPrice": 114.0,
  "currency": "TWD",
  "effectiveFrom": "2025-01-01T00:00:00Z",
  "effectiveTo": "2025-12-31T23:59:59Z",
  "tierMinQty": 10.0
}
```

**邏輯**

1. 找出 `price_list_id` + `sku_id` 下、`effective` 與 `tier_min_qty <= qty` 的所有規則。
2. 以 **最大 `tier_min_qty`** 命中；同層多筆時取 `effective_from` 最近者。
3. 應用折扣後回傳 `netUnitPrice`（未稅）。

**Headers**

- `Idempotency-Key` 非必填（查詢型）；可省略。
- `Accept-Language` 影響錯誤訊息。

### 3.2 價格模擬（批次）

```
POST /api/pricing/preview
```

**Body**

```json
{
  "priceListId": 2001,
  "items": [
    { "skuId": 1001, "qty": 5 },
    { "skuId": 1002, "qty": 12 }
  ],
  "at": "2025-10-15T00:00:00Z"
}
```

**Response**

```json
{
  "totalItems": 2,
  "lines": [
    { "skuId": 1001, "qty": 5, "netUnitPrice": 120.0 },
    { "skuId": 1002, "qty": 12, "netUnitPrice": 114.0 }
  ]
}
```

---

## 4. 應用服務（Service）

### 4.1 `PricingService`

```java
record PriceQuote(
  Long skuId,
  Long priceListId,
  BigDecimal qty,
  Instant at,
  String uom,
  BigDecimal basePrice,
  String discountType,
  BigDecimal discountValue,
  BigDecimal netUnitPrice,
  String currency,
  Instant effectiveFrom,
  Instant effectiveTo,
  BigDecimal tierMinQty
) {}

interface PricingService {
  PriceQuote quote(Long skuId, Long priceListId, BigDecimal qty, Instant at);
  List<PriceQuote> preview(Long priceListId, List<Pair<Long, BigDecimal>> skuQtyList, Instant at);
}

```

### 4.2 快取

- Redis Key：`pricing:{priceListId}:{skuId}:{qtyBucket}:{atBucket}`
  - `qtyBucket`：以 `log2` 或區間切片歸一（避免爆炸）。
  - `atBucket`：依天或小時縮桶。
- 失效策略：當 `item_price` 或 `price_list` 變更，清掉對應 pattern key。

---

## 5. OpenAPI 註解（摘要）

```java
@Operation(summary = "查詢 SKU 單價（未稅）")
@ApiResponses({
  @ApiResponse(responseCode = "200", description = "成功"),
  @ApiResponse(responseCode = "404", description = "找不到匹配的價格規則"),
  @ApiResponse(responseCode = "400", description = "參數錯誤")
})
@GetMapping("/api/pricing/price")
public ResponseEntity<PriceQuoteDTO> getPrice(...)
```

---

## 6. 邏輯與資料完整性檢查

- **UoM 圖形檢查**：新增/更新換算時，檢查是否形成循環。
- **ItemPrice 區間重疊**：在 Repository 層加 Constraint 檢查（或 DB 排他約束 + 友善錯誤）。
- **Barcode 唯一**：DB unique + 友善錯誤訊息（Problem+JSON）。

---

## 7. 測試計畫（Test Plan）

### 7.1 單元測試

- `PricingServiceTest`：
  - 單一價、分級價、多期間、邊界（exact `tier_min_qty`）、無命中（404）。
  - 折扣 `AMOUNT` 與 `RATE`，精度與四捨五入。

### 7.2 整合測試

- `PricingResourceIT`：
  - `/api/pricing/price` 與 `/api/pricing/preview` happy path。
  - Query string 驗證（缺省 `qty`/`at`）。

### 7.3 邊界與例外

- 有效期間重疊 → 422 with violations。
- 條碼重複 → 409 Conflict。
- UoM 形成循環 → 422。

---

## 8. 資料初始化（Seed / Fixtures）

- `price_list`：`STD-RETAIL (TWD, 未稅)`、`STD-B2B (TWD, 未稅)`。
- `item_sku`：`SKU-001`…`SKU-010`。
- `item_price`：每個 SKU 兩個期間（本年/次年），各含兩個 tier（1、10+）。

> CSV 放置 `src/main/resources/config/liquibase/fake-data/`，命名：`price_list.csv`, `item_price.csv`。

---

## 9. 驗收條件（Definition of Done）

- [ ] `/api/pricing/price` 與 `/api/pricing/preview` 可用，Swagger 有範例。
- [ ] `ItemPrice` 期間/階梯不重疊，違規回 422（含 violations）。
- [ ] 條碼唯一；違規回 409。
- [ ] Redis 快取命中率 > 60%（以簡單指標記錄）。
- [ ] 單元 + 整合測試通過；覆蓋率 ≥ 70%。
- [ ] `docs/api/OPENAPI.md` 與 `CONVENTIONS.md` 已補上新端點說明。

---

## 10. 建議 Commit Message

```
feat(product-pricing): add PricingService and price lookup APIs
chore(data): seed price_list and item_price with tiers and periods
test(pricing): unit & integration tests for tiered and dated pricing
docs(api): add phase1-product-pricing spec and openapi examples
```
