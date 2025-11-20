# Legacy React UI Notes

將 `docs/migration/flexora-react/ui` 的 README、wireframe、implementation plan 以及 checklist 轉移到 `docs/ui/`，本檔作為總覽與延續指引。

## `docs/ui` 內容

- `README.md`：說明目錄結構與命名原則，指出 `quotation-ui-wireframe.md`、`quotation-ui-implementation-plan.md`、`react-ui-checklist.md`、`sales-order-ui-notes.md` 與 `components/README.md` 的用途。
- `quotation-ui-wireframe.md`：描述 Phase 4 報價 UI 的 wireframe、管道圖、Workflow Drawer 等，可作為正式 `flexora-react-ui` 的設計依據。
- `quotation-ui-implementation-plan.md`：列出報價 UI 的畫面列表、資料流程、拆分任務與前端資料依賴，可用於部署 backlog。
- `react-ui-checklist.md`：共用 React 檢查清單，含 theme、data scope、Workflow guard、doc updates，PR 時請逐項核對。
- `sales-order-ui-notes.md`：預留 Phase 5 Sales Order UI 筆記，當模組啟動時可延續相同結構。

## 推薦做法

1. 變更 UI 行為時，同步更新該模組的 wireframe/implementation-plan/checklist 與 `docs/specs/<module>/ui-spec`。
2. 若需補充共用元件設計，可在 `docs/ui/components/README.md` 中加入 prop 表、story 或 Figma 連結。
3. 以此 summary 檔作為對 UI 團隊溝通的入口，並在 `docs/AGENTS.md` 和 `docs/specs` 引用本 guide。
