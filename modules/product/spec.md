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

### ERD（摘要）

- `ItemGroup (1) ── (n) Item`
- `Item (1) ── (n) ItemSku`
- `ItemSku (1) ── (n) ItemPrice`
- `PriceList (1) ── (n) ItemPrice`
- `PriceList` 可對應多個客戶群／通路條件；若未來獨立模組則透過 API 提供建議價。

欄位參考（依現有 entity）：
- `Item`: `code`, `name`, `hasVariants`, `status`, `itemGroupId`.
- `ItemSku`: `skuNo`, `uomId`, `taxCode`, `barcode`, `defaultWarehouseId`.
- `PriceList`: `name`, `currency`, `channel`, `ownerId`, `status`.
- `ItemPrice`: `priceListId`, `itemSkuId`, `unitPrice`, `effectiveFrom/To`, `discountType`.

> 待核對 `com.asynctide.flexora.domain` 與 `.jhipster` 定義後補上完整欄位表與關聯圖。

## 關聯模組

| 模組 | 整合說明 |
| --- | --- |
| Quotation / Sales Order | Async Select SKU、顯示 PriceList 建議價、Variant 標識 |
| Inventory | 共享 UoM、倉庫、批號屬性；SKU Drawer 需顯示預設倉庫與序號策略 |
| Procurement | Rfq / PO 行項引用 SKU 資料；Variant 資訊需同步 |
| Pricing | PriceRule/PriceList 與 SKU 的權限、審批聯動 |
| Delivery | 若 SKU 需序號，Product 需提供標記以驅動倉儲流程 |

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

## 審批 / 權限 / Versioning

- **角色**：產品管理（編輯 Item/SKU）、行銷定價（管理 PriceList/ItemPrice）、營運/審批者（核准上架或價格）。  
- **狀態建議**：
  - Item：`DRAFT / IN_REVIEW / ACTIVE / RETIRED`
  - PriceList：`DRAFT / APPROVED / ARCHIVED`
- **流程**：
  1. 新品建立後可直接上架，或進入 `IN_REVIEW` 等候審批。
  2. PriceList 調整時可選擇「立即生效」或「送審」；審批後才同步到報價/訂單。
  3. 審批機制若尚未實作，可先記錄在 TODO，中期再補 workflow handler。
- **樂觀鎖 / 版本**：
  - 目前 entity 的 `version` 欄位未加 `@Version`，僅為 placeholder。  
  - 若要啟用，需在 entity/service 加上樂觀鎖並於 API 回傳 version，前端儲存時攜帶。

## TODO

- [ ] 補齊 ERD 圖像與欄位說明（含 hasVariants 邏輯與 existing table 對照）。
- [ ] 規範 Product / SKU 建立與啟用流程（草稿→審批→上架→停售）。
- [ ] 決定 PriceList 模組歸屬與審批策略，若獨立需設計對應 API。
- [ ] 完整化 ItemPrice 版本控制、幣別與稅別政策，以及權限矩陣。
- [ ] 與 Inventory / Quotation / Procurement 的資料契約、Async Select API。
- [ ] Variant Generator / 匯入流程與 CSV 欄位定義。

## TODO

- [ ] 補齊 ERD 與欄位說明。
- [ ] 定義 Product 與 Item SKU 的建立/啟用流程。
- [ ] 規範 Pricebook 與 ItemPrice 版本控制。
- [ ] 與 Inventory / Quotation 模組的資料交換契約。
- [ ] 製作 Variant / PriceList 流程圖與 wireframe。  
- [ ] 決定 PriceList 是否獨立模組，若是需描述同步策略。

> 本檔暫存為骨架，待後續規劃時依模組需求更新。  
> 若已有相關資料，請在此連結或引用以避免資訊分散。
