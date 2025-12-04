# Product Module – Functional Spec

> 狀態：草稿（2025-12-03）  
> 目的：集中定義 Product、Item SKU、Pricebook 之間的資料模型與行為。詳細敘述將在 Phase 5 展開時補齊。

## 範圍與層級

| 層級 | 說明 | 主要行為 |
| --- | --- | --- |
| `ItemGroup` | 產品線 / 系列（品牌、分類、共用屬性） | Hub filter、匯入對應欄位、批次設定預設稅則 |
| `Item` | 產品主檔，決定是否 `hasVariants` | 決策流程：建立基本資料 → 決定變形策略 → 產出 SKU |
| `ItemSKU` | 銷售/採購單位（含 UoM、稅別、條碼） | 表格 + Drawer 維護；對接庫存、報價、採購 |
| `PriceList` | 價目表主檔（通路、幣別、適用客群） | 可獨立管理，也可由產品模組入口跳轉 |
| `ItemPrice` | SKU 在各價目表的定價紀錄 | 含開始/結束日、折扣型態、版本控制 |

> `PriceList` 是否獨立模組？目前規劃為「共享資源」，可在 Product 模組快速連結，也能在未來抽成 Pricing 模組。文件中先記錄兩種做法的差異。

## 關聯模組

- **Quotation**：建立/編輯報價時需 Async Select SKU、帶入 PriceList 建議價。
- **Inventory**：SKU 與倉庫、批次、MOQ 整合。
- **Procurement / Sales Order**：沿用 SKU 與價目表邏輯。
- **Reporting**：ItemGroup 與 PriceList 是報表分析的基本條件。

## 操作流程（摘要）

1. **建立 Item**  
   - 選 ItemGroup → 填主檔欄位 → 指定是否 `hasVariants`。  
   - 若 `hasVariants=true`，需設定 Variant 維度（如顏色、容量）並由系統產生多個 SKU。
2. **維護 ItemSKU**  
   - 以表格呈現，搭配 Drawer 調整 UoM、稅別、條碼、狀態。  
   - 可複製 SKU 或由 Variant Generator 批次產生。
3. **設定 PriceList / ItemPrice**  
   - 在 SKU Drawer 顯示「各價目表建議價」，可跳轉到 PriceList 模組完整編輯。  
   - 支援匯入或同步 ERP 的價目資料。

## TODO

- [ ] 補齊 ERD 與欄位說明（含 hasVariants 邏輯）。
- [ ] 規範 Product / SKU 建立與啟用流程（草稿→上架→停售）。
- [ ] 定義 PriceList 是否獨立模組或由 Product 代理的決策準則。
- [ ] 完整化 ItemPrice 版本控制、幣別與稅別政策。
- [ ] 與 Inventory / Quotation / Procurement 的資料契約、Async Select API。

## TODO

- [ ] 補齊 ERD 與欄位說明。
- [ ] 定義 Product 與 Item SKU 的建立/啟用流程。
- [ ] 規範 Pricebook 與 ItemPrice 版本控制。
- [ ] 與 Inventory / Quotation 模組的資料交換契約。

> 本檔暫存為骨架，待後續規劃時依模組需求更新。  
> 若已有相關資料，請在此連結或引用以避免資訊分散。
