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
3. **UI Wireframe**：整理庫存工作區的畫面/互動，確立與報價/訂單的串接方式（含匯入/掃碼流程、通知）。

## 客戶（Customer / CRM）
1. **Owner 篩選 / 權限**：定義 Owner scope、Async Select caching、Saved Filter 行為。
2. **Workspace 互動**：確認分頁、篩選、活動紀錄等組件，與報價共用 UI 元件。
3. **API 文檔化**：將 Customer / Contact Resource 的查詢、關聯、權限整理進 API spec。

## 採購（Purchase）
1. **PO/Rfq/Receipt 任務拆解**：依 `modules/purchase/implementation-plan.md` 定義任務工單。
2. **Workflow 與 Pipeline**：決定是否保留 Pipeline；若保留需補拖拉邏輯與權限。
3. **匯入/批次**：定義 CSV 欄位、匯入錯誤提示與批次審批 UI。

## 銷售訂單（Sales Order）
1. **Hub + Pipeline 規格**：完成進階篩選與 Pipeline 行為，確保與報價一致。
2. **整合 Delivery / Invoice**：Detail Panel 提供快速跳轉、顯示出貨收款狀態。
3. **匯入工具**：行項匯入、批次交期調整、批次送審。

## Product / Pricing
1. **Product ERD / Variant**：補充 Variant/PriceList 流程圖與 wireframe，明確 PriceList 是否獨立模組。
2. **Pricing Flow / Trace**：依 Spec 補齊 PriceRule/Trace/Transaction API，並定義 PriceList Workflow。
3. **Transaction API**：與 Product 整單儲存整合（Item + SKU + ItemPrice），確保報價/訂單可共用。

## Delivery / Inventory 交互
1. **交貨流程圖**：Delivery spec 需描述 Issue → Inventory Transaction → 通知序號。
2. **UI Wireframe**：補 Delivery Hub + 任務視圖 + 通知欄位。
3. **API Payload**：定義 `/api/deliveries/transactions` 與 Inventory 的整批格式。

## 待啟動模組
- 財務、售後等模組仍在排程，待 PM 確認優先順序後追加至本計劃。

> 若某模組進度有重大調整，請同步更新本表、`PROJECT_STATUS.md` 以及模組內的 README/plan。 
