# Purchase Module

涵蓋採購需求：Rfq → Purchase Order → 收貨(Purchase Receipt) → 應付流程。  
目前僅建立骨架，詳細規格仍位於原有 `flexora-purchase-management-spec.md` 等檔案，後續會逐步搬遷。

| 文件 | 說明 |
| --- | --- |
| [spec.md](./spec.md) | 功能 / 流程 / 狀態定義（RFQ → PO → Receipt/Return） |
| [ui-spec.md](./ui-spec.md) | 採購 Workspace、PO / Receipt Drawer、Workflow 操作 |
| [api-spec.md](./api-spec.md) | Rfq/Quote/PO/Receipt/Return REST 與 Transaction API 草案 |
| [implementation-plan.md](./implementation-plan.md) | 任務拆解與進度（Workspace、Drawer、整合等） |

相關舊文件：

- [`docs/flexora-purchase-management-spec.md`](../../flexora-purchase-management-spec.md)
- `docs/specs/phase3-inventory/*` 中的採購補貨段落
