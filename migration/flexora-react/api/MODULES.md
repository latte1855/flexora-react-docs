# Flexora ERP 模組開發順序與 API 模組索引

> 文件位置：`docs/api/MODULES.md`  
> 適用版本：JHipster v8.11.0 / Spring Boot + JWT / React / PostgreSQL + Redis  
> 更新日期：2025-10-15（Asia/Taipei）

---

## 0. 開發原則

1. **先底座、後應用**：先完成權限、稅、商品、定價、庫存等共用基礎，後續模組只需對接。
2. **事件驅動、表驅動**：所有流程皆以四表（`*_status_def`, `*_event_def`, `*_state_transition`, `*_status_history`）管理。
3. **單一交易單位（SKU）**：所有交易、稅、庫存、估值均以 SKU 為單位。
4. **JSONB + VO 結構化**：ExtAttr / Address / Metadata 統一使用 VO 對應 JSONB。
5. **Idempotency + Optimistic Lock**：所有「指令型 API」支援重試安全，使用 `If-Match` + `Idempotency-Key`。
6. **Partial Unique + 軟刪除**：唯一鍵採 `WHERE deleted=false`，保留歷史版本。
7. **審計 + JaVers**：所有主要單據支援版本追蹤與稽核。

---

## 1️⃣ Phase 0 — 基礎底座（System Foundation）

| 項目     | 主要實體                                                    | 功能說明                                   | 主要 API / 事件端點                                        | 測試重點                                        |
| -------- | ----------------------------------------------------------- | ------------------------------------------ | ---------------------------------------------------------- | ----------------------------------------------- |
| 資料權限 | `Owner`, `Department`, `Team`, `UserDepartment`, `UserTeam` | 定義權限與資料可見範圍                     | `/api/departments`, `/api/teams`, `/api/users/{id}/assign` | 驗證部門階層（`parentHierarchy`）、權限過濾生效 |
| 系統設定 | `SystemSetting`                                             | 全域設定（稅率模式、預設倉別、安全庫存等） | `/api/system-settings`                                     | 變更設定立即生效於新交易                        |
| 稅務管理 | `TaxCode`, `TaxRateLine`                                    | 稅務定義與複合稅率計算                     | `/api/taxes/codes`, `/api/taxes/rates`                     | 測試稅率運算精度與順序                          |
| 文件管理 | `Document`, `DocumentLink`                                  | 單據附件與關聯                             | `/api/documents/upload`, `/api/documents/{id}/links`       | 上傳、關聯 CRUD                                 |

---

## 2️⃣ Phase 1 — 商品與定價主檔（Product & Pricing）

| 項目       | 主要實體                                       | 功能說明             | 主要 API / 事件端點                          | 測試重點               |
| ---------- | ---------------------------------------------- | -------------------- | -------------------------------------------- | ---------------------- |
| 單位換算   | `Uom`, `UomConversion`                         | 基本單位與換算率管理 | `/api/uoms`, `/api/uom-conversions`          | 換算正確、不可循環定義 |
| 品牌與屬性 | `Brand`, `ItemAttribute`, `ItemAttributeValue` | 品牌、屬性定義       | `/api/brands`, `/api/item-attributes`        | CRUD 正確              |
| 產品群組   | `ItemGroup`, `Item`                            | 產品階層與主檔       | `/api/items`, `/api/item-groups`             | 階層維護正確           |
| SKU 定義   | `ItemSku`, `ItemBarcode`, `SkuAttribute`       | SKU 唯一交易單位     | `/api/skus`                                  | barcode 唯一性         |
| 價目表     | `PriceList`, `ItemPrice`                       | 多階梯、多期間定價   | `/api/pricing/price?skuId=&priceListId=&at=` | 驗證不同日期價格正確   |

---

## 3️⃣ Phase 2 — CRM 基礎（Customer & Contact）

| 項目     | 實體                                      | 功能說明     | API                           | 測試重點            |
| -------- | ----------------------------------------- | ------------ | ----------------------------- | ------------------- |
| 客戶主檔 | `Customer`                                | 客戶基本資料 | `/api/customers`              | CRUD                |
| 聯絡人   | `Contact`                                 | 客戶聯絡人   | `/api/contacts`               | CRUD                |
| 地址管理 | `Address`                                 | 通用地址資料 | `/api/addresses`              | JSONB 轉換          |
| ExtAttr  | `CustomerExtAttrDef`, `ContactExtAttrDef` | 擴充屬性     | `/api/ext-attr-defs/customer` | Partial unique 驗證 |

---

## 4️⃣ Phase 3 — 庫存核心（Inventory Management）

| 功能       | 實體                                | API / 事件端點                                                            | 測試重點          |
| ---------- | ----------------------------------- | ------------------------------------------------------------------------- | ----------------- |
| 倉別與儲位 | `Warehouse`, `Bin`                  | `/api/warehouses`, `/api/bins`                                            | 層級正確          |
| 即時庫存   | `StockByBin`                        | `/api/stock/available?skuId=&warehouseId=`                                | 數量一致性        |
| 庫存預留   | `InventoryReservation`              | `/api/inventory/reservations`, `/api/inventory/reservations/{id}/release` | 預留釋放流程      |
| 庫存交易   | `InventoryTransaction`, `CostLayer` | `/api/inventory/transactions`                                             | 過帳正確性        |
| 估值策略   |                                     | `/api/valuation/sku/{id}`                                                 | FIFO/AVG 切換測試 |

---

## 5️⃣ Phase 4 — 報價（Quotation）

| 功能     | 實體                                                                                            | API / 事件端點                                          | 測試重點    |
| -------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------- | ----------- | ------ | ------ | ------- | -------------- |
| 報價主檔 | `QuotationThread`, `QuotationRevision`                                                          | `/api/quotations`                                       | CRUD        |
| 報價項目 | `QuotationItem`, `QuotationRevisionTax`, `QuotationItemTax`                                     | `/api/quotations/{id}/items`                            | 稅計算正確  |
| 狀態管理 | `QuotationStatusDef`, `QuotationEventDef`, `QuotationStateTransition`, `QuotationStatusHistory` | `/api/quotations/{id}/events/send                       | approve     | reject | cancel | expire` | 狀態轉換與守衛 |
| 轉訂單   | `SalesOrderQuotationLink`                                                                       | `/api/quotations/{id}/revisions/{revNo}:to-sales-order` | SO 生成正確 |

---

## 6️⃣ Phase 5 — 銷售訂單（Sales Order）

| 功能     | 實體                                                                                                | API / 事件端點                         | 測試重點   |
| -------- | --------------------------------------------------------------------------------------------------- | -------------------------------------- | ---------- | ------------ | ------------------ |
| 訂單主檔 | `SalesOrder`, `SalesOrderItem`                                                                      | `/api/sales-orders`                    | CRUD       |
| 稅與折扣 | `SalesOrderTax`, `SalesOrderItemTax`                                                                | `/api/sales-orders/{id}/tax`           | 稅計算正確 |
| 狀態流   | `SalesOrderStatusDef`, `SalesOrderEventDef`, `SalesOrderStateTransition`, `SalesOrderStatusHistory` | `/api/sales-orders/{id}/events/confirm | cancel     | ship.update` | 預留、出貨回寫正確 |
| 擴充屬性 | `SalesOrderExtAttrDef`                                                                              | `/api/ext-attr-defs/sales-order`       | CRUD       |

---

## 7️⃣ Phase 6 — 出貨與客退（Delivery & Customer Return）

| 功能   | 實體                                                                                                                 | API / 事件端點                                                      | 測試重點                |
| ------ | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ----------------------- | ------------ | ------- | ------ | ------------- |
| 出貨   | `DeliveryNote`, `DeliveryNoteItem`, `DeliveryNoteItemSerial`, `DeliveryNoteTax`, `DeliveryNoteItemTax`               | `/api/delivery-notes`, `/api/delivery-notes/{id}/events/ship`       | IM ISSUE / 稅 / SO 回寫 |
| 退貨   | `CustomerReturnNote`, `CustomerReturnItem`, `CustomerReturnItemSerial`, `CustomerReturnTax`, `CustomerReturnItemTax` | `/api/customer-returns`, `/api/customer-returns/{id}/events/receive | inspect.pass            | inspect.fail | restock | scrap` | 庫存/檢驗流程 |
| 狀態流 | `DeliveryNoteStatusDef`, `DeliveryNoteEventDef`, `DeliveryNoteStateTransition`, `DeliveryNoteStatusHistory`          |                                                                     | 四表同步運作            |

---

## 8️⃣ Phase 7 — 採購全流程（RFQ → PO → GRN → SRN）

| 模組       | 實體                                                                             | API / 事件端點                                                    | 測試重點      |
| ---------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------- | ------------ | ------------- | --------------------- |
| 供應商主檔 | `Vendor`, `VendorContact`, `VendorBankAccount`                                   | `/api/vendors`                                                    | CRUD          |
| 詢價單     | `Rfq`, `RfqItem`, `RfqVendorQuote`, `RfqVendorQuoteItem`                         | `/api/rfqs`, `/api/rfqs/{id}/events/publish                       | close         | award`       | 招標/結標正確 |
| 採購單     | `PurchaseOrder`, `PurchaseOrderItem`, `PurchaseOrderTax`, `PurchaseOrderItemTax` | `/api/purchase-orders`, `/api/purchase-orders/{id}/events/confirm | cancel`       | 預留、稅計算 |
| 收貨單     | `PurchaseReceipt`, `PurchaseReceiptItem`                                         | `/api/purchase-receipts/{id}/events/receive                       | iqc.pass      | iqc.fail     | putaway`      | IM RECEIPT / IQC 流程 |
| 供應商退貨 | `SupplierReturn`, `SupplierReturnItem`                                           | `/api/supplier-returns/{id}/events/issue`                         | IM ISSUE 流程 |

---

## 9️⃣ Phase 8 — 製造（Manufacturing: BOM / Routing / MO）

| 功能     | 實體                                                                | API / 事件端點                         | 測試重點           |
| -------- | ------------------------------------------------------------------- | -------------------------------------- | ------------------ | -------------- | -------------- | ------ | ----------------------------- |
| 製程設定 | `WorkCenter`, `Routing`, `RoutingOp`                                | `/api/routings`                        | CRUD               |
| 物料清單 | `Bom`, `BomItem`                                                    | `/api/boms`                            | 階層 BOM 正確      |
| 製令     | `Mo`, `MoItem`                                                      | `/api/mos`, `/api/mos/{id}/events/plan | release            | issue.material | receive.finish | close` | 材料 ISSUE、成品 RECEIPT 正確 |
| 狀態流   | `MoStatusDef`, `MoEventDef`, `MoStateTransition`, `MoStatusHistory` |                                        | 事件與四表正確連動 |

---

## 🔟 Phase 9 — 報表與橫切強化（Reporting / Cross-cutting）

| 功能             | 說明               | API                        | 測試重點             |
| ---------------- | ------------------ | -------------------------- | -------------------- |
| JaVers 版本追蹤  | 單據歷程與差異比對 | `/api/audit/{entity}/{id}` | 版本快照正確         |
| 資料快取         | Redis 分區快取     | `/api/cache/clear`         | 快取失效機制         |
| Idempotency 驗證 | 重試安全           | 所有事件端點               | 重複呼叫不重複過帳   |
| 文件連結         | 文件上傳與關聯     | `/api/documents`           | 文件可關聯至任一實體 |

---

## ✅ 測試與驗收清單（每模組共用）

| 編號 | 驗收項目     | 說明                                        |
| ---- | ------------ | ------------------------------------------- |
| 1    | CRUD 測試    | 透過 Postman 自動化測試 CRUD                |
| 2    | 狀態轉換     | 每個事件端點至少 1 正向 + 1 守衛失敗測試    |
| 3    | ExtAttr CRUD | 驗證 partial unique 與 JSONB 正確性         |
| 4    | 稅與價格     | 單頭、行項、折扣與稅計算一致                |
| 5    | IM 過帳      | Reservation、Issue、Receipt、CostLayer 一致 |
| 6    | 權限篩選     | Owner/Dept/Team 條件正確作用                |
| 7    | OpenAPI 文件 | 所有端點有 @Operation 描述與範例            |
| 8    | JaVers 追蹤  | 修改後產生版本快照                          |
| 9    | Redis Cache  | 預留、稅率、價格快取一致性                  |
| 10   | E2E 流程     | Quotation → SO → DN → CRN → IM 整線通過     |

---

## 📈 擴充建議

- **Phase 10**：財務模組（AR/AP、Invoice、Credit Note）
- **Phase 11**：報表中心（BI、Dashboard、即時庫存與毛利分析）
- **Phase 12**：工作流程擴充（多階簽核、多部門路由）
- **Phase 13**：外部整合（電商 / 第三方物流 / 外部稅務 API）

---

> **作者註**：本文件僅描述模組與端點規劃，詳細 API 參數、請求/回應範例、錯誤代碼定義，請參考 `docs/api/OPENAPI.md` 與 `docs/api/CONVENTIONS.md`。
