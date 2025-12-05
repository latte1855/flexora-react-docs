# Customer Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| CUS-01 | Workspace / Filter Hub | Preset（我的/待跟進…）、進階條件（健康度、Industry、Owner）、List/Pipeline 共用狀態 | TODO |
| CUS-02 | Detail Panel | 基本資料、Owner/Team、健康度、Tabs（Contact、Activity、Ext Attr、Timeline） | TODO |
| CUS-03 | Drawer / Full Editor | Customer Profile、地址、Ext Attr、Owner/Team 指派、Custom Fields | TODO |
| CUS-04 | Contact Tab 整合 | 在 Detail 中嵌入聯絡人列表、Quick Create、主聯絡人標記 | TODO |
| CUS-05 | Activity / Follow-up | ActivityLog API、健康度計算、提醒與 Follow-up Drawer | TODO |
| CUS-06 | Lookup / Async Select | `GET /api/customers/lookup` + 權限 (ownerScope/team) + 關鍵字/Tag 篩選 | TODO |
| CUS-07 | Workflow / 審批 | 若需審批流，定義狀態、事件、流程圖資料、Trigger Event API | TODO |
| CUS-08 | 匯入 / 匯出 | CSV 欄位、驗證、錯誤處理、Mapping、資料遮罩 | TODO |
| CUS-09 | Data Scope / 權限 | Owner/Team/部門層級、欄位遮罩（Email/TaxID）、審計記錄 | TODO |
| CUS-10 | 與其他模組整合 | 與 Opportunity / SalesOrder / Quotation / Billing 的 Link API 與 UI 快捷 | TODO |
| CUS-11 | 測試計畫 | API / Workflow / Lookup / 匯入 / UI 自動化 | TODO |

> 相關 checklist 仍可參考 [`Customer 模組 Codex 開發 Checklist`](../../ui/customer/Customer%20模組%20Codex%20開發%20Checklist（前端%20+%20後端%20+%20資料結構）.md)，此表則用於正式任務拆解。  
