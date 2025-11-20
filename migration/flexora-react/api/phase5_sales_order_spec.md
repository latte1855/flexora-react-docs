# Phase 5 — 銷售訂單（Sales Order）開發規格書（Flexora v2.0）

> 對齊《Flexora ERP 開發規範 – 資料結構與版本管理（含 JaVers）》與 `flexora-salesorder-management-spec.md`。  
> 基礎模組：Spring Boot 3.4.x、React 18 前端、PostgreSQL、Redis（快取）。  
> 共通要求：時間以 UTC 儲存、金額 `DECIMAL(19,4)`、中間運算 `DECIMAL(19,6)`、稅率 `DECIMAL(7,6)`、軟刪 + 樂觀鎖、地址/ExtAttr 皆採 JSONB VO。

---

## 1. 目標與範圍

1. 提供 Sales Order Header/Item CRUD 與查詢 API，支援價目表快照、地址快照、ExtAttr、附件。
2. 完整實作訂單狀態機（SUBMIT → CONFIRM → PARTIAL_SHIP/SHIP → FULFILLED / CANCEL），與 IM 預留、Delivery Note 出貨、AR 請款串接。
3. 支援報價轉訂單（自 Phase 4 Quotation），並回寫 `SalesOrderQuotationLink`。
4. 暫停跨模組整合（CRM 事件 listener、PDF 寄送、模板），於 Phase 5.1 以後統一收斂；QTN-7 類似的 E2E 亦移至全部模組完成後。

---

## 2. 資料模型摘要

| 實體                                       | 說明                                                                                       |
| ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `sales_order`                              | 訂單表頭：訂單號、客戶、幣別、匯率、地址快照、付款條件、運費/手續費、總金額、狀態、ExtAttr |
| `sales_order_item`                         | 訂單行：SKU、描述、數量、單價、折扣、稅、承諾交期、來源 Quotation 行參考                   |
| `sales_order_item_tax` / `sales_order_tax` | 行級/單頭稅明細                                                                            |
| `sales_order_status_*` 四表                | 狀態、事件、轉換、歷史；由 CSV seed 管理                                                   |
| `sales_order_ext_attr_def`                 | 訂單客製欄位定義                                                                           |
| `sales_order_quotation_link`               | SO ↔ Quotation Revision 對應，保存轉單量、備註                                            |
| `document_link`                            | 綁定訂單／訂單版本的附件（沿用共用 Document 模組）                                         |

> 地址欄位採 `AddressSnapshot` VO；`properties`（ExtAttr 值）需為合法 JSON 物件，寫入前由 Service 驗證。

---

## 3. 流程與狀態

### 3.1 狀態碼

| 代碼                | 說明         | 終結 | 備註                      |
| ------------------- | ------------ | ---- | ------------------------- |
| `DRAFT`             | 草稿         | 否   | 建立、可編輯              |
| `CONFIRMED`         | 已確認       | 否   | 完成審核/預留，可進行出貨 |
| `PARTIAL_FULFILLED` | 部分履約     | 否   | DN 已出部分               |
| `FULFILLED`         | 全數出貨完成 | 是   | 出貨完成，待請款          |
| `CANCELLED`         | 已取消       | 是   | 釋放預留、不可再變動      |

### 3.2 事件

| 事件           | 說明         | 守衛 / 備註                                         |
| -------------- | ------------ | --------------------------------------------------- |
| `SUBMIT`       | 提交草稿     | 預檢行項存在與金額合理；僅紀錄歷史                  |
| `CONFIRM`      | 確認（預留） | `order.isValid() && order.hasItems()`，呼叫 IM 預留 |
| `PARTIAL_SHIP` | 部分出貨     | 由 DN 回寫                                          |
| `SHIP`         | 出貨完成     | 由 DN 回寫                                          |
| `CANCEL`       | 取消         | `order.canCancel()`，需釋放預留                     |

### 3.3 狀態轉換（範例）

| From                | Event          | To                  | Action/守衛                                               |
| ------------------- | -------------- | ------------------- | --------------------------------------------------------- |
| `DRAFT`             | `SUBMIT`       | `DRAFT`             | 只寫歷史，不更新狀態                                      |
| `DRAFT`             | `CONFIRM`      | `CONFIRMED`         | 執行預留（`SO_CONFIRM_HANDLER`，metadata `reserve=AUTO`） |
| `DRAFT`             | `CANCEL`       | `CANCELLED`         | 無守衛                                                    |
| `CONFIRMED`         | `PARTIAL_SHIP` | `PARTIAL_FULFILLED` | 由 DN `ship.update` 回寫，更新 shippedQty                 |
| `PARTIAL_FULFILLED` | `SHIP`         | `FULFILLED`         | DN 最後一筆出貨                                           |
| `CONFIRMED`         | `SHIP`         | `FULFILLED`         | 一次性全出貨                                              |
| `CONFIRMED`         | `CANCEL`       | `CANCELLED`         | `order.canCancel()`；需釋放預留／通知 Quotation link      |

### 3.4 IM 自動預留／釋放

- `ReservationStrategy=AUTO` 才會在 `CONFIRM` 事件呼叫 `InventoryCommandService.reserve`，每一個可用行項都送出一筆 `ReserveCommand`（ReservationType=SALES、ReferenceType=SALES_ORDER、ReferenceId=orderId）。
- 若任一行項缺 SKU / 倉庫 / 正數數量，會以 `salesOrderWorkflow` 錯誤中止確認，避免產生不完整的預留。
- `CANCEL` 事件統一透過 `InventoryReservationBulkReleaseRequest` 釋放所有 `SALES_ORDER` 來源預留，優先使用事件傳入的 reason；若無則由系統自動填入 `SO#xxxx cancel auto release`。
- 預留/釋放皆記錄在 `SalesOrderStatusHistory`，並可透過 `GET /api/sales-orders/{id}/status-history` 與 `GET /api/sales-orders/{id}/quotation-links` 回顯。

### 3.5 狀態事件發佈

- `SalesOrderWorkflowService` 每次狀態轉換都會透過 `SalesOrderStatusChangedPublisher` 發佈 `SalesOrderStatusChangedEvent`（ApplicationEvent）。
- Payload 包含 `salesOrderId/orderNo/eventCode/fromStatus/toStatus/fulfillmentStatus/ownerId/customerId/grandTotal/currency/occurredAt`，供 CRM、Dashboard 或通知模組訂閱。
- `ship.update` 透過 Workflow 觸發 `PARTIAL_SHIP` 或 `SHIP` 時同樣會產生事件，確保整體行為一致。

### 3.6 地址 / ExtAttr / 附件 API

- `POST /api/sales-orders/{id}/addresses`：寫入 `billingAddress`、`shippingAddress` 的 `AddressSnapshot` VO，服務端會進行深拷貝並驗證 JSON 結構。
- `POST /api/sales-orders/{id}/ext-attrs`：依 `SalesOrderExtAttrDef` 轉型後寫入 `properties.extAttrs`，支援 STRING/INT/DECIMAL/DATE/BOOL/JSON 等型態。
- `POST /api/sales-orders/{id}/documents`：綁定現有 `Document`，寫入 `DocumentLink`（`relationType`、`isPrimary`、note），供附件模組共用。
- 所有 API 均回傳 400 錯誤訊息以提示欄位缺漏或型態錯誤；整合測試 `SalesOrderCustomizationResourceIT` 針對三個端點驗證成功與失敗情境。

### 3.7 DeliveryNote 回寫（ship.update）

- `POST /api/sales-orders/{id}/events/ship.update`：由 DN 於過帳後呼叫，body 對應 `SalesOrderShipmentUpdateRequestDTO`（`sourceType/sourceId/shippedAt/lines[]` 等欄位）。
- 服務會累加 `SalesOrderItem.shippedQuantity`、計算 `FulfillmentStatus`，並依結果觸發 workflow `PARTIAL_SHIP` 或 `SHIP`，同時發佈 `SalesOrderStatusChangedEvent`。
- 若同一 DN 行要回寫多個 SO 行，可在 `lines[]` 中指定多筆，系統會檢查「不得超出原下單數量」並在超量時回傳 400。

### 3.8 Quotation Link API 與回寫

- Quotation 端 (`QuotationResource`) 已提供 `GET /api/quotations/{threadId}/links[?revisionId=]`，回傳 `SalesOrderQuotationLinkDTO`（含 `salesOrderId/linkType/linkedQuantity/linkedAt/note`）。
- SalesOrder 端 (`SalesOrderResource`) 亦提供 `GET /api/sales-orders/{id}/quotation-links`，便於 UI 在 SO 畫面顯示來源報價與部分轉單紀錄。
- 轉單取消：當 `SalesOrder` 觸發 `CANCEL` 事件時，系統會呼叫 `SalesOrderQuotationLinkService#markAsCancelled`，將相關 Link 的 `linkType` 更新為 `CANCELLED` 並附加取消備註；同時可重新從 Quotation 進行轉單。

---

## 4. API 設計總覽

| 類別             | 路由 / 方法                                                                                   | 說明                                                                      |
| ---------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 訂單 CRUD        | `GET/POST/PUT/PATCH/DELETE /api/sales-orders` + `/{id}:soft-delete`/`:{restore}` + `/_exists` | 基礎 CRUD，支援 Criteria 查詢；新增 orderNo 自動編號、重複檢查、軟刪/還原 |
| 明細維護         | `POST/PUT/DELETE /api/sales-order-items`                                                      | 僅限草稿；行項含 ExtAttr/附件引用                                         |
| 狀態/事件        | `POST /api/sales-orders/{id}/events/{eventCode}`                                              | 觸發 SUBMIT / CONFIRM / CANCEL；ship/partial_ship 由 DN module 觸發       |
| 定價/稅試算      | `POST /api/sales-orders/preview`                                                              | 依行項/整單折扣/運費/手續費計算金額（不落地），回傳行級與總額             |
| ExtAttr          | `POST /api/sales-orders/{id}/ext-attrs`                                                       | 依 ExtAttrDef 寫入 `properties.extAttrs`                                  |
| 地址快照         | `POST /api/sales-orders/{id}/addresses`                                                       | 同 JSON VO 格式                                                           |
| 附件             | `POST /api/sales-orders/{id}/documents`                                                       | `DocumentLink` 綁定                                                       |
| Link 查詢        | `GET /api/sales-orders/{id}/quotation-links`                                                  | 顯示來源報價關聯                                                          |
| Workflow 歷史    | `GET /api/sales-orders/{id}/status-history`                                                   | 供 UI 顯示事件時間軸                                                      |
| 報價轉單（回寫） | 由 Quotation 模組呼叫 `/api/quotations/{threadId}/revisions/{revId}:to-sales-order`           | Phase 4 已完成；本階段提供 link 查詢、SO 端僅需讀取                       |

> orderNo 由 `NumberingService.next("SALES_ORDER", "orderNo")` 產生，POST 時若未帶值由後端自動給號；`GET /api/sales-orders/_exists?orderNo=&excludeId=` 可供 UI 即時檢查。`POST /api/sales-orders/{id}:soft-delete` 只切換 `deleted` 旗標、保留資料，`POST /api/sales-orders/{id}:restore?newOrderNo=` 可在衝突時指定新的單號。`POST /api/sales-orders/preview` 則會依傳入行項、整單折扣與運輸費用，在後端計算並回傳小計、稅額、總額與行級明細，供前端預覽與驗證。`deleted`/`enabled` 等旗標欄位不應由前端直接更新；新增時後端自動補預設值，更新時若需變動請改用對應的 API（例如 `:soft-delete`）。

### 4.1 訂單試算輸入

| 欄位                  | 型態               | 說明                         |
| --------------------- | ------------------ | ---------------------------- |
| `items[]`             | LineItem           | 行序、數量、單價、折扣、稅率 |
| `headerDiscountType`  | `NONE/AMOUNT/RATE` | 整單折扣型態                 |
| `headerDiscountValue` | `BigDecimal`       | 整單折扣值（金額或比率）     |
| `shippingFee`         | `BigDecimal`       | 運費（預設 0）               |
| `handlingFee`         | `BigDecimal`       | 手續費/其他調整（預設 0）    |

LineItem 欄位：`itemId`（選填）、`ordinal`、`quantity`、`unitPrice`、`discountType`、`discountValue`、`taxRate`。試算結果使用 `SalesOrderCalculationResultDTO` 回傳 `subtotal/headerDiscountTotal/taxTotal/grandTotal` 及每行折扣、稅額、含稅金額。

> Swagger 要求：所有新增/調整 Resource 類別與方法需補 `@Tag`、`@Operation`，與 AGENTS.md 規範一致。

> **TODO（Workflow 錯誤訊息）**：目前 `SalesOrderWorkflowService` 透過 `BadRequestAlertException` 回傳錯誤時，`detail` 仍包裹 `ProblemDetailWithCause[...]` 字串，導致前端只能看到 `400 BAD_REQUEST …`。請調整為直接在 `detail`/`message` 填入可讀文字（例如「訂單已逾期，請延展有效期限」），並比照 Quotation Workflow 同步套用，讓 UI 無須額外解析。

---

## 5. 與其他模組整合

| 模組               | 整合點                                                                 | 備註                                                                                                              |
| ------------------ | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Quotation          | `SalesOrderQuotationLink` 追蹤來源；取消訂單時會標記 Link 並釋放轉單量 | Link API 已提供（Thread / Revision / SO 三向查詢），SO 取消會將 linkType 改為 `CANCELLED` 並再次允許部分/全量轉單 |
| Inventory          | Confirm 觸發預留（ReservationCommand）、取消釋放、出貨由 DN 回寫       | 透過 `SalesOrderWorkflowService` action handler 呼叫 IM REST                                                      |
| Delivery Note      | `ship.update` 事件進入 SO workflow，更新 shippedQty / status           | DN 模組需傳遞 shipmentId、行項對應關係                                                                            |
| Document           | 訂單附件（合約、PDF）、DN 生成的 PDF 亦可關聯                          | 沿用共用 `DocumentLink`                                                                                           |
| CRM / Notification | `sales_order.status.changed`（ApplicationEvent）                       | `SalesOrderStatusChangedPublisher` 已發佈；Listener 視模組需求掛載                                                |

---

## 6. 測試與驗收

### 6.1 單元 / 服務

| 類別                           | 重點                                                   |
| ------------------------------ | ------------------------------------------------------ |
| `SalesOrderServiceTest`        | CRUD、計算、JSON 驗證（properties/address）            |
| `SalesOrderWorkflowResourceIT` | 事件 API（SUBMIT/CONFIRM/CANCEL）、歷史/守衛/Link 回傳 |
| `SalesOrderImIntegrationTest`  | Confirm 時呼叫 IM 預留、取消釋放                       |
| `SalesOrderShipmentSyncTest`   | 模擬 DeliveryNote 回寫，驗證部分/全部出貨              |
| `SalesOrderExtAttrResourceIT`  | ExtAttr API、JSONB 寫入                                |
| `SalesOrderDocumentResourceIT` | 附件連結                                               |

### 6.2 Integration / MockMvc

1. CRUD + Workflow Happy Path：建立 → SUBMIT → CONFIRM → 模擬 DN → SHIP → 完結。
2. 守衛：無行項 / 無 channel / 無 reason / 已逾期等情境應回 400。
3. 取消：CONFIRMED 訂單呼叫 CANCEL，確認 IM 預留釋放、Quotation link 更新。
4. 報價轉單：由 Phase 4 測試覆蓋；本模組需提供 API 供 UI 顯示 link 與狀態。

> `com.asynctide.flexora.my.phase5.salesorder.SalesOrderWorkflowResourceIT` 目前涵蓋：Confirm 成功＋歷史查詢、無明細守衛、Cancel 需 reason、Link API 查詢，以及 AUTO 策略下的預留建立/釋放驗證。

### 6.3 驗收清單

1. 訂單 CRUD 完成並符合通用欄位規範。
2. Workflow 事件照 CSV 定義運作，歷史可查。
3. Confirm/CANCEL 與 IM 預留串接。
4. 部分/多次出貨由 DeliveryNote 觸發 `ship.update` 事件並回寫 shippedQty。
5. ExtAttr、地址、附件 API 可用。

### 6.4 測試執行指引

- 單元／整合測試：
  - `SalesOrderCustomizationResourceIT`：覆蓋地址/ExtAttr/附件 API。
  - `SalesOrderWorkflowResourceIT`：確認 SUBMIT/CONFIRM/CANCEL、ship.update、Reservation 建立/釋放等守衛。
  - `QuotationToSalesOrderServiceIT`：部分轉單、轉單量計算、SO 取消後釋放 Link。
  - `SalesOrderStatusChangedPublisherTest`：驗證狀態事件 payload。
- 指令建議：
  - 無需前端資源時可使用 `./mvnw -Dskip.npm -Dskip.installnodenpm test`。
  - 含 Testcontainers 的整合測試需在可使用 Docker 的環境執行 `./mvnw verify`，以便與 PostgreSQL/Redis 等服務整合測試。

---

## 7. 開發任務（建議拆單）

| 編號  | 任務內容                                                           | 依賴              | 目前狀態 |
| ----- | ------------------------------------------------------------------ | ----------------- | -------- |
| SOT-1 | Entity + Repository：`SalesOrder`, `SalesOrderItem`, 稅表、ExtAttr | －                | ✅ Done  |
| SOT-2 | 訂單計價/稅計算服務 + CRUD Resource                                | SOT-1             | ✅ Done  |
| SOT-3 | Workflow：狀態 CSV + `SalesOrderWorkflowService` + 事件 API        | SOT-1             | ✅ Done  |
| SOT-4 | IM 預留整合（Confirm/CANCEL）                                      | SOT-3             | ✅ Done  |
| SOT-5 | Shipment 回寫 & DN 通知（`ship.update` → SHIP/PARTIAL_SHIP）       | SOT-3             | ✅ Done  |
| SOT-6 | ExtAttr / 地址 / 附件 API                                          | SOT-2             | ✅ Done  |
| SOT-7 | 報價 Link API + 轉單取消回寫                                       | Phase 4 Quotation | ✅ Done  |
| SOT-8 | 測試與驗收（單元、整合、守衛）                                     | 全                | ✅ Done  |
| SOT-9 | E2E / Postman（暫緩，與 Phase 5.1 其他模組一併處理）               | 全                | ⏸ TODO  |

> **TODO**：E2E、CRM 事件 Listener、PDF 寄送、模板與 Pricing Trace 串接待後續 Phase 5.1。

---

## 8. 後續工作（Phase 5.1）

1. **CRM 事件 Listener**：訂單狀態變化觸發 Campaign/Opportunity 更新（事件欄位同上，需補 `docs/domain/CRM Events.md`）。
2. **PDF + Mail**：Confirm/SEND 事件產 PDF, 上傳 Document, 經 Mail Service 寄送，保存寄送紀錄。
3. **訂單模板/複製**：從既有訂單或預設模板複製行項/ExtAttr，API `POST /api/sales-orders/{id}:clone`。
4. **Pricing Trace 串接**：Expose `GET /api/sales-orders/{id}/pricing-trace` 供客服檢視定價過程。
5. **E2E 套件**：待 Phase 4〜5 模組完成後，一次性補上 Postman / e2e 測試腳本。
6. **DTO 預設欄位精簡**：待所有模組完成後，統一檢視 `enabled` / `deleted` 等後端給值欄位，批次移除 DTO `@NotNull`／`requiredMode=REQUIRED`，避免前端必填。
7. **DefaultNumberingService 規則整理**：待 Phase 4/5 模組穩定後，彙整各實體給號需求（quotation / sales order / 其他主檔）補強 `DefaultNumberingService`，再行調整。

---

## 9. 範例 Payload

**建立訂單（草稿）**

```json
POST /api/sales-orders
{
  "customerId": 2001,
  "currencyCode": "TWD",
  "orderDate": "2025-11-10",
  "ownerId": 3101,
  "billingAddress": {
    "recipientName": "王大同",
    "companyName": "花見股份有限公司",
    "country": "TW",
    "city": "新北市",
    "line1": "文化路一段100號10樓"
  },
  "shippingAddress": {
    "recipientName": "江小美",
    "recipientPhone": "+886-2-12345678#308",
    "country": "TW",
    "city": "高雄市",
    "line1": "成功二路123號"
  },
  "items": [
    {
      "skuId": 1001,
      "quantity": "5",
      "unitPrice": "120.000000",
      "taxRate": "0.05",
      "needByDate": "2025-11-20"
    }
  ],
  "properties": { "CHANNEL": "WEB" }
}
```

**觸發事件**

```http
POST /api/sales-orders/9001/events/confirm
{
  "channel": "SYSTEM",
  "reason": null,
  "note": "自動轉單確認"
}
```

---

## 10. 參考文件

- `docs/flexora-salesorder-management-spec.md`
- `docs/api/phase4_quotation_spec.md`
- `docs/api/final/API 清單（含用途說明）.md`
- `docs/api/README.md`（Sales Order 章節）
- `docs/庫存管理 Inventory Management（IM）規格書.md`（預留、出貨流程）

---

## 11. 前端 UI 執行指引

1. **沿用並擴充 JHipster UI**：Sales Order 的新增頁、列表、詳情、Workflow 等既有畫面需繼續可用；新 ERP 操作（drawer、pipeline、Dashboard 小工具）必須在既有 route/component 基礎上延伸，而非替換或下架原組件。
2. **Materia Theme**：UI 顏色、陰影、按鈕等應沿用 Bootswatch Materia token，必要時僅在 `scss` 透過 `var(--bs-*)` 自訂，確保與 Quotation 模組一致。
3. **元件優先順序**：優先使用專案現有的 Bootstrap 5、JHipster Table/Form、Redux hooks、`app/shared/components`。如需導入第三方 UI Library，需評估與 Materia Theme、權限控制、i18n 的相容性與安全性。
4. **欄位/旗標處理**：畫面需與 DTO 欄位一致，但可採「核心 + 詳細」佈局。`deleted/enabled` 等旗標不提供一般表單編輯，改用 Soft-delete/Restore API；Numbering 允許空值，由後端補號並在 UI 顯示提示。
5. **Workflow / Quick Action**：Toolbar 預設包含 Submit、Confirm、Cancel、Release Reservation、Export；寄送 PDF、Excel 匯入等進階功能以 disabled menu 預留並標註 Phase 5.1。Workflow 表單需檢查 channel/reason/line item 守衛。
6. **權限與 DataScope**：列表使用後端 Criteria 回傳的結果，若需 owner filter，僅提供條件欄位而非前端自行過濾；詳情頁、workflow 按鈕與 Quotation Link 區塊需依 API 回傳 `canEdit/canTriggerEvent` 決定可見性。Ship/Pipeline 操作若遭拒絕須顯示 toast 並還原畫面。
7. **跨模組共用**：地址、ExtAttr、附件、Workflow Timeline 等元件需與報價模組共用（位於 `shared` 或 `sales/quotation` 共用層），確保體驗一致並減少維護成本。
8. **檢查清單同步**：所有 UI 需求與實作細節，需同步記錄在 `docs/ui/quotation-ui-wireframe.md`、`docs/ui/sales-order-ui-notes.md`（或對應的 wireframe）與 React UI Checklist（`docs/ui/react-ui-checklist.md`），必要時更新 `docs/ui/README.md` 以保持結構說明最新，並於 PR 敘述說明與後端 API 的對應關係。
9. **守衛對齊**：前端必須對 `deleted/enabled`、workflow 守衛、Numbering 自動給號、Quotation Link 權限等規則進行相同處理，避免 UI 與後端邏輯產生落差。
