# Pricing Module

集中管理 PriceRule、PriceList Assignment、Pricing Preview/Trace 等功能。  
目前多數規格散落於 `flexora-product-management-spec.md`、`PricingService` 相關文件，此處先建立骨架。

| 文件 | 說明 |
| --- | --- |
| [spec.md](./spec.md) | 定價規則、適用條件、計價流程 |
| [ui-spec.md](./ui-spec.md) | 價目表維護、PriceRule 編輯、Trace Viewer |
| [api-spec.md](./api-spec.md) | PricingService、PriceRule、Trace API |
| [implementation-plan.md](./implementation-plan.md) | 任務拆解 |

參考：

- `docs/flexora-product-management-spec.md` 中的定價章節
- 後端 `PriceRuleResource`, `PricingResource`, `PriceCalcTraceResource`
