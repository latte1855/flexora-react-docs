# 銷售訂單模組 Legacy 規格摘要

依 `docs/migration/flexora-react/flexora-salesorder-management-spec.md` 整理，並與 `flexora-react/src/main/java/com/asynctide/flexora/domain/SalesOrder*`, `Quotation`、`DeliveryNote` 等實作對照。

## 1. 核心流程

| 階段 | 描述 |
| --- | --- |
| 建立 / Pricing | 建立 Draft → 呼叫 Pricing Engine 計算金額 / 稅 / 折扣（参考 `pricing-rules.md`） |
| Confirm | 對 IM 預留 / 計算 Promise Date → 更新 `sales_order_status_def`（DB 驅動） |
| 出貨 | 透過 Delivery Note，出貨完畢後狀態走向 `FULFILLED`；部分/多單支援 `PARTIALLY_SHIPPED` |
| 轉單 | 由 Quotation Revision (或 Sales Order) 觸發 `sales_order_quotation` 關聯；可附 `CalculationTrace` 做稽核。 |

## 2. 資料與狀態

- Header (`sales_order`) + Line (`sales_order_item`) + 稅表 + ExtAttr + 狀態四表（`status_def/event_def/state_transition/status_history`）為模組基礎。
- 快照欄位提供 `sku_code/product_name/unit_price/tax_rate/discount` 等數據，避免後續主檔變更影響歷史資料。
- `document`/`document_link` 供附件；`calculation_trace` 為共用稽核表（可能由 `PricingService` 產生）。
- Enum `discount_type` 與 `sales_order_status_def.code`（DRAFT, CONFIRMED, PARTIALLY_SHIPPED, FULFILLED, CANCELLED）與 `phase4` workflow 保持一致。

## 3. 交互與 NEXT

- 與 IM/Cost Layer 連動：出貨觸發 `InventoryTransaction`，並依 `CostingMethod` 計算 COGS；請確認 `docs/specs/phase3-inventory/costing-rules.md` 目前公式是否一致。
- 與 Quotation/Delivery：`SalesOrderQuotationLink` 與 `DeliveryNote` 轉單需要同步 `status/event`。
- 參考 `flexora-react/src/main/java/com/asynctide/flexora/web/rest/SalesOrderResource.java` 了解 API 端點，並同步 `docs/specs/sales-order`（待新增）供正式前端使用。
