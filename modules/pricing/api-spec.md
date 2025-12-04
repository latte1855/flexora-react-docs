# Pricing Module – API Spec (骨架)

| 資源 | Controller / Path | 備註 |
| --- | --- | --- |
| PriceRule | `PriceRuleResource` (`/api/price-rules`) | CRUD, 優先順序 |
| Pricing Trace | `PriceCalcTraceResource` (`/api/pricing/trace`) | 查看試算結果 |
| Pricing Service | `PricingResource` (`/api/pricing`) | Preview / Calc |
| PriceList Assignment | `PriceListAssignmentResource`, `PriceListAssignmentSyncResource` | 管理客戶與價目表關係 |

## TODO

- [ ] 梳理 `PricingResource` 的 request/response（線上試算、批次計算）。
- [ ] 定義 Trace API 的輸出欄位，方便 UI 呈現。
- [ ] 若 PriceRule 要支援 Workflow / 審批，需補 API。
- [ ] 與 Product Module 的 Transaction API 整合（回寫價格時的流程）。
