# Product Module – API Spec

> Version: 2025-12-08  
> 所有端點皆應回傳 Problem JSON 以統一錯誤處理。

## 1. Product / Item

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/items` | 列表 / Hub 用，支援分頁、排序 | ✅ `ItemResource# getAllItems` |
| POST | `/api/items` | 建立產品（含 `hasVariants`） | ✅ `ItemResource#create` |
| GET | `/api/items/{id}` | 詳細資料，含 ItemGroup、Ext Attr、SKU 摘要 | ✅ |
| PUT/PATCH | `/api/items/{id}` | 更新/部分更新 | ✅ |
| DELETE | `/api/items/{id}` | 軟刪除；保留 `deletedBy` | ✅ |

**查詢參數**：`keyword`, `itemGroupId`, `enabled`, `ownerId`, `priceListId`, `hasVariants`, `preset`, `page`, `size`, `sort`.

### Preset 支援 (2025-12-08)

| Preset | 後端狀態 | 說明 |
|--------|---------|------|
| `ALL` | ✅ 完成 | 無額外篩選 |
| `ENABLED` | ✅ 完成 | `enabled=true` |
| `DISABLED` | ✅ 完成 | `enabled=false` |
| `LOW_STOCK` | ⚠️ 待實作 | 需要庫存資料整合 |
| `MISSING_PRICELIST` | ⚠️ 待實作 | 需要 `item_price` 表 LEFT JOIN |

**未實作原因**：

- **LOW_STOCK**：目前 `ItemSku` 沒有直接的庫存欄位，庫存資料需從外部來源整合（如 InventoryBalance 表或 WMS 系統），待後續模組開發時補充。
- **MISSING_PRICELIST**：需要在 `ItemSkuQueryService` 中實作 LEFT JOIN `item_price` 表的自定義查詢，判斷 SKU 是否存在有效價格記錄。涉及較複雜的 JPA Criteria 或 Native Query。

### Item DTO 回應格式

```json
{
  "id": 1,
  "itemNo": "TSHIRT",
  "itemName": "素面T恤",
  "enabled": true,
  "hasVariants": true,
  "variantBasedOn": "ITEM_ATTRIBUTE",
  "properties": "{}",
  "deleted": false,
  "version": 0,
  "owner": {
    "id": 2,
    "ownerType": "USER",
    "name": "user"
  },
  "itemGroup": {
    "id": 1,
    "itemGroupNo": "APPAREL",
    "itemGroupName": "服飾"
  },
  "brand": null,
  "variantOf": null
}
```

**欄位說明**：
- `itemNo`: 產品編號
- `itemName`: 產品名稱
- `enabled`: 是否啟用（true=已上架, false=已停用）
- `hasVariants`: 是否有變體
- `variantBasedOn`: 變體依據類型
- `owner`: 負責人物件（nested）
- `itemGroup`: 商品群組物件（nested）
- `brand`: 品牌物件（nested, 可 null）

---

## 2. Item SKU

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/item-skus` | 列表；可依 Item / ItemGroup 過濾 | ✅ `ItemSkuResource#getAllItemSkus` |
| POST | `/api/item-skus` | 新增 SKU | ✅ |
| PUT/PATCH | `/api/item-skus/{id}` | 編輯 UoM、稅別、條碼、狀態 | ✅ |
| DELETE | `/api/item-skus/{id}` | 軟刪除 | ✅ |
| POST | `/api/item-skus/variants` | Variant Generator，輸入維度與值 → 產生多筆 | ⛔ 尚未實作 |

**查詢參數**：`keyword`, `itemId`, `enabled`, `priceListId`, `page`, `size`, `sort`.

### SKU DTO 回應格式

```json
{
  "id": 1001,
  "skuNo": "TSHIRT-RED-M",
  "skuName": "素面T恤_紅_M",
  "enabled": true,
  "productBundle": false,
  "allowSales": true,
  "maintainStock": true,
  "allowNegativeStock": false,
  "supplyMode": "MTS",
  "itemKind": "FINISHED_GOOD",
  "warrantyPeriodInDays": 30,
  "shelfLifeInDays": null,
  "defaultSalesQuantity": 1.0,
  "leadTimeInDays": 7,
  "scrapPercent": 0.0,
  "properties": "{}",
  "deleted": false,
  "version": 0,
  "owner": {
    "id": 2,
    "name": "user"
  },
  "item": {
    "id": 1,
    "itemNo": "TSHIRT",
    "itemName": "素面T恤",
    "enabled": true,
    "hasVariants": true
  },
  "defaultUom": null
}
```

**欄位說明**：
- `skuNo`: SKU 編號
- `skuName`: SKU 名稱
- `enabled`: 是否啟用（true=已上架, false=已停用）
- `supplyMode`: 供應模式（MTS/MTO）
- `itemKind`: 商品類型（FINISHED_GOOD 等）
- `item`: 所屬款式物件（nested）
- `owner`: 負責人物件（nested）
- `defaultUom`: 預設單位物件（nested, 可 null）

---

## 3. Lookup / Async Select

| Path | 說明 | 回傳欄位 | 現況 |
| --- | --- | --- | --- |
| `GET /api/items/lookup` | 用於報價/訂單選擇產品 | `id`, `itemNo`, `itemName`, `hasVariants`, `itemGroup { id, itemGroupName }` | ⛔ 尚未提供 |
| `GET /api/item-skus/lookup` | SKU 選擇，支援 keyword、priceListId | `id`, `skuNo`, `skuName`, `item { id, itemNo, itemName }`, `defaultUom` | ⛔ |

參數：`keyword`, `itemId`, `priceListId`, `enabled`, `limit=20`.

---

## 4. PriceList / ItemPrice

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/price-lists` | 列表；支援通路、幣別、狀態篩選 | ✅ `PriceListResource` |
| POST | `/api/price-lists` | 建立價目表（含 owner、適用客群） | ✅ |
| PUT/PATCH | `/api/price-lists/{id}` | 更新價目表；若需審批則帶 `status` | ✅（但目前無審批欄位） |
| GET | `/api/price-lists/{id}` | 讀取單筆 | ✅ |
| POST | `/api/item-prices` | 新增/更新單一 SKU 的價格 | ✅ |
| POST | `/api/item-prices/bulk-import` | 匯入 CSV；需提供批次 ID 追蹤 | ✅ 透過 `ItemPriceMaintenanceResource /items:bulk-upsert` |

---

## 5. 前端欄位對應表

### Item (款式)

| API 欄位 | 前端顯示 |
| --- | --- |
| `itemNo` | 產品編號 |
| `itemName` | 產品名稱 |
| `enabled` | 狀態（true=已上架, false=已停用） |
| `hasVariants` | 多變體標籤 |
| `itemGroup?.itemGroupName` | 商品群組 |
| `owner?.name` | 負責人 |
| `brand?.brandName` | 品牌 |
| `variantBasedOn` | 變體依據 |

### SKU

| API 欄位 | 前端顯示 |
| --- | --- |
| `skuNo` | SKU 編號 |
| `skuName` | SKU 名稱 |
| `enabled` | 狀態（true=已上架, false=已停用） |
| `item?.itemNo` | 所屬款式編號 |
| `item?.itemName` | 所屬款式名稱 |
| `supplyMode` | 供應模式 |
| `itemKind` | 商品類型 |
| `owner?.name` | 負責人 |
| `defaultUom?.uomName` | 預設單位 |
| `leadTimeInDays` | 前置時間 |

---

## 6. TODO / 待確認

- [ ] 批次匯入/匯出的檔案格式、欄位清單與錯誤回饋。
- [ ] Variant Generator API 的輸入/輸出定義（可能需返回衝突清單）。
- [ ] PriceList 權限與審批流程細節（狀態轉換、通知）。
- [ ] Lookup API 實作。
- [ ] 庫存、單價欄位目前未從 API 回傳，需評估是否由其他 API 補充。
