# Product Module

聚焦產品主檔、Item SKU、Pricebook 相關資料，作為報價、庫存與採購的共用基礎。  
此模組將逐步承接舊有 `specs/phase3-*` 的資訊，並以 Hub + Drawer + Transaction API 的方式規劃前後端。

| 文件 | 說明 |
| --- | --- |
| [spec.md](./spec.md) | 功能 / 資料模型需求（ItemGroup、Item、SKU、PriceList、審批） |
| [ui-spec.md](./ui-spec.md) | Workspace、SKU 表格、Variant Generator、PriceList 提示 |
| [api-spec.md](./api-spec.md) | Item / SKU / PriceList REST 端點與整單 Transaction API 草案 |
| [implementation-plan.md](./implementation-plan.md) | 任務拆解 / 進度追蹤（MVP 補貨、PriceList 版本、Lookup 等） |

> PriceList 目前視為本模組的共享子系統；若未來要獨立為 Pricing 模組，請於 implementation plan 中更新決策。  
> 若要查看最新的報價/整單編輯需求，可參考 [`docs/ui/quotation-ui-wireframe.md`]，其中的 SKU/PriceList 互動與本模組共用。

## 關聯模組

- **Quotation / Sales Order**：使用 SKU Lookup、PriceList 建議價。  
- **Inventory**：共享 UoM、倉庫預設與批號屬性。  
- **Pricing**：若 PriceRule 套用 SKU 層級，需與本模組同步欄位。  
- **Purchase**：從 Item/Variant 切換到 PO 行項時需帶入描述、稅別。

## 待搬遷 / 補充資料

- Phase3 時期的詳細規格仍在 `docs/specs/phase3-inventory/`；未完成部分請在本資料夾補上。  
- 如需補 Worflow / 審批圖，請在 `spec.md` 補充，並在 README 中更新連結。
