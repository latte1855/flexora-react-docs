# Sales Order Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| SO-01 | Workspace UI 實作 | Hub + List + Detail + Workflow 行為，沿用報價樣式 | TODO |
| SO-02 | Detail / Drawer | 客戶資訊、行項表格、Ext Attr、Workflow 按鈕 | TODO |
| SO-03 | Transaction API / Payload | 定義一次提交 SO + Items + Ext Attr 的 API，含錯誤格式 | TODO |
| SO-04 | Lookup / Async Select | `GET /api/sales-orders/lookup` for Delivery / CRM | TODO |
| SO-05 | SO ↔ Quotation Link | 梳理 link API、UI 操作（從 Quote 建 SO、顯示關聯） | TODO |
| SO-06 | Workflow / Timeline | 收斂事件、狀態、歷史 API，提供流程圖資料 | TODO |
| SO-07 | 與 Delivery / Invoice 整合 | 交期、出貨、發票連動（Link + UI 快捷） | TODO |
| SO-08 | 匯入/批次交期更新 | 決定是否需要 CSV / Command API | TODO |
| SO-09 | 測試計畫 | API/Integration/前端驗收用例 | TODO |

> 待正式啟動開發後再補充優先順序、預估成本與負責人。
