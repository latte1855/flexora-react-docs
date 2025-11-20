# Phase 0 — 系統基礎導向（Data Scope / System Setting / Tax / Document）說明

本檔以 `docs/migration/flexora-react/api/phase0-system-foundation.md` 為基礎，整理「資料權限」「系統設定」「稅務」「文件管理」等 Phase 0 核心行為，並附上真正程式碼位置做為驗證來源（如 `com.asynctide.flexora.security.datascope.DataScopeService`、`SystemSettingService`、`TaxService`、`DocumentResource`）。

## 1. Definition of Done（可交付條件）

以下功能為 Phase 0 最小可交付成果，實作完畢才可稱為「完成 Phase 0」：

1. Owner/Department/Team 三層的 CRUD 與階層維護（含 `parentHierarchy`／`depth`）。
2. 使用者可查詢「可見 owner」的資料域 (`v_owner_visible`)；API/Service 皆能拿到 `DataScopeContext` 並自動套用。
3. 所有業務實體預設帶 `owner_id`，列表/查詢會套用資料域；只有 `ROLE_ADMIN` 或明確 bypass header 才可忽略。
4. `SystemSetting` CRUD + Redis 快取，提供 `getInt/getJson/getDecimal` 便利查詢並能驗證 `valueType`。
5. `TaxCode`、`TaxRateLine` 與 `TaxService` 提供預覽/應用 API（`POST /api/taxes/preview`）與複合稅計算。
6. `Document`/`DocumentLink` 支援上傳、下載、關聯綁定且遵循資料域檢查；可切換 `FileStorage` 實作（Local / S3）。
7. OpenAPI、Postman 與單元測試涵蓋上述行為（Querydsl、Security、Storage、Tax）。

## 2. 資料權限架構（Owner / Data Scope）

### 2.1 組件

| 元件 | 位置 | 作用 |
| --- | --- | --- |
| `DataScopeService` | `com.asynctide.flexora.security.datascope` | 決策可見 owner id；提供 `resolveForCurrentUser()`、`withBypass()`、`isBypassed()`。 |
| `@DataScoped` / `HibernateFilterAspect` | `security.datascope.aop` | AOP 掛在 Service/Resource，自動啟 `ownerFilter`。 |
| `DataScopePredicateBuilder` / `DataScopeSpecificationBuilder` | `security.datascope.builder` | 提供 Querydsl Predicate 與 Specification，供自訂查詢套入 owner 條件。 |
| `v_owner_visible` view | DBA 層面 | 汇總使用者可見 owner，供 filter/statement 參考。 |

### 2.2 查詢範例（參考 `SalesOrderQuery` / `QuotationThreadQueryService`）

```java
Predicate ownerPred = dsp.forRoot(QSalesOrder.salesOrder, SalesOrder_.owner);
if (ownerPred != null) {
    builder.and(ownerPred);
}
```

若 `DataScopeService` 判斷 `DataScopeRule.ALL` 或 `isBypassed()`，回傳 `null` 表示不加 owner 條件；若回傳 predicate，則附加 `owner.in(...)`。`HibernateFilterAspect` 會再於 Session 層啟用 filter，確保 lazy loading / repository 皆受保護。

### 2.3 `DataScopeContext`

```java
public record DataScopeContext(DataScopeRule rule, Set<Long> ownerIds, Instant resolvedAt, String reason) {}
```

所有 Service 在進入查詢前可透過 `DataScopeService.resolveForCurrentUser()`，獲得 owner 清單與決策依據；必要時可 `withBypass()` 讓特定流程臨時解除資料域限制。

## 3. 系統設定與快取

### 3.1 服務與 Repository

| 元件 | 位置 | 備註 |
| --- | --- | --- |
| `SystemSettingService` | `com.asynctide.flexora.service.SystemSettingService` | 提供 `getInt`, `getDecimal`, `getJson`, `validate` 等 API；更新時會同步失效 Redis 快取。 |
| `SystemSettingResource` | `web.rest.SystemSettingResource` | `GET /api/system-settings?code=...`, `PUT /api/system-settings/{code}`, `POST /api/system-settings/validate`。 |
| Cache | `SystemSettingCacheService` | 使用 Redis，key 範例如 `sys:setting:defaultWarehouseId`。 |

### 3.2 常見設定

| key | 說明 |
| --- | --- |
| `defaultWarehouseId` | 預設倉庫，用於 UI 預設值。 |
| `lowStockThreshold` | 補貨提醒閾值。 |
| `maxUploadMB` | 文件上傳大小限制。 |

設定更新需要額外驗證 `valueType`（如 `NUMBER`, `JSON`, `TEXT`）並可選擇 dry-run 進行 `validate` API。

## 4. 稅務管理

### 4.1 資料模型

| 表 | 內容 |
| --- | --- |
| `tax_code` | `code`, `name`, `isCompound`, `jurisdiction`, `defaultRate`, `metadata`。 |
| `tax_rate_line` | `tax_code_id`, `sequence`, `component_code`, `isPercentageBase`, `rate`, `fixedAmount`, `applyOn (HEAD/LINE)`, `metadata`。 |

### 4.2 `TaxService`

`TaxService.preview()` / `applyToLines()` 提供：  
* `TaxBreakdown`：列出每個稅組件金額。  
* `applyOn`：可選 `HEAD` 或 `LINE`，`isCompound` 會依 `sequence` 計算；`isPercentageBase`=true 時以當前 subtotal 作為基底。  
精度為金額 `DECIMAL(19,4)`、稅率 `DECIMAL(7,6)`，所有計算使用 `RoundingMode.HALF_UP`。

## 5. 文件管理

### 5.1 API

| Method | Path | 說明 |
| --- | --- | --- |
| `POST /api/documents/upload` | 上傳檔案（multipart），需 owner match。 |
| `GET /api/documents/{id}/download` | 下載前檢查資料域與授權。 |
| `POST /api/documents/{id}/links` | 對應 `document_link`，使文件關聯到 SalesOrder/Quotation。 |
| `GET /api/documents?entity=x&id=y` | 列出特定單據的文件列表。 |

文件存放採 `FileStorage` interface（Local/S3），路徑格式 `{bucket}/{yyyy}/{MM}/{dd}/{uuid}.{ext}`。可選擇在上傳後觸發 asynchronous 檔案驗證或掃毒。

### 5.2 DocumentLink

`DocumentLink` 表包含 `entity`, `entity_id`, `document_id`，可串接 `SalesOrder`、`QuotationRevision`、`InventoryTransaction` 等。`DocumentResource` 會驗證 owner 與 data scope 才允許建立或查詢。

## 6. 其他補充

### 6.1 測試 / Seed

1. Owner/Department/Team 可透過 `departments.csv`、`teams.csv`、`user_department.csv`, `user_team.csv` 建立測試資料。  
2. `system_setting.csv` 預設值：`defaultWarehouseId`, `lowStockThreshold=100`, `maxUploadMB=20`。  
3. 稅務：`tax_code.csv` + `tax_rate_line.csv` 建立台灣常見稅種（VAT 5%）。

### 6.2 建議實作順序

1. 建立 Owner hierarchy + Department/Team API。  
2. 實作 `DataScopeService` + Querydsl PredicateBuilder + `HibernateFilterAspect`。  
3. `SystemSettingService` + Redis 快取。  
4. `TaxService` + `/api/taxes/preview`。  
5. Document upload/download + Storage Adapter。  
6. OpenAPI 註解與 Postman 測試腳本。

### 6.3 Commit Message 建議

```
feat(security): add data scope filter with querydsl + hibernate filter
feat(setting): add system setting service + cache + validation
feat(tax): introduce tax service + preview endpoint
feat(document): implement file storage + upload/download/links
docs(api): document phase0 data domain spec
```
