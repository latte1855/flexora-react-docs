---
codex:
  purpose: 'Flexora ERP API 規格（彙整版）'
  context: 'Pricing + CRM（Customer/Contact/Address）核心 API 對外行為規範'
  lastUpdated: 2025-11-04
  language: zh-TW
  related:
    - './final/API 清單（含用途說明）.md'
---

# Flexora ERP API 規格（彙整版）

> 更新日期：2025-10-30（Asia/Taipei）  
> 適用版本：JHipster v8.11.0 基礎（Spring Boot 3.4/3.5）  
> 本版重點：
>
> - 定價模組（Pricing / Price List / Item Price Maintenance / Assignment Sync）
> - Phase 2：CRM（Customer / Contact / Address）
> - **唯一鍵規則**：`deleted=false` 範圍 + `LOWER(code)` 全域唯一
> - 補齊 **\_exists**、**:soft-delete**、**:restore** API

---

## 0) 全域規範

### 0.1 認證與標頭

- `Authorization: Bearer <JWT>`
- 追蹤：`X-Request-Id`（可選）
- 冪等：建議對 **新增/批次上傳** 支援 `Idempotency-Key`（伺服器需保存 key 與結果一段時間）

### 0.2 分頁/排序

- 參數：`page`、`size`、`sort=field,(asc|desc)`（JHipster 標準）
- 回應：`X-Total-Count` + `Link` header（第一頁/上一頁/下一頁/最後一頁）

### 0.3 軟刪/還原語意

- 軟刪：設 `deleted=true` 與 `deletedAt/by`；**不可被列表（預設條件）取回**
- 還原：設 `deleted=false`；**若 code 與現存（deleted=false）衝突 → 400**

### 0.4 唯一鍵規則（非常重要）

- **規則**：在 `deleted=false` 範圍，**`LOWER(code)` 全域唯一**
- **適用**：`Customer.customerNo`、`Contact.contactNo`、（可延伸至）Brand、UoM、Item…等
- **錯誤格式**（Problem+JSON）：

  ```json
  {
    "type": "https://www.jhipster.tech/problem/problem-with-message",
    "title": "code.duplicate",
    "status": 400,
    "entityName": "customer",
    "errorKey": "code.duplicate",
    "message": "error.code.duplicate"
  }
  ```

### 0.5 常見錯誤碼

- `400 BadRequest`：參數錯誤 / 唯一鍵衝突（`code.duplicate`）
- `404 NotFound`：目標不存在或已軟刪
- `409 Conflict`：批次/同步處理的版本衝突
- `422 UnprocessableEntity`：業務規則未滿足（例如：價表時效性、UoM 不可換算）
- `429 TooManyRequests`：節流
- `500`：非預期錯誤（帶追蹤 ID）

---

## 1) 定價與價表（Pricing）

### 1.1 API 一覽表（含 Controller）

| 方法與路徑                                                            | 中文名稱           | 功能用途                                                                                         | 使用場景 / 呼叫對象                          | Controller 類別                   |
| --------------------------------------------------------------------- | ------------------ | ------------------------------------------------------------------------------------------------ | -------------------------------------------- | --------------------------------- |
| **POST `/api/pricing/preview`**                                       | 價格試算（多行）   | 依輸入（客戶/群組/通路/幣別/日期/SKU/數量/UoM/稅別）執行定價引擎，回傳行單價、稅額、總額與折扣。 | Quotation / SO / DN 建立或重算；管理端試算頁 | `PricingResource`                 |
| **GET `/api/pricing/price-lists/applicable`**                         | 查詢可適用價目表   | 依 Assignment 規則排序回傳（CUSTOMER→GROUP→CHANNEL→DEFAULT；priority 升冪）。                    | 建立單據前帶出建議價目表                     | `PricingResource`                 |
| **GET `/api/pricing/traces`**                                         | 分頁查詢試算追蹤   | 依時間/關鍵字查詢 `PriceCalcTrace`。                                                             | 稽核/除錯                                    | `PricingResource`                 |
| **GET `/api/pricing/traces/{traceNo}`**                               | 取得試算追蹤       | 查詢單一 trace（遮罩敏感欄位）。                                                                 | 稽核/除錯                                    | `PricingResource`                 |
| **POST `/api/pricing/reprice`**                                       | 重算（Reprice）    | 以 Pricing 請求重算定價結果。                                                                    | 後台重算                                     | `PricingResource`                 |
| **GET `/api/price-lists`**                                            | 查詢價目表         | 關鍵字/幣別/通路分頁查詢。                                                                       | 管理端列表                                   | `PriceListResource`               |
| **POST `/api/price-lists`**                                           | 新增價目表         | 建立價目表基本資訊（支援 `Idempotency-Key`）。                                                   | 管理端                                       | `PriceListResource`               |
| **PUT `/api/price-lists/{id}`**                                       | 更新價目表         | 編輯名稱、通路、期間、屬性。                                                                     | 管理端                                       | `PriceListResource`               |
| **DELETE `/api/price-lists/{id}`**                                    | 刪除價目表（軟刪） | 標記 deleted。                                                                                   | 管理端                                       | `PriceListResource`               |
| **GET `/api/price-lists/{id}`**                                       | 取得價目表         | 讀取單一價目表。                                                                                 | 管理端                                       | `PriceListResource`               |
| **GET `/api/price-lists/{id}/preview`**                               | 指定價目表預覽     | 鎖定價表取樣 SKU 做試算。                                                                        | 管理端驗證價表                               | `ItemPriceMaintenanceResource`    |
| **POST `/api/price-lists/{id}/items:bulk-upsert`**                    | 價表明細批次上傳   | JSON/CSV 批次維護階梯價格（含 UoM/稅別）。                                                       | 管理端大量調價                               | `ItemPriceMaintenanceResource`    |
| **POST `/api/price-lists/{id}/skus/{skuId}/item-prices/bulk-upsert`** | 單 SKU 批次上傳    | 維護單一 SKU 的多階梯價。                                                                        | 管理端                                       | `ItemPriceMaintenanceResource`    |
| **POST `/api/price-lists/{id}/assignments:sync`**                     | 同步價目表指派     | 依前端排序（D&D）重寫 priority=1..N；驗證所屬價表一致。                                          | 管理端                                       | `PriceListAssignmentSyncResource` |
| **GET `/api/brands`**                                                 | 查詢品牌           | 分頁/條件查詢 Brand。                                                                            | 管理端品牌總管                               | `BrandResource`                   |
| **POST `/api/brands`**                                                | 新增品牌           | 建立品牌主檔。                                                                                   | 管理端                                       | `BrandResource`                   |
| **PUT `/api/brands/{id}`**                                            | 更新品牌           | 編輯品牌資訊。                                                                                   | 管理端                                       | `BrandResource`                   |
| **DELETE `/api/brands/{id}`**                                         | 刪除品牌           | 軟刪（建議加上被參照檢查）。                                                                     | 管理端                                       | `BrandResource`                   |
| **GET `/api/item-groups`**                                            | 查詢產品群組       | 支援 by parentId、關鍵字。                                                                       | 類別管理                                     | `ItemGroupResource`               |
| **POST `/api/item-groups`**                                           | 新增產品群組       | 建立層級節點。                                                                                   | 類別管理                                     | `ItemGroupResource`               |
| **PUT `/api/item-groups/{id}`**                                       | 更新產品群組       | 調整名稱、父節點、排序。                                                                         | 類別管理                                     | `ItemGroupResource`               |
| **DELETE `/api/item-groups/{id}`**                                    | 刪除產品群組       | 軟刪（禁止刪除有子節點/被 Item 參照）。                                                          | 類別管理                                     | `ItemGroupResource`               |
| **GET `/api/items`**                                                  | 查詢產品           | 依品牌/群組/關鍵字。                                                                             | 產品主檔                                     | `ItemResource`                    |
| **POST `/api/items`**                                                 | 新增產品           | 建立 Item 主檔。                                                                                 | 產品主檔                                     | `ItemResource`                    |
| **PUT `/api/items/{id}`**                                             | 更新產品           | 編輯產品資訊。                                                                                   | 產品主檔                                     | `ItemResource`                    |
| **DELETE `/api/items/{id}`**                                          | 刪除產品           | 軟刪（有 SKU 時禁止）。                                                                          | 產品主檔                                     | `ItemResource`                    |
| **GET `/api/uoms`**                                                   | 查詢單位           | 分頁/條件查詢 UoM。                                                                              | 基礎主檔                                     | `UomResource`                     |
| **POST `/api/uoms`**                                                  | 新增單位           | 建立 UoM。                                                                                       | 基礎主檔                                     | `UomResource`                     |
| **PUT `/api/uoms/{id}`**                                              | 更新單位           | 編輯 UoM。                                                                                       | 基礎主檔                                     | `UomResource`                     |
| **DELETE `/api/uoms/{id}`**                                           | 刪除單位           | 軟刪（被使用時禁止）。                                                                           | 基礎主檔                                     | `UomResource`                     |
| **GET `/api/uom-conversions`**                                        | 查詢換算           | 依 from/to/期間查詢換算率。                                                                      | 定價/庫存計量                                | `UomConversionResource`           |
| **POST `/api/uom-conversions`**                                       | 新增換算           | 建立 `(from→to)` 換算率。                                                                        | 定價/庫存計量                                | `UomConversionResource`           |
| **PUT `/api/uom-conversions/{id}`**                                   | 更新換算           | 編輯換算與有效期。                                                                               | 定價/庫存計量                                | `UomConversionResource`           |
| **DELETE `/api/uom-conversions/{id}`**                                | 刪除換算           | 軟刪（避免中斷已用路徑）。                                                                       | 定價/庫存計量                                | `UomConversionResource`           |

#### 1.2 價表批次上傳（CSV）格式

```csv
skuId;uomCode;minQty;unitPrice;taxCode
1001;;0;100.000000;TWN_VAT_5
1001;;10;95.000000;TWN_VAT_5
```

- `uomCode` 可留空代表 SKU 基本單位
- 金額建議 `DECIMAL(19,6)`；最終呈現可四捨五入 2 位
- 稅別依據系統 `tax_code` 參照

#### 1.3 幣別與價型

- `price_list.currency_code` 必須與試算請求相容
- `price_type` 控制儲存價是否含稅（引擎計算時要對準）

---

## 2) Phase 2：CRM（Customer / Contact / Address）

### 2.1 Address

| 方法   | 路徑                  | 中文名稱   | 用途              | Controller        |
| ------ | --------------------- | ---------- | ----------------- | ----------------- |
| POST   | `/api/addresses`      | 新增地址   | 建立 Address 主檔 | `AddressResource` |
| GET    | `/api/addresses`      | 查詢地址   | 分頁/條件查詢     | `AddressResource` |
| GET    | `/api/addresses/{id}` | 讀取地址   | 讀單筆            | `AddressResource` |
| PATCH  | `/api/addresses/{id}` | 局部更新   | Merge-Patch       | `AddressResource` |
| PUT    | `/api/addresses/{id}` | 覆寫更新   | 全量更新          | `AddressResource` |
| DELETE | `/api/addresses/{id}` | **硬刪除** | 物理刪除（謹慎）  | `AddressResource` |

> 若需軟刪/還原，可追加：`POST /api/addresses/{id}:soft-delete`、`POST /api/addresses/{id}:restore`

---

### 2.2 Customer

| 方法     | 路徑                                  | 中文名稱         | 用途                                                                               | 備註                                                   | Controller         |
| -------- | ------------------------------------- | ---------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------ | ------------------ |
| POST     | `/api/customers`                      | 新增客戶         | 建立 Customer                                                                      | 儲存前檢查 `LOWER(customerNo)` 唯一（`deleted=false`） | `CustomerResource` |
| GET      | `/api/customers`                      | 查詢客戶         | 分頁/條件查詢                                                                      | 依 Criteria                                            | `CustomerResource` |
| GET      | `/api/customers/{id}`                 | 讀取客戶         | 讀單筆                                                                             | —                                                      | `CustomerResource` |
| PATCH    | `/api/customers/{id}`                 | 局部更新         | Merge-Patch                                                                        | 儲存前檢查唯一                                         | `CustomerResource` |
| PUT      | `/api/customers/{id}`                 | 覆寫更新         | 全量更新                                                                           | 儲存前檢查唯一                                         | `CustomerResource` |
| DELETE   | `/api/customers/{id}`                 | **硬刪除**       | 物理刪除                                                                           | 謹慎使用                                               | `CustomerResource` |
| **POST** | **`/api/customers/{id}:soft-delete`** | **軟刪除**       | 設 `deleted=true`、填 `deletedAt/by`                                               | 新增                                                   | `CustomerResource` |
| **POST** | **`/api/customers/{id}:restore`**     | **還原**         | 設 `deleted=false`（**遇碼重複→400**）                                             | 新增                                                   | `CustomerResource` |
| **GET**  | **`/api/customers/_exists`**          | **代碼唯一檢查** | `?customerNo=...` 或 `?customerNo=...&excludeId=123` 回 `{ "exists": true/false }` | 大小寫不分、只看 `deleted=false`                       | `CustomerResource` |

---

### 2.3 Contact

| 方法     | 路徑                                 | 中文名稱         | 用途                                                                             | 備註                                                  | Controller        |
| -------- | ------------------------------------ | ---------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------- | ----------------- |
| POST     | `/api/contacts`                      | 新增聯絡人       | 建立 Contact                                                                     | 儲存前檢查 `LOWER(contactNo)` 唯一（`deleted=false`） | `ContactResource` |
| GET      | `/api/contacts`                      | 查詢聯絡人       | 分頁/條件查詢                                                                    | 依 Criteria                                           | `ContactResource` |
| GET      | `/api/contacts/lookup`               | 聯絡人下拉       | `?keyword=&customerId=&limit=` 提供 1~100 筆精簡清單，僅顯示未刪除且符合 Owner 能見度 | 預設 limit=20、支援 `fullName/contactNo/email/phone/mobile` 模糊搜尋 | `ContactResource` |
| GET      | `/api/contacts/{id}`                 | 讀取聯絡人       | 讀單筆                                                                           | —                                                     | `ContactResource` |
| PATCH    | `/api/contacts/{id}`                 | 局部更新         | Merge-Patch                                                                      | 儲存前檢查唯一                                        | `ContactResource` |
| PUT      | `/api/contacts/{id}`                 | 覆寫更新         | 全量更新                                                                         | 儲存前檢查唯一                                        | `ContactResource` |
| DELETE   | `/api/contacts/{id}`                 | **硬刪除**       | 物理刪除                                                                         | 謹慎使用                                              | `ContactResource` |
| **POST** | **`/api/contacts/{id}:soft-delete`** | **軟刪除**       | 設 `deleted=true`、填 `deletedAt/by`                                             | 新增                                                  | `ContactResource` |
| **POST** | **`/api/contacts/{id}:restore`**     | **還原**         | 設 `deleted=false`（**遇碼重複→400**）                                           | 新增                                                  | `ContactResource` |
| **GET**  | **`/api/contacts/_exists`**          | **代碼唯一檢查** | `?contactNo=...` 或 `?contactNo=...&excludeId=123` 回 `{ "exists": true/false }` | 大小寫不分、只看 `deleted=false`                      | `ContactResource` |

---

## 3) 參考請求格式

### 3.1 價格試算（多行）

```http
POST /api/pricing/preview
Content-Type: application/json
Authorization: Bearer <JWT>

{
  "currency": "TWD",
  "priceDate": "2025-10-30",
  "customerNo": "C-001",
  "channelCode": "B2B",
  "lines": [
    { "skuId": 1001, "uomCode": null, "qty": 12, "taxCode": "TWN_VAT_5" },
    { "skuId": 2009, "uomCode": "BOX", "qty": 3, "taxCode": "TWN_VAT_5" }
  ]
}
```

回應（摘要）：

```json
{
  "traceNo": "PRC-20251030-000123",
  "currency": "TWD",
  "lines": [
    { "skuId": 1001, "uomCode": "EA", "qty": 12, "unitPrice": 95.0, "net": 1140.0, "tax": 57.0, "gross": 1197.0, "appliedPriceListId": 17 },
    { "skuId": 2009, "uomCode": "BOX", "qty": 3, "unitPrice": 320.0, "net": 960.0, "tax": 48.0, "gross": 1008.0, "appliedPriceListId": 17 }
  ]
}
```

### 3.2 價表明細批次上傳（CSV）

```http
POST /api/price-lists/17/items:bulk-upsert
Content-Type: text/csv
Idempotency-Key: 20251030-PL17

skuId;uomCode;minQty;unitPrice;taxCode
1001;;0;100.000000;TWN_VAT_5
1001;;10;95.000000;TWN_VAT_5
```

---

## 4) 版本與變更紀錄

- **2025-10-30**

  - 新增 `_exists` / `_:exists` 說明與範例
  - 明確化 **唯一鍵規則** 與還原衝突錯誤格式
  - 彙整 Pricing + CRM API 成單一文件，補上批次上傳 CSV 規範
  - 增補冪等建議與錯誤碼表

- **2025-11-04**

  - 補充「🧭 文件關係與衝突解決原則」尾註；與最終 API 清單建立主從關係

---

## 🧭 文件關係與衝突解決原則

| 項目                                  | 主檔（權威來源）                                | 說明                                                 |
| ------------------------------------- | ----------------------------------------------- | ---------------------------------------------------- |
| API 方法 / 路徑 / Controller 類別     | **`/docs/api/final/API 清單（含用途說明）.md`** | 作為「唯一真相來源」（SSOT），所有端點簽章以此為主。 |
| API 行為 / 規則 / 錯誤碼 / 範例 / CSV | **本文件**                                      | 作為行為層說明與測試依據，更新時應同步修清單備註。   |
| 發現不一致時                          | **以清單為先，規格為輔**                        | 先修清單 → 再補規格；若為行為邏輯變動則相反。        |
| 維護責任                              | API Owner / Backend Team                        | PR 務必勾選：`[ ] 已同步更新清單與規格文件`。        |

---

## 5) Sales / Quotation（Phase 4）

### 5.1 Workflow 事件 API 一覽

| 方法與路徑                                               | 說明                                                                                  | Controller          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------- |
| **POST `/api/quotations/{threadId}/events/{eventCode}`** | 依 `quotation_state_transition` 驗證守衛後執行事件並寫入 `quotation_status_history`。 | `QuotationResource` |
| **GET `/api/quotations/{threadId}/events/options`**      | 依最新 Revision 狀態列出可執行事件、目標狀態與 guard 需求供 Workflow Drawer 使用。    | `QuotationResource` |

### 5.2 `QuotationWorkflowOptionDTO`

- `eventCode / eventName`：事件代碼與名稱（來源：`quotation_event_def`）。
- `toStatusCode / toStatusName`：對應的目標狀態。
- `requiresLineItems`：是否需至少一筆行項（SEND / APPROVE 等）。
- `requiresChannel`：是否需指定 `channel`（SEND）。
- `requiresReason`：是否需輸入 `reason`（REJECT / CANCEL / LOST）。
- `requiresValidUntil`：是否需檢查 `validUntil` 未逾期（APPROVE / ACCEPT）。

> 前端 Pipeline 與 Workflow Drawer 僅需依此 DTO 判斷事件與必填欄位，避免硬編事件清單；當狀態機新增事件時能即時反映。

---

- ### Stock Availability API

  - `GET /api/stock/available`：依 `skuId + warehouseId`（可選 `binId/lotNo`）取得單點存量，回傳 `StockAvailabilityDTO`，欄位含 onHand / reserved / available / stockStatus / lastCost / averageCost / breakdown 與 `asOf`。
  - `GET /api/stock/available/by-warehouse`：指定 `skuId`，依倉庫彙總 onHand/reserved/available，`status=ALL`。可透過 `includeQuarantine=true` 額外呈現 QUARANTINE/HOLD 層。
  - 兩個端點共用 Redis/JCache 快取（`stock.available.point`、`stock.available.byWarehouse`）；所有寫入 API 完成後由 `InventoryCacheService` 精準驅逐對應 key，確保下一次查詢即為最新值。
  - 錯誤碼：`404`（SKU 或倉庫不存在）、`400`（缺少必要參數）、`403`（資料權限）。

- ### Inventory Valuation API

  - `GET /api/inventory/valuation/by-sku?skuId=&warehouseId=&lotNo?=`：回傳 `SkuValuationDTO`，包含 `lastCost`（最新成本層單價）與 `averageCost`（現值加權平均成本）。`lotNo` 可選，用於特定批次估值。
  - 估值結果緩存在 `valuation.sku` cache；當收貨、出庫、調整或成本調整觸發 `inventoryEventPublisher.publishValuationChanged` 時，由 `InventoryCacheService.evictValuation` 驅逐，使查詢反映最新層資料。
  - 若查無成本層，兩欄位皆為 `null`，前端應提供預設顯示。

- ### Inventory Posting API 更新
  - `POST /api/inventory/transactions/receipt|issue|adjust` 改以 `@IdempotentEndpoint` 進行冪等控制，請求需附 `Idempotency-Key` 才會建立 `(endpoint, key)` 記錄並在重送時回放。
  - 新增 `POST /api/inventory/transactions/reclassify`：專責處理 QUARANTINE/HOLD → AVAILABLE 等狀態轉換，payload 為 `{ skuId, warehouseId, binId?, qty, fromStatus, toStatus, note }`，成功後寫入 `InventoryTxType.RECLASSIFY` 交易與對應 `StockByBin`。
  - 查詢型 API（可用量、估值）採 Redis/JCache 快取，cache key 為 `stock.available.point|warehouse`、`valuation.sku`。所有寫入 API 完成後由 `InventoryCacheService` 精準驅逐，以確保下一次查詢即為最新值。
