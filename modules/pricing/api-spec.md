# Pricing Module – API Spec (草稿)

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| PriceRule | `PriceRuleResource` (`/api/price-rules`) | 規則 CRUD、優先順序 |
| Pricing Engine | `PricingResource` (`/api/pricing`) | 即時計價、預覽 |
| Trace | `PriceCalcTraceResource` (`/api/pricing/trace`) | 條件/结果追蹤 |
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

- `POST /api/pricing/preview` 或 `/api/pricing`：輸入 `skuId`, `quantity`, `customerId`, `priceListId`, `channel`，回傳 `unitPrice`, `discount`, `traceId`。  
- 需支援批次預覽（`lines: []`），供報價/訂單使用。  
- 建議定義 Request/Response DTO，包含 `priceListName`, `ruleApplied`, `messages`。

## 3. Trace API

- `GET /api/pricing/trace/{traceId}`：取得計算步驟，欄位包含 `rules`, `conditions`, `calcSteps`, `finalPrice`。  
- 若要提供即時 trace，可允許 `POST /api/pricing/trace`，輸入條件後直接回傳 trace。

## 4. PriceList Assignment

| Method | Path | 說明 |
| --- | --- | --- |
| GET | `/api/price-list-assignments` | 列表（客戶/通路/region 對應） |
| POST | `/api/price-list-assignments` | 建立對應（可含有效日期） |
| PUT/PATCH | `/api/price-list-assignments/{id}` | 更新 |
| DELETE | `/api/price-list-assignments/{id}` | 刪除 |
| POST | `/api/price-list-assignments/sync` | 透過 `PriceListAssignmentSyncResource` 與外部系統同步 |

## 5. TODO

- [ ] 確認 `PriceRule` 是否需要審批/版本欄位；若有，新增 `/submit`, `/approve`.  
- [ ] 規劃 `/reorder` 或 `priority` 更新 API。  
- [ ] 定義 Pricing Preview 的詳細 payload / 錯誤碼並記錄在文件中。  
- [ ] 若 Trace 需長期儲存，定義分頁/查詢參數。  
- [ ] 與 Product Transaction API 的整合：當價格更新時是否要呼叫 PricingEngine。
