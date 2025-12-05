# Product Module – Implementation Plan

> 狀態：待展開  
> 目的：拆解 Product / Item SKU / Pricebook 的開發任務，並追蹤進度。

## 1. 近期目標

1. 建立 Product / Item SKU 資料表、REST API。
2. 提供 SKU Lookup / Async Select 供報價、庫存等模組使用。
3. 規劃 Pricebook & ItemPrice CRUD，支援多幣別與批次匯入。

## 2. 任務草稿

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| PROD-01 | `ItemGroup` / `Item` Domain | 建立產品線、`hasVariants` 欄位與 workflow | ⏳ |
| PROD-02 | Variant 定義與 Generator | CRUD for `VariantDimension/Value`、SKU 預覽/產生邏輯 | ⏳ |
| PROD-03 | `ItemSKU` 表格 + Drawer | 支援 UoM、稅別、條碼、Ext Attr；含複製 SKU | ⏳ |
| PROD-04 | PriceList vs. Product 邏輯 | 決定是否獨立模組 / 共享入口，設計 API | ⏳ |
| PROD-05 | `ItemPrice` CRUD / 匯入 | 多幣別、通路、有效日期，並供報價查詢 | ⏳ |
| PROD-06 | SKU Lookup Service | Async Select + 篩選（關鍵字、ItemGroup、PriceList） | ⏳ |
| PROD-07 | UI – Catalog Workspace | Hub + List + Drawer；Pipeline 需求另列 TODO | ⏳ |
| PROD-08 | PriceList UI/提示 | 在 SKU Drawer 顯示建議價，連結 PriceList 頁面；提供 ASCII Wireframe（暫代 Figma） | ⏳ |
| PROD-09 | Product Transaction API | 規劃整單儲存 payload（Item + Variant + SKU + ItemPrice），列出欄位驗證 | ⏳ |
| PROD-10 | PriceList 審批/歷史 | 若啟用 workflow，新增 `submit/approve/history` 端點，定義通知/權限 | ⏳ |
| PROD-11 | Variant/PriceList Wireframe | 產出 Hub/Drawer/Variant ASCII Wireframe，標題與 UI Spec 同步 | ⏳ |
| PROD-12 | API 欄位/審批協作 | 與後端確認 Transaction API 欄位與 Variant/Price 審批策略；紀錄 meeting note、open issue | ⏳ |
| PROD-13 | 實作工單拆解 | 依最終欄位/審批決策拆成後端/前端工單：Variant Generator、Transaction API、PriceList Workflow | ⏳ |
| PROD-14 | Price Transaction QA | 擬定整單 transaction 測試案例（CRUD、審批、rollback） | ⏳ |

## 3. TODO / 風險

- [ ] 確認 `item`, `item_sku`, `price_list`, `item_price` 的現況與遷移策略。
- [ ] Variant Generator 與匯入流程（CSV / ERP 同步）。
- [ ] 與 Inventory 模組共享 UoM、稅別定義。
- [ ] 行銷/產品組對價格管理的權限需求、版本審核。
- [ ] Lookup 與 Transaction API 需評估與報價模組共用 DTO/Service，避免重複邏輯。

> 本檔為骨架，待正式展開時再逐項填入細節與進度。
