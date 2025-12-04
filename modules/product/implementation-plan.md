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
| PROD-01 | Product Entity / Repository / REST | 定義產品主檔欄位、狀態 | ⏳ |
| PROD-02 | Item SKU + Default UoM/Tax | 供報價/庫存使用的 SKU 主檔 | ⏳ |
| PROD-03 | Pricebook / ItemPrice API | 多幣別、通路、有效日期 | ⏳ |
| PROD-04 | SKU Lookup Service | Async Select + 篩選（關鍵字、價目表） | ⏳ |
| PROD-05 | UI – Catalog / SKU Workspace | 列表 + Drawer 編輯 | ⏳ |

## 3. TODO / 風險

- [ ] 確認現有資料表（`item_sku`, `price_list`, `item_price`）是否需要遷移或擴充。
- [ ] 與 Inventory 模組共享 UoM、稅別定義。
- [ ] 行銷/產品組對價格管理的權限需求。

> 本檔為骨架，待正式展開時再逐項填入細節與進度。
