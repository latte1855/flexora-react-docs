# 採購模組 Legacy 規格概要

本檔引用 `docs/migration/flexora-react/flexora-purchase-management-spec.md`，紀錄 PO / GRN / SRN / RFQ 四大流程的設計原則。實作以 `flexora-react/src/main/java/com/asynctide/flexora/domain/purchase`（如 `PurchaseOrder`, `PurchaseReceipt` 等 Entity）與 `service`、`web.rest` 的 API 版本為準。

## 1. 流程核心

| 單據 | 目的 | 成本/庫存影響 |
| --- | --- | --- |
| `PurchaseOrder` | 凍結商務條件（價量、付款條件），不直接動庫存 | - |
| `PurchaseReceipt`（GRN） | 收貨/檢驗 → 產生 `InventoryTransaction` (RECEIPT) → 更新成本 | 會動成本與庫存 |
| `SupplierReturn`（SRN） | 對應退供/不良 → 產生出庫分錄 | 觸發 `InventoryTransaction` ISSUE |
| `Rfq` | 多供應商詢價 → 計算 `SupplierQuote` → 選擇最佳報價後轉 PO | 只影響商務流程 |

## 2. 資料規則

- 所有表皆含 `created_by/created_at`, `last_modified_by/last_modified_at`, `deleted`, `deleted_at`, `deleted_by`, `version`、`owner_id`，且唯一鍵採 `partial unique`（`WHERE deleted=false`）。
- 稅/折扣/金額遵循 `phase0` 規範（`DECIMAL(19,4)`、`decimal(7,6)`）；時間一致使用 UTC。
- PO/GRN/ SRN/ RFQ 均採 DB 驅動的狀態與事件表（`*_status_def`, `*_event_def`, `*_state_transition`, `*_status_history`）。
- PaymentTerms 與 TaxCode 為共用主檔，POS/SO/Quotation 皆可引用 `payment_terms`。

## 3. 具體建議

- 表格規範採 `欄位代碼｜型態｜預設｜欄位名稱｜必填｜說明｜注意事項` 格式，離線進行實作時可依序填補。
- RFQ 以 `RfqVendorQuote`、`RfqVendorQuoteItem` 存供應商回覆，並提供 `GET /api/rfqs/{id}/vendor-quotes` 等 API；實作可參考 `RfqResource`。
- 稅務採行銷售+採購通用的 `tax_code`/`tax_rate_line`，PO line 同時存 `tax_type` 與 `tax_rate` 快照，避免後續稅率變動影響歷史。

## 4. 下一步

- 若需詳細欄位表或 Merimd ER，可直接閱讀原始 legacy 文檔並將必要欄位逐步拆入 `docs/specs/purchase-order/` 中的具體 files。
- 同步確認 `docs/specs/phase3-inventory/costing-rules.md` 及 `docs/architecture/DATA_MODEL.md`，確保 Inventory / AP 交集欄位一致。
