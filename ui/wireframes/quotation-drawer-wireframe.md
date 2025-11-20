# Wireframe — Quotation Detail Drawer

> 為了避免右側固定欄位被 list 表格擠壓（字體需要放大給年長使用者），改採 Drawer 顯示詳細資料。列表畫面可維持全寬，點擊 row 後再滑出 Drawer 呈現 Summary & Associations。

```markdown
--------------------------------------------------------
| Quotes List (Full width table)                       |
--------------------------------------------------------

Row selected:
- Table area 保持 100% 寬度（避免 list欄位被擠），highlight 選取 row。
- 點擊 row 時呼叫 `/workflow-info` / `/api/quotation-revisions/{id}` 取得 detail，右側 Drawer 開啟。

[ Drawer Panel (slide-in from right, width ~35%) ]
--------------------------------------------------------
| Header: QTN-2025-0001  [Close X]                     |
| Tabs: Summary | Lines | Workflow | Attachments       |
|------------------------------------------------------|

Summary tab layout:
--------------------------------------------------------
| Quote Summary Card                                  |
| Thread No / Customer / Owner / Status / Valid Until |
| Total Amount / Margin                               |
|------------------------------------------------------|
| Workflow Guard Card (list guards + action buttons)  |
|------------------------------------------------------|
| Associations (accordion)                            |
|  - Contacts (top 3 + View All link)                 |
|  - Sales Orders (top 2 + Create SO button)          |
|  - Delivery Notes (top 2 + View timeline)           |
|  - Documents (list + Upload button)                 |
--------------------------------------------------------

Drawer actions:
- Close icon or Escape key = 關閉 Drawer（Table 回到全寬）。
- Buttons (Submit/Approve/Cancel...) 在 Drawer 內直接呼叫 workflow API。
- 字體預設 14px~16px，保證 readability。
```
