# Sales Order (SO) Module

負責銷售訂單、交期管理、與報價/出貨的串接。現階段為骨架，詳細內容仍在 `flexora-salesorder-management-spec.md` 等文件內。

| 文件 | 說明 |
| --- | --- |
| [spec.md](./spec.md) | SO 流程、狀態、資料模型 |
| [ui-spec.md](./ui-spec.md) | Sales Order Workspace、Detail Drawer |
| [api-spec.md](./api-spec.md) | SO CRUD、Workflow、Link API |
| [implementation-plan.md](./implementation-plan.md) | 任務拆解與進度 |

參考資料：

- [`docs/flexora-salesorder-management-spec.md`](../../flexora-salesorder-management-spec.md)
- 報價模組：`docs/modules/quotation/README.md`

> **Pipeline 決議**：是否沿用 Pipeline 視圖仍待 PM 確認。若最終僅保留 List/Detail，請在 `ui-spec` 及此 README 註記；若採 Pipeline，需定義狀態順序、拖拉權限與通知行為。
