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

> **Pipeline 說明**：PM 尚在評估是否採用 Pipeline 視圖。若未啟用，本模組僅提供 List + Detail；若啟用則狀態將使用 `DRAFT → RFQ → IN_REVIEW → APPROVED → RECEIVING → CLOSED`，並需實作拖拉規則。請在規格確認後同步本檔與 UI spec。
