---
codex:
  purpose: 'API 索引與用途對照表（最終清單 / SSOT）'
  context: '所有 API 的 method/path/controller 變更，必須先同步至本清單；Agent 以此為端點來源'
  lastUpdated: 2025-11-04
  language: zh-TW
  schema:
    columns:
      - method_path
      - name_zh
      - purpose
      - usage_context
      - controller_class
      - remarks
---

# Flexora 最終 API 清單（含用途說明）

> **維護規則**
>
> - 任何 API 的 **新增/修改（method、path、Controller 類別）**，**必須**立即更新此表
> - 與規格檔（`/docs/api/*`）不一致時，以此清單為準，並回補該模組規格
> - 建議 PR 加入 changelog：`docs: api list updated`

## 1) Pricing & Price List

| 方法與路徑                                    | 中文名稱         | 功能用途                                                                          | 使用場景 / 呼叫對象                          | Controller 類別名稱 | 備註                                   |
| --------------------------------------------- | ---------------- | --------------------------------------------------------------------------------- | -------------------------------------------- | ------------------- | -------------------------------------- |
| **POST `/api/pricing/preview`**               | 價格試算（多行） | 依客戶/通路/幣別/日期/SKU/數量/UoM/稅別執行定價引擎，回傳行單價、稅額、總額與折扣 | Quotation / SO / DN 建立或重算；管理端試算頁 | `PricingResource`   | 支援 `Idempotency-Key`；回傳 `traceNo` |
| **GET `/api/pricing/price-lists/applicable`** | 查詢可適用價目表 | 依 Assignment 規則排序（CUSTOMER→GROUP→CHANNEL→DEFAULT；priority 升冪）           | 建單前帶出建議價目表                         | `PricingResource`   |                                        |
| **GET `/api/pricing/traces`**                 | 分頁查詢試算追蹤 | 依時間/關鍵字查 `PriceCalcTrace`                                                  | 稽核/除錯                                    | `PricingResource`   |                                        |
| **GET `/api/pricing/traces/{traceNo}`**       | 取得試算追蹤     | 單一 trace（遮罩敏感欄位）                                                        | 稽核/除錯                                    | `PricingResource`   |                                        |
| **POST `/api/pricing/reprice`**               | 重算（Reprice）  | 以 Pricing 請求重算定價結果                                                       | 後台重算                                     | `PricingResource`   |                                        |

### 1.1 Price List / Item Price Maintenance / Assignment

| 方法與路徑                                                            | 中文名稱           | 功能用途                                       | 使用場景 / 呼叫對象 | Controller 類別名稱               | 備註             |
| --------------------------------------------------------------------- | ------------------ | ---------------------------------------------- | ------------------- | --------------------------------- | ---------------- |
| **GET `/api/price-lists`**                                            | 查詢價目表         | 關鍵字/幣別/通路分頁查詢                       | 管理端列表          | `PriceListResource`               |                  |
| **POST `/api/price-lists`**                                           | 新增價目表         | 建立價目表（支援 `Idempotency-Key`）           | 管理端              | `PriceListResource`               |                  |
| **PUT `/api/price-lists/{id}`**                                       | 更新價目表         | 名稱、通路、期間、屬性                         | 管理端              | `PriceListResource`               |                  |
| **DELETE `/api/price-lists/{id}`**                                    | 刪除價目表（軟刪） | 標記 `deleted=true`                            | 管理端              | `PriceListResource`               |                  |
| **GET `/api/price-lists/{id}`**                                       | 取得價目表         | 讀單一價目表                                   | 管理端              | `PriceListResource`               |                  |
| **GET `/api/price-lists/{id}/preview`**                               | 指定價目表預覽     | 鎖定價表取樣 SKU 做試算                        | 管理端驗證價表      | `ItemPriceMaintenanceResource`    |                  |
| **POST `/api/price-lists/{id}/items:bulk-upsert`**                    | 價表明細批次上傳   | JSON/CSV 維護階梯價格（含 UoM/稅別）           | 管理端大量調價      | `ItemPriceMaintenanceResource`    | CSV 規格見規格檔 |
| **POST `/api/price-lists/{id}/skus/{skuId}/item-prices/bulk-upsert`** | 單 SKU 批次上傳    | 維護單一 SKU 多階梯價                          | 管理端              | `ItemPriceMaintenanceResource`    |                  |
| **POST `/api/price-lists/{id}/assignments:sync`**                     | 同步價目表指派     | 依前端排序重寫 priority=1..N；驗證所屬價表一致 | 管理端              | `PriceListAssignmentSyncResource` |                  |

### 1.2 基礎主檔（Brand / ItemGroup / Item / UoM / UoM Conversion）

| 方法與路徑                             | 中文名稱     | 功能用途                      | 使用場景 / 呼叫對象 | Controller 類別名稱     | 備註 |
| -------------------------------------- | ------------ | ----------------------------- | ------------------- | ----------------------- | ---- |
| **GET `/api/brands`**                  | 查詢品牌     | 分頁/條件查詢 Brand           | 管理端品牌總管      | `BrandResource`         |      |
| **POST `/api/brands`**                 | 新增品牌     | 建立品牌主檔                  | 管理端              | `BrandResource`         |      |
| **PUT `/api/brands/{id}`**             | 更新品牌     | 編輯品牌資訊                  | 管理端              | `BrandResource`         |      |
| **DELETE `/api/brands/{id}`**          | 刪除品牌     | 軟刪（建議檢查被參照）        | 管理端              | `BrandResource`         |      |
| **GET `/api/item-groups`**             | 查詢產品群組 | 支援 by parentId、關鍵字      | 類別管理            | `ItemGroupResource`     |      |
| **POST `/api/item-groups`**            | 新增產品群組 | 建立層級節點                  | 類別管理            | `ItemGroupResource`     |      |
| **PUT `/api/item-groups/{id}`**        | 更新產品群組 | 名稱、父節點、排序            | 類別管理            | `ItemGroupResource`     |      |
| **DELETE `/api/item-groups/{id}`**     | 刪除產品群組 | 軟刪（禁止刪有子節點/被參照） | 類別管理            | `ItemGroupResource`     |      |
| **GET `/api/items`**                   | 查詢產品     | 依品牌/群組/關鍵字            | 產品主檔            | `ItemResource`          |      |
| **POST `/api/items`**                  | 新增產品     | 建立 Item 主檔                | 產品主檔            | `ItemResource`          |      |
| **PUT `/api/items/{id}`**              | 更新產品     | 編輯產品資訊                  | 產品主檔            | `ItemResource`          |      |
| **DELETE `/api/items/{id}`**           | 刪除產品     | 軟刪（有 SKU 禁止）           | 產品主檔            | `ItemResource`          |      |
| **GET `/api/uoms`**                    | 查詢單位     | 分頁/條件查詢 UoM             | 基礎主檔            | `UomResource`           |      |
| **POST `/api/uoms`**                   | 新增單位     | 建立 UoM                      | 基礎主檔            | `UomResource`           |      |
| **PUT `/api/uoms/{id}`**               | 更新單位     | 編輯 UoM                      | 基礎主檔            | `UomResource`           |      |
| **DELETE `/api/uoms/{id}`**            | 刪除單位     | 軟刪（被使用時禁止）          | 基礎主檔            | `UomResource`           |      |
| **GET `/api/uom-conversions`**         | 查詢換算     | 依 from/to/期間查換算率       | 定價/庫存計量       | `UomConversionResource` |      |
| **POST `/api/uom-conversions`**        | 新增換算     | 建立 `(from→to)` 換算率       | 定價/庫存計量       | `UomConversionResource` |      |
| **PUT `/api/uom-conversions/{id}`**    | 更新換算     | 編輯換算與有效期              | 定價/庫存計量       | `UomConversionResource` |      |
| **DELETE `/api/uom-conversions/{id}`** | 刪除換算     | 軟刪（避免中斷已用路徑）      | 定價/庫存計量       | `UomConversionResource` |      |

## 2) CRM（Customer / Contact / Address）

### 2.1 Address

| 方法與路徑                       | 中文名稱   | 功能用途          | 使用場景 / 呼叫對象 | Controller 類別名稱 | 備註             |
| -------------------------------- | ---------- | ----------------- | ------------------- | ------------------- | ---------------- |
| **POST `/api/addresses`**        | 新增地址   | 建立 Address 主檔 | 基礎主檔            | `AddressResource`   |                  |
| **GET `/api/addresses`**         | 查詢地址   | 分頁/條件查詢     | 基礎主檔            | `AddressResource`   |                  |
| **GET `/api/addresses/{id}`**    | 讀取地址   | 讀單筆            | 基礎主檔            | `AddressResource`   |                  |
| **PATCH `/api/addresses/{id}`**  | 局部更新   | Merge-Patch       | 基礎主檔            | `AddressResource`   |                  |
| **PUT `/api/addresses/{id}`**    | 覆寫更新   | 全量更新          | 基礎主檔            | `AddressResource`   |                  |
| **DELETE `/api/addresses/{id}`** | **硬刪除** | 物理刪除（謹慎）  | 基礎主檔            | `AddressResource`   | 可選改為軟刪 API |

### 2.2 Customer

| 方法與路徑                                 | 中文名稱     | 功能用途                                                          | 使用場景 / 呼叫對象 | Controller 類別名稱 | 備註                                                   |
| ------------------------------------------ | ------------ | ----------------------------------------------------------------- | ------------------- | ------------------- | ------------------------------------------------------ |
| **POST `/api/customers`**                  | 新增客戶     | 建立 Customer                                                     | 客戶主檔            | `CustomerResource`  | 儲存前檢查 `LOWER(customerNo)` 唯一（`deleted=false`） |
| **GET `/api/customers`**                   | 查詢客戶     | 分頁/條件查詢（Criteria）                                         | 客戶主檔            | `CustomerResource`  |                                                        |
| **GET `/api/customers/{id}`**              | 讀取客戶     | 讀單筆                                                            | 客戶主檔            | `CustomerResource`  |                                                        |
| **PATCH `/api/customers/{id}`**            | 局部更新     | Merge-Patch                                                       | 客戶主檔            | `CustomerResource`  |                                                        |
| **PUT `/api/customers/{id}`**              | 覆寫更新     | 全量更新                                                          | 客戶主檔            | `CustomerResource`  |                                                        |
| **DELETE `/api/customers/{id}`**           | **硬刪除**   | 物理刪除（謹慎）                                                  | 客戶主檔            | `CustomerResource`  | 建議改為軟刪                                           |
| **POST `/api/customers/{id}:soft-delete`** | 軟刪除       | 設 `deleted=true`、填 `deletedAt/by`                              | 客戶主檔            | `CustomerResource`  |                                                        |
| **POST `/api/customers/{id}:restore`**     | 還原         | 設 `deleted=false`（遇碼重複→400）                                | 客戶主檔            | `CustomerResource`  |                                                        |
| **GET `/api/customers/_exists`**           | 代碼唯一檢查 | `?customerNo=...` / `&excludeId=...` → `{ "exists": true/false }` | 客戶主檔            | `CustomerResource`  | 大小寫不分、只看 `deleted=false`                       |

### 2.3 Contact

| 方法與路徑                                | 中文名稱     | 功能用途                                                         | 使用場景 / 呼叫對象 | Controller 類別名稱 | 備註                                                  |
| ----------------------------------------- | ------------ | ---------------------------------------------------------------- | ------------------- | ------------------- | ----------------------------------------------------- |
| **POST `/api/contacts`**                  | 新增聯絡人   | 建立 Contact                                                     | 聯絡人主檔          | `ContactResource`   | 儲存前檢查 `LOWER(contactNo)` 唯一（`deleted=false`） |
| **GET `/api/contacts`**                   | 查詢聯絡人   | 分頁/條件查詢（Criteria）                                        | 聯絡人主檔          | `ContactResource`   |                                                       |
| **GET `/api/contacts/{id}`**              | 讀取聯絡人   | 讀單筆                                                           | 聯絡人主檔          | `ContactResource`   |                                                       |
| **PATCH `/api/contacts/{id}`**            | 局部更新     | Merge-Patch                                                      | 聯絡人主檔          | `ContactResource`   |                                                       |
| **PUT `/api/contacts/{id}`**              | 覆寫更新     | 全量更新                                                         | 聯絡人主檔          | `ContactResource`   |                                                       |
| **DELETE `/api/contacts/{id}`**           | **硬刪除**   | 物理刪除（謹慎）                                                 | 聯絡人主檔          | `ContactResource`   | 建議改為軟刪                                          |
| **POST `/api/contacts/{id}:soft-delete`** | 軟刪除       | 設 `deleted=true`、填 `deletedAt/by`                             | 聯絡人主檔          | `ContactResource`   |                                                       |
| **POST `/api/contacts/{id}:restore`**     | 還原         | 設 `deleted=false`（遇碼重複→400）                               | 聯絡人主檔          | `ContactResource`   |                                                       |
| **GET `/api/contacts/_exists`**           | 代碼唯一檢查 | `?contactNo=...` / `&excludeId=...` → `{ "exists": true/false }` | 聯絡人主檔          | `ContactResource`   | 大小寫不分、只看 `deleted=false`                      |

## 2) Inventory Management（IM）

| 方法與路徑                                          | 中文名稱         | 功能用途                                                                    | 使用場景 / 呼叫對象                 | Controller 類別名稱                   | 備註                                                                                     |
| --------------------------------------------------- | ---------------- | --------------------------------------------------------------------------- | ----------------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------- |
| **GET `/api/warehouses/_filter`**                   | 倉庫輕量查詢     | 依代號/名稱/啟用/Bin 追蹤開關（含軟刪除選項）取得倉庫下拉資料               | SO/PO/IM 表單倉庫選擇；設定畫面     | `WarehouseResource`                   | 忽略資料權限時仍需搭配 `@DataScoped`；結果未分頁                                         |
| **GET `/api/bins/_filter`**                         | 倉位輕量查詢     | 依倉庫/倉位代號/類型/啟用/軟刪除取得倉位列表                                | SO/PO/IM 表單倉位選擇；尋找可用倉位 | `BinResource`                         | 會檢查倉庫是否啟用 Bin 追蹤；結果未分頁                                                  |
| **GET `/api/stock/available`**                      | 單點可用量查詢   | 依 SKU×倉庫（可選倉位/批號）回傳 onHand/reserved/available 及最新成本資訊   | 建立 SO/PO、Dashboard 即時查詢      | `StockAvailabilityResource`           | `includeQuarantine` 控制是否納入 QUARANTINE/HOLD；結果含 `StockAvailabilityDTO` 詳細欄位 |
| **GET `/api/stock/available/by-warehouse`**         | 倉別彙總可用量   | 依 SKU 對所有倉庫彙總可用量（ALL status），可選納入 QUARANTINE/HOLD 層      | 製造/補貨決策、跨倉調撥評估         | `StockAvailabilityResource`           | 回傳 `List<StockAvailabilityDTO>`；為減少負載支援 Redis 快取                             |
| **POST `/api/inventory/transactions/receipt`**      | 庫存收貨過帳     | 以倉庫/倉位/批序號過帳收貨量，支援 QUARANTINE 狀態與單位成本                | 進貨收貨、退貨入庫、盤盈入庫        | `InventoryPostingResource`            | 支援 `Idempotency-Key`；回傳 `InventoryTxResult`                                         |
| **POST `/api/inventory/transactions/issue`**        | 庫存出庫過帳     | 依可用量進行出庫扣減，遵循倉庫 `allowNegative` 設定                         | 銷貨出庫、領料、退貨出庫            | `InventoryPostingResource`            | 支援 `Idempotency-Key`；`allowNegative=false` 時不足量回 `im.insufficient`               |
| **POST `/api/inventory/transactions/adjust`**       | 庫存調整過帳     | 支援數量或成本調整（`mode=QUANTITY` / `COST_ADJUST`），建立對應交易與成本層 | 盤點調整、成本調整                  | `InventoryPostingResource`            | 支援 `Idempotency-Key`；`costDelta` 不一致時回 409                                       |
| **POST `/api/inventory/transactions/reclassify`**   | 庫存狀態轉換     | 將 QUARANTINE/HOLD 等狀態轉為 AVAILABLE，記錄 `RECLASSIFY` 交易並更新可用量 | IQC PASS、質檢放行                  | `InventoryPostingResource`            | 支援 `Idempotency-Key`；寫入後自動驅逐可用量快取                                         |
| **POST `/api/inventory/reservations`**              | 建立庫存預留     | 鎖定指定 SKU/倉庫/倉位可用量並建立預留單，提供剩餘可用量回饋                | SO / 預留指令 / WMS 同步            | `InventoryReservationCommandResource` | 支援 `Idempotency-Key`；預留成功落 `InventoryTransaction(RESERVATION)`                   |
| **POST `/api/inventory/reservations/{id}/release`** | 釋放預留         | 依預留單 ID 釋放保留量，可選填原因；支援冪等回放                            | 拋單取消、訂單結案                  | `InventoryReservationCommandResource` | 務必傳相同 body 以便冪等比對                                                             |
| **POST `/api/inventory/reservations/release`**      | 預留批次釋放     | 依 SKU/倉庫/類型/參考編號批次釋放，回傳釋放筆數與失敗列表                   | 系統批次、排程釋放                  | `InventoryReservationCommandResource` | 支援 `Idempotency-Key`；提供成功/失敗統計                                                |
| **GET `/api/inventory/reservations`**               | 預留查詢         | 分頁+條件查詢預留單，支援 `activeOnly` 快速篩選未取消/未履約                | 訂單追蹤、稽核                      | `InventoryReservationCommandResource` | 條件含 SKU、倉庫、預留類型、參考資料                                                     |
| **GET `/api/inventory/valuation/by-sku`**           | SKU 庫存估值查詢 | 回傳指定 SKU（可選倉庫）之移動平均與 FIFO 概況、剩餘層資訊                  | 財務/成本分析                       | `ValuationQueryResource`              | 回傳 `SkuValuationDTO`；`asOf` 取 `Instant.now()`                                        |

## 3) Sales / Quotation（Phase 4）

| 方法與路徑                                                                  | 中文名稱           | 功能用途                                                                               | 使用場景 / 呼叫對象        | Controller 類別名稱       | 備註                                                      |
| --------------------------------------------------------------------------- | ------------------ | -------------------------------------------------------------------------------------- | -------------------------- | ------------------------- | --------------------------------------------------------- |
| **POST `/api/quotations/preview`**                                          | 報價預覽           | 呼叫 Pricing 試算 + QuotationCalculationService，回傳行級與總額（不落地）              | 報價建立前預覽 / API 呼叫  | `QuotationResource`       | `properties` 需為 JSON；回傳行級含稅/未稅資訊             |
| **POST `/api/quotations`**                                                  | 建立報價草稿       | 建 Thread + 第 1 個 Revision，並落地 Items / Taxes                                     | 報價建立                   | `QuotationResource`       | 內部再跑一次 preview；`properties` 驗證 JSON              |
| **POST `/api/quotations/{threadId}/revisions`**                             | 新增報價版本       | 以既有 Thread 建下一版（複製上一版快照、重算 totals）                                  | 版控/協調                  | `QuotationResource`       | 僅允許最新 Revision 修改                                  |
| **POST `/api/quotations/{threadId}/events/{eventCode}`**                    | 報價事件           | 執行 send / approve / reject / cancel / expire 等狀態機事件，寫入歷史                  | Workflow / 守衛            | `QuotationResource`       | 依 state_transition 守衛；輸入 `QuotationEventRequestDTO` |
| **GET `/api/quotations/{threadId}/events/options`**                         | Workflow 可用事件  | 依 Thread 目前狀態列出可執行事件、目標狀態與 Guard 需求（行項/Channel/Reason/逾期）    | Workflow Drawer / Pipeline | `QuotationResource`       | 回傳 `QuotationWorkflowOptionDTO` 提供前端 UI 參考        |
| **POST `/api/quotations/{threadId}/revisions/{revisionId}:to-sales-order`** | 報價轉 SalesOrder  | 將 APPROVED 版本轉為 SalesOrder（支援部分/多次轉單），並新增 `SalesOrderQuotationLink` | 報價→訂單轉換              | `QuotationResource`       | 自動觸發 SO `SUBMIT`/`CONFIRM`；`items[]` 可指定行與數量  |
| **GET `/api/quotations/{threadId}/links`**                                  | 轉單紀錄清冊       | 查詢 Thread（或指定 Revision）對應的 SalesOrder 關聯紀錄                               | 前端顯示轉單歷史           | `QuotationResource`       | `revisionId` 選填；僅回傳 Link 摘要                       |
| **POST `/api/quotations/{threadId}/revisions/{revisionId}/addresses`**      | 更新地址快照       | 更新 Revision 的 billing/shipping `AddressSnapshot`                                    | 自訂地址 / CRM 同步        | `QuotationResource`       | DTO 採 `AddressSnapshot` 結構，僅 non-null 欄位會覆寫     |
| **POST `/api/quotations/{threadId}/revisions/{revisionId}/ext-attrs`**      | 更新 ExtAttr       | 依 `QuotationRevisionExtAttrDef` 批次寫入 `properties.extAttrs`                        | 報價客製欄位維護           | `QuotationResource`       | 後端依 dataType 驗證；空陣列或未帶值會回 400              |
| **POST `/api/quotations/{threadId}/documents`**                             | 新增附件連結       | 建立 Thread/Revision 與 Document 的 `DocumentLink`                                     | 報價附件 / PDF / 照片      | `QuotationResource`       | `documentId` 必填；`revisionId` 選填（缺省掛 Thread）     |
| **POST `/api/quotation-threads/{id}:soft-delete`**                          | 報價主線軟刪       | 僅更新 `deleted=true/deletedAt/deletedBy`，保留資料供還原                              | 後台手動刪除               | `QuotationThreadResource` | 需具備刪除權限；不影響已建立的 Revision                   |
| **POST `/api/quotation-threads/{id}:restore`**                              | 報價主線還原       | 取消軟刪並可選擇指定新的 `threadNo`（避免碰撞）                                        | 還原被刪除的報價           | `QuotationThreadResource` | `newThreadNo` 選填；若為空則沿用舊編號或重新給號          |
| **GET `/api/quotation-threads/_exists`**                                    | 查詢 threadNo 重複 | 檢查未刪除資料內是否存在相同 `threadNo`，提供前端即時驗證                              | 前端輸入驗證               | `QuotationThreadResource` | 支援 `excludeId` 用於更新情境                             |

## 4) Sales Order（Phase 5）

| 方法與路徑                                           | 中文名稱          | 功能用途                                                                                   | 使用場景 / 呼叫對象  | Controller 類別名稱  | 備註                                                       |
| ---------------------------------------------------- | ----------------- | ------------------------------------------------------------------------------------------ | -------------------- | -------------------- | ---------------------------------------------------------- |
| **POST `/api/sales-orders`**                         | 建立銷售訂單      | 建立 SalesOrder Header/Items；未帶 `orderNo` 時由 NumberingService 自動給號                | SO 建單 / API 呼叫   | `SalesOrderResource` | `properties` 驗證 JSON；回傳 DTO                           |
| **POST `/api/sales-orders/preview`**                 | 銷售訂單試算      | 依行項、整單折扣、運輸費用計算金額（不落地），回傳行級與總額                               | 建單前預覽 / 客製頁  | `SalesOrderResource` | 輸入 `SalesOrderCalculationRequestDTO`，回傳 summary       |
| **POST `/api/sales-orders/{id}:soft-delete`**        | 銷售訂單軟刪      | 僅更新 deleted 旗標，保留資料供還原                                                        | 後台刪單             | `SalesOrderResource` | 變更 `deleted/deletedAt/deletedBy`                         |
| **POST `/api/sales-orders/{id}:restore`**            | 銷售訂單還原      | 還原軟刪訂單，可視需要指定新的 `orderNo`                                                   | 還原示範/錯刪訂單    | `SalesOrderResource` | `newOrderNo` 選填；留空則沿用舊號或由後端重新給號          |
| **GET `/api/sales-orders/_exists`**                  | 查詢 orderNo 重複 | 檢查未刪除資料內是否已有該 `orderNo`                                                       | 前端輸入驗證         | `SalesOrderResource` | `excludeId` 可排除目前編輯中的訂單                         |
| **POST `/api/sales-orders/{id}/events/ship.update`** | DN 出貨回寫       | DeliveryNote 過帳後回寫 `shippedQuantity`、同步履約狀態並觸發 `PARTIAL_SHIP/SHIP` workflow | 出貨後回寫 / DN 模組 | `SalesOrderResource` | body 為 `SalesOrderShipmentUpdateRequestDTO`；檢查不可超量 |

---

## 🧭 文件關係與衝突解決原則

| 項目                                  | 主檔（權威來源）                                  | 說明                                                 |
| ------------------------------------- | ------------------------------------------------- | ---------------------------------------------------- |
| API 方法 / 路徑 / Controller 類別     | **`/docs/api/final/API 清單（含用途說明）.md`**   | 作為「唯一真相來源」（SSOT），所有端點簽章以此為主。 |
| API 行為 / 規則 / 錯誤碼 / 範例 / CSV | **`/docs/api/Flexora ERP API 規格（彙整版）.md`** | 作為行為層說明與測試依據，更新時應同步修清單備註。   |
| 發現不一致時                          | **以清單為先，規格為輔**                          | 先修清單 → 再補規格；若為行為邏輯變動則相反。        |
| 維護責任                              | API Owner / Backend Team                          | PR 務必勾選：`[ ] 已同步更新清單與規格文件`。        |
