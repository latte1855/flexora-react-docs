# Phase 0 — 基礎底座（System Foundation）落地規劃

> 位置：`docs/api/phase0-system-foundation.md`  
> 對齊：`docs/api/MODULES.md` 的 Phase 0（資料權限／系統設定／稅務管理／文件管理）  
> 更新：2025-10-15（Asia/Taipei）

本文件提供 **可直接實作的規格**、**API 端點**、**服務設計**、**測試與驗收**，
特別聚焦「資料權限（Owner/Department/Team）」部分，補齊 Querydsl 與 JPA/Hibernate 共存設計。

---

## 0. 完成定義（DoD）

- [ ] 部門/團隊 CRUD 與階層維護完成（含 parentHierarchy 與 depth 自動維護）。
- [ ] 使用者與部門／團隊關聯指派 API 完成，並可查詢使用者有效資料域（data scope）。
- [ ] Repository 層「**自動套用資料權限過濾**」：清單與讀取一律以 Owner 可見域篩選。
- [ ] SystemSetting CRUD 與設定快取完成，提供系統層設定（例如：預設倉別、安全庫存、檔案大小限制…）。
- [ ] TaxCode / TaxRateLine CRUD 與 TaxService（單頭/行、複合稅計算）雛形完成。
- [ ] Document / DocumentLink 完成，上傳檔案存放策略（本機/雲端）與下載授權檢查。
- [ ] OpenAPI 註解與 Postman 契約測試腳本可直接跑。

---

## 1. 資料權限（Owner / Department / Team）

### 1.1 目標與範圍

- **資料域（Data Scope）**：SELF / DEPT / DEPT_AND_SUB / TEAM / ALL / CUSTOM
- 所有業務實體（如 SalesOrder、Quotation…）均應具有 `owner_id` 欄位，指向 `Owner` 主鍵。
- 清單/查詢/報表預設套用資料域；明確要求時才可加上 `includeDeleted` 或 `ignoreDataScope`（僅 ADMIN）。

---

### 1.2 技術架構：Querydsl + AOP + Hibernate Filter + Specification 共存

> Flexora 系統 Repository 皆 `extends QuerydslPredicateExecutor<T>`，  
> 採「**Querydsl 主線** + **AOP/Filter 保險網** + **Specification 轉接層**」的共存架構。

| 層                | 角色                 | 用途                                                                 |
| ----------------- | -------------------- | -------------------------------------------------------------------- |
| Querydsl          | 主力查詢層           | 所有主要清單/報表查詢皆統一用 Querydsl；AOP 自動組出 owner predicate |
| Hibernate Filter  | 保險網               | 覆蓋 Repository 衍生方法 / Lazy loading / 原生 SQL                   |
| JPA Specification | 舊程式或第三方轉接層 | 由共用的 `OwnerScopeSpecs` 提供                                      |
| ThreadLocal Flag  | 雙重過濾防呆         | 標記當前 Session 是否已套用 Filter                                   |

---

### 1.3 統一資料域解析器（唯一權威）

```java
public enum DataScopeRule {
  SELF,
  DEPT,
  DEPT_AND_SUB,
  TEAM,
  ALL,
  CUSTOM,
}

public record DataScopeContext(DataScopeRule rule, Set<Long> ownerIds, Instant resolvedAt, String reason) {}

public interface DataScopeService {
  DataScopeContext resolveForCurrentUser(); // 算出可見 ownerIds
  boolean isBypassed(); // 是否被明確要求忽略（ADMIN + Header）
  void withBypass(Runnable r); // 內部操作暫時忽略
}

```

- 結果快取在 Request 或 ThreadLocal，避免重複計算。
- `resolveForCurrentUser()` 需展開部門樹（以 `parentHierarchy like '/10/%'` 查找子層）。

---

### 1.4 Querydsl 主線（建議作法）

#### PredicateBuilder：組出 owner 條件

```java
@Component
public class DataScopePredicateBuilder {

  private final DataScopeService dataScopeService;

  public <T> Predicate forRoot(EntityPathBase<T> root, NumberPath<Long> ownerIdPath) {
    DataScopeContext ctx = dataScopeService.resolveForCurrentUser();
    if (ctx.rule() == DataScopeRule.ALL || dataScopeService.isBypassed()) return null;
    return ownerIdPath.in(ctx.ownerIds());
  }
}

```

#### Service 查詢範例

```java
@Service
public class SalesOrderQuery {
  private final JPAQueryFactory qf;
  private final DataScopePredicateBuilder dsp;

  public Page<SalesOrderDTO> find(Pageable pageable, SalesOrderFilter f) {
    QSalesOrder so = QSalesOrder.salesOrder;
    BooleanBuilder where = new BooleanBuilder();
    if (f.status()!=null) where.and(so.status.eq(f.status()));

    Predicate ownerPred = dsp.forRoot(so, so.owner.id);
    if (ownerPred != null) where.and(ownerPred);

    List<SalesOrderDTO> content = qf.select(Projections.constructor(...))
        .from(so)
        .where(where)
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    long total = qf.select(so.id.count()).from(so).where(where).fetchOne();
    return new PageImpl<>(content, pageable, total);
  }
}
```

---

### 1.5 Hibernate Filter：保險網層

#### Entity 層宣告

```java
@FilterDef(name = "ownerFilter", parameters = @ParamDef(name = "ownerIds", type = Long.class))
@Filter(name = "ownerFilter", condition = "owner_id in (:ownerIds)")
@Entity
public class SalesOrder { ... }
```

#### AOP 自動開關

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface DataScoped {
}

@Aspect
@Component
public class HibernateFilterAspect {

  private final EntityManager em;
  private final DataScopeService dataScope;

  @Around("@within(DataScoped) || @annotation(DataScoped)")
  public Object around(ProceedingJoinPoint pjp) throws Throwable {
    if (dataScope.isBypassed()) return pjp.proceed();
    Set<Long> ids = dataScope.resolveForCurrentUser().ownerIds();
    Session session = em.unwrap(Session.class);
    Filter filter = session.enableFilter("ownerFilter").setParameterList("ownerIds", ids);
    try {
      return pjp.proceed();
    } finally {
      session.disableFilter("ownerFilter");
    }
  }
}

```

**適用情境：**

- Repository 衍生方法（`findByStatus(...)`）
- Lazy collection 載入
- 原生 SQL 或舊代碼查詢

---

### 1.6 JPA Specification：轉接與共用

```java
public final class OwnerScopeSpecs {

  public static <T> Specification<T> visibleTo(Set<Long> ownerIds) {
    return (root, query, cb) -> root.get("owner").get("id").in(ownerIds);
  }
}

```

> 讓舊式 Repository `findAll(Specification<T>)` 也能吃到 owner 過濾。

---

### 1.7 防呆與 ThreadLocal 控制

- ThreadLocal 旗標：AOP 已開啟 Hibernate Filter 時，Querydsl helper 不再套 predicate。
- 允許 Header 覆蓋：

  ```
  X-Ignore-DataScope: true
  ```

- `DataScopeService.withBypass()`：在系統批次、管理操作中暫時關閉資料域。

---

### 1.8 選擇建議表

| 情境                     | 建議作法                        | 原因             |
| ------------------------ | ------------------------------- | ---------------- |
| 新的清單、報表、複雜查詢 | Querydsl + PredicateBuilder     | 效能高、型別安全 |
| Repository 衍生方法      | Hibernate Filter + @DataScoped  | 低侵入           |
| 舊程式 / 第三方          | Specification + OwnerScopeSpecs | 快速轉接         |
| 原生 SQL                 | Hibernate Filter 或顯式 where   | 保險網           |
| 批次或管理員             | DataScopeService.withBypass     | 安全忽略         |

---

### 1.9 效能與索引建議

- 所有業務表皆建立 `owner_id` 索引。
- `department.parent_hierarchy` 使用 PostgreSQL `varchar_pattern_ops` btree 索引。
- Querydsl predicate 先行過濾，避免重複條件。

---

### 1.10 測試矩陣

| 測試項目                                                       | 驗收結果 |
| -------------------------------------------------------------- | -------- |
| Querydsl path：`SalesOrderQuery.find()` 只回 ownerIds 範圍資料 | ✅       |
| Repo 衍生方法 + `@DataScoped` 可過濾                           | ✅       |
| Specification path 結果一致                                    | ✅       |
| Lazy collection 受 Filter 過濾                                 | ✅       |
| 不會雙重加 owner_id 條件                                       | ✅       |
| `X-Ignore-DataScope` 可略過過濾（log 有紀錄）                  | ✅       |

---

## 2. 系統設定（SystemSetting）

### 2.1 設計

- `SystemSetting { id, code, value, valueType(STRING/INT/DECIMAL/JSON/BOOL), description, updatedBy, updatedAt }`
- partial unique：`code`。
- 設定以 JSONB 支援結構化值（`valueType=JSON` 時），例如：

```json
{ "defaultWarehouseId": 1, "lowStockThreshold": 100, "maxUploadMB": 20 }
```

### 2.2 API

```
GET    /api/system-settings?code={code}     # 取單一或多筆
PUT    /api/system-settings/{code}          # 更新（依 valueType 驗證）
POST   /api/system-settings/validate        # 檢核設定（dry-run）
```

### 2.3 快取

- Redis key：`sys:setting:{code}`，更新時失效。
- 提供 `SystemSettingService#getInt/getDecimal/getJson(...)` 封裝解析。

### 2.4 測試

- 型別錯誤回 422（violations 指出 `valueType mismatch`）。
- 更新後新交易讀到新值（快取已失效）。

---

## 3. 稅務管理（TaxCode / TaxRateLine + TaxService）

### 3.1 稅模型

- `TaxCode { id, code, name, isCompound, jurisdiction, defaultRate, metadata }`
- `TaxRateLine { id, taxCodeId, sequence, componentCode, isPercentageBase, rate, fixedAmount, applyOn(HEAD/LINE), metadata }`
- 稅率計算順序：依 `sequence`，`isPercentageBase`=true 時以「目前小計」為基礎。

### 3.2 API（CRUD 以外）

```
POST /api/taxes/preview
{
  "applyOn": "LINE",
  "netAmount": 114.00,
  "taxCodeId": 10
}
→ { "components": [ { "code":"VAT", "amount":5.70 } ], "taxTotal":5.70 }
```

### 3.3 服務（TaxService）

```java
TaxBreakdown preview(TaxCode code, BigDecimal netAmount, ApplyOn applyOn);
TaxBreakdown applyToLines(List<Line>, ApplyOn applyOn);
```

- 支援行稅 → 彙總到單頭；或單頭稅 → 分攤到行（可選）。

### 3.4 測試

- compound 稅計算順序與精度。
- 四捨五入：行→單頭一致。

---

## 4. 文件管理（Document / DocumentLink）

### 4.1 存放策略

- Adapter 介面：`FileStorage`（實作 `LocalStorage`, `S3Storage`）。
- 路徑規則：`{bucket}/{yyyy}/{MM}/{dd}/{uuid}.{ext}`。
- 以 SystemSetting 控制：`storage.provider=LOCAL|S3`、`maxUploadMB`。

### 4.2 API

```
POST   /api/documents/upload          (multipart)    # 權限：擁有對應 owner 的寫入權
GET    /api/documents/{id}/download                 # 檢查資料域後下發簽名或串流
POST   /api/documents/{id}/links                   # body: { entity:"salesOrder", entityId:100 }
GET    /api/documents?entity=salesOrder&id=100     # 列出關聯文件
```

### 4.3 安全與掃毒（可選）

- 上傳後觸發 async 任務做檔案型態驗證與（可選）AV 掃描，失敗則標記 quarantined。

### 4.4 測試

- 非資料域用戶無法下載/連結檔案。
- maxUpload 超限回 413。

---

## 5. OpenAPI 摘要（關鍵端點）

- `GET /api/users/{id}/data-scope`
- `POST /api/departments/{id}/move`
- `GET /api/departments/tree`
- `GET /api/system-settings?code=` / `PUT /api/system-settings/{code}`
- `POST /api/taxes/preview`
- `POST /api/documents/upload` / `GET /api/documents/{id}/download` / `POST /api/documents/{id}/links`

---

## 6. 種子資料（Seed）

- `departments.csv`：建立 1~2 層部門樹並產生對應 `Owner`
- `teams.csv`：建立 3~5 個團隊（`Owner` 對應）
- `user_department.csv` / `user_team.csv`：建立若干關聯
- `system_setting.csv`：`defaultWarehouseId`, `lowStockThreshold=100`, `maxUploadMB=20`
- `tax_code.csv` / `tax_rate_line.csv`：台灣常見稅別（VAT 5%）

---

## 7. 建議實作順序（Phase 0 內部）

1. **OwnerHierarchyService** + 部門移動與樹狀 API
2. **DataScopeService** + Querydsl PredicateBuilder + HibernateFilterAspect
3. **SystemSettingService** + Redis 快取與驗證
4. **TaxService** + `/api/taxes/preview`
5. **Document 上傳/下載/連結** + Storage Adapter
6. OpenAPI 註解與 Postman 契約測試

---

## 8. Commit Message 建議

```
feat(security): implement data scope (querydsl + hibernate filter) with repository filtering
feat(setting): add SystemSetting service, cache, and validation
feat(tax): introduce TaxService and /api/taxes/preview
feat(document): file upload/download and entity linking with storage adapter
docs(api): update phase0-system-foundation spec with querydsl integration
```
