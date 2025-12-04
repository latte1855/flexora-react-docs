# Product Module – API Spec (草稿)

> Version: Draft 2025-12-03  
> 所有端點皆應回傳 Problem JSON 以統一錯誤處理；`version` 欄位若未啟用樂觀鎖，前端可忽略。

## 1. Product / Item

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/items` | 列表 / Hub 用，支援分頁、排序 |
| POST | `/api/items` | 建立產品（含 `hasVariants`） |
| GET | `/api/items/{id}` | 詳細資料，含 ItemGroup、Ext Attr、SKU 摘要 |
| PUT | `/api/items/{id}` | 更新產品主檔 |
| DELETE | `/api/items/{id}` | 軟刪除；保留 `deletedBy` |

**查詢參數**：`keyword`, `itemGroupId`, `status`, `ownerId`, `priceListId`, `hasVariants`, `page`, `size`, `sort`.

**範例 Payload（建立）**

```json
{
  "code": "ITEM-1001",
  "name": "Flexora Notebook",
  "itemGroupId": 12,
  "hasVariants": true,
  "status": "DRAFT",
  "ownerId": 5,
  "defaultTaxCode": "VAT",
  "description": "主產品描述",
  "extAttrs": {
    "BRAND": "Flexora",
    "SERIES": "2025"
  }
}
```

## 2. Item SKU

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/item-skus` | 列表；可依 Item / ItemGroup 過濾 |
| POST | `/api/item-skus` | 新增 SKU（亦可由 Variant Generator 批次） |
| PUT | `/api/item-skus/{id}` | 編輯 UoM、稅別、條碼、狀態 |
| DELETE | `/api/item-skus/{id}` | 軟刪除 |
| POST | `/api/item-skus/variants` | Variant Generator，輸入維度與值 → 產生多筆 |

**SKU 範例**

```json
{
  "itemId": 100,
  "skuNo": "ITEM-1001-BLK-L",
  "name": "Notebook / Black / Large",
  "uomId": 3,
  "taxCode": "VAT",
  "barcode": "1234567890123",
  "defaultWarehouseId": 9,
  "status": "ACTIVE",
  "variantAttributes": {
    "COLOR": "BLACK",
    "SIZE": "L"
  }
}
```

## 3. Lookup / Async Select

| Path | 說明 | 回傳欄位 |
| --- | --- | --- |
| `GET /api/items/lookup` | 用於報價/訂單選擇產品 | `id`, `code`, `name`, `hasVariants`, `itemGroup { id, name }` |
| `GET /api/item-skus/lookup` | SKU 選擇，支援 keyword、priceListId、status | `id`, `skuNo`, `name`, `uom { id, name }`, `taxCode`, `defaultPrice` |

參數：`keyword`, `itemId`, `priceListId`, `status`, `limit=20`.

## 4. PriceList / ItemPrice

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/price-lists` | 列表；支援通路、幣別、狀態篩選 |
| POST | `/api/price-lists` | 建立價目表（含 owner、適用客群） |
| PUT | `/api/price-lists/{id}` | 更新價目表；若需審批則帶 `status` |
| GET | `/api/price-lists/{id}/items` | 該價目表下所有 ItemPrice |
| POST | `/api/item-prices` | 新增/更新單一 SKU 的價格 |
| POST | `/api/item-prices/bulk-import` | 匯入 CSV；需提供批次 ID 追蹤 |

**ItemPrice 範例**

```json
{
  "priceListId": 20,
  "itemSkuId": 3005,
  "currency": "TWD",
  "unitPrice": 1200,
  "discountType": "PERCENT",
  "discountValue": 5,
  "effectiveFrom": "2025-12-01",
  "effectiveTo": null,
  "status": "APPROVED"
}
```

## 5. 審批 / Workflow API（可選）

- `POST /api/price-lists/{id}/submit`：送審。
- `POST /api/price-lists/{id}/approve` / `/reject`：審批動作。
- `GET /api/price-lists/{id}/history`：查看版本與審批紀錄。

若暫不實作，可保留為 TODO 並在 spec 中紀錄。

## 6. TODO / 待確認

- [ ] 批次匯入/匯出的檔案格式、欄位清單與錯誤回饋。
- [ ] Variant Generator API 的輸入/輸出定義（可能需返回衝突清單）。
- [ ] PriceList 權限與審批流程細節（狀態轉換、通知）。
- [ ] 與報價模組的交易 API（例如 `POST /api/quotations/preview` 時需要 SKU 值/價目表）是否需要 cache。
- [ ] 若 PriceList 最終獨立成模組，需要在 `README` 與 `PROJECT_STATUS` 更新對應關係。
