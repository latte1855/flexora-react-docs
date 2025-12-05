# Contact Module – API Spec (草稿)

| 資源 | Controller / Path | 說明 |
| --- | --- | --- |
| Contact | `ContactResource` (`/api/contacts`) | CRUD + Lookup |
| Contact Transaction（建議） | `/api/contacts/transactions` | 一次提交聯絡人 + 地址 + Ext Attr |
| Ext Attr | `ContactExtAttrDefResource` | 客製欄位定義 |
| Owner / Team | `OwnerResource`, `TeamResource` | 供下拉使用 |

## 1. CRUD

| Method | Path | 說明 | 現況 |
| --- | --- | --- | --- |
| GET | `/api/contacts` | 列表 / Criteria（customerId、ownerId、status、keyword、tag） | ✅ |
| POST | `/api/contacts` | 建立聯絡人 | ✅ |
| GET | `/api/contacts/{id}` | 詳細資料 | ✅ |
| PUT/PATCH | `/api/contacts/{id}` | 更新 / 部分更新 | ✅ |
| DELETE | `/api/contacts/{id}` | 軟刪除 | ✅ |

DTO 欄位：`fullName`, `firstName`, `lastName`, `title`, `department`, `email`, `mobile`, `phone`, `customerId`, `ownerId`, `isPrimary`, `extAttrs`, `tags`.

建議新增 Transaction API，以便 Customer Drawer 在一次儲存時一併寫入聯絡人與地址：

```
POST /api/contacts/transactions
{
  "contact": { ...ContactDTO... },
  "addresses": { ...OptionalAddress... },
  "extAttrs": { ... }
}
```

## 2. Lookup

`ContactResource#lookupContacts` 已存在：

- `GET /api/contacts/lookup?customerId=&keyword=&limit=`  
  - 參數：`customerId` (可選)、`keyword`、`limit` (default 20)。  
  - 權限：只回傳使用者有權限看到的客戶。  
  - 回傳：`id`, `fullName`, `email`, `mobile`, `customer { id, name }`.

如果未來要提供 Owner Scope（如只搜我的聯絡人），可新增 `ownerScope` 參數。

## 3. Ext Attr

- `GET /api/contact-ext-attr-defs`：欄位定義與型別。  
- 若需單獨更新 Ext Attr，可新增 `PATCH /api/contacts/{id}/ext-attrs`（目前尚未提供）。  
- 匯入/匯出需依 Ext Attr 定義動態解析。

## 4. 匯入 / 批次

- 目前沒有官方匯入 API；若要支援 CSV 匯入，可新增 `POST /api/contacts/import`，欄位至少包含 `customerId`, `fullName`, `email`.  
- 匯出可透過 `/api/contacts` + `size=-1` 或另建 `/export`.

## 匯入 / 批次

- `POST /api/contacts/import`：CSV 匯入，欄位包含 `customerId`, `fullName`, `email`, `phone`, `owner`, `isPrimary`。  
- `POST /api/contacts/{id}/owner`：批次變更 Owner（若需要）。  
- 匯出可透過 `/api/contacts/export?filter=...`。

## TODO

- [ ] 若 Contact 需要 Workflow/審批（黑名單、GDPR），須擴充欄位與 API。  
- [ ] 決定是否開放批次更新（例如更換 Owner/標籤）。  
- [ ] 與 Customer Workspace 的 Activity / Follow-up API 銜接。  
- [ ] 定義 Address/ExtAttr 在 Transaction API 的格式。  
