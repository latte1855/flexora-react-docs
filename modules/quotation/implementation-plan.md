# Quotation Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| QTN-01 | Workspace / Filter Hub | Hub Preset、進階篩選、List/Pipeline 共用 filter state，避免重複 API call | TODO |
| QTN-02 | Pipeline 體驗 | 拖拉互動、Stage 設定、API Debounce、提示訊息、狀態覆寫 | TODO |
| QTN-03 | QuickCreate Drawer | SKU Lookup、UOM/Tax 自動帶出、地址/Owner Async Select、錯誤提示 | TODO |
| QTN-04 | Quote Editor (Full) | 整單編輯含交易 API、行項表格優化、預覽、自動計價 | TODO |
| QTN-05 | Transaction API 統一 | `POST /api/quotations/transactions` 供 QuickCreate / Quote Editor / Workflow 共用 | TODO |
| QTN-06 | Workflow Drawer | Trigger Event 行為、流程圖 / Timeline 切換、Revision Segments | TODO |
| QTN-07 | Attachment / Document | Upload/Preview/Download API、權限、流程上的附件統計 | TODO |
| QTN-08 | Lookup / Integration | 提供 Delivery/SO 使用的 Lookup API、與 Customer/Contact/PriceList 整合 | TODO |
| QTN-09 | 匯入 / 複製 / 匯出 | CSV 匯入、複製版本、PDF/報價匯出等工具 | TODO |
| QTN-10 | 測試計畫 | API、Pricing Preview、UI 自動化、Workflow 事件測試 | TODO |

> 其他細節仍可參考 `docs/ui/quotation-ui-implementation-plan.md`、`docs/ui/quotation-ui-redesign-plan.md`；後續可將各條目拆成任務後，同步更新此表與 `PROJECT_STATUS.md`。  
