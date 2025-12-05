# Pricing Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| PRC-01 | PriceRule 規格整理 | 梳理欄位、條件、優先順序、公式範本 | TODO |
| PRC-02 | PriceRule UI | Hub + List + Drawer（條件、公式、審批、Trace 預覽） | TODO |
| PRC-03 | Priority / Reorder API | 設計拖拉/批次調整優先權的服務 | TODO |
| PRC-04 | Pricing Preview / Trace | 定義 API、回傳欄位與 Trace Viewer；支援儲存/分享 | TODO |
| PRC-05 | PriceList Assignment / 匯入 | UI + API（匯入、批次套用客戶/通路、權限提示） | TODO |
| PRC-06 | ItemPrice Transaction Flow | 與 Product Transaction API 串接，決定一次寫入 Item + SKU + Price | TODO |
| PRC-07 | 與 Quotation / SO 整合 | 規劃報價/訂單呼叫 Pricing Preview、失敗提示、cache 策略 | TODO |
| PRC-08 | 權限 / 審批 | 定義角色、狀態、Workflow API、審批通知 | TODO |
| PRC-09 | Trace 儲存策略 / Log | 決定 Trace 保存期限、查詢 API、清理排程 | TODO |
| PRC-10 | Formula Editor / Trace UI | 依 ASCII/Wireframe 製作 Monaco + Trace Viewer 元件；規畫開發票 | TODO |
| PRC-11 | ItemPrice 匯入/匯出 | 設計 CSV 模板、錯誤回報、背景 Job API；規畫開發票 | TODO |
| PRC-12 | PriceList Workflow API | `submit/approve/history` 實作，與通知整合 | TODO |
| PRC-13 | 測試計畫 | PricingEngine 單元測試、整合測試、回歸腳本 | TODO |
| PRC-14 | Cache/Failover 策略 | 設計 Pricing 結果快取、fallback、熔斷 | TODO |
| PRC-15 | Observability | Trace/Preview 的 log 及 APM 指標定義 | TODO |

> 待開始開發時再補細節。
