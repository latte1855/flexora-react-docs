# Quotation Module – API Spec

> API/Workflow 規格目前請參考：  
> [`docs/specs/phase4-quotation/api-spec.md`](../../specs/phase4-quotation/api-spec.md)  
> 以及 Workflow 相關補充：[`workflow-spec.md`](../../specs/phase4-quotation/workflow-spec.md)、[`workflow-guards.md`](../../specs/phase4-quotation/workflow-guards.md)。

涵蓋內容：

- QuotationThread / QuotationRevision REST 介面  
- `/api/quotations/transactions` 整批儲存  
- Workflow 事件 (`/events/{code}`)、Status History、Guard API  
- Pricing/Preview endpoints

後續若有新的 API 或 DTO 調整，請在舊檔與本目錄同時更新。  
