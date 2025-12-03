# 全模組高層計劃（Draft）

> 最後更新：2025-12-03  
> 目的：彙總各模組下一階段的主要目標，供 PM / 架構師 / 實作人員快速掌握。  
> 詳細任務請參考各模組目錄下的 `implementation-plan.md`。

## 報價（Quotation / Phase 4）
1. **Hub 篩選穩定化**：調整 preset 與進階篩選的 state/URL guard，避免 VIP→ALL 重複 API。
2. **行項表格重構**：導入表格視圖、匯入與行折扣/稅別提示，並提供預覽行明細/PDF。
3. **Workflow Drawer / Timeline**：若業主要求整批儲存，再評估改用 transaction API；同時補齊事件分類/流程圖互動。
4. **文件搬遷**：Phase 4 舊文件逐步移至 `modules/quotation/`。

## 庫存（Inventory / Phase 3）
1. **規格補齊**：完成 Replenishment / Reservation / Costing 詳細規格，定義 MVP 範圍。
2. **API 設計**：產出補貨、保留、批次作業 API 草圖，明確 DTO 與 Job 流程。
3. **UI Wireframe**：整理庫存工作區的畫面/互動，確立與報價/訂單的串接方式。

## 客戶（Customer / CRM）
1. **Owner 篩選 / 權限**：定義 Owner scope、Async Select caching、Saved Filter 行為。
2. **Workspace 互動**：確認分頁、篩選、活動紀錄等組件，與報價共用 UI 元件。
3. **API 文檔化**：將 Customer / Contact Resource 的查詢、關聯、權限整理進 API spec。

## 待啟動模組
- 採購、銷售訂單、財務等模組仍在排程，待 PM 確認優先順序後追加至本計劃。

> 若某模組進度有重大調整，請同步更新本表、`PROJECT_STATUS.md` 以及模組內的 README/plan。 
