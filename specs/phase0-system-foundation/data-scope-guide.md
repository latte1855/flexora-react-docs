# Data Scope / 資料權限指南（Legacy Summary）

本檔整理 `docs/migration/flexora-react/security/data-scope.md` 的要點，並提醒後續模組在需要 owner 過濾時可以依此實作。實作請參考 `flexora-react/src/main/java/com/asynctide/flexora/security/datascope/*`。

## 核心機制

1. `ownerFilter` (Hibernate Filter)：每個 entity 標註 `@Filter(name = "ownerFilter", condition = "owner_id IN (SELECT owner_id FROM v_owner_visible WHERE user_id = :userId)")`，保護 row-level security。
2. `@DataScoped` AOP：`HibernateFilterAspect` 攔截標註方法，自動啟 `ownerFilter` 並帶入 current user ID，可搭配 `@DataScoped` 搭配 `@Service`/Controller`。
3. Helper：無法套 filter 時，使用 `DataScopePredicateBuilder` 或 `DataScopeSpecificationBuilder` 在 QueryDSL/Specification 中手動加入 `owner_id` 條件。

## 行為摘要

| 場景 | Filter 啟用 | 備註 |
| --- | --- | --- |
| 方法加 `@DataScoped` | ✅ | Auto enable，可搭配 `Session` 取得 `userId` |
| Admin (`ROLE_ADMIN`) | ❌ | 自動 bypass |
| 未登入 | ❌ | 若需禁止，可手動丟 401/403 |
| 維運腳本 | 可呼 `HibernateFilterAspect.withBypass(() -> ...)` | 臨時關閉 filter |

## 建議

- 所有查詢 `owner_id` 資料的 Service/Controller 預設標註 `@DataScoped`。
- 若需直接查 owner list，可增加 `GET /api/owners/visible` endpoint 查 `v_owner_visible`。
- 若新增 `ownerFilter` 事件（例如 `inventory.tx.posted` 需要 owner metadata），請同步更新 `docs/architecture/legacy-inventory-domain-events.md`。
