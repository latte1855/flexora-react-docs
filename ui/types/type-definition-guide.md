# 前端類型定義指南

本指南說明如何在前端定義與後端對應的 TypeScript 類型 (Interface/Type)，以確保前後端資料串接的一致性與型別安全。

## 1. 基本原則

1. **命名一致性**: 前端 Interface 命名應盡量與後端 DTO 保持一致（或去除 `DTO` 後綴）。
2. **對齊 DTO**: 欄位名稱與結構必須對齊後端 Swagger/OpenAPI 定義。
3. **明確的可選性**: 使用 `?` 標記可選欄位，區分 `null` 與 `undefined`。

## 2. DTO 對應範例

### 後端 Java DTO

```java
public class CustomerDTO {
    private Long id;
    @NotNull
    private String name;
    private String email;          // 可空
    private AddressDTO address;    // 關聯物件
    private List<ContactDTO> contacts;
    private Instant createdAt;
}
```

### 前端 TypeScript Interface

```ts
export interface Customer {
    id: string; // ID 統一轉為 string (避免大數問題)
    name: string;
    email?: string; // 可選欄位
    
    // 關聯物件
    address?: Address;
    contacts?: Contact[];
    
    // 時間欄位通常為 ISO String
    createdAt?: string; 
}
```

## 3. 特殊類型處理

| 資料類型 | 前端處理 | 說明 |
|---------|---------|------|
| `Long` / `BigInteger` | `string` | 為了避免 JavaScript 精度遺失，ID 與大數應轉為字串。 |
| `BigDecimal` | `number` 或 `string` | 金額運算建議用 `number`，若需高精度展示則維持 `string`。 |
| `LocalDate` | `string` | 格式 `YYYY-MM-DD`。 |
| `Instant` / `ZonedDateTime` | `string` | ISO 8601 格式 `YYYY-MM-DDTHH:mm:ss.sssZ`。 |
| `Enum` | `string` 或 Union Type | 例: `type Status = 'OPEN' \| 'CLOSED'` |
| `JSONB` | `Record<string, any>` | 或定義具體 Interface。 |

## 4. 常用通用類型

### 分頁回應 (List Response)

```ts
export interface PageResponse<T> {
    data: T[];
    total: number;
    pageIndex: number;
    pageSize: number;
}
```

### 關聯物件參照 (Relation)

若後端僅回傳 ID，定義為 `string`；若回傳完整物件或快照，定義為物件類型。

```ts
interface Order {
    customerId: string; // 僅 ID
    customer?: Customer; // 完整物件 (若有 expand)
}
```

## 5. 檢查清單

- [ ] 所有必填欄位是否**沒有**加 `?`。
- [ ] 所有 ID 欄位是否定義為 `string`（若後端是 Long）。
- [ ] 列舉值 (Enum) 是否定義了 Union Type 以獲取自動補全。
- [ ] 日期欄位是否註解了格式 (e.g. `// YYYY-MM-DD`)。
