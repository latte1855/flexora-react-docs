# Pricing Module – Functional Spec

> 參考文件：`flexora-product-management-spec.md`、`PricingService` 相關文件。  
> 本模組聚焦 PriceRule、PriceList Assignment、Pricing Preview/Trace。

## 範圍

| 範疇 | 說明 |
| --- | --- |
| PriceRule | 定價規則（條件、優先順序、公式） |
| PriceList Assignment | 管理價目表與客戶/通路/Region 的關係 |
| Pricing Engine | `PricingResource` 提供試算/即時計價 |
| Trace / Debug | `PriceCalcTraceResource` 顯示命中規則與計算過程 |
| PriceList / ItemPrice | 價目表主檔、SKU 價格、有效期間與審批 |
| Integration | 與 Quotation / Sales Order / Product 的資料交換 |

## PriceRule 資料摘要

- 欄位：`ruleCode`, `name`, `priority`, `channel`, `customerGroup`, `itemGroup`, `effectivity`, `discountType`, `formula`。  
- 狀態：`DRAFT / ACTIVE / ARCHIVED`。  
- 規則比對流程：依 `priority` + 條件過濾 → 套用 `formula`（Ex: fixed price, percent discount, markup）。  
- 需支援審批/版本？（TODO）

## Pricing Flow（概念）

```
Input: skuId, quantity, customerId, priceListId, channel
 → 撈 PriceList / ItemPrice
 → 依 PriceRule 條件過濾
 → 套用 Formula (discount/markup/custom)
 → 回傳 unitPrice, discountDetail, trace info
```

## 整合

- 報價、報價預覽 (`/api/quotations/preview`) 透過 PricingService 試算 → 需確保 API 穩定。  
- Product 模組的 Transaction API 更新價格時，可自動觸發 PricingEngine 重新計算或寫入 Trace。  
- Sales Order / Contract 也可呼叫 Pricing Preview 取得建議價。

## TODO

- [ ] 完整列出 PriceRule 欄位與資料字典。  
- [ ] 補上流程圖（輸入 → 取 PriceList → 套 Rule → 輸出）。  
- [ ] 審批需求：若 PriceRule 調整需要審批，定義狀態/事件。  
- [ ] 記錄 Trace/Debug 所需欄位與存放時程。  
- [ ] 若 PricingEngine 需支援批量試算、排程同步，補充說明。
- [ ] PriceList 與 Product Transaction API 的同步策略（例：同時更新 SKU 與價格）。  
- [ ] 權限 / 角色矩陣：誰可以建立規則、調整優先順序、審批價目。  
