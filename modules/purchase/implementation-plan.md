# Purchase Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| PUR-01 | ERD / Domain 梳理 | RFQ → PO → Receipt → Billing / GRN 的資料表、Workflow、關聯 | TODO |
| PUR-02 | Workspace / Filter Hub | Hub + List + Pipeline（依狀態/供應商/Buyer）及 Detail Panel | TODO |
| PUR-03 | Purchase Order Drawer | 基本資料、行項、稅別、附件、審批按鈕；支援子 Drawer（供應商、Item） | TODO |
| PUR-04 | Receipt / Invoice Flow | 規劃「建立 Receipt」「建立 AP Invoice」等衍生動作與 Drawer | TODO |
| PUR-05 | Workflow / Transition API | 整合 `PurchaseOrderStateTransitionResource` 等端點到 Transaction API | TODO |
| PUR-06 | 匯入 / 批次處理 | RFQ/PO/Receipt 匯入格式、驗證、錯誤回饋、排程 | TODO |
| PUR-07 | 供應商 / Contract 連動 | 從 Vendor 模組帶入條款、付款條件、PriceList | TODO |
| PUR-08 | Inventory / Accounting 整合 | 收貨入庫、凍結/扣帳、Invoice Matching、GL Posting | TODO |
| PUR-09 | Lookup / Async Select | Vendor / Item / PriceList Lookup API，支援權限與狀態過濾 | TODO |
| PUR-10 | 測試計畫 | API、Workflow、匯入工具、與 Inventory / AP 模組的整合測試 | TODO |

> 待正式開發時，對應 spec / ui-spec / api-spec 的欄位與流程再補細節與負責人。  
