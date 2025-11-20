# Flexora ERP — 錯誤處理規格（Error Handling Specification）
Phase 0 — System Foundation

本文件定義 Flexora ERP 在後端/前端之間 **錯誤回傳格式與處理方式的統一規範**，  
以確保：

- API 回傳錯誤 **穩定且可預期**  
- 前端可依錯誤碼進行 i18n 顯示  
- Codex / Agent 在自動產生程式與文件時有明確依據

---

## 1. 設計目標

1. **統一錯誤格式**：避免每個 Controller/Service 以不同方式回傳錯誤。
2. **易於前端處理**：前端不需要解析複雜結構，只要根據 `errorKey` 顯示對應訊息。
3. **可追蹤 / 可記錄**：重要錯誤應可在 Log / APM 中追蹤。
4. **與 JHipster 預設錯誤模型兼容**：沿用 JHipster 已有的 ErrorVM, Problem API 模型。

---

## 2. 全域錯誤模型（ErrorVM / Problem）

Flexora ERP 會沿用 JHipster 的錯誤模型（以 Spring MVC + Problem API 為主），  
常見的錯誤回應結構如下（簡化版）：

```json
{
  "type": "https://example.com/problem/constraint-violation",
  "title": "Bad Request",
  "status": 400,
  "path": "/api/quotations",
  "message": "error.validation",
  "fieldErrors": [
    {
      "objectName": "quotationDTO",
      "field": "validUntil",
      "message": "quotation.invalid_valid_until"
    }
  ]
}
```

或使用 ErrorVM（較舊型式）：

```json
{
  "message": "error.badRequest",
  "errorKey": "quotation.invalid_valid_until",
  "params": {
    "entity": "quotation",
    "field": "validUntil"
  }
}
```

> ⚠ 實際使用哪一種形式，可依 JHipster 版本與專案設定而定。
> Flexora ERP 的目標是：**錯誤碼（errorKey）一致、可預測**。

---

## 3. 錯誤類型分類（Error Categories）

Flexora ERP 主要錯誤類型可分為：

1. **驗證錯誤（Validation Errors）**

   * 欄位未填、格式錯誤、長度超過、數值範圍不合法等。
2. **業務邏輯錯誤（Business Rules Errors）**

   * 狀態不允許此操作、數量不足、重複資料、不允許刪除等。
3. **權限與認證錯誤（Authorization/Authentication Errors）**

   * JWT 失效、無權限存取某資源等。
4. **系統錯誤（System Errors）**

   * NullPointerException, DB 連線問題、非預期 Exception 等。

---

## 4. 錯誤碼命名規則（Error Key Naming）

所有錯誤碼應為 **可讀、具語意、可 i18n** 的 key，
建議使用：

```text
<領域>.<錯誤描述>
```

範例：

| 類型 | 錯誤碼（errorKey）                   | 說明             |
| -- | ------------------------------- | -------------- |
| 全域 | `error.validation`              | 一般驗證錯誤         |
| 全域 | `error.badRequest`              | 通用 Bad Request |
| 報價 | `quotation.invalid_state`       | 報價狀態不允許目前操作    |
| 報價 | `quotation.no_lines`            | 報價沒有明細         |
| 報價 | `quotation.invalid_valid_until` | 有效期限未填或已過期     |
| 編號 | `error.numbering.failed`        | 編號產生失敗         |
| 庫存 | `inventory.insufficient_stock`  | 庫存不足           |
| 採購 | `purchase.cannot_cancel`        | 不可作廢的訂單        |

**命名原則：**

* 全域錯誤以 `error.` 開頭：

  * `error.validation`
  * `error.badRequest`
  * `error.unauthorized`
* 模組專屬錯誤則以模組名稱開頭：

  * `quotation.*`
  * `salesOrder.*`
  * `inventory.*`
  * `numbering.*` 或 `error.numbering.*`（依用途）

這些錯誤碼應被定義於：

* 後端：丟 Exception 時使用
* 前端：i18n 訊息檔中對應顯示文字
* 文件：本 spec 或各模組 spec 中的「錯誤碼表」

---

## 5. BadRequestAlertException 使用規範

JHipster 通常提供 `BadRequestAlertException`、`ErrorConstants` 等類別。
Flexora ERP 建議統一使用 `BadRequestAlertException` 來處理「已知可預期」錯誤。

### 5.1 範例：報價 Guard 錯誤

```java
if (thread.getLines().isEmpty()) {
    throw new BadRequestAlertException(
        "quotation.no_lines",          // errorKey
        "quotationThread",             // entityName
        "noLines"                      // default message / more specific key
    );
}
```

對應的錯誤 JSON（示意）：

```json
{
  "message": "error.quotation",
  "errorKey": "quotation.no_lines",
  "params": {
    "entity": "quotationThread"
  }
}
```

> 實際結構會依後端 ErrorVM/Problem 實作而略有差異，但 `errorKey` 應明確存在。

---

## 6. 驗證錯誤（Validation Errors）

### 6.1 Bean Validation（後端）

後端實作中，應使用：

* `@NotNull`
* `@NotBlank`
* `@Size`
* `@Min` / `@Max`
* `@Pattern`
* …

當驗證失敗時，框架會產生 `ConstraintViolationException` 或類似錯誤，
JHipster 通常會把這些轉換為標準的 Problem JSON，
內容包含 `fieldErrors` 陣列。

**範例：**

```json
{
  "message": "error.validation",
  "fieldErrors": [
    {
      "objectName": "quotationDTO",
      "field": "validUntil",
      "message": "NotNull"
    }
  ]
}
```

> 前端可以依 `field` 與 `message` 做對應顯示。
> 如果需要更精準錯誤碼，可以在錯誤轉換時把 `message` 映射為 `quotation.invalid_valid_until`。

### 6.2 前端驗證（不取代後端）

* 前端可以做基本防呆（必填、格式、範圍、日期先後…），提高 UX。
* **但後端 Bean Validation 仍是最終防線**，不可移除。

---

## 7. 業務邏輯錯誤（Business Errors）

這類錯誤通常不是 Bean Validation 能處理的，而是：

* 狀態不合法（例如：已作廢的單據不能修改）
* 單據缺漏必要資料（報價無明細、訂單無客戶…）
* 邏輯衝突（例如：庫存不足）

**實作方式：**

* 使用 `BadRequestAlertException`（帶明確 errorKey）
* 或自訂 Exception → 統一在 Exception Handler 中轉為 Problem / ErrorVM

**範例：**

```java
if (quotation.getStatus() != QuotationStatus.DRAFT) {
    throw new BadRequestAlertException(
        "quotation.invalid_state",
        "quotation",
        "invalidState"
    );
}
```

---

## 8. 權限與認證錯誤（Security Errors）

常見情況：

* 未附 JWT 或 JWT 無效 → HTTP 401（Unauthorized）
* 有 JWT 但角色不足 → HTTP 403（Forbidden）

**規範：**

* HTTP Status：

  * 401 → 未認證
  * 403 → 已認證但無權限
* 錯誤碼建議：

  * `error.unauthorized`
  * `error.accessDenied`

**前端行為：**

* 401 → 導向登入頁或顯示「登入已過期」
* 403 → 顯示「沒有權限執行此操作」

---

## 9. 系統錯誤（System Errors）

指未預期錯誤，例如：

* NullPointerException
* 資料庫連線失敗
* 未捕捉的 RuntimeException

**規範：**

* HTTP Status：500（Internal Server Error）
* 回傳內容：

  * message 一律用 `error.internalServerError` 或類似
  * 不暴露 stack trace 或 DB 詳細錯誤給前端
* Log：

  * 記錄詳細 stack trace 於伺服器 log
  * 可搭配 APM / Logging 平台（例如 ELK, Loki）進行追蹤

---

## 10. 前端錯誤處理行為（General Frontend Behavior）

前端（`flexora-react-ui`）在收到錯誤時應遵守以下原則：

1. **優先讀取 errorKey 或 message**

   * 若有 `errorKey` → 使用 i18n 顯示對應語系訊息
   * 若無 `errorKey` 但有一般 message → 顯示通用錯誤訊息

2. **針對 Validation Errors**

   * 若有 `fieldErrors` → 將錯誤對回對應欄位（顯示紅字/提示）

3. **針對 Business Errors**

   * 如 `quotation.invalid_state`、`inventory.insufficient_stock` 等
     → 顯示 toast / dialog，文字由 i18n 決定

4. **針對 Security Errors（401/403）**

   * 401 → 重導登入或重新整理 Token
   * 403 → 顯示「無權限」訊息，不重導

5. **針對 System Errors（500）**

   * 僅顯示友善錯誤訊息（例如「系統發生錯誤，請稍後再試」）
   * 不顯示技術細節、SQL 或 Stack trace

---

## 11. 文件與程式碼同步要求

* 每新增一個 **模組特定錯誤碼**（例如 `quotation.xxx`）：

  * 必須在該模組 spec 中補上錯誤碼表
  * 前端 i18n 檔需增加對應 key
* 若調整 JHipster ErrorVM / Problem 實作方式：

  * 必須更新本文件 `error-handling-spec.md`
  * 並更新 `docs/architecture/OVERVIEW.md`（若屬架構級變更）

---

## 12. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                       |
| ---- | ---------- | ----- | ------------------------ |
| v0.1 | 2025-11-19 | Jimmy | 建立 Flexora ERP 錯誤處理規格初版。 |
