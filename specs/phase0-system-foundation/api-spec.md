# Flexora ERP — API 規格（API Specification）
Phase 0 — System Foundation

本文件定義 Flexora ERP 在 Phase 0 的 **REST API 共用規則**，  
包含路徑命名、HTTP 方法、分頁 / 排序、過濾、日期格式、錯誤回傳與一些核心端點示意。

> 之後各模組（Product / Inventory / Quotation / Sales…）的 API 規格，  
> 若無特別說明，均應遵守本文件規則。

---

## 1. 基本風格（REST Style & Base Path）

### 1.1 Base Path

- 所有後端 REST API 一律以：

```text
/ api/**
```

為前綴。

例：

* `GET /api/users`
* `POST /api/owners`
* `GET /api/quotations/{id}`（Phase 4）

> Phase 0 主要聚焦在使用者 / Owner 等基礎 API，
> 其他模組於各自 Phase spec 中補充。

### 1.2 HTTP 方法語意

| HTTP 方法  | 用途                                            |
| -------- | --------------------------------------------- |
| `GET`    | 讀取資源（列表或單一）                                   |
| `POST`   | 建立新資源                                         |
| `PUT`    | 完整更新現有資源                                      |
| `PATCH`  | 部分更新（若採用，建議使用 `application/merge-patch+json`） |
| `DELETE` | 刪除資源（多為軟刪除）                                   |

---

## 2. 資源路徑命名規則

### 2.1 一般命名

* 一律使用複數名詞（lower-kebab-case）：

  * `/api/users`
  * `/api/owners`
  * `/api/teams`
  * `/api/quotations`（Phase 4）
* ID 採 path parameter：

  * `/api/users/{id}`
  * `/api/owners/{id}`

### 2.2 關聯資源（後續模組用）

通用模式：

```text
/api/<parentResource>/{parentId}/<childResource>
```

例（示意）：

* `/api/quotation-threads/{id}/revisions`
* `/api/sales-orders/{id}/lines`

Phase 0 本身關聯較少，可先以單一資源為主。

---

## 3. Request / Response 格式

### 3.1 Content-Type

* 所有 API Request／Response 一律採用：

```text
Content-Type: application/json
Accept: application/json
```

> 若未來實作檔案下載 / 上傳，會在對應模組 spec 中另行說明。

### 3.2 DTO 結構

* API 使用 DTO（Data Transfer Object），**不直接暴露 Entity**。
* DTO 屬性命名使用 camelCase：

  * `id`, `ownerId`, `createdDate`, `lastModifiedBy`

### 3.3 ID vs No

* `id`：後端主鍵（Long / UUID），僅供系統使用
* `xxxNo`：業務可讀編號（例如 `quotationNo`），由 Numbering Service 產生

> 相關規則詳見：`numbering-spec.md`

---

## 4. 分頁、排序與過濾（Paging / Sorting / Filtering）

### 4.1 分頁參數（JHipster 預設）

列表 API 應支援標準分頁參數：

| 參數     | 說明           | 範例                      |
| ------ | ------------ | ----------------------- |
| `page` | 第幾頁（0-based） | `page=0`                |
| `size` | 每頁筆數         | `size=20`               |
| `sort` | 排序欄位與方向      | `sort=createdDate,desc` |

**範例：**

```http
GET /api/users?page=0&size=20&sort=login,asc
```

### 4.2 分頁回應 Header

列表 API 應回傳：

* `X-Total-Count`：符合條件的總筆數
* `Link`：分頁導覽（first / prev / next / last）

---

## 5. 過濾（Filtering）— Phase 0 基礎規則

Phase 0 先定義基本過濾原則，細部規則可由各模組擴充。

### 5.1 Query String Filtering

常見過濾方式：

```http
GET /api/users?activated=true
GET /api/owners?active=true&code.contains=TW
```

建議遵循 JHipster 的 Criteria 動態過濾模式（如有啟用）：

* `<field>.equals=xxx`
* `<field>.contains=xxx`
* `<field>.in=a,b,c`
* `<field>.specified=true/false`
* `<field>.greaterThan=...` / `<field>.lessThan=...`

**建議：**

* Phase 0 中之 users / owners 若實作 criteria 查詢，請在本文件或對應模組 spec 補充常用過濾欄位。

---

## 6. 日期與時間格式

### 6.1 後端型別

* Java 後端：

  * `LocalDate`：無時間的日期（例如生日、生效日）
  * `Instant` 或 `ZonedDateTime`：含時間與時區（例如審計欄位）

### 6.2 JSON 序列化格式

**日期（LocalDate）：**

```json
"2025-11-19"
```

**時間戳（Instant / ZonedDateTime）：**

採用 ISO-8601：

```json
"2025-11-19T08:30:00Z"
```

或（含偏移）：

```json
"2025-11-19T16:30:00+08:00"
```

> 前端應採用相同格式解析，不應依賴瀏覽器 locale default。

---

## 7. Enum / 狀態欄位

### 7.1 Enum 傳輸格式

Enum 欄位在 JSON 中以 **字串** 形式傳輸：

```json
{
  "status": "DRAFT"
}
```

> 詳細 Enum 列表與說明，應於各模組 spec（如 Quotation、Sales）中列出。

### 7.2 Enum 的 i18n

* 後端使用 Enum 名稱（全大寫英文字）與程式內文控制轉換。
* 前端／文件需建立 i18n 對應表，例如：

  * `enum.quotationStatus.DRAFT = 草稿`
  * `enum.quotationStatus.SUBMITTED = 已送出`

---

## 8. 錯誤回傳格式（Errors）

錯誤格式完整規範請參考：

* `error-handling-spec.md`

本節僅摘要 Phase 0 API 規定的重點：

### 8.1 HTTP Status 使用

| 情境             | HTTP Status               |
| -------------- | ------------------------- |
| 驗證錯誤（欄位不合法）    | 400 Bad Request           |
| 業務規則錯誤         | 400 Bad Request           |
| 找不到資源          | 404 Not Found             |
| 未登入 / Token 無效 | 401 Unauthorized          |
| 無權限            | 403 Forbidden             |
| 系統錯誤           | 500 Internal Server Error |

### 8.2 錯誤回應內容

錯誤 JSON 需包含：

* `message`：全域錯誤訊息代碼（例如 `error.validation`）
* `errorKey`（若適用）：模組級錯誤代碼（例如 `quotation.invalid_state`）
* `fieldErrors`（若為欄位驗證錯誤）

示意：

```json
{
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

或（使用 ErrorVM）：

```json
{
  "message": "error.badRequest",
  "errorKey": "quotation.no_lines",
  "params": {
    "entity": "quotationThread"
  }
}
```

---

## 9. 建立 / 更新 / 刪除語意（Create / Update / Delete）

### 9.1 建立（Create）

* 方法：`POST /api/<resource>`
* Request DTO **不得包含 id**（或需為 null）
* 成功回傳：

  * HTTP 201 Created
  * Body：建立後的 DTO（含 id, no 等）
  * `Location` header：指向新建立資源，如 `/api/owners/123`

### 9.2 更新（Update）

* 方法：`PUT /api/<resource>/{id}`
* Request DTO 必須包含 id，且需與 path 參數 id 一致：

```http
PUT /api/owners/5
```

```json
{
  "id": 5,
  "code": "HQ",
  "name": "Headquarters"
}
```

* 成功回傳：

  * HTTP 200 OK
  * Body：更新後 DTO

### 9.3 部分更新（Partial Update）— 若採用

* 方法：`PATCH /api/<resource>/{id}`
* Content-Type：

```text
application/merge-patch+json
```

* Body 只需要包含要更新的欄位。

### 9.4 刪除（Delete）

* 方法：`DELETE /api/<resource>/{id}`
* 多數採「軟刪除」策略：

  * 將 `deleted = true`，不直接刪除資料列
* 成功回傳：

  * HTTP 204 No Content（不含 body）

---

## 10. 安全與 Token（與 Security 規格關聯）

> 詳細安全規格請參考 `security-and-auth.md`，
> 這裡僅列出與 API 呼叫相關的要點。

### 10.1 認證 Header

所有受保護 API，必須在 Header 中附上：

```http
Authorization: Bearer <JWT_TOKEN>
```

未附或 Token 無效：

* 回傳 HTTP 401（`error.unauthorized`）

### 10.2 CSRF

* 由於採 Token-based（JWT），一般不啟用瀏覽器 Cookie-based CSRF。
* 若未來採 HttpOnly Cookie 攜帶 Token，需重新評估 CSRF 策略。

---

## 11. Phase 0 典型 API 範例（示意）

> 以下為「概念示例」，實際路徑與 DTO 以程式碼為準，
> 後續可視情況在此文件補上更精確的欄位定義。

### 11.1 取得目前登入帳號資訊

* `GET /api/account`
* 回傳當前 User 的基本資訊（login, email, langKey, authorities 等）

### 11.2 User CRUD（管理端）

* `GET /api/users`（分頁、可依 `activated` 過濾）
* `GET /api/users/{login}`（或 `{id}`）
* `POST /api/users`
* `PUT /api/users`
* `DELETE /api/users/{login}`（或 `{id}`）

> 按 JHipster 預設產生的 UserResource 為基礎。

### 11.3 Owner CRUD（若已實作）

* `GET /api/owners`
* `GET /api/owners/{id}`
* `POST /api/owners`
* `PUT /api/owners/{id}`
* `DELETE /api/owners/{id}`（實務上多為軟刪除）

---

## 12. 文件與程式碼同步規則

每當修改以下項目時，應同步更新本文件或模組級 spec：

* API 路徑（Path）
* Request / Response DTO 結構
* 資源命名（Resource naming）
* 分頁與過濾參數格式
* 標準錯誤回傳格式

**優先權：**

1. 程式碼（實際 Controller / DTO）
2. 本文件與模組 spec
3. OpenAPI / Swagger（由程式自動產生）

---

## 13. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                   |
| ---- | ---------- | ----- | -------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 0 API 規格初版。 |

