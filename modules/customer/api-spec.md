# Customer Module – API Spec

> 聚焦 Customer 主檔、聯絡人關聯、Activity/Follow-up 與權限。既有端點多在 `CustomerResource`，本檔整理缺口與建議新增的 Transaction/Lookup API。

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| Customer | `CustomerResource` (`/api/customers`) | CRUD + Criteria |
| Customer Transaction（建議） | `/api/customers/transactions` | 一次提交客戶 + 聯絡人 + 地址 + Ext Attr |
| Customer Ext Attr | `CustomerExtAttrDefResource` | 動態欄位定義 |
| Address Snapshot | `AddressResource` 或在 DTO 中嵌入 | Billing / Shipping 共用結構 |
| Owner / Team | `OwnerResource`, `TeamResource` | Hub 與 Drawer 下拉使用 |
| Activity / Follow-up | `ActivityLogResource`, 建議新增 `CustomerFollowUpResource` | 活動紀錄、提醒任務 |

## 1. Customer CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/customers` | 列表（支援 `ownerId`, `status`, `customerGroupId`, `keyword`, `healthStatus`…） | ✅ |
| POST | `/api/customers` | 建立客戶 | ✅ |
| GET | `/api/customers/{id}` | 詳細資料（含地址 / Ext Attr / 主聯絡人） | ✅ |
| PUT/PATCH | `/api/customers/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/customers/{id}` | 軟刪除 | ✅ |

> Criteria 物件：`CustomerCriteria`，具 `ownerId`, `customerGroupId`, `industryId`, `status`, `createdDate`, `page`, `sort` 等欄位。

## 2. Transaction API（建議新增）

```
POST /api/customers/transactions
{
  "customer": { ...CustomerDTO... },
  "contacts": [ ...ContactDTO... ],
  "addresses": {
    "billing": { ...AddressSnapshotDTO... },
    "shipping": { ...AddressSnapshotDTO... }
  },
  "extAttrs": { "CUSTOMER_PO": "AB-123" }
}
```

- 目的：一次儲存客戶、主聯絡人、地址與 Ext Attr，類似報價的 Transaction API。  
- 回傳：`CustomerDTO` + 新增聯絡人資訊。  
- 若更新既有客戶，可提供 `PUT /api/customers/{id}/transactions`。  
- 需在 Service 層以 `@Transactional` 確保失敗 rollback。

## 3. Lookup / Async Select

```
GET /api/customers/lookup
  ?keyword=acme
  &ownerScope=SELF|TEAM|ALL
  &customerGroupId=
  &status=ACTIVE
  &healthStatus=GREEN
  &limit=20
```

- 回傳：`id`, `customerNo`, `name`, `owner { id, name }`, `status`, `customerGroup`, `mainContact`.  
- 另可提供 `GET /api/customer-groups/lookup`、`GET /api/customer-tags/lookup` 等輔助端點。  
- 供報價、訂單、活動模組的 Async Select 使用。

## 4. Ext Attr / Address

- `CustomerExtAttrDefResource`：取得欄位定義與資料型別。  
- 可新增 `PATCH /api/customers/{id}/ext-attrs` 以單獨更新自訂欄位。  
- 地址建議統一使用 AddressSnapshot DTO；若需共用 AddressResource，須定義 `type=BILLING/SHIPPING`。

## 5. Activity / Follow-up

- `ActivityLogResource`: `GET/POST /api/activity-logs`，可帶 `customerId`, `contactId`, `type`.  
- 建議新增 `CustomerFollowUpResource`：  
  - `GET /api/customers/{id}/follow-ups` – 查詢未完成提醒  
  - `POST /api/customers/{id}/follow-ups` – 新增提醒（欄位：`title`, `dueDate`, `ownerId`, `notes`）  
  - `POST /api/customer-follow-ups/{id}/complete` – 完成提醒  
- 可整合通知（Email/SMS/Push）或日曆同步。

## 6. Workflow / Permissions / Data Scope

- 若需要審批流程，新增 `POST /api/customers/{id}/submit`, `/approve`, `/reject`, `GET /api/customers/{id}/status-history`。  
- 權限：依 Owner/Team/Data Scope 控管；可參考 `DataScopeDebugResource`。  
- 敏感欄位（稅號、Email、信用額度）需依權限決定是否遮罩。  
- 黑名單客戶：可新增 `POST /api/customers/{id}/blacklist`、`/unblacklist`。

## 7. 匯入 / 匯出 / GDPR

- `POST /api/customers/import`：CSV 匯入（欄位：客戶編號、名稱、Owner、群組、Email、Phone、地址…），回傳批次 ID 與錯誤行。  
- `GET /api/customers/export`：依 filter 匯出。  
- GDPR / 個資請求：可新增 `POST /api/customers/{id}/anonymize` 或提供遮罩 API。

## TODO

- [ ] 確認是否已有 `lookup` endpoint；若無需實作。  
- [ ] 設計 Transaction API Payload 與 Error 格式。  
- [ ] 匯入/匯出 API 詳細規格與權限。  
- [ ] Workflow 與黑名單流程是否啟用。  
- [ ] 若 Customer 建立時需要同步建立 Contact/Address，請在 Service 端提供 helper 以減少重複程式。  
