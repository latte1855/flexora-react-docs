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

> **Pipeline**：SO 模組沿用報價 Pipeline 規則（`DRAFT → IN_REVIEW → APPROVED → FULFILLED`）。拖曳僅允許下一階狀態並需權限，`APPROVED` → `FULFILLED` 需確認 Delivery/Invoice 完成，取消 Pipeline 時請同步此檔與 UI spec。
