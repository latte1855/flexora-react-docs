# Flexora 定價模組（Product Pricing）Phase 1 規格書 v1.0

> 更新：2025-10-21（Asia/Taipei）  
> 適用規範：**《Flexora ERP 開發規範 – 資料結構與版本管理（含 JaVers）》**（時間=UTC、金額與比率精度、軟刪、樂觀鎖、地址 JSONB VO、**不使用 `tenant_id`、唯一鍵 partial unique (WHERE deleted=false)**）。  
> Phase 1 目標：提供 **價目表（Price List）+ 數量階梯 + 客戶/群組/通路 指派** 的可落地 **定價引擎**，可被 **Quotation / Sales Order / Delivery Note** 以「快照」模式引用。

---

## 目錄

1. [範圍（Phase 1）與非範圍](#範圍phase-1與非範圍)
2. [名詞與計價流程（高階）](#名詞與計價流程高階)
3. [Enum 值定義（集中）](#enum-值定義集中)
4. [資料表定義（完整表格）](#資料表定義完整表格)
   - 4.1 `price_list`
   - 4.2 `price_list_item`
   - 4.3 `price_list_assignment`
   - 4.4 `price_rule`（Phase 1 最小可用）
   - 4.5 `price_calc_trace`（計算追蹤）
5. [定價演算法（Phase 1）](#定價演算法phase-1)
6. [精度、捨入與稅前/稅後價](#精度捨入與稅前稅後價)
7. [API 介面（對外 / 供內部模組呼叫）](#api-介面對外--供內部模組呼叫)
8. [與其他模組整合](#與其他模組整合)
9. [Migration 與索引](#migration-與索引)
10. [Seed 資料（CSV 範例）](#seed-資料csv-範例)
11. [驗收測試（UAT）與單元測試清單](#驗收測試uat與單元測試清單)
12. [開發分支、Commit 規範、交付物](#開發分支commit-規範交付物)

---

## 範圍（Phase 1）與非範圍

### 1.1 範圍（Phase 1）

- **價目表（Price List）**：多幣別、有效期間、基準價型別（未稅 / 含稅）、通路碼。
- **價目表明細（Price List Item）**：SKU 層級單價，支援 **數量階梯（quantity breaks）** 與 **UoM**。
- **指派（Assignment）**：價目表可指派至 **客戶**、**客戶群組**、**銷售通路**，並有優先序；支援「預設價目表」。
- **最小化規則（Price Rule）**：提供 **整單折扣碼** 與 **每 SKU 依群組比例** 兩種簡單規則（可關閉）。
- **定價引擎**：以輸入（客戶、通路、日期、幣別、SKU、數量、UoM、稅別）回傳「未稅單價」、「含稅單價」、「行淨額」、「稅額」；同時產出 `price_calc_trace` 便於稽核。
- **被動快照**：Quotation/SalesOrder/DN 取回 **單價與稅率快照**，後續不回寫主檔。

### 1.2 非範圍（未納入 Phase 1）

- 複雜 **促銷規則（滿額、第二件、贈品、區域門市排除）**。
- 動態毛利守門（成本變動即時重算）— Phase 2 評估。
- 價目表 **分層繼承/覆寫** 與 **自動匯率換算**（Phase 2）。

---

## 名詞與計價流程（高階）

**輸入**：customerId / customerGroup / channel / currency / orderDate / skuId / qty / uom / taxCode。  
**查找**：

1. 以 **customer → group → channel → default** 優先序挑選 **適用的價目表**（有效期間/幣別相符）。
2. 在價目表內，尋找 SKU 的 **最佳數量階梯** 與最貼近 UoM（找不到則使用 base UoM 換算）。
3. 套用 **price_rule**（若開啟）。
4. 依 `price_list.price_type`（未稅/含稅）與 `tax_rate` 轉算另一種價並計算行稅額。  
   **輸出**：unitPriceExclTax / unitPriceInclTax / netAmount / taxAmount / traceId。

---

## Enum 值定義（集中）

- `price_type`：`EXCL_TAX`（未稅）、`INCL_TAX`（含稅）。
- `assignment_level`：`CUSTOMER` / `CUSTOMER_GROUP` / `CHANNEL` / `DEFAULT`。
- `rule_type`（Phase 1）：`ORDER_DISCOUNT_RATE`（整單率折）、`SKU_GROUP_RATE`（SKU 群組率折）。

---

## 資料表定義（完整表格）

> 所有表均含通用欄位：`created_by/created_at/last_modified_by/last_modified_at/deleted/deleted_at/deleted_by/version`。金額 `DECIMAL(19,4)`；中間運算/單價/數量 `DECIMAL(19,6)`；比率 `DECIMAL(7,6)`。

### 4.1 `price_list`（Header）

| 欄位代碼        |         型態 |     預設值 | 欄位名稱   | 必填 | 說明            | 注意事項                                  |
| --------------- | -----------: | ---------: | ---------- | :--: | --------------- | ----------------------------------------- |
| id              |    BIGINT PK |            | 主鍵       |  Y   |                 |                                           |
| price_list_code |  VARCHAR(64) |            | 價目表代碼 |  Y   | 對外唯一        | **partial unique**（WHERE deleted=false） |
| price_list_name | VARCHAR(255) |            | 名稱       |  Y   | 顯示名稱        |                                           |
| currency_code   |  VARCHAR(16) |            | 幣別       |  Y   | ISO 4217        |                                           |
| price_type      |  VARCHAR(16) | `EXCL_TAX` | 價格型態   |  Y   | 未稅/含稅       | 與稅別互相轉算                            |
| valid_from      |         DATE |       NULL | 生效起     |  N   |                 |                                           |
| valid_to        |         DATE |       NULL | 生效迄     |  N   |                 |                                           |
| channel_code    |  VARCHAR(64) |       NULL | 通路碼     |  N   | WEB/RETAIL/B2B… | 字典                                      |
| description     | VARCHAR(500) |       NULL | 說明       |  N   |                 |                                           |
| properties      |        JSONB |         {} | 其他參數   |  N   |                 |                                           |

### 4.2 `price_list_item`（SKU 單價＋數量階梯）

| 欄位代碼      |          型態 | 預設值 | 欄位名稱              | 必填 | 說明                      | 注意事項                 |
| ------------- | ------------: | -----: | --------------------- | :--: | ------------------------- | ------------------------ |
| id            |     BIGINT PK |        | 主鍵                  |  Y   |                           |                          |
| price_list_id |     BIGINT FK |        | 所屬價目表            |  Y   | FK→`price_list`           |                          |
| sku_id        |     BIGINT FK |        | SKU                   |  Y   | 請參照既有 `item_sku`     |                          |
| uom_id        |     BIGINT FK |   NULL | 單位                  |  N   | 參照 `uom`；NULL=基礎單位 |                          |
| min_qty       | DECIMAL(19,6) |      0 | 最小數量              |  Y   | 此階梯下限（含）          | 同 SKU/UoM 不可重疊      |
| unit_price    | DECIMAL(19,6) |      0 | 單價（依 price_type） |  Y   | 階梯單價                  |                          |
| tax_code_id   |     BIGINT FK |   NULL | 稅別                  |  N   | FK→`tax_code`             | 僅於 `INCL_TAX` 轉算需要 |
| properties    |         JSONB |     {} | 其他                  |  N   |                           |                          |

> **唯一鍵**：`(price_list_id, sku_id, COALESCE(uom_id,-1), min_qty)`（WHERE deleted=false）。  
> **階梯規則**：同一 `(sku,uom)` 範圍內 `min_qty` 升冪、不得重疊。

### 4.3 `price_list_assignment`（指派與優先）

| 欄位代碼         |        型態 |    預設值 | 欄位名稱 | 必填 | 說明                                          | 注意事項                |
| ---------------- | ----------: | --------: | -------- | :--: | --------------------------------------------- | ----------------------- |
| id               |   BIGINT PK |           | 主鍵     |  Y   |                                               |                         |
| price_list_id    |   BIGINT FK |           | 價目表   |  Y   |                                               |                         |
| assignment_level | VARCHAR(32) | `DEFAULT` | 指派層級 |  Y   | CUSTOMER / CUSTOMER_GROUP / CHANNEL / DEFAULT |                         |
| ref_id           |      BIGINT |      NULL | 參照 ID  |  N   | CUSTOMER/CUSTOMER_GROUP 對應 id               | CHANNEL/DEFAULT 為 NULL |
| priority         |         INT |       100 | 優先序   |  Y   | 值愈小優先                                    | 用於多表衝突            |
| valid_from       |        DATE |      NULL | 生效起   |  N   |                                               |                         |
| valid_to         |        DATE |      NULL | 生效迄   |  N   |                                               |                         |
| is_fallback      |     BOOLEAN |     false | 是否預設 |  Y   | **僅 DEFAULT 可設 true**                      | 系統保底                |

> **選取優先序**：CUSTOMER（精確）→ CUSTOMER_GROUP → CHANNEL → DEFAULT；同級以 priority 升冪、最晚生效者優先。

### 4.4 `price_rule`（Phase 1 最小可用）

| 欄位代碼   |         型態 | 預設值 | 欄位名稱 | 必填 | 說明                                                                                             | 注意事項           |
| ---------- | -----------: | -----: | -------- | :--: | ------------------------------------------------------------------------------------------------ | ------------------ |
| id         |    BIGINT PK |        | 主鍵     |  Y   |                                                                                                  |                    |
| rule_code  |  VARCHAR(64) |        | 規則代碼 |  Y   | 唯一                                                                                             | **partial unique** |
| name       | VARCHAR(255) |        | 名稱     |  Y   |                                                                                                  |                    |
| rule_type  |  VARCHAR(32) |        | 規則型態 |  Y   | `ORDER_DISCOUNT_RATE` / `SKU_GROUP_RATE`                                                         |                    |
| enabled    |      BOOLEAN |   true | 啟用     |  Y   |                                                                                                  |                    |
| properties |        JSONB |     {} | 參數     |  Y   | `ORDER_DISCOUNT_RATE`：`{"rate":0.05}`；`SKU_GROUP_RATE`：`{"groupCode":"ACCESSORY","rate":0.1}` |                    |

### 4.5 `price_calc_trace`（計算追蹤）

| 欄位代碼     |        型態 | 預設值 | 欄位名稱 | 必填 | 說明                          |
| ------------ | ----------: | -----: | -------- | :--: | ----------------------------- |
| id           |   BIGINT PK |        | 主鍵     |  Y   |                               |
| trace_no     | VARCHAR(64) |        | 追蹤代號 |  Y   | 供 UI/稽核查詢                |
| requested_at |   TIMESTAMP |    now | 請求時間 |  Y   | UTC                           |
| payload_req  |       JSONB |     {} | 請求內容 |  Y   | {customer, channel, items[]…} |
| payload_resp |       JSONB |     {} | 計算結果 |  Y   | 含階梯命中、規則套用明細      |

---

## 定價演算法（Phase 1）

1. **挑選價目表**：依 `customer → group → channel → default` 迭代，篩選幣別/期間相符者，取 **priority 最小** 並 **距離 orderDate 最近** 的一筆。
2. **命中 SKU 階梯**：在該價目表內，使用 `(sku,uom)` 尋找 `min_qty ≤ qty` 的 **最大 min_qty**；找不到則嘗試 UoM 換算成 base UoM 查找。
3. **計稅換算**：
   - 若 `price_type=EXCL_TAX`：`unitIncl = unitExcl × (1 + taxRate)`。
   - 若 `price_type=INCL_TAX`：`unitExcl = unitIncl ÷ (1 + taxRate)`（精度規則見下一節）。
4. **套用規則（可選）**：
   - `ORDER_DISCOUNT_RATE`：對 **整單淨額** 乘以 `(1 - rate)`，分攤回各行（按行淨額占比）。
   - `SKU_GROUP_RATE`：對命中 `groupCode` 的 **行單價** 乘以 `(1 - rate)`。
5. **輸出**：每行 `unitPriceExclTax, unitPriceInclTax, lineNet, taxAmount`；輸出 `trace_no` 供調試。

---

## 精度、捨入與稅前/稅後價

- **比率**：`DECIMAL(7,6)`；**金額**：`DECIMAL(19,4)`；**單價/數量**：`DECIMAL(19,6)`。
- **捨入**：計算過程一律指定 `RoundingMode.HALF_UP`；行內中間值以 s=6，最終金額以 s=4；顯示可 round 2。
- **含稅→未稅** 轉換：先算未稅單價至 s=6，再計行稅額（s=4），避免累計誤差。

---

## API 介面（對外 / 供內部模組呼叫）

> REST + **Idempotency-Key**（含寫入 API）；輸入/輸出金額以 **字串** 避免 JS 浮點誤差。OpenAPI 置於 `docs/api/pricing-openapi.yaml`。

### 7.1 試算（單筆 / 多筆）

`POST /api/pricing/preview`  
**Request**

```json
{
  "customerId": 123,
  "customerGroupId": 45,
  "channel": "B2B",
  "currency": "TWD",
  "orderDate": "2025-10-21",
  "items": [
    { "skuId": 1, "uomId": null, "qty": "10", "taxCode": "TWN_VAT_5" },
    { "skuId": 2, "uomId": 7, "qty": "3.5", "taxCode": "TWN_VAT_5" }
  ]
}
```

**Response**

```json
{
  "traceNo": "PRC-20251021-0001",
  "lines": [
    {
      "skuId": 1,
      "unitPriceExcl": "100.000000",
      "unitPriceIncl": "105.000000",
      "taxRate": "0.050000",
      "netAmount": "1000.000000",
      "taxAmount": "50.0000"
    },
    {
      "skuId": 2,
      "unitPriceExcl": "250.000000",
      "unitPriceIncl": "262.500000",
      "taxRate": "0.050000",
      "netAmount": "875.000000",
      "taxAmount": "43.7500"
    }
  ],
  "discountTotal": "-93.7500",
  "grandTotal": "1874.9999"
}
```

### 7.2 價目表 CRUD（管理端）

- `GET /api/price-lists?keyword=&currency=&channel=&page=&size=`
- `POST /api/price-lists`（Idempotency-Key）
- `PUT /api/price-lists/{id}` / `DELETE /api/price-lists/{id}`（軟刪）
- `POST /api/price-lists/{id}/items:bulk-upsert`（明細批次上傳，CSV/JSON）
- `POST /api/price-lists/{id}/assignments:sync`（指派同步寫入）

### 7.3 規則 CRUD（最小）

- `GET /api/price-rules` / `POST /api/price-rules` / `PUT /api/price-rules/{id}` / `DELETE /api/price-rules/{id}`

> **安全**：以資料權限（Owner/Dept/Team）和角色判斷允許之價目表。

---

## 與其他模組整合

- **Quotation**：建立/重算時呼叫 `/pricing/preview` 取得 **行單價/稅率**，寫入 `quotation_item` 與 `quotation_item_tax`；Revision 不可變。
- **Sales Order**：Confirm 前最後試算；SO 行快照欄位（`unit_price/discount/tax_rate/line_total`）對齊既有規格。
- **Delivery Note**：一般不變價；僅在「後台修正」時可重算，產生差額（Phase 2 評估）。
- **稅**：引用既有 `tax_code` / `tax_rate_line` 主檔；含稅/未稅轉換依 taxRate 處理。

---

## Migration 與索引

- Liquibase 變更檔：
  - `20251021170000_added_entity_PriceList.xml`
  - `20251021170100_added_entity_PriceListItem.xml`
  - `20251021170200_added_entity_PriceListAssignment.xml`
  - `20251021170300_added_entity_PriceRule.xml`
  - `20251021170400_added_entity_PriceCalcTrace.xml`
- 重要索引：
  - `price_list(price_list_code) PARTIAL WHERE deleted=false`
  - `price_list_item(price_list_id, sku_id, COALESCE(uom_id,-1), min_qty)`
  - `price_list_assignment(assignment_level, ref_id, priority, valid_from, valid_to)`

---

## Seed 資料（CSV 範例）

**price_list.csv**（欄位以 `;` 分隔）

```
id;price_list_code;price_list_name;currency_code;price_type;valid_from;valid_to;channel_code;description;properties;deleted;deleted_at;deleted_by;version
1;PL_TWD_STD;台幣標準售價;TWD;EXCL_TAX;2025-01-01;;B2B;台幣基準表;{};false;;;
2;PL_TWD_WEB;台幣官網價;TWD;INCL_TAX;2025-01-01;;WEB;官網含稅價;{};false;;;
```

**price_list_item.csv**

```
id;price_list_id;sku_id;uom_id;min_qty;unit_price;tax_code_id;properties;deleted;deleted_at;deleted_by;version
1;1;1001;;0;100.000000;;{};false;;;
2;1;1001;;10;95.000000;;{};false;;;
3;2;1001;;0;105.000000;1;{};false;;;
```

**price_list_assignment.csv**

```
id;price_list_id;assignment_level;ref_id;priority;valid_from;valid_to;is_fallback;deleted;deleted_at;deleted_by;version
1;1;DEFAULT;;9999;;true;true;false;;;
2;2;CHANNEL;;50;;true;false;false;;;
```

**price_rule.csv**

```
id;rule_code;name;rule_type;enabled;properties;deleted;deleted_at;deleted_by;version
1;RULE_ORDER_5OFF;整單95折;ORDER_DISCOUNT_RATE;true;{"rate":0.05};false;;;
2;RULE_ACC_10OFF;配件九折;SKU_GROUP_RATE;true;{"groupCode":"ACCESSORY","rate":0.1};false;;;
```

---

## 驗收測試（UAT）與單元測試清單

- **價目表挑選**：同時存在 CUSTOMER/CHANNEL/DEFAULT，應以 CUSTOMER 優先；同級比較 priority、有效期間。
- **數量階梯**：qty=9 命中 0 階梯，qty=10 命中 10 階梯；UoM 換算可命中基礎階梯。
- **含稅/未稅轉換**：以 s=6 計單價換算，行稅額 s=4；整單捨入不超過 ±0.01（以 TWD 測）。
- **規則套用**：
  - `ORDER_DISCOUNT_RATE`：整單 5% 折扣，分攤後各行合計誤差 < 0.01。
  - `SKU_GROUP_RATE`：僅影響屬於 group 的 SKU 行單價。
- **權限**：非擁有者不可見未授權之價目表。
- **API**：`/pricing/preview` 壓力測試 100 RPS、P95 < 80ms（以 1k SKU、10 階梯為基準）。

---

## 開發分支、Commit 規範、交付物

### 12.1 分支命名（依本專案慣例）

- **分支**：`feature/phase1-product-pricing-20251021`（或精簡：`feat/product-pricing-phase1`）。
- **首個 Commit Message（Conventional Commits）**：
  - `feat(pricing): introduce Phase 1 price lists, assignments, quantity breaks and preview API`
  - `chore(db): add liquibase changelogs for price_list* and price_rule tables`
  - `docs(api): add pricing-openapi.yaml and module spec`

### 12.2 交付物清單

- `/docs/spec/phase1-product-pricing.md`（本檔）
- `/docs/api/pricing-openapi.yaml`（API 定義）
- `/src/main/resources/config/liquibase/changelog/2025102117xxxx_added_entity_*.xml`（5 個）
- `/src/main/java/.../pricing/*`（定價服務、查詢、DTO、Mapper、Controller）
- 單元測試 `/src/test/java/.../pricing/*`；Seed CSV 置於 `/src/main/resources/config/liquibase/fake-data/`。

---

> **備註**：Phase 1 僅提供「價目表＋階梯＋簡單規則」的穩定落地；Phase 2 可擴充「繼承/覆寫、促銷條件、毛利守門、FX 轉換、區域/倉別差異」等。
