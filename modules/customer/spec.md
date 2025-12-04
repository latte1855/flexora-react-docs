# Customer Module – Functional Spec

> 覆蓋 Customer Workspace、客戶主檔、Ext Attr、活動紀錄、資料權限等。過往資訊散落於 SRS/Checklist，本檔為新版彙整。

## 範圍與關聯

| 範疇 | 說明 |
| --- | --- |
| Customer 主檔 | `Customer`、`CustomerExtAttr`、Address、Owner、狀態 |
| Contact | 與 Customer 1-n，於 Contact 模組詳細定義 |
| Activity Log | `ActivityLogResource`：紀錄拜訪、備註 |
| Workflow / 健康狀態 | 若需審批或健康指標，可於此定義 |
| 與 Sales/Opportunity | 儲存 Sales Thread、Quotation 連結、SO/Invoice 參考 |

## 主要欄位

| 類別 | 欄位 | 備註 |
| --- | --- | --- |
| 基本資料 | customerNo, name, type (法人/個人), taxId, status (`ACTIVE/INACTIVE/BLACKLIST`), ownerId, channel | Status 需與 Workflow 配合 |
| 聯絡方式 | mainContactId, phone, email, website, preferredLanguage | mainContact 指向 Contact 模組 |
| 地址 | billingAddress, shippingAddress；支援多地址 | 與 AddressSnapshot 結構共用 |
| 財務 | paymentTerm, creditLimit, currency | 影響報價/訂單 |
| 標籤 / 分類 | customerGroup, industry, accountTier | Hub/Filter 使用 |
| Ext Attr | 客製欄位（CustomerExtAttr） | 三欄網格呈現 |

## Customer Workspace 流程

1. 透過 Hub 篩選客戶（我的客戶、潛在、黑名單…）。  
2. 展開任何客戶 → Detail Panel 顯示摘要、關聯記錄、Activity。  
3. 透過 Drawer 建立 / 編輯；可快速建立 Contact、Address。  
4. Activity / Follow-up：可在 Detail 直接記錄或分頁顯示。

## Workflow / 資料權限

- 目前尚未有正式 Workflow；若需要審核流程（例如新客戶需審批），可新增 `status` 欄位與 `CustomerStateTransitionResource`。  
- 權限：`Owner` + `Team` + Data Scope（參考 `docs/modules/inventory/...`）。  
- 預計：`Owner` 可編輯；同部門可檢視；黑名單可限制。

## TODO

- [ ] 製作 ERD（Customer、Contact、Address、ExtAttr）。  
- [ ] 決定是否啟用 Workflow/健康度（healthStatus, lastActivityAt）。  
- [ ] 補上與 Sales/Opportunity 的連結說明。  
- [ ] 記錄匯入/匯出流程、資料驗證（稅號、Email 等）。  
- [ ] 若需 GDPR/個資控管，在此標註流程。*** End Patch***
