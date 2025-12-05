# Sales Order Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| SO-01 | Workspace / Filter Hub | Hub Preset（我的/待出貨…）、進階篩選、List/Pipeline 共用狀態 | TODO |
| SO-02 | Detail Panel | 摘要卡、客戶/聯絡人、金額、Workflow 按鈕、Tab（行項/計價/附件/Timeline） | TODO |
| SO-03 | Drawer / Full Editor | 客戶選擇、行項表格（SKU Lookup、稅別、折扣）、出貨 / Invoice 設定 | TODO |
| SO-04 | Transaction API / Payload | 單次提交 `SalesOrder + Items + Ext Attr + Link`，定義錯誤格式、Trace | TODO |
| SO-05 | QuickCreate 轉換 | 從報價單建立 Sales Order 的 API / UI 串接與欄位映射 | TODO |
| SO-06 | Workflow / Timeline | 事件/狀態整理、流程圖資料、Trigger Event API 統一 | TODO |
| SO-07 | Delivery / Invoice Integration | 建立 Delivery / Invoice 快捷、Link API、狀態同步 | TODO |
| SO-08 | Lookup / Async Select | `GET /api/sales-orders/lookup` + 權限 + ownerScope，供 Delivery/CRM 使用 | TODO |
| SO-09 | 匯入 / 批次更新 | CSV/Excel 匯入行項、交期、客戶條件；Command API | TODO |
| SO-10 | 規則 / 權限 | Owner / Team 範圍、審批需求、敏感欄位遮罩 | TODO |
| SO-11 | 測試計畫 | API / Workflow / QuickCreate / Integration 測試 | TODO |

> 待正式啟動開發後再補充優先順序、預估成本與負責人。
