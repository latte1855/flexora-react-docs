# Phase 1 — Item / SKU 資料模型參考（Item Schema）

本檔根據 `flexora-react/src/main/java/com/asynctide/flexora/domain/Item.java` 與 `ItemSku.java` 之實作撰寫，作為 Phase 1 具體欄位參考。所有欄位精度、限制、必填邏輯均以程式碼為權威；若有變更需同步更新後端 entity 並註記於本檔。

## 1. Item（款式 / SPU）

| 欄位 | 型態 | 必填 | 說明 | 備註 |
| --- | --- | --- | --- | --- |
| `item_no` | `VARCHAR(50)` | Y | 款式代號 | 建議 partial unique，跨 `owner_id` 唯一。 |
| `item_name` | `VARCHAR(100)` | Y | 款式名稱 | |
| `enabled` | `BOOLEAN` | Y | 是否啟用 | |
| `has_variants` | `BOOLEAN` | Y | 是否有變體 | |
| `variant_based_on` | Enum (`VariantBasedOn`) | N | 變體產生依據 | |
| `properties` | `JSONB` | N | 自訂 metadata（ExtAttr 載體） | |
| `deleted`, `deleted_at`, `deleted_by` | `BOOLEAN`/`TIMESTAMP`/`VARCHAR` | Y/N | 軟刪欄位，所有 entity 通用 | `deleted` 預設 `false` |
| `version` | `Long` | N | 樂觀鎖 | |
| `owner_id` | `Long` | Y | 所屬 Owner | 需配合 DataScope |
| `item_group_id` | `Long` | Y | 所屬 ItemGroup | |
| `brand_id` | `Long` | N | 品牌 | |
| `variant_of_id` | `Long` | N | 參照母品 | 代表此 Item 是某 Item 的變體 |

**註**：以上欄位皆可在 `flexora-react/src/main/java/com/asynctide/flexora/domain/Item.java` 找到，Entity 上標註 `@Filter(name = "ownerFilter", ...)` 以套入資料域。

## 2. ItemSku（SKU / 最小交易單位）

| 欄位 | 型態 | 必填 | 說明 | 備註 |
| --- | --- | --- | --- | --- |
| `sku_no` | `VARCHAR(64)` | Y | SKU 代號 | partial unique `WHERE deleted=false` 建議由 Numbering Service / UI 避免衝突 |
| `sku_name` | `VARCHAR(150)` | Y | 可包含屬性值 | |
| `enabled` | `BOOLEAN` | Y | 是否啟用 | |
| `product_bundle` | `BOOLEAN` | Y | 是否組合品（ProductBundleItem） | |
| `allow_sales` | `BOOLEAN` | N | 是否允許缺貨仍能銷售（MTO） | 建議搭配 `allow_negative_stock` |
| `maintain_stock` | `BOOLEAN` | N | 是否有庫存控制 | 服務品可 false |
| `allow_negative_stock` | `BOOLEAN` | N | 是否允許負庫 | 須審批 |
| `supply_mode` | Enum `SupplyMode` | N | 供應策略（MTS/MTO/ATO/CTO） | |
| `item_kind` | Enum `ItemKind` | N | 物料類型（成品/原料/半成品/虛擬/服務） | |
| `warranty_period_in_days` | `INTEGER` | N | 保固天數 | |
| `shelf_life_in_days` | `INTEGER` | N | 保存天數 | |
| `default_sales_quantity` / `default_purchase_quantity` | `DECIMAL(21,6)` | N | 建議單位數量 | |
| `lead_time_in_days` | `INTEGER` | N | 製造前置期 | |
| `scrap_percent` | `DECIMAL(21,6)` | N | 製程損耗（0~1） | |
| `properties` | `JSONB` | N | 自訂 metadata | ExtAttr 也可額外儲存 |
| `deleted`, `deleted_at`, `deleted_by`, `version` | ... | ... | 軟刪與樂觀鎖 | |
| `owner_id`, `item_id`, `default_uom_id` | `Long` | Y | 所屬 Owner、Item、預設 UoM | 需與 DataScope/Inventory 對齊 |
| `tax_type`, `tax_code`, `import_duty` etc. | `VARCHAR`/`DECIMAL` | 視實作 | 參考 `priceList` / `tax_type` 與 `ItemPrice` | |

## 3. ExtAttr / DataScope 備註

* `ItemExtAttrDef` / `ItemSkuExtAttrDef` 與 `Item*AttrValue` 均實作 `ownerFilter` 與 soft delete，可在 `flexora-react/domain` 中查詢欄位。  
* `Item` / `ItemSku` 上皆有 `@Filter(name = "ownerFilter", ...)`。若要呈現 owner 範圍請同時查 `v_owner_visible`。  
* ExtAttr API 由 `ItemResource`、`ItemSkuResource` 提供（`/api/items/{id}/ext-attrs`、`/api/item-skus/{id}/ext-attrs`），需搭配 `DataScopePredicateBuilder` 進行 owner 過濾。

## 4. 參考程式碼

- `flexora-react/src/main/java/com/asynctide/flexora/domain/Item.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/domain/ItemSku.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/web/rest/ItemResource.java`  
- `flexora-react/src/main/java/com/asynctide/flexora/web/rest/ItemSkuResource.java`
