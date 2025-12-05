# Inventory Module – Implementation Plan

| ID | 任務 | 說明 / 備註 | 狀態 |
| --- | --- | --- | --- |
| INV-01 | Inventory Workspace | Hub + List + Pipeline（倉庫 / Bin / 狀態切換）＋ Detail Panel | TODO |
| INV-02 | Stock Ledger & Snapshot | UI 與 API 統一定義（Qty On Hand / Reserved / Available） | TODO |
| INV-03 | Adjustment / Transfer Flow | Drawer + Transaction API，支援附件 / 審批 | TODO |
| INV-04 | Receiving / Put-away 結點 | 與 Purchase / Delivery 模組串接的入庫流程 | TODO |
| INV-05 | Reservation / Allocation | Sales Order 保留貨量邏輯、批次釋放、警示 | TODO |
| INV-06 | Lookup / Async Select | `GET /api/inventories/lookup` 提供 SKU + 倉別 + 數量資訊 | TODO |
| INV-07 | Batch Job / Replenishment | 補貨策略、Min/Max 設定、排程產生任務 | TODO |
| INV-08 | 匯入 / 導入工具 | 初始盤點、調整 CSV、與 ERP 同步策略、掃碼 | TODO |
| INV-09 | Notification / Webhook | 補貨提醒、保留到期、WMS 回傳通知 | TODO |
| INV-10 | 權限 / 資料範圍 | 倉庫層級權限、日誌追蹤、活動記錄 | TODO |
| INV-11 | 測試計畫 | API / UI / 整合測試與模擬批次任務 | TODO |

> 相關舊資料可參考 `docs/phase3-inventory`；後續在 spec / ui-spec / api-spec 中細化欄位與流程，再回寫到此表。  
