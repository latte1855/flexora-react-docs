# Phase 4 — 報價計價邏輯規格（Quotation Pricing Rules）
本文件定義 Flexora ERP **報價模組（Quotation）** 的計價流程、欄位邏輯、折扣與稅額計算方式。  
此規格為計算明細行（Line）、主檔（Revision）、整體折扣與稅務邏輯的唯一權威（Single Source of Truth）。

> 本文件與 Phase 1「價格（Pricing）」模組密切相關，  
> 一旦 PriceList / ItemPrice 的規格更新，需同步回寫本文件。

---

# 1. 計價流程（Pricing Flow Overview）

報價版本（`QuotationRevision`）的計價流程分成三層：

```

(1) 明細層：Line
(2) 主檔層：Revision
(3) 加總層：Subtotal / Discount / Tax / Total

```

每一層都有明確的公式，禁止前端與後端使用不一致的計算方式。

> ⚠ 計價結果之後端為最終權威。  
> 前端計算僅供使用者預覽，提交時仍以後端計算覆寫。

---

# 2. 明細層（Line Level Pricing）

每一行計算以下欄位：

- `unitPrice`（單價）
- `quantity`（數量）
- `discountRate`（行折扣％）
- `discountAmount`（行折扣金額）
- `netUnitPrice`（折後單價）
- `lineSubtotal`（行小計，不含稅）
- `taxRate`（稅率，如 0.05）
- `taxAmount`（稅額）
- `lineTotal`（含稅金額）

## 2.1 行折扣計算

### 2.1.1 優先順序

折扣計算採以下順序：

1. 若 `discountRate` > 0 → 使用折扣率計算  
2. 若 `discountRate` = 0 且 `discountAmount` > 0 → 使用折扣金額  
3. 兩者皆未填 → 無折扣  

### 2.1.2 公式

1. **折扣金額（由折扣率計算）**

```

discountAmount = unitPrice * discountRate

```

2. **折後單價**

```

netUnitPrice = unitPrice - discountAmount

```

3. **行小計（不含稅）**

```

lineSubtotal = netUnitPrice * quantity

```

---

# 3. 稅務計算（Tax Calculation）

若 `taxIncluded = false`（未稅報價）：

```

taxAmount = lineSubtotal * taxRate
lineTotal = lineSubtotal + taxAmount

```

若 `taxIncluded = true`（含稅報價）：

```

lineTotal = netUnitPrice * quantity
taxAmount = lineTotal * (taxRate / (1 + taxRate))
lineSubtotal = lineTotal - taxAmount

```

> ⚠ 保證「含稅」與「未稅」都能算出一致的含稅總額 / 稅額。

---

# 4. 主檔層（Revision Level Pricing）

主檔層維護以下欄位（可選擇存 DB 或 runtime 計算）：

- `subtotalAmount`（所有 lineSubtotal 加總）
- `discountRate`（整份報價的總體折扣％）
- `discountAmount`（整體折扣金額）
- `taxAmount`（所有行稅額加總）
- `totalAmount`（整份報價含稅金額）

## 4.1 加總行小計

```

subtotalAmount = Σ(lineSubtotal)

```

## 4.2 整體折扣（Overall Discount）

整體折扣與「行折扣」是**不同層次**的概念。

### 4.2.1 若使用整體折扣率：

```

discountAmount = subtotalAmount * discountRate

```

### 4.2.2 若前端提供折扣金額而非折扣率：

```

discountRate = discountAmount / subtotalAmount

```

> 必須二擇一，以避免出現矛盾數值。

## 4.3 小計（扣除整體折扣後）

```

discountedSubtotal = subtotalAmount - discountAmount

```

## 4.4 合併行稅額＋折扣計算邏輯

稅額基本來自各行合計：

```

taxAmount = Σ(lineTaxAmount)

```

但若整體折扣需要反映在稅內價格，需依稅法與公司政策決策：

| 公司規則 | 稅底應減少？ | 行為 |
|-----------|-------------|------|
| 整體折扣視為降價 | 是 | 稅額需重算 |
| 整體折扣視為總額折讓 | 否 | 稅額不動，只減少總額 |

### 建議規則（多數 ERP 採用）

**整體折扣要重新反映在含稅金額上，但不拆行重新計算稅額。**

所以 Revision 層最終 total：

```

totalAmount = discountedSubtotal + taxAmount

```

> 若公司政策要「折扣後重新算稅額」，必須寫入 DECISION_LOG.md。

---

# 5. 價格來源（Price Source）與 PriceList 整合

以下由 Phase 1 Pricing 模組決定，但本文件需同步規則。

## 5.1 單價來源優先順序（建議）

1. 若有綁定 `priceListId` → 使用 PriceList 尋價  
2. 若 PriceList 無對應 → 使用 `ItemSku.defaultPrice`  
3. 若前端手動覆寫 → 以手動輸入為主  
4. 若未提供 → 錯誤：`quotation.unit_price_missing`

## 5.2 PriceList 規則（摘要）

若使用 PriceList：

- 價格名稱：`ItemPrice`
- 必須含：
  - SKU
  - 幣別
  - 價格（含稅或未稅需寫入 MDM）
- 若找不到 → 錯誤碼：`quotation.pricelist_no_price_found`

> 完整規範請見 Phase 1 Pricing spec。

---

# 6. 新增明細行時的預設行為

- 新增 Line → 預設：
  - quantity = 1
  - unitPrice = 從 PriceList 或 SKU 帶入
  - discountRate = 0
  - taxType = PriceList 或 SKU 預設稅別
- 若 SKU 有多規格（EAN, 產線區分等） → 編輯畫面需顯示完整 name/spec

---

# 7. 前端計算 vs 後端計算（前端僅預估）

### 7.1 前端計算需求

- 在 UI 中能即時計算：
  - lineSubtotal  
  - taxAmount  
  - lineTotal  
  - subtotalAmount  
  - discountAmount（整體）  
  - totalAmount  

作為使用者參考。

### 7.2 後端最終計算規則

- 後端收到 API（PUT/POST/Submit Workflow）時：
  1. 重新跑所有明細計算 → 覆寫前端值
  2. 再跑 Revision-level 計價 → 覆寫前端總額
  3. 存 DB（若有存金額）
  4. 回傳完整 DTO（金額、折扣、稅額全部為後端計算）

### 7.3 不一致時的處理

若前端傳入的金額與後端計算不一致：

- 後端不視為錯誤（除非明顯不合理）  
- 後端覆寫 → 回傳最終計算值  
- 若差異過大，可回錯誤：  
  - `quotation.amount_mismatch`（需寫入 DECISION_LOG）

---

# 8. 數值精度（Precision）與四捨五入

### 8.1 建議精度

- 單價：小數第 4 位（BigDecimal(18, 4)）
- 數量：小數第 3 位（BigDecimal(18, 3)）
- 稅率：小數第 4 位
- 金額：小數第 0~2 位（依公司政策）

### 8.2 四捨五入規則

典型 ERP 採用：

```

小計與稅額 → 四捨五入到小數第 0 位（整數金額）
折扣與整體金額 → 四捨五入到小數第 0 位

```

若公司保留小數 → 必須更新本文件，並回寫 DECISION_LOG.md。

---

# 9. 折扣組合問題（行折扣 + 整體折扣）

ERP 中最常遇到的爭議是：

「到底先算行折扣？還是先算整體折扣？」

強制規則：

### **先算行折扣 → 再算整體折扣**

理由：

- 行折扣常由 SKU 本身或業務依品項談定
- 整體折扣通常為銷售總體讓利（如尾數折扣、總量折扣）
- 此方式更符合一般公司報價行為

---

# 10. 稅別（Tax Type）邏輯

報價 Line 可指定 `taxType`，數值邏輯如下：

| 稅別 | 稅率 | 行為 |
|------|------|------|
| 一般稅 | 5%（例） | 依「含稅 / 未稅」計算 |
| 零稅率 | 0% | 免稅，計算正常（乘 0） |
| 免稅 | 0% | 通常不顯示稅額欄位 |
| 自定 | 依 ItemSku 或 PriceList | 由 Phase 1 Tax 模組決定 |

未來若引入 B2C / 海外稅制，需更新本規格。

---

# 11. 計價錯誤碼（與 error-codes.md 同步）

以下是本文件使用到的錯誤碼摘要：

| errorKey | 說明 |
|----------|------|
| `quotation.amount_calculation_failed` | 計算過程發生錯誤 |
| `quotation.amount_mismatch` | 前後端計算差異過大 |
| `quotation.unit_price_missing` | 單價不可為空 |
| `quotation.pricelist_no_price_found` | 無法從 PriceList 找到單價 |
| `quotation.line_quantity_invalid` | 數量小於等於 0 |
| `quotation.line_unit_price_invalid` | 單價為負數 |

必要時請補到 `error-codes.md`。

---

# 12. 與其他模組的關聯

- **Phase 1 Pricing**：  
  PriceList / ItemPrice / Currency / TaxRate 等。
- **Phase 0 Validation & Error Handling**：  
  前端與後端的欄位與錯誤一致性。
- **Workflow**：  
  SUBMIT / APPROVE 時需跑完整計算流程。

---

# 13. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立報價計價規格（Pricing Rules）初稿。 |

## 14. 程式碼對照與錯誤碼

| 對應程式 | 說明 |
| --- | --- |
| `flexora-react/src/main/java/com/asynctide/flexora/service/pricing/QuotationPricingService.java` | `calculateLineAmounts`, `calculateTax`, `applyOverallDiscount`, `verifyAmounts` 用以計算 line/revision 金額；所有 `BigDecimal` 使用 `RoundingMode.HALF_UP`、`scale=4` 公差。 |
| `QuotationPricingValidator` | `verifyAmounts` 比對前端與後端總額，若差距過大拋出 `quotation.amount_mismatch`；同時檢查 `lineQty`、`unitPrice`。 |
| `QuotationRevisionResource`, `QuotationThreadResource` | Create/update 時都會透過 PricingService 重算金額，DTO response 值皆由後端計算。 |

### 14.1 錯誤碼對應

* `quotation.amount_calculation_failed`：`calculateLineAmounts` 遇到缺資料/稅率錯誤時拋出。  
* `quotation.amount_mismatch`：`QuotationPricingValidator.verifyAmounts` 拋出；API 會回 400。  
* `quotation.unit_price_missing` / `quotation.line_unit_price_invalid`：`QuotationLineDTO` 的 `@NotNull` / `@Positive` 驗證。  
* `quotation.pricelist_no_price_found`：`ItemPriceService` 找不到對應 price list entry。

> 若 Pricing 公式/精度更動，請以 `QuotationPricingService` 為最終依據，並在此節註記 `Updated per flexora-react/src/main/java/com/asynctide/flexora/service/pricing/QuotationPricingService.java`。
