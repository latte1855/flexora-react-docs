# Flexora ERP — 產品管理＋製造（BOM/Routing）規格（Product & Manufacturing Spec）
> 版本：2025-09-21 00:00:00Z  
> 作者：Chia-Ming Liu <latte1855@gmail.com>（彙整 by GPT）

---

## 目標
以 **SKU 為唯一交易單位**，建立可擴充的商品資料模型與製造基本模型，支援：
- 變體（屬性衍生）
- 條碼（多碼）
- **組合/套裝（Sales Bundle BOM）** 與 **製造用 BOM（Manufacturing BOM）**
- 製程（Routing / RoutingOp）與工作中心（WorkCenter）
- 價目表與期間價格
- 分類/品牌/單位與換算
- 軟刪除、審計欄位、**樂觀鎖 `version`**
- JaVers 版本管理（審計快照）

> **關鍵決策**：所有銷售/採購/報價行 **一律對應 `ItemSku`**。  
> 無論商品是否有變體，皆建立「預設 SKU」，以簡化條碼、庫存、定價、報表與序號管理。  
> **Sales Bundle**（銷售綁品）與 **Manufacturing BOM**（製造用用料表）**分流**：前者用於銷售打包，後者用於工單投料。

---

## 名詞對應
- **Item（款式 / SPU）**：商品的母體，不一定直接交易。
- **SKU（`ItemSku`）**：可交易/可庫存的最小單位（如 `T恤_紅_L`）。
- **屬性（`ItemAttribute`）/屬性值（`ItemAttributeValue`）**：用以衍生 SKU 的維度；另支援連續參數（`numberValue`/`textValue`）。
- **組合（`ProductBundleItem`）**：主 SKU 由多個元件 SKU 與數量構成（**銷售用**）。
- **製造用 BOM（`Bom`/`BomItem`）**：針對 **某 SKU** 的用料版本與期間。
- **製程（`Routing`/`RoutingOp`）**：加工途程版本與工序；工序綁定工作中心（`WorkCenter`）。
- **價目表（`PriceList`）/價格（`ItemPrice`）**：(價目表, SKU, 期間) → 牌價。

---

## ExtAttr（每模組一表，與報價模組一致）
> 為支援客製欄位且避免單表膨脹，採 **每模組一表**：`ItemAttrDef/ItemAttrValue`、`ItemSkuAttrDef/ItemSkuAttrValue`。  
> 結構一致、易於共用 Service 與驗證邏輯。

- **定義表（Def）**：`id`, `code`, `data_type(STRING|INT|DECIMAL|DATE|BOOL|JSON)`, `label`, `required`, `regex`, `searchable`, `deleted`, `version`
- **值表（Value）**（擇一填入對應型別欄位）：  
  `owner_id(item_id 或 sku_id)`, `attr_code`, `data_type`,  
  `value_string|value_number|value_decimal|value_date|value_bool|value_json`,  
  `deleted`, `created_at`, `updated_at`, `version`
- **唯一鍵**：`unique(owner_id, attr_code) WHERE deleted=false`
- **常用索引**：`(attr_code, value_string)`、`(attr_code, value_decimal)`（依需求加）
- **API 建議**：  
  `GET /api/items/{id}/ext-attrs`、`PUT /api/items/{id}/ext-attrs/{code}`  
  `GET /api/skus/{id}/ext-attrs`、`PUT /api/skus/{id}/ext-attrs/{code}`

---

## 關係示意（Mermaid ER）
```mermaid
erDiagram
  ITEM ||--o{ ITEMSKU : has
  ITEM }o--|| ITEMGROUP : categorized-by
  ITEM }o--|| BRAND : branded-by
  ITEM }o--|| UOM : default-uom
  ITEM ||--o| ITEM : variant-of

  ITEMSKU }o--|| ITEM : belongs-to
  ITEMSKU }o--|| UOM : default-uom
  ITEMSKU ||--o{ ITEMBARCODE : has

  SKUATTRIBUTE }o--|| ITEMSKU : of
  SKUATTRIBUTE }o--|| ITEMATTRIBUTE : uses-attribute
  SKUATTRIBUTE }o--|| ITEMATTRIBUTEVALUE : uses-value

  PRICELIST ||--o{ ITEMPRICE : contains
  ITEMPRICE }o--|| ITEMSKU : prices

  PRODUCTBUNDLEITEM }o--|| ITEMSKU : component
  PRODUCTBUNDLEITEM }o--|| ITEMSKU : bundle

  UOMCONVERSION }o--|| UOM : fromUom
  UOMCONVERSION }o--|| UOM : toUom

  WORKCENTER ||--o{ ROUTINGOP : used-by
  ITEMSKU ||--o{ BOM : has-bom
  BOM ||--o{ BOMITEM : has
  ITEMSKU ||--o{ ROUTING : has-routing
  ROUTING ||--o{ ROUTINGOP : has
````

> 若 Mermaid 無法渲染，可參考下方 ASCII 示意：

```
Item (SPU) 1 ── * ItemSku (SKU)
   │               ├─ * ItemBarcode
   │               └─ * SkuAttribute ──> ItemAttribute ── * ItemAttributeValue
   │
   ├─ o1 ItemGroup（樹狀）
   ├─ o1 Brand
   ├─ o1 Uom（預設）
   └─ o1 Item（variantOf；可為空）

PriceList 1 ── * ItemPrice ── o1 ItemSku

ItemSku（主） 1 ── * ProductBundleItem ── o1 ItemSku（元件）   ←（銷售綁品）

ItemSku 1 ── * Bom（版本/期間） ── * BomItem（元件）         ←（製造用料）
ItemSku 1 ── * Routing（版本/期間） ── * RoutingOp（工序）─o1 WorkCenter
```

---

## 交易指向（報價/訂單/採購/出入庫）

* **行項目 FK**：`sku_id` (**NOT NULL → `ItemSku`**)
* 可帶：`uom_id`（預設從 SKU 帶出，可覆寫）、`price_list_id`（追溯）、`quantity`、`unitPrice`、折扣欄位等。
* 若 `sku` 為 **Sales Bundle** 主 SKU，報價可不展開；出貨/扣帳再展開（策略可配置）。

---

## 業務情境與製造模式（MTO/MTS/ATO/CTO）

* 客戶產業：**接單生產（MTO）為主**，庫存低甚至 0，也要能 **允許零庫存接單**。

  * 對應：`ItemSku.allowSales = true` 時可在零庫存下接單；生產由 SO 觸發 MO。
  * `ItemSku.supplyMode`（MTS/MTO/ATO/CTO）決定後續流程：

    * **MTO**：SO → 觸發 MO（抓 `Bom` + `Routing`）
    * **MTS**：MO 依預測/補庫計畫產生
    * **ATO/CTO**：SO 時組裝/配置，BOM 可用選配料與參數（未來擴充）
* `ItemKind`（成品/半成品/原料/虛擬件/服務）：

  * **PHANTOM 虛擬件**：不入庫、僅展開其下元件；`BomItem.phantomExpand=true` 時於爆展時直接替換為其子件。
  * **SERVICE 服務**：不走庫存，通常以工序或工時呈現，可用 SKU 表達成本計算（不回沖存貨）。

---

## 實體說明（含欄位建議）

> **欄位共通規則**
>
> * 時間：事件時間用 `Instant (UTC)` → DB `TIMESTAMP WITH TIME ZONE`。
> * 數值：金額/數量用 `BigDecimal`；金額 `numeric(19,4)`、數量 `numeric(19,6)`、比率 `scale=6`。
> * 軟刪：`deleted(Boolean)`, `deletedAt(Instant)`, `deletedBy(String)`（**預設值建議 `false`**）。
>   **建議 partial unique 皆加入 `WHERE deleted=false`。**
> * 樂觀鎖：`version(Long)`（JPA `@Version`）。
> * DB 檢核（check）以 **PL/pgSQL `DO $$ ... $$;`** 建立，避免多次執行衝突。

### 1) Item（款式 / SPU）

* 用途：商品母體，承載分類、品牌、預設單位、是否有變體。
* 主要欄位：`itemNo`（唯一；partial unique）、`itemName`、`hasVariants`、`variantBasedOn`、`enabled`、`deleted*`、`version`
* 關聯：`itemGroup`、`brand`、`defaultUom`、`variantOf`

### 2) ItemGroup（產品分類；樹）

* **路徑設計**：`parentHierarchy` **含自身、以 `/` 分隔**；子樹查詢：`= '/1' OR LIKE '/1/%'`。
* 欄位：`itemGroupNo`, `itemGroupName`, `parentHierarchy(1024)`, `depth`, `itemGroupSequence`, `enabled`, `deleted*`, `version`。
* 索引：`unique(itemGroupNo) WHERE deleted=false`、`(parentHierarchy)`、`(parentHierarchy text_pattern_ops)`、`(owner_id)`。

### 3) Brand

* 欄位：`brandNo`（**partial unique**）、`brandName`、`enabled`、`deleted*`、`version`。

### 4) Uom / UomConversion

* `Uom`：`uomCode`（**partial unique**）、`uomName`、`precision`、`enabled`、`deleted*`、`version`。
* `UomConversion`：`fromUom_id`, `toUom_id`, `factor(numeric(19,6))`、`deleted*`、`version`。

  * 唯一鍵：`(fromUom_id, toUom_id) WHERE deleted=false`。
  * 檢核：`factor > 0`、`from != to`。

### 5) ItemAttribute / ItemAttributeValue

* `ItemAttribute`：`attributeName`（**partial unique**）等；`deleted*`、`version`。
* `ItemAttributeValue`：`attribute_id`、`value`、`displaySequence`、`deleted*`、`version`。

  * 唯一鍵：`(attribute_id, value) WHERE deleted=false`。

### 6) ItemSku（SKU / 最小交易與庫存單位）

* 欄位：`skuNo`（**partial unique**）、`skuName`、`enabled`、`allowSales`、`maintainStock`、`allowNegativeStock`、`supplyMode(MTS|MTO|ATO|CTO)`、`itemKind(FINISHED|SEMI|RAW|PHANTOM|SERVICE)`、`warrantyPeriodInDays`、`shelfLifeInDays`、`defaultSalesQuantity`、`defaultPurchaseQuantity`、`leadTimeInDays(>=0)`、`scrapPercent(0~1)`、`deleted*`、`version`。
* 關聯：`item_id`（母款）、`defaultUom_id`、`owner_id`。
* 檢核：`leadTimeInDays >= 0`、`scrapPercent` 在 `[0,1]`（允許 NULL）。
* 索引：`unique(skuNo) WHERE deleted=false`、`(item_id)`、`(default_uom_id)`、`(owner_id)`。

### 7) ItemBarcode（SKU 條碼；多對一 SKU）

* 欄位：`barcode`、`type(BarcodeType)`、`active`、`deleted*`、`version`。
* 唯一：**partial unique** `unique(barcode) WHERE deleted=false AND active=true`。
* 索引：`(sku_id)`。

### 8) SkuAttribute（SKU 的屬性指定）

* 三選一：`attribute_value_id` / `numberValue` / `textValue` 其一必填（**應用層檢核**）。
* 唯一：`(sku_id, attribute_id) WHERE deleted=false`（同一 SKU 的同屬性僅一筆）。
* 索引：`(sku_id)`、`(attribute_id)`、`(attribute_value_id)`。

### 9) ProductBundleItem（銷售綁品 BOM）

* 關聯：`bundleSku_id`（主 SKU）、`componentSku_id`（元件 SKU）。
* 欄位：`quantity(numeric(19,6))`（**> 0**）、`displaySequence`、`deleted*`、`version`。
* 檢核：`quantity > 0`、`bundle != component`。
* 唯一：`(bundleSku_id, componentSku_id) WHERE deleted=false`。
* 索引：`(bundle_sku_id)`、`(component_sku_id)`。

### 10) PriceList / ItemPrice

* `PriceList`：`priceListCode`（**partial unique**）、`priceListName`、`type(SALES|PURCHASE)`、`currencyCode(ISO 4217)`、`enabled`、`isDefault`、`deleted*`、`version`。

  * 唯一：每 `(type,currency)` 僅一個 `isDefault=true`（`WHERE deleted=false`）。
* `ItemPrice`：`priceList_id`、`sku_id`、`listPrice(numeric(19,4))`、`validFrom(LocalDate)`、`validTo(LocalDate)`、`deleted*`、`version`。

  * 檢核：`validFrom/validTo` 至少其一為 NULL，或 `from <= to`。
  * 唯一：避免「完全相同期間」重複：`unique(price_list_id, sku_id, valid_from, valid_to) WHERE deleted=false`。
  * 索引：`(price_list_id, sku_id)`、`(valid_from)`、`(valid_to)`。

### 11) **WorkCenter（工作中心）**

* 欄位：`workCenterNo`（**partial unique**）、`workCenterName`、`enabled`、`deleted*`、`version`。
* 關聯：`owner_id`（責任歸屬）。
* 索引：`(owner_id)`。

### 12) **Bom（製造用 BOM 表頭） / BomItem（用料）**

* **Bom（表頭）**：

  * 指向：`sku_id`（為該成品/半成品定義 BOM）。
  * 欄位：`versionCode`（可空，便於預設版）、`isDefault`、`effectiveFrom/To`、`remarks`、`deleted*`、`version`。
  * 檢核：`effectiveFrom/To` 至少其一為 NULL，或 `from <= to`。
  * 唯一：每 SKU 僅一個 `isDefault=true`（`WHERE deleted=false`）；`(sku_id, version_code)` 在未刪除資料內唯一（`versionCode IS NOT NULL`時）。
  * 索引：`(owner_id)`、`(sku_id)`。
* **BomItem（明細）**：

  * 指向：`parent_bom_id`、`component_sku_id`、（選）`uom_id`。
  * 欄位：`quantity(numeric(19,6))`（**> 0**）、`scrapPercent`（0\~1，可空）、`optional`、`phantomExpand`、`displaySequence`、`remarks`、`deleted*`、`version`。
  * 檢核：`quantity > 0`、`scrapPercent` 在 `[0,1]`（或 NULL）。
  * 唯一：同一 BOM 不可重複相同 `component_sku_id`（`WHERE deleted=false`）。
  * 索引：`(parent_bom_id)`、`(component_sku_id)`、`(uom_id)`。

### 13) **Routing（製程） / RoutingOp（工序）**

* **Routing（表頭）**：

  * 指向：`sku_id`。
  * 欄位：`versionCode`（可空）、`isDefault`、`effectiveFrom/To`、`remarks`、`deleted*`、`version`。
  * 檢核：`effectiveFrom/To` 至少其一為 NULL，或 `from <= to`。
  * 唯一：每 SKU 僅一個 `isDefault=true`（`WHERE deleted=false`）；`(sku_id, version_code)` 在未刪除資料內唯一（`versionCode IS NOT NULL`時）。
  * 索引：`(owner_id)`、`(sku_id)`。
* **RoutingOp（工序）**：

  * 指向：`routing_id`、`work_center_id`。
  * 欄位：`opNo(Integer)`、`setupTimeMinutes`、`runTimePerUomMinutes`、`queueTimeMinutes`、`moveTimeMinutes`、`remarks`、`deleted*`、`version`。
  * **opNo 唯一**：同一 `routing_id` 底下 `opNo` 不得重複（`WHERE deleted=false`）。
  * 檢核：各時間參數需 **>= 0**（允許空）。

---

## BOM 與 Routing 的**選版規則**（建議）

當 **工單（MO）** 需要選用 BOM / Routing 版本時，依序：

1. 若指定 `versionCode`，且符合日期區間（落於 `effectiveFrom/To`），則使用。
2. 否則若存在 `isDefault=true` 且日期符合，則使用。
3. 否則，選用 **最新有效** 的一版（按 `effectiveFrom` 近者優先；若同日，取 `id` 大者）。

> 選版時請同時檢查 `deleted=false`。若均找不到，應回報錯誤並阻擋發單。

---

## BOM 與 UoM / 換算

* `BomItem` 可填 `uom_id`（元件的用料單位）。
* 投料/回沖時：以 `UomConversion` 轉為元件 SKU 的庫存主單位；若無對應換算，阻擋並提示。
* `scrapPercent`：系統應以 **需求量 ÷ (1 - scrap)** 方式放大用量（四捨五入規則請在製造參數定義）。

---

## Phantom（虛擬件）與 **展開策略**

* **虛擬件（PHANTOM）**：只作為群組，不入庫；在爆展時直接以其子階元件替代自身。
* `BomItem.phantomExpand = true` 時，MO 展開結構將**遞迴**替換該元件為其子件。
* 若 `itemKind = PHANTOM` 但 `phantomExpand = false`，則視為需發料但不建議入庫 → 視需求定義為「直接消耗」。

---

## MTO 與客製參數（尺寸/文字）

* `SkuAttribute` 支援 `numberValue`、`textValue`，可記錄門寬/高度、客製字樣等。
* **延伸（非當前版實作）**：BOM 參數化（例如 `quantity = 材料寬度 × 係數`），建議將公式儲存在 BOM 明細層（或 RoutingOp）並於 MO 產生時計算。
* 當前版：先以 **固定 BOM** 應對 MTO，客製尺寸以 SKU 屬性記錄，並在製令備註/附檔同步給現場。

---

## 報價選價流程（建議）

1. 取得 **Sales PriceList**（客戶綁定或預設）。
2. 以 **SKU + 報價日期** 查 `ItemPrice`（落在 `validFrom/validTo` 區間）。
3. 找不到時的策略：

   * 用 `Item.defaultSalesListPrice` 作為 fallback，或
   * 給出「未定價」錯誤交由人工處理。

---

## 路徑型階層（ItemGroup）設計取捨

* 採用 **`/1/3`** 路徑（含自身），優點：

  * 子樹查詢單純：`= '/1' OR LIKE '/1/%'`。
  * 移動節點時只需更新**子樹**路徑（可批次）。
* `text_pattern_ops` 索引改善 `LIKE 'prefix%'` 的效能。

---

## 索引與唯一鍵建議（PostgreSQL 摘要）

```sql
-- SKU 唯一（排除軟刪）
CREATE UNIQUE INDEX uq_itemsku_sku_no
ON item_sku (sku_no)
WHERE deleted = false;

-- 條碼唯一（僅限啟用且未刪除）
CREATE UNIQUE INDEX uq_itembarcode_active
ON item_barcode (barcode)
WHERE deleted = false AND active = true;

-- ItemGroup 子樹查詢加速
CREATE INDEX idx_itemgroup_path ON item_group (parent_hierarchy text_pattern_ops);

-- UoM 換算唯一、檢核
CREATE UNIQUE INDEX uq_uomconv_from_to_active ON uom_conversion(from_uom_id, to_uom_id) WHERE deleted=false;

-- Sales Bundle：主/元件 唯一
CREATE UNIQUE INDEX uq_bundle_component_active ON product_bundle_item(bundle_sku_id, component_sku_id) WHERE deleted=false;

-- BOM 表頭：每 SKU 僅一個 is_default
CREATE UNIQUE INDEX uq_bom_default_per_sku ON bom(sku_id) WHERE is_default = true AND deleted=false;
-- BOM 表頭：版本代號唯一（有填版號時）
CREATE UNIQUE INDEX uq_bom_version_per_sku ON bom(sku_id, version_code) WHERE deleted=false AND version_code IS NOT NULL;

-- BOM 明細：同 BOM 不重複元件
CREATE UNIQUE INDEX uq_bomitem_component_unique ON bom_item(parent_bom_id, component_sku_id) WHERE deleted=false;

-- Routing 表頭：每 SKU 僅一個 is_default
CREATE UNIQUE INDEX uq_routing_default_per_sku ON routing(sku_id) WHERE is_default = true AND deleted=false;
-- Routing 表頭：版本代號唯一（有填版號時）
CREATE UNIQUE INDEX uq_routing_version_per_sku ON routing(sku_id, version_code) WHERE deleted=false AND version_code IS NOT NULL;

-- Routing 工序：同路徑下 opNo 唯一
CREATE UNIQUE INDEX uq_rtop_opno_per_routing ON routing_op(routing_id, op_no) WHERE deleted=false;
```

> **Check 建議**（以 PL/pgSQL `DO $$` 建立）：
>
> * `uom_conversion.factor > 0`、`from <> to`
> * `product_bundle_item.quantity > 0`、`bundle <> component`
> * `bom_item.quantity > 0`、`scrapPercent ∈ [0,1]`
> * `item_sku.leadTimeInDays >= 0`、`scrapPercent ∈ [0,1]`
> * `*_validFrom/To` 需 `from <= to`（或至少其一為 NULL）
> * `routing_op.*TimeMinutes >= 0`

---

## 資料品質與防錯

* **循環參照**：BOM 禁止環；DB 階層巡訪較難檢出，建議**應用層**建立 DFS 檢查（建/改版時驗證）。
* **刪除策略**：一律軟刪（`deleted=true`），並以 **partial unique index** 避免歷史資料卡住唯一鍵。
* **欄位預設**：所有 `deleted` 欄位 **預設 `false`**；`enabled` 欄位預設依業務。
* **審計**：採 JaVers，重要變更（BOM/Routing）必須有操作人與備註。

---

## 成本與損耗（預留）

* 現階段不建標準成本表，但 `scrapPercent` 已支援用料放大。
* 後續可擴：材料、人工、製造費（按 RoutingOp 時間×工資/時 + WorkCenter 費率）→ 標準成本。

---

## 與外部系統一致性

* **ERPNext**：以具體可交易項目（Item Variant / Item）為單位；我們以 `ItemSku` 對應。其 BOM/Routing 多版本與本設計一致。
* **vtiger**：報價/訂單指向產品實體；實務亦以 SKU 層進行。

---

## 未來擴充

* **BOM 參數化** 與 **替代料（Alternates）**、**生效優先序**。
* **工時薪級/排程能力**（WorkCenter Capacity, Calendars）。
* **序號/批號（SN/Batch）**、**先到先出**。
* **期間重疊價格**的 DB 約束（Exclusion Constraint）或觸發器。

---

## 審計與版本（JaVers）

* Repository 加 `@JaversSpringDataAuditable` 或於 Service 呼叫 `javers.commit(author, entity)`。
* `commitProperties` 建議帶：`tenantId`, `source`, `ip`, `reqId`。
* SQL Schema 由 Liquibase 管（`audit.jv_*`）；建議以 `jv_commit.commit_date` 作月分割（後續工作）。

---

## 變更紀錄（Changelog）

* 2025-09-21 00:00:00Z：新增 **製造用 BOM/Routing/WorkCenter** 模組與相關規範、MTO 支援要點、Phantom 展開、選版規則、索引/檢核摘要；補入 **ItemExtAttr / ItemSkuExtAttr（每模組一表）**。
* 2025-09-19 03:27:12Z：初版彙整，確認「報價行指向 SKU」與 ER 關係；加入路徑型階層、索引、JaVers 要點。

