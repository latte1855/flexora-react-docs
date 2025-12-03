# Flexora 模組文件索引

為了避免 `specs/`、`ui/`、`guides/` 等資料分散難以搜尋，  
自 2025-12-03 起每個模組都會在 `docs/modules/<module>/` 建立統一入口。  
目前仍逐步搬遷中，若內容尚未複製會先連回原始檔。  
整體高層計劃請參考：[GLOBAL_PLAN.md](./GLOBAL_PLAN.md)。

| 模組 | 內容摘要 | 入口 |
| --- | --- | --- |
| 報價（Quotation / Phase 4） | 報價流程、Workflow、UI/交易 API、文件整合 | [modules/quotation](./quotation/README.md) |
| 庫存（Inventory / Phase 3） | 庫存補貨/保留、成本、API 草案 | [modules/inventory](./inventory/README.md) |
| 客戶（Customer / CRM） | 客戶/聯絡人工作區、Async Select、資料結構 | [modules/customer](./customer/README.md) |

> 未列出的模組（如採購、銷售訂單）仍在規劃中，待開始實作時再建立目錄。  
> 這些 README 只是一個入口；實際內容仍暫存於 `specs/`、`ui/` 既有檔案，等所有模組完成後再整併。  
