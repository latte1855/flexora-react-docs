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

## ERD / 流程示意

```
PriceRule ──< RuleCondition
     │
     └─< RuleFormula

PriceList ──< ItemPrice >── ItemSku
     │
     └─< PriceListAssignment (客戶 / 通路 / Region)
```

## Pricing Flow（概念）

```
Input: skuId, qty, customerId, priceListId, channel
 → 取 PriceListAssignment 決定 PriceList
 → 讀取 ItemPrice（有效期、幣別）
 → 套 PriceRule (依 priority + condition)
 → 執行 Formula (fixed / percent / markup / script)
 → 回傳 unitPrice, discountDetail, trace info
 → 若啟用通知，推送 PriceTrace 供 UI 調試
```

## 整合

- 報價、報價預覽 (`/api/quotations/preview`) 透過 PricingService 試算 → 需確保 API 穩定。  
- Product 模組的 Transaction API 更新價格時，可自動觸發 PricingEngine 重新計算或寫入 Trace。  
- Sales Order / Contract 也可呼叫 Pricing Preview 取得建議價。

- [ ] 匯入/匯出 PriceRule 與 ItemPrice 的 CSV 欄位定義與範例。
- [ ] Notification：價格異動審批完成時是否通知報價/訂單。
