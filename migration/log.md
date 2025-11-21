# Documentation Migration Log

Entries track each migration step, including source files in `flexora-react/docs`, destinations within this repository, and any follow-up actions needed.

## 2025-11-19
- Copied all files from `flexora-react/docs/` into `docs/migration/flexora-react/` as the canonical archive for this migration wave.
- Next: create per-module summaries/legacy notes in `docs/specs/` (Phase 3, 4, etc.) and link them to the archived sources.
- Added legacy summary documents in the target repo:
  - `docs/specs/phase4-quotation/legacy-flexora-quotation-management.md` and `docs/specs/phase3-inventory/legacy-flexora-inventory.md` to surface Sales/Inventory guidelines.
  - New per-module folders (`phase1-product-pricing`, `purchase-order`, `sales-order`, `delivery-note`, `crm`) with `legacy-*` notes pulling from the migration archive.
  - Copied the old UI docs into `docs/ui/` and documented the relocation in `docs/guides/legacy-ui-documentation.md`.
  - Archived API/component/security/domain summaries under `docs/api-spec/legacy-flexora-react-api-index.md`, `docs/architecture/legacy-inventory-domain-events.md`, and `docs/specs/phase0-system-foundation/{data-scope-guide,data-structure-versioning}.md`.
- Added `docs/specs/phase0-system-foundation/data-domain-guide.md` to capture the full Phase 0 API / Data Scope requirements (DoD, Querydsl filter, SystemSetting, Tax, Document) from the legacy `phase0-system-foundation.md`.
- Documented the Phase 3 & Phase 4 README files to note that the merged content derives from the legacy inventory/quotation specs and must stay aligned with the actual service/controller implementations in `flexora-react/src/main/java/com/asynctide/flexora/...`.
- Added Phase 1 module overview/references: `docs/specs/phase1-product-pricing/README.md`, introduced `item-schema.md` summarizing `Item`/`ItemSku` fields, and recorded module consolidation next steps as per the migration plan.
- Documented Phase 3 integration plan in `docs/specs/phase3-inventory/integration-plan.md` with step-by-step breakdown (costing, workflow, reservation, replenishment) aligned to the corresponding services/repositories.
- Added CostLayer+code reference section to `docs/specs/phase3-inventory/costing-rules.md`, noting `InventoryCostingService` / `InventoryTransactionService` / `CostLayerRepository` as the ultimate sources and specifying derived columns/error codes.
- Added new `docs/specs/phase3-inventory/workflow-spec.md` summarizing workflow transitions/guards with code references (`InventoryTransactionService`, `InventoryCostingService`, `InventoryReservationService`, `InventoryTransactionResource`, `InventoryReservationResource`).
- Documented reservation rules/ APIs in `docs/specs/phase3-inventory/reservation-rules.md` (from Inventory spec) and created `phase3-inventory/error-codes.md` with inventory-specific error keys tied to the corresponding services.
- Added replenishment spec `docs/specs/phase3-inventory/replenishment-rules.md` covering `InventoryReplenishmentService` logic, APIs, and data model references.
- Added Phase 4 integration plan in `docs/specs/phase4-quotation/integration-plan.md` outlining API/Workflow/Pricing/UI steps tied to `QuotationThreadService`, `QuotationRevisionService`, `QuotationPricingService`, and workflow resources.
- Added introductory `docs/specs/phase4-quotation/api-spec.md` summarizing key endpoints, DTO validation, guard error codes, workflow info payload, and pointing to exact resource/service files.
- Added new `docs/specs/phase4-quotation/workflow-guards.md` mapping each workflow action to the corresponding `QuotationWorkflowService` guard/errors, noting `quotation.*` error keys and referencing `QuotationPricingService`.
- Updated `docs/specs/phase4-quotation/pricing-rules.md` to include `QuotationPricingService`/`QuotationPricingValidator` references, formula/rounding notes, and error code mapping keyed to the actual service implementations.
- Added UI redesign plan `docs/ui/quotation-ui-redesign-plan.md` outlining the new List + Detail layout, summary guards, association cards, and integration with backend workflow/pricing APIs.
- Expanded `docs/ui/wireframes/quotation-detail-wireframe.md` to include Pipeline mode toggle, jump-to-page, and Detail pane wireframe (Summary/Associations/Workflow guard). Added `docs/specs/phase4-quotation/ui-spec.md` as a detailed UI spec referencing the wireframe, plus recorded notes in `docs/ui/revisions.md`.
- Added Drawer wireframe (`docs/ui/wireframes/quotation-drawer-wireframe.md`) per user preference for full-width list + slide-out detail, and updated plan/checklist accordingly.
- Added pipeline HTML prototype at `docs/ui/wireframes/quotation-pipeline-prototype.html` for drag-and-drop columns + drawer detail interactions.
- Added `docs/migration/flexora-react/ui/react-ui-checklist.md` with updated items (List/Pipeline toggle, Drawer detail, workflow guards, pricing/attachments).
- Implemented actual UI at `flexora-react-ui/src/views/sales/QuotesWorkspace.tsx` with List/Pipeline/Drawer layout and updated `/sales/quotes` route + UI spec reference.
- Added `docs/ecme-template-document/README.md` summarizing Ecme template guides, and updated `docs/AGENTS.md` to require reading此目錄 before UI development.

## 2025-11-20
- 串接正式登入：`flexora-react-ui` 以 `/api/authenticate` 為唯一入口，新增 `rememberMe` 選項並依狀態選擇將 JWT 存於 `sessionStorage`（預設）或 `localStorage`，同時統一 axios interceptor 的 Token 讀取邏輯。
- 登入後會呼叫 `/api/account` 同步帳號資訊，並透過 `WebsocketTrackerService` 連線後端 `/websocket/tracker` 回報使用者活動；登出時會清除 Token 與 websocket 連線。
- `docs/specs/phase0-system-foundation/security-and-auth.md` 已補充登入欄位、Token 儲存策略及 websocket 追蹤流程，確保文件與程式碼一致。

## 2025-11-21
- 新增 Vite 開發環境設定檔 `.env.development`，透過 `VITE_BACKEND_URL` 管控 API 與 Websocket 連線的後端位址，避免寫死 localhost。
- `WebsocketTrackerService` 依據 `VITE_BACKEND_URL` 或 `window.location.origin` 建構 tracker URL，確保 dev/prod 皆能連到對應的 `/websocket/tracker`。
