# Flexora 系統開發進度總覽

> 最後更新：2025-12-03  
> 目的：集中記錄各模組（前/後端）的完成度、主要里程碑與下一步。每次完成功能或新增 TODO 時請同步更新，避免資訊分散在各 spec。

## 概覽表

| 模組 | 前端狀態 | 後端狀態 | 下一步 / TODO | 參考文件 |
| --- | --- | --- | --- | --- |
| 報價（Phase 4） | ✅ Quick Create / Quote Editor / Workflow Tab 改用 `/api/quotations/transactions`；Timeline Segment + 事件 chips 已上線；Workflow Drawer 仍使用 `triggerEvent`（有備註）。 | ✅ Transaction API / 交易落地完成；工作流事件維持 `triggerEvent`；QuotationThread preset enum 已初步實作。 | • 行項表格/匯入、Preview 行明細與 PDF<br>• Workflow Drawer 若需整批儲存再改造<br>• Saved Filters / Hub UX（篩選收藏、owner scope 標示） | [`modules/quotation/README.md`](modules/quotation/README.md) |
| 庫存（Phase 3） | ⏳ Workspace/Drawer/Wireframe 草稿完成；欄位驗證、匯入/掃碼與通知需求已補在 spec。 | ⏳ Replenishment / Reservation / Transaction API 規格撰寫中。 | • 根據 ASCII Wireframe 推進 UI<br>• 依 spec 實作 Reservation/Transaction DTO 與排程 | [`modules/inventory/README.md`](modules/inventory/README.md) |
| 採購（Purchase） | ⏳ UI Spec 已含欄位驗證/匯入與 Pipeline 設計；尚未實作。 | ⏳ Domain/Workflow 梳理中。 | • 拆分 PO/Rfq/Receipt 任務<br>• 與 Inventory/Billing 整合、批次審批/匯入流程 | [`modules/purchase/README.md`](modules/purchase/README.md) |
| 銷售訂單（Sales Order） | ⏳ Workspace/Drawer/欄位驗證與 Pipeline 規則已補；匯入範本待 UI(flow) 實作。 | ⏳ Transaction API/Workflow 尚未建置。 | • Hub + Pipeline 設計稿<br>• 匯入行項/交期批次調整、Delivery/Invoice 快捷 | [`modules/sales-order/README.md`](modules/sales-order/README.md) |
| 客戶/聯絡人模組 | ✅ Workspace/UI spec 已補欄位驗證、Filter、匯入匯出流程；Async Select 行為記錄完成。 | ✅ `CustomerResource` / `ContactResource` 具 lookup 與 Data Scope；Owner scope 控制待加。 | • Lazy load / caching<br>• Workflow / 黑名單需求 | [`modules/customer/README.md`](modules/customer/README.md) |
| Pricing / Product | ⏳ Product ERD/流程、Pricing Flow/Trace 文件已補；ASCII Wireframe 與 Transaction 驗證完成。 | ⏳ PriceRule/PriceList workflow 與 Transaction API 規劃中。 | • 與後端確認 Transaction 欄位/審批並拆工單<br>• Pricing 端實作 Formula Editor、Trace Viewer、`item-prices` 匯入/匯出與 workflow API | [`modules/pricing/README.md`](modules/pricing/README.md) |
| 其他模組 | ⏸ 尚未開發 | ⏸ 尚未開發 | • 依專案時程逐步展開 | `docs/architecture/OVERVIEW.md` |

## 使用方式

1. 每次完成功能或新增 TODO，請更新本表的狀態欄與「下一步」欄，並加上日期或註解，避免遺漏。
2. 如需追蹤更細的報價模組事項，請參考 `ui-spec` 與 `quotation-ui-implementation-plan`，兩者已同步維護清單。
3. 若後續模組（例如財務、售後）啟動，請在本檔新增列並連結對應 spec/README。 

> 建議每次 PR 都檢查是否需要調整此表，確保團隊可一眼掌握整體進度。 
