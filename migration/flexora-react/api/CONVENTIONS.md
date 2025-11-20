# Flexora ERP API 命名與介面慣例規範

> 文件位置：`docs/api/CONVENTIONS.md`  
> 適用版本：JHipster v8.11.0  
> 更新日期：2025-10-15（Asia/Taipei）

---

## 1️⃣ 一般命名規則

| 類別      | 命名規則           | 範例                            |
| --------- | ------------------ | ------------------------------- |
| Entity    | 駝峰命名，首字大寫 | `SalesOrder`, `QuotationItem`   |
| Table     | 全小寫 + `_` 分隔  | `sales_order`, `quotation_item` |
| REST Path | 小寫 + 複數        | `/api/sales-orders`             |
| DTO       | `{Entity}DTO`      | `SalesOrderDTO`                 |
| Enum      | 全大寫 + `_`       | `APPROVED`, `CANCELLED`         |
| JSON 欄位 | camelCase          | `createdBy`, `taxAmount`        |

---

## 2️⃣ 日期與時間格式

| 類型       | 格式         | 範例                     |
| ---------- | ------------ | ------------------------ |
| 時間戳     | ISO 8601 UTC | `"2025-10-15T07:55:00Z"` |
| 日期       | `YYYY-MM-DD` | `"2025-10-15"`           |
| 帶時區顯示 | `Z` 代表 UTC | `"2025-10-15T07:55:00Z"` |

所有時間欄位後端使用 `Instant`，資料庫型態為 `TIMESTAMP WITH TIME ZONE`。

---

## 3️⃣ 數值與精度

| 類型        | 精度            | 範例       |
| ----------- | --------------- | ---------- |
| 金額        | `DECIMAL(19,4)` | 12345.6789 |
| 稅率 / 折扣 | `DECIMAL(9,6)`  | 0.050000   |
| 數量        | `DECIMAL(19,6)` | 12.345678  |
| 顯示精度    | 2 位四捨五入    | 12.35      |

---

## 4️⃣ 分頁 / 排序 / 篩選

### 4.1 分頁

```

GET /api/customers?page=0&size=20

```

回應格式：

```json
{
  "content": [...],
  "totalElements": 124,
  "totalPages": 7,
  "pageable": { "pageNumber": 0, "pageSize": 20 }
}
```

### 4.2 排序

```
GET /api/items?sort=createdDate,desc&sort=skuCode,asc
```

### 4.3 篩選

```
GET /api/sales-orders?_search=customerName:like:Tech|status:eq:CONFIRMED
```

---

## 5️⃣ 樂觀鎖（Optimistic Lock）

所有主要業務表皆具 `version` 欄位。
更新時需附帶：

```http
PUT /api/sales-orders/100
If-Match: W/"3"
```

若版本不符，回傳：

```json
{
  "status": 409,
  "title": "Optimistic Lock Failed",
  "detail": "Version mismatch"
}
```

---

## 6️⃣ 錯誤碼對照表

| 狀態碼 | 類型                  | 說明                 |
| ------ | --------------------- | -------------------- |
| 200    | OK                    | 成功                 |
| 201    | Created               | 建立成功             |
| 204    | No Content            | 刪除成功             |
| 400    | Bad Request           | 參數錯誤或無法解析   |
| 401    | Unauthorized          | JWT 無效或逾期       |
| 403    | Forbidden             | 無權存取資源         |
| 404    | Not Found             | 找不到資源           |
| 409    | Conflict              | 樂觀鎖或業務衝突     |
| 422    | Unprocessable Entity  | 驗證失敗（表單錯誤） |
| 429    | Too Many Requests     | 超出速率限制         |
| 500    | Internal Server Error | 系統錯誤             |
| 503    | Service Unavailable   | 系統維護中           |

---

## 7️⃣ 統一回應格式（Standard Response）

```json
{
  "data": { ... },
  "meta": {
    "timestamp": "2025-10-15T07:55:00Z",
    "traceId": "123abc456",
    "status": "OK"
  }
}
```

查詢類 API 以 `data[]` 回傳陣列，指令類 API 以 `data` 單筆物件為主。

---

## 8️⃣ 狀態機事件回應格式

範例：

```http
POST /api/sales-orders/100/events/confirm
```

回應：

```json
{
  "event": "confirm",
  "fromStatus": "DRAFT",
  "toStatus": "CONFIRMED",
  "timestamp": "2025-10-15T07:55:00Z",
  "performedBy": "admin"
}
```

---

## 9️⃣ 國際化（i18n）

- 支援 `Accept-Language: zh-TW` 或 `en-US`
- 狀態、錯誤訊息、Enum 名稱皆可本地化
- 錯誤訊息範例：

  ```json
  { "message": "報價單已過期，無法批准" }
  ```

---

## 🔟 日誌與追蹤

- 每筆 API 呼叫記錄 `traceId`（UUID）
- 日誌格式：

  ```
  [2025-10-15T07:55:00Z][TRACEID=abc123][USER=admin][URI=/api/quotations/100/events/approve]
  ```

---

## 11️⃣ 驗收標準

| 編號 | 項目           | 驗收方式                    |
| ---- | -------------- | --------------------------- |
| 1    | 命名一致性     | 實體 / DTO / REST Path 對應 |
| 2    | 日期與金額格式 | ISO 8601 UTC + DECIMAL 精度 |
| 3    | 樂觀鎖測試     | 衝突時 409                  |
| 4    | 分頁 / 排序    | 正確傳遞與解析              |
| 5    | 錯誤碼         | 對應 Problem+JSON 規格      |
| 6    | i18n           | 中文與英文版本訊息皆正確    |

---
