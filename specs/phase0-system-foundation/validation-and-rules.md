# Flexora ERP — 驗證與規則規格（Validation & Business Rules）
Phase 0 — System Foundation

本文件定義 Flexora ERP 的 **後端驗證（Bean Validation）與業務驗證（Business Rules / Guard）** 的基本原則、實作模式與命名慣例。

> 目標：  
> - 清楚區分「欄位層級」與「業務邏輯層級」驗證  
> - 確保前後端與文件對於「什麼情況會被拒絕」有共同理解  
> - 為後續模組（Product / Inventory / Quotation / Sales…）打下通用驗證框架

---

## 1. 驗證的分類

Flexora ERP 的驗證邏輯分為三層：

1. **欄位與 DTO 驗證（Field & DTO Validation）**
   - 使用 Java Bean Validation（JSR 380，如 `@NotNull`、`@Size`）
   - 主要處理「資料格式正確性」
2. **業務規則驗證（Business Rules / Guard）**
   - 由 Service 層負責
   - 包含狀態轉換合法性、關聯資料是否存在、數值邏輯是否合理等
3. **前端 UI 驗證（Client-side Validation）**
   - 提升 UX（例如必填欄位即時提示）
   - 不取代後端驗證

---

## 2. 欄位與 DTO 驗證（Bean Validation）

### 2.1 使用範圍

- 用於 REST API 的 Request DTO（如 `QuotationDTO`, `ItemDTO`）
- 不建議直接在 JPA Entity 上堆疊過多驗證（以 DTO 為主）

### 2.2 常用註解與建議用途

| 註解 | 用途 |
|------|------|
| `@NotNull` | 欄位不得為 null（後端必要欄位） |
| `@NotBlank` | 字串不得為 null / 空白（trim 後不得為空） |
| `@Size(min, max)` | 字串長度限制（例如代碼、名稱） |
| `@Min` / `@Max` | 整數範圍限制（數量、分數等） |
| `@Positive` / `@PositiveOrZero` | 價格/數量等需為正數 |
| `@Pattern` | 需要符合特定 regex 的欄位（例如手機、統編） |
| `@Email` | 電子郵件格式檢查 |

**原則：**

- 能用 Bean Validation 處理的檢查 → 優先用 Bean Validation
- 避免在 Service 自己再檢查「是否為 null」這種低階錯誤

### 2.3 DTO 與 Entity 欄位對應

- DTO：負責 API 入口驗證
- Entity：負責持久化與 Domain 行為

當兩者規則不完全一致時：

- 以 DTO 驗證為主（例如有些欄位在某些 API 可選填）
- 在 spec 中註明「此欄位在建立時必填，在修改時可選」

---

## 3. 業務規則驗證（Business Rules / Guard）

Bean Validation 無法處理的「跨欄位、多實體、流程相關」邏輯，  
統一在 **Service 層** 以「Guard 方法」實作。

### 3.1 Guard 的設計原則

1. **每個重要動作都應有對應 Guard**
   - 例如：submitQuotation, approveQuotation, cancelOrder 等

2. **Guard 與核心業務方法分離**
   - 以 `validateXxxGuard()` 形式存在，提高可讀性與可測試性

3. **Guard 失敗時一律丟出明確錯誤碼**
   - 使用 `BadRequestAlertException` 或專用 Exception
   - 搭配 `errorKey`（如 `quotation.invalid_state`）

### 3.2 範例：報價送出 Guard

```java
@Transactional
public QuotationThread submit(Long id) {
    QuotationThread thread = findByIdOrThrow(id);
    validateSubmitGuard(thread);
    thread.setStatus(QuotationStatus.SUBMITTED);
    // ... 其他後續處理
    return thread;
}

private void validateSubmitGuard(QuotationThread thread) {
    if (thread.getStatus() != QuotationStatus.DRAFT) {
        throw new BadRequestAlertException(
            "quotation.invalid_state",
            "quotationThread",
            "invalidState"
        );
    }

    if (thread.getLines().isEmpty()) {
        throw new BadRequestAlertException(
            "quotation.no_lines",
            "quotationThread",
            "noLines"
        );
    }

    if (thread.getValidUntil() == null || thread.getValidUntil().isBefore(LocalDate.now())) {
        throw new BadRequestAlertException(
            "quotation.invalid_valid_until",
            "quotationThread",
            "invalidValidUntil"
        );
    }
}
```

> 上述 Guard 屬「業務邏輯層級」，不可只依賴前端檢查。

---

## 4. 驗證 vs 錯誤處理的分工

* **Validation（驗證）**：檢查輸入資料是否「合理」
* **Error Handling（錯誤處理）**：將錯誤轉成統一的回應格式給前端

**關係：**

* Bean Validation 失敗 → 由全域 Exception Handler 統一轉成 `error.validation`
* Guard 驗證失敗 → 手動丟 `BadRequestAlertException` + 對應 `errorKey`

詳見：`error-handling-spec.md`

---

## 5. 前端驗證（Client-side Validation）

### 5.1 目的

* 提升使用者體驗（及時提示）
* 避免不必要的網路請求
* 不取代後端保護邏輯

### 5.2 原則

* 與後端規則「盡量一致」

  * 必填欄位、格式、數值範圍，在前端也應有相同限制
* 若前端規則比後端「寬鬆」：

  * 可能導致後端拒絕，使用者 UX 變差
* 若前端規則比後端「嚴格」：

  * 要確定是刻意的（例如 UI 設計要求）

### 5.3 碰撞處理

若前端疏忽導致規則未同步：

* 以後端結果為準
* 前端應能解析後端錯誤（尤其是 `fieldErrors` 與 `errorKey`）
* 後續調整前端規則使之與後端一致

---

## 6. 常見驗證場景與建議實作

### 6.1 必填欄位（Required Fields）

* 後端：

  * DTO 使用 `@NotNull` 或 `@NotBlank`
* 前端：

  * 表單加上「*」標示與即時提示
* 文件：

  * 在 spec 的欄位表中標明「必填」

### 6.2 數值與金額（Quantity / Amount）

* 後端：

  * `@Positive` 或 `@PositiveOrZero`（視需求）
* 前端：

  * 數值輸入框（避免自由文字）
* Guard：

  * 像庫存不足、超過信用額度等屬業務規則，由 Service 層檢查

### 6.3 日期範圍（ValidFrom / ValidUntil, Date Range）

* 後端：

  * Bean Validation 可檢查非空，但「前後順序」應由 Guard 檢查：

    * `validFrom <= validUntil`
* 前端：

  * 日期選擇元件（DatePicker）搭配簡單前後順序檢查
* 錯誤碼：

  * `error.date.range.invalid` 或模組專用，如 `quotation.invalid_valid_until`

### 6.4 唯一性（Unique Constraint）

* DB 層：

  * Unique Index（可搭配 partial index，例如「未刪除 + code」）
* 後端：

  * 建議實作 `assertUniqueOrThrows(...)` 類輔助方法
* 錯誤碼：

  * 如 `customer.code.duplicate`, `item.sku.duplicate`

---

## 7. 驗證規則文件化要求

每個模組在其 spec 下應：

1. **列出欄位與對應驗證規則**

   * 必填 / 選填
   * 型別（文字、數字、日期…）
   * 長度限制
   * 範圍限制
2. **列出主要 Guard 規則**

   * 每個重大動作（submit, approve, cancel…）需要的條件
3. **列出常見錯誤碼**

   * 對應 `error-handling-spec.md` 中的錯誤碼規範

範例（以 Quotation 為例，放在 Phase 4 spec 中）：

* 欄位：

  * `customerId`：必填，須存在於 Customer
  * `validUntil`：必填，且 `>= today`
* Guard：

  * 送出前須有至少一行明細
  * 草稿以外狀態不得再次送出
* 錯誤碼：

  * `quotation.no_lines`
  * `quotation.invalid_valid_until`

---

## 8. 驗證與單元測試

為避免驗證規則悄悄被改壞，建議：

1. **對 Guard 方法寫單元測試**

   * 測試「合法情境」
   * 測試「非法情境（應拋出錯誤）」
2. **對 Bean Validation 寫簡易測試**

   * 確認 DTO 欄位 Missing 時會觸發 ConstraintViolation
3. 若使用 MockMvc 或 WebTestClient：

   * 可針對 REST API 測試 400 Bad Request 行為與錯誤結構

---

## 9. 與其他文件的關聯

* 錯誤格式與錯誤碼命名 → `error-handling-spec.md`
* 資料模型與欄位定義 → `../../architecture/DATA_MODEL.md`
* Workflow / 狀態機 → `../../architecture/WORKFLOW.md`
* 各模組詳細規格 → `/docs/specs/<module>/...`

---

## 10. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                    |
| ---- | ---------- | ----- | --------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 0 驗證與規則規格初版。 |
