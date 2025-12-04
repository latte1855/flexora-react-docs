# Contact Module – API Spec

> 撰寫目的：整理聯絡人 CRUD / Lookup / Ext Attr API 契約。  
> 狀態：TODO，先列出預期端點與欄位。

## 1. Lookup / Async Select

- `GET /api/contacts/lookup?customerId=&keyword=&limit=`  
  - 權限：僅可查詢自己可見的客戶。
  - 回傳欄位：`id`, `fullName`, `email`, `mobile`, `customer { id, name }`.

## 2. CRUD

- `POST /api/contacts` / `PUT /api/contacts/{id}`  
  - Request：基本欄位 + `extAttrs`.
  - Validation：Email/Phone 格式、ownerId 必填、customerId 必須存在。
- `GET /api/contacts/{id}`：含活動紀錄、Ext Attr。
- `DELETE /api/contacts/{id}`：軟刪除，需紀錄 `deletedBy`.

## 3. Ext Attr

- `GET /api/contact-ext-attr-defs` 取得欄位定義。
- `PATCH /api/contacts/{id}/ext-attrs` 部分更新。

## TODO

- [ ] 與 `ContactResource` 既有行為比對，確認欄位命名一致。
- [ ] 決定批次匯入（CSV）的 API 或流程。
- [ ] 測試案例：lookup paging、權限過濾、Ext Attr 驗證。
