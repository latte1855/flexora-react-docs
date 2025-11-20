# Documentation Migration Plan

> 目標：將 `flexora-react/docs` 與 `flexora/docs` 緊密整合，內容以程式碼實作作為裁定依據，並把最終資料放在 `flexora/docs`。

## 1. 調整目標與原則
1. 以「內容與行為為主」：只要 code 還在用（如 Convert Drawer 稅額、Workflow resend），就把對應文本整理進 `flexora/docs`；非同步僅依 `git status`。
2. 按主題集中：
   * `docs/specs/phase4-quotation`: 報價 Workflow、Pricing、Convert、Guard、error codes。
   * `docs/specs/phase3-inventory`: Inventory/共用 domain 說明。
   * `docs/guides`: Ecme React UI template、Workflow usage hints、流程指南。
   * `docs/architecture`: 資料權限 / Workflow 架構文件。
3. 若 `flexora-react/docs` 只有稀疏筆記，可從中摘錄至新 doc（並加注來源），不再維持舊檔。

## 2. 目錄與分類
```
docs/
  specs/
    phase4-quotation/
    phase3-inventory/
  guides/
  architecture/
  migration/
```
- 每個子目錄須含 `README.md` 說明內容範圍、源文件、維護者、code reference。
- `migration/` 為本計畫紀錄，包含 log 與 checklist（本檔即 v1.0）。

## 3. 重整流程（內容為主）
1. 清點 `flexora-react/docs`:
   * `find flexora-react/docs -type f`，列出檔名 + 檔案用途。
   * 對照 `flexora/docs` 先前有哪些對應（如 `workflow-spec`、`pricing-rules`），標示「需整合」、「可存入 archive」、「需新建」。
2. 整合方式：
   * 若 `flexora/docs/<topic>` 已存在主檔，將 `flexora-react/docs` 的相關段落重寫進主檔並註記來源（e.g. `<!-- derived from flexora-react/docs/... -->`），同時刪除 react repo 原檔以避免混亂。
   * 若無主檔，則直接在 `flexora/docs/<topic>` 新建檔案（例如 `docs/guides/ecme-react-ui-template.md`），內容以目前 code 為依據，並在 doc 備註原始來源。
   * 內容重寫原則：以程式碼行為（Convert Drawer、timeline helper、pricing summary）為 anchor，確保 doc 直接映射到實作；必要時加入 `References` section 連到 `src/...`。
3. 整合後，在 `flexora-react/docs/README` 顯示「最新 docs 請參考 `../docs/...`」，並可把舊檔案移進 `flexora-react/docs/archive` 作 history。

## 4. 驗證與交付
- 每輪整合後，可 rerun `npm run lint -- --fix` 與 `mvn test`（如 repo 有相關配置）以防格式/引用錯誤。
- 在 `docs/specs/README.md` 追加 table，列出整合後主要文檔、最後更新時間與維護狀態。
- 若 spec 涉及 error codes 或 guard，需同步更新 `docs/specs/phase4-quotation/error-codes.md` 與 `docs/specs/error-codes.md`。

## 5. 後續維護
- `docs/migration/log.md` 可記錄後續每次整合的內容與負責人（本檔即 v1.0）。
- 可考慮 `docs/scripts/doc-collect.sh`，自動把 `flexora-react/docs` relevant 檔案 extract 到 `flexora/docs` 並輸出 log。
- 所有引用 `docs/specs/...` 的 repo 請在 README 明確標記新位置。

## 6. 目前待辦
- [ ] 盤點 `flexora-react/docs` 檔案並更新對應目的地。
- [ ] 整合 `pricing-rules.md`、`workflow-spec.md`、`ecme-react-ui-template.md` 等內容到 `docs/specs/phase4-quotation`/`docs/guides`。
- [ ] 補 `docs/specs/README.md` 與 `flexora-react/docs/README.md` 的路徑指引。
- [ ] 記錄每輪整合於 `docs/migration/log.md`。

## 7. Next actions before merge
1. 完整盤點 `flexora-react/docs` 每一份檔案，確認目前對應的 `docs/` 目錄（Phase 4 workflow/pricing、Phase 3 inventory、guides、architecture 等）。
2. 以程式碼為最終權威，重寫/整合內容進 `flexora/docs`，並註明原始來源（如需，先加 `<!-- derived from ... -->` 或 References）。
3. 若該主題已有主檔（如 `pricing-rules.md`、`workflow-spec.md`、`ecme-react-ui-template.md` 等），就以該檔為整合集點；若尚無，則新增 `docs/` 下符合分類的新檔並記錄維護者、code reference。

## 8. Code alignment and verification
- 整合後要比對 `flexora-react` / `flexora-react-ui` 的實際 code（Controller/Service/API/React components）與 docs 內容，修正任何明顯不符的錯誤。
- 若遇到無法直接從程式碼判斷的衝突，列出待用戶判斷的條目。
- 持續把每次整合的結果註記於 `docs/migration/log.md`，包括移動的檔案、參考代碼路徑與驗證狀態。

## 9. Modules consolidation sequence
1. **Phase 1 — Product & Pricing**：將 `flexora-react/docs/flexora-product-management-spec.md` 的欄位表、ExtAttr、PriceList/BOM/Routing 計畫分割到 `docs/specs/phase1-product-pricing/<topic>.md`（例如 `item-model.md`, `pricing-system.md`），每一章都附上 code reference（`ItemResource`, `PriceListService`, `BomService`, `RoutingService`）。優先完成 `Item`/`SKU` 的欄位表與 API 行為，然後再補 `ExtAttr` 与 `PriceList` 設定。
2. **Phase 3 — Inventory Management**：從 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md` 拆出 `costing-rules`, `workflow-spec`, `reservation-rules`, `replenishment-rules` 的細節，採用 `InventoryTransactionService`, `CostLayerRepository`, `InventoryCostingService` 等實作作為權威。每個章節末尾加註 `Derived from ...`.
3. **Phase 4 — Quotation**：將 `flexora-quotation-management-spec.md` 內容拆入 `docs/specs/phase4-quotation/api-spec.md`、`workflow-spec.md`、`pricing-rules.md`、`error-codes.md` 等，並用 `QuotationThreadService`, `QuotationRevisionService`, `QuotationWorkflowService`, `QuotationThreadResource`, `QuotationRevisionResource` 做驗證；必要時直接抓取 `flexora-react/src/main/java/...` 的 Guard/DTO 實作補到文件中。
4. **Phase 2 / CRM**：擇一時機把 `docs/migration/flexora-react/flexora-campaign-lead-opportunity-spec.md` 内容拆進 `docs/specs/crm/` 各檔（Campaign/Lead/Opportunity），並驗證 `com.asynctide.flexora/domain/crm` 的欄位/API。
5. 每完成一個 module，立即在對應 README 中註記原始來源與程式碼 reference，並在 `docs/migration/log.md` 增加條目描述完成細節與代碼檢查結果。
