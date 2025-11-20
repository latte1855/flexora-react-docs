# Phase 1 — 商品與定價模組概覽（Product & Pricing Overview）

本文件以 `docs/specs/phase1-product-pricing/legacy-flexora-product-management.md` 與 `flexora-react` 的 domain/Service 實作為依據（例如 `com.asynctide.flexora.domain.Item*`、`PriceListService`、`BomService`），說明 Phase 1 的範圍、核心決策、資料模型以及與其他模組的協作方式。

## 1. 目標與邊界

1. **SKU-first**：所有交易行（報價/採購/銷售）皆指向 `ItemSku`，即便一款商品只有單一變體也需建立至少一個 SKU，以利條碼、庫存、定價與報表一致。
2. **價目表與價格有效期間**：`PriceList` + `ItemPrice` 組成「PriceListEntry」，可依日期區間（`startDate`/`endDate`）與 `owner` 分流。
3. **Sales Bundle 與 Manufacturing BOM 分離**：前者（`ProductBundleItem`）應用於銷售打包，後者（`Bom`/`BomItem`）提供製造用料階層；兩者不可混用。
4. **Routing / WorkCenter**：製程版本（`Routing`）與工序（`RoutingOp`）定義加工流程，工序綁定 `WorkCenter`，用於計算 lead time、MO 預估時間。
5. **ExtAttr**：每個模組（Item / SKU / BOM 等）皆有 `AttrDef`/`AttrValue`，透過 `owner_id + attr_code` partial unique 控制可見性、並提供 API `GET /api/items/{id}/ext-attrs` 及 `PUT /api/items/{id}/ext-attrs/{code}`。

## 2. 資料模型精華

### 2.1 主要 Entity

| 實體 | 關聯 | 說明 |
| --- | --- | --- |
| `Item` | `ItemSku` | 商品 SPU（母體），儲存 `item_code`、`brand`、`category`、`description`。 |
| `ItemSku` | `Item`, `Uom`, `ItemBarcode` | 可交易最小單位，包含 `sku_code`、`defaultUom`、`taxType`、`priceList` 快照。 |
| `ProductBundleItem` | `ItemSku` | 組合商品，定義 bundle 與 component SKU + 數量。 |
| `Bom` / `BomItem` | `ItemSku` | 製造用 BOM，每版可搭配 `effectiveFrom/To` 與 `Active` flag。 |
| `Routing` / `RoutingOp` | `WorkCenter` | 製程版本與工序，提供 `opSequence`、`standardTime`、`setupTime`。 |
| `PriceList` / `ItemPrice` | `ItemSku` | 價目表 + 該 SKU 的特定期間價格、有效幣別、折扣策略。 |

### 2.2 ExtAttr 機制

1. 每個模組自有 `AttrDef`、`AttrValue`（例如 `ItemAttrDef`/`ItemAttrValue`、`ItemSkuAttrDef`/`ItemSkuAttrValue`）。
2. `AttrValue` 透過資料型別欄位（`value_string`、`value_number` 等）存值，使用 `owner_id` + `attr_code` partial unique（`WHERE deleted=false`）來確保租戶隔離。
3. API 由 `ItemResource` `GET/PUT /ext-attrs` 實作，前端可依此動態擴充欄位。

## 3. 與其他模組協作

- **Phase 0 System Foundation**：共用 `Owner`/`User`/`Team`、`DataScope` 以及 `Numbering`（`ItemSku`、`PriceList` 等需帶 `owner_id` 與 `ownerScope`）。
- **Phase 3 Inventory**：商品 SKU 與定價會被 InventoryTransaction、CostLayer、InventoryBalance 參照，PriceList 的單價會影響成本估值（Heading `COSTING RULES`）。
- **Phase 4 Quotation / Sales Order**：報價與訂單會從 `ItemSku` 取得 `taxType`、`unitPrice`、`bundle` 資訊，需確保 `PriceList` 的 `validDate` 與 `QuotationPricingService` 計算一致。

## 4. 實作建議

1. 從 `Item` 建立 `ItemSku` 時，必須指定 `defaultUom`、`taxType`、`basePrice`，並可選擇 `priceListId`。
2. BOM 與 Routing 建議提供版本控制（`version`, `effectiveFrom`），配合 `MoStatus` 以自動套用 BOM。
3. ExtAttr API 應支援 `owner` 權限與 `DataScope` 過濾；若特定 SKU 有自訂欄位，可在 `ItemSkuAttrDef` 設定 `searchable`、`required`。

## 5. 目前進度與下一步

- 已依 code 實作新增 `item-schema.md`，詳細列出 `Item` / `ItemSku` 欄位與 owner filter/ExtAttr 相依性（參考 `Item.java`、`ItemSku.java`）；後續會基於此拆 `price-list.md`、`bom-routing.md` 等更細節檔案。
- 下一步將針對 `PriceList`、`ItemPrice`、`ItemBarcode` 與 `Bom`/`Routing` 各自建 structure doc，並引用 `PriceListService`, `ItemPriceService`, `BomService`, `RoutingService` 的 DTO/API。

## 5. 版本與歷程（History）

| 版本 | 日期       | 說明 |
| ---- | ---------- | ---- |
| v0.1 | 2025-11-20 | 由 legacy doc 初版拆解；後續視實作細節補齊 BOM/PriceList API。 |
