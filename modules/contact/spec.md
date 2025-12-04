# Contact Module – Functional Spec

> 聯絡人資料由 Contact 模組維護，與 Customer、Sales/Quotation 共享。此為新版規格。

## 範圍

| 範疇 | 說明 |
| --- | --- |
| Contact 主檔 | 姓名、職稱、Email、Phone、Owner、主要聯絡人 flag |
| Ext Attr | 自訂欄位（專案代碼、客戶 PO 編號等） |
| 關聯 | `customerId`, `team`, `owner`, 活動紀錄 |
| Async Select | 報價、訂單、PO 等使用的搜尋服務 |
| Follow-up / 活動 | 在 Customer Detail 中顯示、建立活動 |

## 資料摘要

| 類別 | 欄位 | 備註 |
| --- | --- | --- |
| 基本 | firstName/lastName/fullName, title, department, ownerId, status (`ACTIVE/INACTIVE`) | 支援主聯絡人 `isPrimary` |
| 聯絡方式 | email, mobile, phone, assistant, preferredLanguage | Email/Phone 需驗證 |
| 地址 | mailingAddress | 可選 |
| 客戶關聯 | `customerId`, `customerName` | 必填 |
| Ext Attr | ContactExtAttr | 例：客戶 PO 編號、專案代碼 |

## 行為

- Contact Workspace：可依 Owner、Customer、標籤、狀態篩選。  
- Drawer：建立/編輯聯絡人，同時可指派 Owner、標註為主聯絡人。  
- 與 Customer Detail 整合：在 Customer Detail 的聯絡人 Tab 列出、可 Quick Edit。  
- Async Select：支援關鍵字 + ownerScope + customerId 過濾。

## TODO

- [ ] 製作 ERD：Contact ↔ Customer ↔ Activity。  
- [ ] 欄位驗證（Email 需正規化、Phone 遮罩）。  
- [ ] 匯入/匯出需求（CSV 篩選權限）。  
- [ ] 若需審批或禁用流程，定義狀態與權限。
