# Customer Module – API Spec (草稿)

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| Customer | `CustomerResource` (`/api/customers`) | CRUD + Criteria |
| Customer Ext Attr | `CustomerExtAttrDefResource` | 客製欄位定義 |
| Address | `AddressResource` | 儲存/查詢地址 |
| Owner / Team | `OwnerResource`, `TeamResource` | 供下拉選擇 |
| Activity Log | `ActivityLogResource` | 客戶活動紀錄 |

## 1. Customer CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/customers` | 列表，支援 Criteria（ownerId、status、customerGroup、keyword…） | ✅ |
| POST | `/api/customers` | 建立客戶 | ✅ |
| GET | `/api/customers/{id}` | 詳細資料（含地址/ExtAttr） | ✅ |
| PUT/PATCH | `/api/customers/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/customers/{id}` | 軟刪除 | ✅ |

> Criteria 物件：`CustomerCriteria`，可帶 `ownerId`, `customerGroupId`, `status`, `industryId`, `createdBy`, `page`, `sort` 等欄位。

## 2. Lookup / Async Select

- 建議提供 `GET /api/customers/lookup?keyword=&ownerScope=SELF&status=ACTIVE&limit=20`。  
- 回傳欄位：`id`, `customerNo`, `name`, `owner { id, name }`, `status`, `customerGroup`.  
- 亦可提供 `GET /api/customer-groups/lookup` 供 Filter Hub 使用。

## 3. Ext Attr / Address

- `CustomerExtAttrDefResource`：列出可用欄位、型別。  
- 實際資料放在 `Customer` 的 DTO 中; 若需要單獨更新可新增 `PATCH /api/customers/{id}/ext-attrs`.  
- `AddressResource` 或 AddressSnapshot 概念：為 Billing/Shipping 提供統一結構。

## 4. Activity / Follow-up

- `ActivityLogResource`: `GET/POST /api/activity-logs`；可帶 `customerId`.  
- 若需要 Follow-up Drawer API，需定義 `GET /api/customers/{id}/follow-ups`.  
- 可整合通知/提醒。

## 5. Permissions / Data Scope

- 客戶資料通常受 Owner/Team 控制。API 需套用 Data Scope（可參考 `DataScopeDebugResource`）。  
- 若有 Workflow/審批，可在 spec 中加上 `CustomerStateTransitionResource`（目前尚無）。

## TODO

- [ ] 審視 `CustomerResource` 中是否已有 `lookup` 或 `search`，若無需新增。  
- [ ] 若要求整單 Transaction API（Customer + Contact + Address），需設計 payload。  
- [ ] 匯入/匯出 API（CSV）需求。  
- [ ] GDPR / 個資相關 API（遮罩、刪除請求）。*** End Patch
