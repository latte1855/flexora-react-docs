# Flexora ERP API 文件產出與維運規範（OpenAPI 3.1）

> 文件位置：`docs/api/OPENAPI.md`  
> 適用版本：JHipster v8.11.0 / Spring Boot 3.x / SpringDoc OpenAPI  
> 更新日期：2025-10-15（Asia/Taipei）

---

## 1️⃣ 目的

本文件說明 Flexora ERP API 文件的產出方式、版本管理策略、Header 與安全性規範。  
所有 API 定義皆符合 **OpenAPI 3.1.0**，可供：

- React 前端自動產生型別 (`openapi-typescript`)
- Postman / Insomnia 匯入測試
- Swagger UI / ReDoc 展示
- 契約測試（Contract Test）對齊

---

## 2️⃣ 工具與設定

### 2.1 SpringDoc 設定

在 `pom.xml` 中已納入：

```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.6.0</version>
</dependency>
```

### 2.2 常用端點

| 端點                     | 說明               |
| ------------------------ | ------------------ |
| `/v3/api-docs`           | JSON 格式完整定義  |
| `/v3/api-docs.yaml`      | YAML 格式輸出      |
| `/swagger-ui/index.html` | Swagger UI 預覽    |
| `/api-docs`              | ReDoc 文件（選用） |

---

## 3️⃣ 文件產出流程

1. 開發者在 Controller 或 Resource 類別中撰寫註解：

   ```java
   @Operation(summary = "取得報價單詳情", description = "依 ID 取得完整報價單資料")
   @ApiResponses({
       @ApiResponse(responseCode = "200", description = "成功"),
       @ApiResponse(responseCode = "404", description = "找不到報價單")
   })
   public ResponseEntity<QuotationDTO> getQuotation(@PathVariable Long id) { ... }
   ```

2. CI/CD 於建置階段執行：

   ```bash
   mvn clean verify -DskipTests
   curl -o docs/api/openapi.json http://localhost:8080/v3/api-docs
   ```

3. 將 `openapi.json` 轉為 TypeScript：

   ```bash
   npx openapi-typescript docs/api/openapi.json -o frontend/src/api/types.ts
   ```

4. 文件輸出版本對應 Git Tag，例如：

   ```
   docs/api/openapi-v2025.10.15.yaml
   ```

---

## 4️⃣ 命名慣例（Naming Conventions）

| 類型         | 規則                                                                                  |
| ------------ | ------------------------------------------------------------------------------------- |
| REST Path    | `/api/{entity-name}`，以小寫、複數命名（例：`/api/customers`）                        |
| 事件端點     | `/api/{module}/{id}/events/{eventCode}`（例：`/api/sales-orders/100/events/confirm`） |
| 查詢端點     | `/api/{module}/_search?...`                                                           |
| 內部服務端點 | `/internal/{service}/{action}`（需 JWT Admin 權限）                                   |
| Enum 命名    | 全大寫 + `_`（例：`APPROVED`, `CANCELLED`）                                           |
| DTO 命名     | `{Entity}DTO`                                                                         |
| VO 命名      | `{Function}VO`（例：`AddressVO`, `TaxSummaryVO`）                                     |

---

## 5️⃣ API Header 規範

| Header            | 說明                               | 範例                     |
| ----------------- | ---------------------------------- | ------------------------ |
| `Authorization`   | JWT 權杖（必填）                   | `Bearer eyJhbGciOiJI...` |
| `Accept-Language` | 語系設定                           | `zh-TW`, `en-US`         |
| `X-API-Version`   | API 版本控制                       | `1.0`                    |
| `Idempotency-Key` | 指令型端點避免重複執行（POST/PUT） | `UUID`                   |
| `If-Match`        | 樂觀鎖版本控制                     | `W/"4"`                  |
| `Content-Type`    | MIME 類型                          | `application/json`       |

---

## 6️⃣ 版本控制（Versioning）

| 類型     | 使用方式                                   | 範例         |
| -------- | ------------------------------------------ | ------------ |
| Header   | `X-API-Version`（推薦）                    | `1.0`, `1.1` |
| URL      | `/api/v1/quotations`（僅當重大破壞改動時） | -            |
| 內部代碼 | `@ApiVersion("1.0")` 標註於 Controller     | -            |

當前版本策略：

- **主版號**：重大架構改動
- **次版號**：API 欄位增加（向後相容）
- **修訂號**：Bug 修正或範例更新

---

## 7️⃣ Idempotency 與重試安全

### 7.1 使用場景

針對「非查詢類 API（POST/PUT/PATCH）」使用 `Idempotency-Key` header。

### 7.2 實作建議

- 建立資料表 `api_idempotency_record`
  欄位：`key`, `user_id`, `uri`, `payload_hash`, `response_code`, `response_body`, `expire_at`
- 若同一組 Key 再次呼叫，直接回傳第一次的結果。

---

## 8️⃣ 錯誤格式（Problem+JSON）

範例：

```json
{
  "type": "https://api.flexora.io/errors/validation",
  "title": "Validation Failed",
  "status": 422,
  "detail": "欄位驗證錯誤",
  "violations": [{ "field": "price", "message": "不得為負數" }],
  "timestamp": "2025-10-15T07:55:00Z",
  "traceId": "a1b2c3d4"
}
```

---

## 9️⃣ 安全性與權限控制

- 採用 **JWT Bearer Token**。
- 角色範例：`ROLE_ADMIN`, `ROLE_MANAGER`, `ROLE_USER`, `ROLE_VENDOR`。
- 權限繫結：

  - **資料層（Data Scope）**：依 `Owner`、`Department`、`Team`。
  - **動作層（Action Scope）**：依狀態機定義的 `allow_assigned_to` 欄位。

---

## 🔟 文件驗收標準

| 編號 | 驗收項目                                     | 說明 |
| ---- | -------------------------------------------- | ---- |
| 1    | 所有端點皆有 `@Operation` 與 `@ApiResponse`  |      |
| 2    | DTO / VO 皆具 `@Schema` 描述                 |      |
| 3    | Swagger UI 可完整渲染                        |      |
| 4    | `openapi.json` 產出無錯誤                    |      |
| 5    | 前端可使用 `openapi-typescript` 自動生成型別 |      |
| 6    | 每個指令端點支援 `Idempotency-Key`           |      |

---
