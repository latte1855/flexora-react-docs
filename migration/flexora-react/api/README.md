# Flexora ERP API 設計與 Roadmap（JHipster 8.11.0 / Spring Boot + JWT + React + PostgreSQL + Redis）

> 依據現況設定（Monolith、JWT、Redis Cache、React、i18n：zh-tw/en）與既有模組規格彙整而成，所有時間以 UTC、金額/數量精度遵循專案規範。

## 0) 設計總綱（Design Tenets）

1. **資料一致性優先**：所有業務異動以「DB 驅動狀態機 + 交易分錄實體」為準；如 IM 的庫存/成本以 `InventoryTransaction` 與 `CostLayer` 做最終一致。
2. **一切可設定、少改程式**：流程採「`*_status_def / *_event_def / *_state_transition / *_status_history` 四表」，事件/轉換改表即可上線；只有新增全新 `action_handler` 才需要改程式。`metadata` 一律 JSONB 對應 VO，不用 Map。
3. **SKU 為唯一交易單位**：報價/訂單/採購/出貨/退貨所有行都對 `ItemSku`；BOM 分為銷售綁品與製造用 BOM，各自獨立。
4. **地址／ExtAttr 規範**：單據上的地址用 JSONB 值物件；每模組自帶 ExtAttr 定義/值兩表；MapStruct 進出 DTO 必須 deep copy。
5. **部分唯一鍵與軟刪**：所有 `*_no` 與代碼類欄位皆採 `WHERE deleted=false` 的 partial unique；內建樂觀鎖 `version`。
6. **估值與庫存**：預設 Moving Average，SKU 可覆寫 FIFO；**永遠產生成本層**，報表可切換 AVG/FIFO。

---

## 1) API 分層（Layering）與檔案佈局

### 1.1 專案目錄（建議新增）

```
docs/
  api/
    README.md               ← 本文件
    OPENAPI.md              ← 產生與維運（springdoc / maven plugin）
    CONVENTIONS.md          ← 命名、分頁、排序、錯誤格式、Idempotency、樂觀鎖
    MODULES.md              ← 各模組端點索引（連到子檔）
    flows/
      sales-order.md
      purchase.md
      inventory.md
      delivery-return.md
      quotation.md
```

### 1.2 Controller/Service 切分原則

- **CRUD**：沿用 JHipster 產生的 `*Resource`（`/api/*`）。
- **流程事件**：新增「事件端點」：`POST /api/{module}/{id}/events/{eventCode}`，由「表驅動狀態機」解析與執行（守衛/副作用）。
- **查詢/報表**：專用 `QueryResource`（避免塞進 CRUD）；必要時提供 `GET /api/{module}/_search?...`。
- **定價/稅/庫存服務**：抽成 Application Service（`PricingService`, `TaxService`, `InventoryService`）以便被多模組共用；稅/價目表快照落在行或表頭（各模組規格已定）。

---

## 2) API 共通規範（Conventions）

- **語言/在地化**：接受 `Accept-Language`（zh-tw/en），錯誤訊息與狀態名可回傳 i18n label。環境已開啟多語系。
- **時間與數值**：時間以 ISO-8601 (UTC)；金額 `DECIMAL(19,4)`、數量 `DECIMAL(19,6)`、稅率/折扣 `scale=6`；計算必定指定 RoundingMode（HALF_UP）。
- **分頁/排序/篩選**：沿用 JHipster 預設 `?page=..&size=..&sort=field,asc`；複雜查詢另開 `_search`。
- **錯誤格式**：統一 `Problem+JSON`（`type/title/status/detail/violations[]`）；事件守衛違反回 409 / 422。
- **Idempotency**：對「指令型端點（事件/過帳）」支援 `Idempotency-Key`（header）；伺服器側以 `(key,user,resource,event)` 做 5–15 分鐘短期去重。
- **樂觀鎖**：要求修改/事件帶 `If-Match: W/"{version}"` 或在 body 帶 `version`；衝突回 409。
- **授權與資料權限**：每筆業務資料帶 `owner_id`（`Owner` 模型）；RLS 規則以目前登入者的 `deptOwnerIds/teamOwnerIds` 做篩選。

---

## 3) 模組端點與事件（重點）

> 下列只列「非 CRUD 的行為端點」與「必備查詢」，CRUD 由 JHipster 既有 `*Resource` 提供。資料與流程說明完全沿用各模組規格。

### 3.1 產品與製造（Product & MFG）

- **定價**

  - `GET /api/pricing/price?skuId=&priceListId=&at=` → 回單價（含 tier/期間），僅查價不落地。

- **BOM/Routing 查詢**

  - `GET /api/skus/{id}/boms?activeAt=` / `GET /api/skus/{id}/routings?activeAt=`（版本/期間過濾）。

### 3.2 庫存（IM）

- **預留/釋放**

  - `POST /api/inventory/reservations`（批次預留：來源=SO/MO/TRANSFER；支援 bin/lot/serial）
  - `POST /api/inventory/reservations/{id}/release`（原因：出貨/取消/到期）。

- **可用量／安全量**

  - `GET /api/stock/available?skuId=&warehouseId=`
  - `GET /api/stock/low-safety?warehouseId=`（比較 `Certificate.safeQuantity`/或 SKU safety 設定；門檻預設 100 可由設定覆寫）—此為你近期需求的 API 化版本。

- **估值設定**

  - `PUT /api/valuation/sku/{skuId}`（FIFO/AVG 切換，需審批）

### 3.3 報價（Quotation）

- **事件**

  - `POST /api/quotations/{threadId}/events/{eventCode}`（`send/approve/reject/expire/cancel`）
  - `POST /api/quotations/{threadId}/revisions`（建立新版本；舊版不可變）

- **轉單**

  - `POST /api/quotations/{threadId}/revisions/{revNo}:to-sales-order`（部分/整單轉 SO，保留 `sales_order_quotation_link`）

### 3.4 訂單（Sales Order）

- **事件**

  - `POST /api/sales-orders/{id}/events/confirm` → 建立/更新 Reservation、計算 `promise_date`
  - `POST /api/sales-orders/{id}/events/cancel`
  - `POST /api/sales-orders/{id}/events/ship.update` → 由 DN 回寫已出貨/完成狀態。

- **查詢**

  - `GET /api/sales-orders/{id}/fulfillment`（已出/未出/待補）

### 3.5 出貨/退貨（DN/CRN）

- **DN 事件**

  - `POST /api/delivery-notes/{id}/events/confirm`
  - `POST /api/delivery-notes/{id}/events/ship`（過帳：IM `ISSUE`、釋放 Reservation、計稅、回寫 SO）
  - `POST /api/delivery-notes/{id}/events/cancel`。

- **CRN 事件**

  - `POST /api/customer-returns/{id}/events/receive`（入 QUARANTINE）
  - `POST /api/customer-returns/{id}/events/inspect.pass|inspect.fail|restock|scrap|cancel|close`。

### 3.6 採購（PO/GRN/SRN/RFQ）

- **PO 事件**：`confirm/cancel`
- **GRN 事件**：`receive/putaway/iqc.pass/iqc.fail`（對 IM 產生 `RECEIPT`，隔離到 AVAILABLE 的轉移）
- **SRN 事件**：`issue`（IM `ISSUE`）
- **RFQ**：`publish/close/award`（選定供應商後產生 PO 草稿）。

---

## 4) OpenAPI 與文件產生（OPENAPI.md 摘要）

- **工具**：採 `springdoc-openapi`（JHipster 8 預設支援），自動掃描 `@Operation/@Schema`（目前 DTO 已有 `@Schema` 註解，延續使用）。
- **路徑策略**

  - CRUD：`/api/{entities}`（現成）
  - 事件：`/api/{module}/{id}/events/{eventCode}`（POST）
  - 查價/稅/庫存：`/api/pricing/*`、`/api/tax/*`、`/api/stock/*`

- **版本**：以 HTTP Header `X-API-Version` 控制，必要時對外暴露 `/v1/**` Alias（Gateway 前置可再談）。
- **示例與契約測試**：為每個事件端點提供最小請求/回應範本（JSON Examples）；用 Cucumber 覆蓋 happy/guard/edge。

---

## 5) 差異與風險清單（針對 9 份檔）

1. **流程 `metadata` 的型別與對映**

   - 你的規範要求 **JSONB → 固定 VO**，避免 Map；請檢查現有四表的實體是否已改成 `@Type(JsonType.class)` + VO（`StatusMetadataVO`/`EventMetadataVO`/`TransitionMetadataVO`）。目前規範文件明確，但程式是否已對齊需再確認。

2. **ExtAttr 一致性**

   - 產品、報價、訂單、採購等模組都定義了 ExtAttrDef/Value；請確認命名一致（`*_ext_attr_def/value`）與唯一鍵規則（`owner_id + attr_code` partial unique）。

3. **安全庫存 API 的來源**

   - IM 規格以 SKU/WH 設定安全量；你在近期需求中也提到「若查不到設定則預設 100」的行為，請決定權威來源（`InventorySetting`/`SkuPolicy`/`Certificate.safeQuantity` 何者為準）並寫回規格；否則跨模組會互相覆蓋。

4. **Moving Average + FIFO**

   - 「永遠建 CostLayer」很好，但 API 必須明示「估值切換」的可見行為（切換點前後報表一致性、禁止在未對齊期間切換）。規格已定方向，落地要在 API 加保護。

5. **單據地址用 JSONB VO**

   - 報價/訂單/DN/CRN 皆應一致；若目前 entity 還是用關聯 Address 表，需要依規格改為 VO，並調整 MapStruct。

6. **Owner / Department / Team 主鍵共享**

   - 你的資料權限規格採 `Owner.id` 為共同主鍵來源（`@MapsId`）；請再次確認目前 `User/Department/Team` 是否已完成「建立順序」與「禁止更換依附」的檢查，否則會導致 FK 破壞。

---

## 6) Roadmap（8 週、四階段）

> 預設你已具備所有 CRUD；以下專注「**流程事件、計算引擎、查詢與一致性**」。每階段都附最小交付物（PR/測試/文件）。

### Phase 0 — 基線與文件（第 0–0.5 週）

- 建立 `docs/api/` 結構與三份基礎文件：`README.md`（本檔）、`OPENAPI.md`、`CONVENTIONS.md`。
- 啟用 springdoc OpenAPI UI + JSON 輸出；把事件端點的 `@Operation` 與範例補齊。
- **交付**：OpenAPI JSON 可被前端（React）與 Postman 匯入；README 有完整端點索引與守衛條件摘要。

### Phase 1 — 狀態機事件層（第 1–2 週）

- 新增通用「事件端點模板」與「Guard/Handler SPI」；將 **Quotation / SalesOrder / DeliveryNote / CustomerReturn / Purchase** 套用。
- `metadata` VO 與 JSONB 型別落地；加上快取與「重新載入流程定義」API。
- **交付**：每模組至少 3–5 個事件流可跑，含 409/422 守衛測試；OpenAPI 範例完成。

### Phase 2 — 計價/稅/庫存引擎（第 3–5 週）

- `PricingService`：支援 (priceList, sku, at)；Sales/Quotation 共用。
- `TaxService`：讀 `tax_code/tax_rate_line`，依行與單頭折扣順序計算與捨入（各模組規格一致）。
- `InventoryService`：Reservation（建立/釋放）、IM 分錄過帳（RECEIPT/ISSUE/ADJUSTMENT）、CostLayer 持久化。
- **交付**：SO Confirm 能預留；DN Ship 能扣帳計稅回寫；CRN Receive 能入 QUARANTINE。

### Phase 3 — 報表與查詢、權限與稽核（第 6–8 週）

- 典型查詢：可用量、待補、低於安全量、已出/未出、採購到貨進度、BOM 爆展清單。
- 權限：以 `Owner` 與 Dept/Team 關係計算能見度；將條件下沉到 Repository 自動附掛。
- 稽核：JaVers 快照（至少對單據表頭/行）；事件結果寫入 `*_status_history` 與分錄。
- **交付**：前端能透過 REST 完成從報價→訂單→出貨/退貨的完整 Happy Path，並可用查詢 API 驗證庫存與成本一致。

---

## 7) 模組索引（Modules Quick-Index）

- **產品/製造**：Item/ItemSku/Barcode/UoM/PriceList/ItemPrice、WorkCenter、BOM/Routing、Bundle。
- **庫存（IM）**：Warehouse/Bin、StockByBin、Reservation、InventoryTransaction、CostLayer、Replenishment。
- **報價（Quotation）**：Thread/Revision/Item/Tax、Status 四表、ExtAttr、轉單規則。
- **訂單（SO）**：Header/Item/Tax、Status 四表、ExtAttr、DN 回寫。
- **出貨/退貨（DN/CRN）**：Header/Item/Serial/Tax、與 IM 的 ISSUE/RECEIPT 關聯。
- **採購（PO/GRN/SRN/RFQ）**：Vendor/Contact/Bank、PO/GRN/SRN/RFQ 全流程與稅務。
- **資料權限**：Owner/Department/Team、UserDepartment/UserTeam 關聯與 `@MapsId` 流程、RLS 規則。

---

## 8) 驗收清單（API 面）

- [ ] OpenAPI JSON/HTML 成功匯入；所有事件端點有範例。
- [ ] `metadata` 欄位皆為 VO（非 Map），有單元測試驗證序列化。
- [ ] SO Confirm 會建立/更新 Reservation；DN Ship 會釋放/扣帳/計稅/回寫；CRN Receive 入 QUARANTINE。
- [ ] 稅計算順序與捨入一致（行 → 單頭；折扣先於稅）。
- [ ] ExtAttr API 可對 Item/SKU/SO/Quotation 等模組做 CRUD，且唯一定義符合 partial unique。
- [ ] Owner/Dept/Team 權限：典型清單（SO、PO、DN）能依登入者自動過濾。

---

### 附錄 A — 我已核對並採用的 9 份檔案（與用途）

- `.yo-rc.json`：Monolith + JWT + Redis + React、i18n 語系、實體清單（對照 API 範圍）。
- `資料權限管理 Domain 規格.md`：Owner/@MapsId、Dept/Team/關聯與 RLS 規範。
- `Flexora ERP 開發規範 – 資料結構與版本管理（含 JaVers）.md`：時間/數值規範、狀態機四表、JSONB VO、ExtAttr、審計/樂觀鎖。
- `Flexora ERP — 產品管理＋製造（BOM/Routing）規格.md`：SKU 唯一交易、BOM/Routing/PriceList。
- `庫存管理（IM）完整規格書.md`：Reservation、InventoryTransaction、CostLayer、估值法。
- `Flexora 報價模組（Quotation）完整規格書.md`：Thread/Revision、事件、轉 SO。
- `Flexora 訂單模組（Sales Order）完整規格書.md`：事件 `confirm/cancel/ship.update`、DN 回寫。
- `Flexora 出貨/客退 完整規格書.md`：DN/CRN 事件與稅/IM 整合。
- `Flexora CRM 三大模組整合規格書.md`：與 CRM 對接（Customer/Contact/Opportunity 等）。

> 另外我也閱讀了 `dto.txt` 的樣例，確認 DTO 已有 `@Schema` 註解與欄位說明，可直接被 springdoc 掃描。

---

## 建議下一步（立即可做）

1. 我直接把本檔當作 `docs/api/README.md` 初稿。
2. 新增 `docs/api/OPENAPI.md` 與 `docs/api/CONVENTIONS.md` 兩檔（我可以先寫第一版）。
3. 依 Phase 1 實作共用「事件端點模板」與 Guard/Handler SPI，把 **Quotation** 先串起來（最小流：`send/approve/cancel`）。

---

如果你要，我也可以馬上把 `OPENAPI.md` 與 `CONVENTIONS.md` 的第一版草稿一起補上。接著我們用這個 Roadmap 開分支逐步推進就行。
