# 產品與製造模組 Legacy 規格摘要

摘錄自 `docs/migration/flexora-react/flexora-product-management-spec.md`，並指向 `flexora-react/src/main/java/com/asynctide/flexora/domain/Item*`, `BOM`, `Routing` 等類的實作。該規格強調 SKU 為交易核心、ExtAttr 分表、BOM 分流與價目表/價格時間維度。

## 1. 關鍵決策

- **SKU-first**：所有銷售/採購/報價行都指向 `ItemSku`，即便商品只有單一變體也建立對應 SKU。
- **Sales Bundle vs Manufacturing BOM**：銷售組合與製造用 BOM 分流，前者畫面呈現 `ProductBundleItem`，後者供 MO 投料。
- **Routing / WorkCenter**：製程版本與工序由 `Routing`、`RoutingOp` 定義，工序綁 `WorkCenter`；工單 `MoStatus` 控流程。
- **ExtAttr**：每個模組（一致的 `*AttrDef/Value`）各自存放客製欄位，透過 `unique(owner_id, attr_code) WHERE deleted=false` 管理，與 UI/Service 同步。

## 2. Data & ER Highlights

- 主要 Entity：`Item`（母體）、`ItemSku`（最小交易單位）、`ItemGroup`、`Brand`、`UOM`、`PriceList` + `ItemPrice`、`ProductBundleItem`、`Bom/BomItem`、`Routing/RoutingOp`。
- Data rules：金額/rate 同 `phase0` 規範，`Item`/`SKU` 須支援多條碼 `ItemBarcode`，`UOMConversion` 處理單位轉換。
- ER：`ItemSku` 連 `UOM`、`priceList`、`bundle`、`BOM`；`RoutingOp` 搭 `WorkCenter` ；`ExtAttr` 透過 `owner_id` 與 `attr_code` 鍵值查詢。

## 3. API / Process hints

- ExtAttr API 由 `GET /api/items/{id}/ext-attrs`、`PUT /api/items/{id}/ext-attrs/{code}` 等 endpoint 實作；實際 Controller 於 `flexora-react/src/main/java/com/asynctide/flexora/web/rest/ItemResource.java`。
- 價格模組（PriceList/ItemPrice）應能查詢特定期間價格，並支援 `latest` fallback；詳見 `flexora-react/src/main/java/com/asynctide/flexora/service/ItemPriceService`.

## 4. Next steps

- 若需更多細節（如 `ItemAttribute` 值允許何種 datatype），請參考原始規格檔或 `flexora-react` domain class。
- 新增/調整 ExtAttr 或 BOM 時，請同步更新 `docs/architecture/DATA_MODEL.md` 以及 `phase1` spec。
