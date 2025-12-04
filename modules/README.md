# Flexora 模組文件索引

為了避免 `specs/`、`ui/`、`guides/` 等資料分散難以搜尋，  
自 2025-12-03 起每個模組都會在 `docs/modules/<module>/` 建立統一入口。  
目前仍逐步搬遷中，若內容尚未複製會先連回原始檔。  
整體高層計劃請參考：[GLOBAL_PLAN.md](./GLOBAL_PLAN.md)。

| 模組 | 內容摘要 | 入口 |
| --- | --- | --- |
| 報價（Quotation / Phase 4） | 報價流程、Workflow、UI/交易 API、文件整合 | [quotation](./quotation/README.md) |
| 銷售訂單（Sales Order） | SO 與報價、出貨、Workflow 串接 | [sales-order](./sales-order/README.md) |
| 採購（Purchase） | Rfq → PO → Receipt／Supplier Return | [purchase](./purchase/README.md) |
| 出貨 / Delivery | Delivery Note、包裝、追蹤、倉儲整合 | [delivery](./delivery/README.md) |
| 庫存（Inventory / Phase 3） | 庫存補貨/保留、成本、API 草案 | [inventory](./inventory/README.md) |
| 客戶（Customer / CRM） | 客戶資料、Customer Workspace、Ext Attr | [customer](./customer/README.md) |
| 聯絡人（Contact） | Async Select、Drawer、Follow-up Flow | [contact](./contact/README.md) |
| 產品（Product / Item & SKU） | ItemGroup / Item / SKU / PriceList | [product](./product/README.md) |
| 定價（Pricing） | PriceRule、Pricing Preview、PriceList 指派 | [pricing](./pricing/README.md) |

> 未列出的模組（如採購、銷售訂單）仍在規劃中，待開始實作時再建立目錄。  
> 這些 README 只是一個入口；實際內容仍暫存於 `specs/`、`ui/` 既有檔案，等所有模組完成後再整併。  

## 待建模組（骨架預計開立）

- Campaign / Lead / Opportunity
- Manufacturing / MO
- 服務 / 維修（如有）

之後啟動這些模組時，請依本資料夾格式建立 `README + spec + ui-spec + api-spec + implementation-plan`，並在上表補上連結。
