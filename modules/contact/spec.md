# Contact Module – Functional Spec

> 狀態：骨架（2025-12-03）  
> 目的：集中聯絡人主檔、擴充欄位與權限邏輯，避免 scattered spec。

## 範圍

- **Contact 主檔**：基本資料（姓名、職稱、Email、Phone、Owner）。
- **Ext Attr**：模組化客製欄位，與 Customer/Opportunity 共用行為。
- **關聯**：與 Customer、Sales Owner、活動追蹤的連結。
- **讀取情境**：Async Select（報價、訂單等）、Workspace 詳細頁、Follow-up Drawer。

## TODO

- [ ] 繪製 ERD：Contact ↔ Customer ↔ ActivityLog ↔ Ownership。
- [ ] 列出 CRUD 欄位驗證、遮罩與去識別需求。
- [ ] 規範「大量資料」時的搜尋 / 分頁策略。
- [ ] 定義 Ext Attr 的繼承與版本控制。
- [ ] 決定匯入/匯出的欄位映射與權限檢查。

> 目前詳細內容仍在 `docs/ui/customer/*.md` 與 `docs/specs/` 之間；待模組開發展開後搬遷統整。
