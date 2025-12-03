# Flexora 系統開發進度總覽

> 最後更新：2025-12-03  
> 目的：集中記錄各模組（前/後端）的完成度、主要里程碑與下一步。每次完成功能或新增 TODO 時請同步更新，避免資訊分散在各 spec。

## 概覽表

| 模組 | 前端狀態 | 後端狀態 | 下一步 / TODO | 參考文件 |
| --- | --- | --- | --- | --- |
| 報價（Phase 4） | ✅ Quick Create / Quote Editor / Workflow Tab 改用 `/api/quotations/transactions`；Timeline Segment + 事件 chips 已上線；Workflow Drawer 仍使用 `triggerEvent`（有備註）。 | ✅ Transaction API / 交易落地完成；工作流事件維持 `triggerEvent`；QuotationThread preset enum 已初步實作。 | • 修正 Hub preset 切換造成的多次 API 呼叫<br>• 行項表格/匯入、Preview 行明細與 PDF<br>• Workflow Drawer 若需整批儲存再改造 | `docs/specs/phase4-quotation/ui-spec.md`<br>`docs/ui/quotation-ui-implementation-plan.md` |
| 庫存 / 採購（Phase 3） | ⏳ Wireframe 與規格撰寫中，尚未進入實作。 | ⏳ Replenishment/Reservation 規則撰寫中，尚未實作。 | • 補齊 `phase3-inventory` 規格<br>• 根據 spec 拆工單 | `docs/specs/phase3-inventory/*` |
| 客戶/聯絡人模組 | ✅ Async Select 元件已提供篩選與 debounced 查詢，於報價模組 reuse。 | ✅ `CustomerResource` / `ContactResource` lookup API 可依關鍵字過濾；尚需補 owner scope 控制。 | • 大量資料下的 lazy load / caching 機制<br>• Saved filter / 權限控管 | `docs/ui/customer/*` |
| 其他模組 | ⏸ 尚未開發 | ⏸ 尚未開發 | • 依專案時程逐步展開 | `docs/architecture/OVERVIEW.md` |

## 使用方式

1. 每次完成功能或新增 TODO，請更新本表的狀態欄與「下一步」欄，並加上日期或註解，避免遺漏。
2. 如需追蹤更細的報價模組事項，請參考 `ui-spec` 與 `quotation-ui-implementation-plan`，兩者已同步維護清單。
3. 若後續模組（例如財務、售後）啟動，請在本檔新增列並連結對應 spec/README。 

> 建議每次 PR 都檢查是否需要調整此表，確保團隊可一眼掌握整體進度。 
