# 資料權限 / Data Scope 指南

> 版本：2025-11-16  
> 主要 package：`com.asynctide.flexora.security.datascope.*`

本文件說明 Flexora 目前的資料權限機制：如何啟動 `ownerFilter`、在 QueryDSL / Specification 中手動套用，以及 Admin / 維運繞過方式。所有需要查詢 Owner 資料的模組皆可引用本指南。

## 1. 架構總覽

1. **Owner + v_owner_visible**：每個資料實體都有 `owner_id`；資料庫 view `v_owner_visible` 會列出「登入者可見的 owner_id」（包含本人、部門、團隊等）。
2. **Hibernate Filter (`ownerFilter`)**：在 entity 上透過 `@Filter(name = "ownerFilter", condition = ... v_owner_visible ...)` 實作 Row-level Security，條件為 `exists (select 1 from v_owner_visible ...)`。
3. **AOP 啟動 (`@DataScoped`)**：`com.asynctide.flexora.security.datascope.aop.HibernateFilterAspect` 會攔截所有標註 `@DataScoped` 的 Service/Controller，於方法進入時自動 `session.enableFilter("ownerFilter")` 並帶入 `SecurityUtils.getCurrentUserId()`。
4. **Helper (Predicate / Specification)**：若無法使用切面，可改用 `DataScopePredicateBuilder` 或 `DataScopeSpecificationBuilder` 在 QueryDSL / Specification 中手動加入過濾條件。

## 2. 預設行為

| 場景                   | 是否套用 `ownerFilter` | 說明                                                                   |
| ---------------------- | ---------------------- | ---------------------------------------------------------------------- |
| 方法標註 `@DataScoped` | 會                     | AOP 會啟用 Hibernate Filter，所有 Repository / lazy loading 均受保護。 |
| 未標註 `@DataScoped`   | 不會                   | 請改用 `@DataScoped` 或在 QueryDSL / Specification 中自行呼叫 helper。 |
| Admin (`ROLE_ADMIN`)   | 不會                   | Admin 自動 bypass，可看全部資料。                                      |
| 未登入/無 userId       | 不會                   | 切面直接放行（可自行改成拋 401/403）。                                 |

## 3. QueryDSL / Specification Helper

### QueryDSL

```java
Predicate predicate = dataScopePredicateBuilder.forRoot(QQuotationThread.quotationThread, QQuotationThread.quotationThread.owner.id);
if (predicate != null) {
    query.where(predicate);
}
```

- 回傳 `null`：代表當前執行緒已啟動 Hibernate Filter 或使用者為 Admin。
- 回傳 `Expressions.FALSE.isTrue()`：未登入，查詢將得不到任何資料。

### Specification

```java
Specification<Order> dataScope = dataScopeSpecificationBuilder.forRoot(Order_.owner);
if (dataScope != null) {
    spec = spec.and(dataScope);
}
```

## 4. Bypass / 維運

- **Admin**：自動繞過。
- **程式暫時繞過**：`HibernateFilterAspect.withBypass(() -> {...})` 可關閉 filter（僅當前執行緒），適合維運或批次任務。
- **ThreadLocal Flag**：`HibernateFilterAspect.isFilterApplied()` 可判斷 filter 是否已啟動，helper 會用此 flag 避免重複疊加條件。

## 5. 測試案例

`com.asynctide.flexora.my.phase0.security.datascope.DataScopeE2EIT` 包含多個情境：

- 一般使用者 / Admin 差異。
- 部門、團隊繼承對 `v_owner_visible` 的影響。
- `@DataScoped` + helper 混合使用時的行為。

新增功能時建議依此測試，確保持續符合資料域策略。

## 6. 使用指南

1. **REST / Service 方法一律標註 `@DataScoped`**，除非確定要開放全部資料。
2. **Repository / QueryDSL 自訂查詢** 如無法套 `@DataScoped`，請呼叫 `DataScopePredicateBuilder/DataScopeSpecificationBuilder`。
3. **維運腳本** 若需繞過資料域，請包在 `HibernateFilterAspect.withBypass(...)`。
4. **新模組文件**：在各模組 Spec 或 README 中引用本檔案，以保持一致。

## 7. 待辦 / 建議

- **Owner 查詢 API**：若 UI 需要直接列出使用者可見 owner，可包一支 API 讀 `v_owner_visible`。
- **自動掃描 `@DataScoped`**：未來可把 Service template 預設標註，減少遺漏風險。
- **文件引用**：`docs/資料權限管理 Domain 規格.md`、`AGENTS.md` 等文件請引用本指南。

## 8. 實際範例：報價 Hub

- **Owner 篩選**：`QuotationThreadQueryService` 新增 `ownerScope` criteria；當 `ownerScope=MINE` 且使用者非 Admin 時，會透過 `SecurityUtils.getCurrentUserOwnerIds()` 取得 `v_owner_visible` 中的 ownerId 清單並加入 Specification。若清單為空則直接回傳 `Predicate#disjunction()`，確保無權限時查不到資料。
- **Owner ID 解析**：`UserAuthorizationService` 於 `@PostConstruct` 註冊到 `SecurityUtils`，實作 `resolveVisibleOwnerIds(userId)` -> `OwnerRepositoryCustom#getOwnerIdsVisibleToUser`，而 Repository 會以 JPQL 查 `VOwnerVisible` view。
- **前端用法**：Quotation Hub List / Pipeline View 只需在查詢參數帶 `ownerScope=MINE`，伺服器會依資料權限自動過濾；UI 不再需要預先查詢 ownerId 或在客戶端手動比對。

---

若有新的資料權限策略（例如跨租戶、臨時授權），請同步更新本文件與 `DataScopeE2EIT`，確保行為一致。
