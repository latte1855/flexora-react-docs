# Pricing Module – API Spec (草稿)

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| PriceRule | `PriceRuleResource` (`/api/price-rules`) | 規則 CRUD、優先順序 |
| Pricing Engine | `PricingResource` (`/api/pricing`) | 即時計價、預覽 |
| Trace | `PriceCalcTraceResource` (`/api/pricing/trace`) | 條件/结果追蹤 |
| PriceList / ItemPrice | `PriceListResource`, `ItemPriceResource` | 價目表與 SKU 價格 |
| PriceList Assignment | `PriceListAssignmentResource`, `PriceListAssignmentSyncResource` | 管理價目表與客戶關係 |

## 1. PriceRule API

| Method | Path | 說明 | 備註 |
| --- | --- | --- | --- |
| GET | `/api/price-rules` | 列表 / Criteria | ✅ |
| POST | `/api/price-rules` | 建立規則 | ✅ |
| PUT/PATCH | `/api/price-rules/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/price-rules/{id}` | 軟刪除 | ✅ |
| POST | `/api/price-rules/reorder` | 調整優先順序（若未實作需新增） | ⛔ |

欄位需包含條件（channel、customerGroup、itemGroup）、公式（fixed price、percent、markup）與有效日期。

## 2. Pricing Engine

- `POST /api/pricing/preview``：輸入 `lines` 陣列，並可選擇 `returnTrace=true`。  
```json
{
  "customerId": 501,
  "priceListId": 2,
  "channel": "WEB",
  "currency": "TWD",
  "lines": [
    { "lineId": "Q1L1", "skuId": 1001, "qty": "5", "basePrice": "500.00" }
  ],
  "options": { "returnTrace": true, "dryRun": true }
}
```
- Response：
```json
{
  "lines": [
    {
      "lineId": "Q1L1",
      "unitPrice": "450.00",
      "discountAmount": "50.00",
      "ruleApplied": "VIP-5OFF",
      "traceId": "PRC-TRACE-20251203-001"
    }
  ],
  "priceListName": "VIP-2025",
  "messages": ["pricing.preview.success"]
}
```
- 需支援批次預覽與錯誤列表（當某行錯誤，不影響其他行，回傳 `errors:[{lineId,message}]`）。

## 3. Trace API

- `GET /api/pricing/trace/{traceId}`：取得計算步驟，欄位包含 `rules`, `conditions`, `calcSteps`, `finalPrice`。  
- 若要提供即時 trace，可允許 `POST /api/pricing/trace`，輸入條件後直接回傳 trace。  
- Response 範例：
```json
{
  "traceId": "PRC-TRACE-20251203-001",
  "input": { "skuId": 1001, "qty": "5", "customerId": 501 },
  "steps": [
    { "ruleCode": "VIP-5OFF", "matched": true, "condition": "customerGroup == VIP", "calc": "basePrice * 0.95", "result": "450.00" },
    { "ruleCode": "PROMO-XMAS", "matched": false }
  ],
  "finalPrice": "450.00"
}
```

## 4. PriceList / ItemPrice API

| Method | Path | 說明 | 備註 |
| --- | --- | --- | --- |
| GET | `/api/price-lists` | 查詢價目表（支援 Criteria） | ✅ |
| POST | `/api/price-lists` | 建立價目表 | ✅ |
| PATCH | `/api/price-lists/{id}` | 更新（幣別、channel、有效期） | ✅ |
| POST | `/api/price-lists/{id}/submit` | 送審（若啟用 workflow） | ⛔ |
| POST | `/api/price-lists/{id}/approve` | 審批/退回 | ⛔ |
| GET | `/api/price-lists/{id}/history` | workflow 歷史/評論 | ⛔ |
| GET | `/api/item-prices` | 依價目表、SKU 查詢價格 | ✅ |
| POST | `/api/item-prices` | 建立/更新 SKU 價格（支援整批 lines） | ⛔ |
| POST | `/api/item-prices/transactions` | 整批寫入（Item + Price） | ⛔ |
| POST | `/api/item-prices/import` | 匯入（CSV/XLSX），回傳 traceNo | ⛔ |
| GET | `/api/item-prices/import-jobs/{traceNo}` | 查詢匯入結果與錯誤 | ⛔ |
| GET | `/api/item-prices/export` | 匯出目前價格 | ⛔ |

**Transaction payload 建議**
```json
{
  "priceList": { "id": 2, "currency": "TWD", "channel": "WEB" },
  "items": [
    { "skuId": 1001, "unitPrice": "450.00", "effectiveFrom": "2025-01-01", "effectiveTo": null, "discountType": "PERCENT" }
  ]
}
```

**匯入/匯出格式建議**
```
priceListCode,skuNo,unitPrice,discountType,effectiveFrom,effectiveTo,currency
WEB-2025,P-100-RED-500,450,PERCENT,2025-01-01,,TWD
```
- `/import` 回傳 `{"traceNo":"PRC-IMPORT-001","errors":[{"row":3,"message":"pricing.error.invalidSku"}]}`。  
- 匯入完成後可透過 `GET /api/item-prices/import-jobs/{traceNo}` 查詢狀態。  
- `/export` 支援 filter：`priceListId`, `skuNo`, `onlyActive`。  
- 若啟用審批，可在 payload 加上 `submit=true` 由服務端決定是否觸發 workflow。

## 5. PriceList Assignment

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/price-list-assignments` | 列表（客戶/通路/region 對應） |
| POST | `/api/price-list-assignments` | 建立對應（可含有效日期） |
| PUT/PATCH | `/api/price-list-assignments/{id}` | 更新 |
| DELETE | `/api/price-list-assignments/{id}` | 刪除 |
| POST | `/api/price-list-assignments/sync` | 透過 `PriceListAssignmentSyncResource` 與外部系統同步 |

## 6. TODO

- [ ] 確認 `PriceRule` 是否需要審批/版本欄位；若有，新增 `/submit`, `/approve`.  
- [ ] 規劃 `/reorder` 或 `priority` 更新 API。  
- [ ] 定義 Pricing Preview 的詳細 payload / 錯誤碼並記錄在文件中。  
- [ ] 若 Trace 需長期儲存，定義分頁/查詢參數。  
- [ ] 與 Product Transaction API 的整合：當價格更新時是否要呼叫 PricingEngine。
- [ ] `item-prices` 匯入/匯出與整批儲存 API。  
- [ ] PriceList workflow（submit/approve/history）與通知。  
