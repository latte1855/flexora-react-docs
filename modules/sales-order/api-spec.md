# Sales Order Module – API Spec (骨架)

| 資源 | Controller / Path | 備註 |
| --- | --- | --- |
| Sales Order | `SalesOrderResource` (`/api/sales-orders`) | 標準 CRUD + Criteria |
| SO Item | `SalesOrderItemResource` (`/api/sales-order-items`) | 行項資料 |
| Workflow | `SalesOrderStateTransitionResource`, `SalesOrderStatusHistoryResource`, `SalesOrderEventDefResource` | 用於流程/歷史 |
| SO ↔ Quotation Link | `SalesOrderQuotationLinkResource` | 維護 SO 與 Quotation 之間的連結 |

## TODO

- [ ] 梳理 `SalesOrderResource` 中的特殊端點（如 `openWorkflow`, `cancel`, `preview` 等）。
- [ ] 若要提供整單 Transaction API（SO + Items + Ext Attr），在此定義 payload。
- [ ] Async Select：供其它模組選擇 Sales Order。
- [ ] 匯入 / 批次更新 API（如更新交期）。
