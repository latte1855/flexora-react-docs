# Legacy API Specs from `flexora-react/docs/api`

此索引列出原始 `flexora-react/docs/api` 中的彙整文件與各 Phase API 草案，已全數備份在 `docs/migration/flexora-react/api/`。若要對照舊版本或復刻其中的 API 範例，可在該資料夾中尋找對應章節。

| 檔案 | 說明 | 對應位置 |
| --- | --- | --- |
| `phase0-system-foundation.md` | System Foundation API 風格與範例 | 對應 `docs/specs/phase0-system-foundation/api-spec.md` |
| `phase1-product-pricing.md` / `phase1_product_pricing_202510221657.md` | Product/Pricing API 概念 | 目前尚未拆分細節，可依 `phase1` spec/DTO 補充 |
| `phase2-crm-foundation-spec_20251029.md` | CRM API（Campaign/Lead/Opportunity） | 對應 `docs/specs/crm/legacy-flexora-campaign-lead-opportunity.md` |
| `phase3_im_api.md` | Inventory Management REST API（包含 Cost Layer、Stock） | 可與 `docs/specs/phase3-inventory/api-spec` 同步 |
| `phase4_quotation_spec.md` | Quotation API / Workflow（Thread/Revision/Workflow Info） | 與 `docs/specs/phase4-quotation/api-spec.md` 整合中 |
| `phase5_sales_order_spec.md` | Sales Order API（SO/Create/Workflow） | 對應 `docs/specs/sales-order/legacy-...` |
| `Flexora ERP API 規格（彙整版）` | 全域 API 風格與 BadRequest 範例 | 可當做 narrative summary |
| `CONVENTIONS`, `MODULES`, `OPENAPI`, `README`, `final` | 記錄 API 編寫 conventions、模組架構、OpenAPI 程式產出方式、最終目錄 | 參考 `docs/README.md`、`docs/architecture` |

## 建議

1. 若某 phase API 需要細節（欄位、Request body 範例），可直接打開對應 legacy file 並依 `phaseX` spec 撰寫新版 MD。
2. 對照 `flexora-react/src/main/java/.../web/rest/*.java` 確認 DTO field 與 sample request 是否一致。
