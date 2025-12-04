# Purchase Module – API Spec (骨架)

| 資源 | Controller / Path | 備註 |
| --- | --- | --- |
| Rfq | `RfqResource` (`/api/rfqs`) | 已存在標準 CRUD、Criteria 查詢 |
| Rfq Vendor Quote | `RfqVendorQuoteResource` | - |
| Purchase Order | `PurchaseOrderResource` (`/api/purchase-orders`) | 具備 Workflow / State Transition 資源 |
| Purchase Receipt | `PurchaseReceiptResource` (`/api/purchase-receipts`) | - |
| Supplier Return | `SupplierReturnResource` | - |

## TODO / 待補

- [ ] 把各資源的主要 API（列表、審批、附檔）整理為表格。
- [ ] 列出 Workflow API (`PurchaseOrderStateTransitionResource` 等) 的可用事件。
- [ ] 補充 Lookup / Async Select（供報價或其他模組選擇供應商、PO）。
- [ ] 若有 CSV 匯入（如 PO 明細），標註 API 與 payload 範例。
