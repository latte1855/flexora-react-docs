# React UI 開發檢查清單（Flexora ERP）

> 適用於在 JHipster 8.11 React 前端上擴充 ERP 模組（Quotation / Sales Order / 其他業務模組）。  
> 目的：確保新介面與既有 CRUD/導航共存、遵守組件策略並同步文件。  
> 相關檔案：`docs/ui/README.md`（目錄規劃）、`docs/ui/*-ui-wireframe.md`（各模組 wireframe）。

---

## 1. 架構與相容性

- [ ] **保留原生畫面**：JHipster 產生的 Entity List / Detail / CRUD 頁必須保持可用；若新增 pipeline、drawer、dashboard 等進階 UI，須透過額外 route/component 實現，而非刪除原頁面。
- [ ] **App Shell 不變**：沿用現有 `App.tsx`、`routes.tsx`、側邊/頂部導覽；如需新增 Module Rail 或快捷按鈕，採新增選項或插槽，不覆蓋既有功能。
- [ ] **Theme/套件一致**：優先使用專案內建的 Bootstrap 5（Materia theme）、Reactstrap、JHipster DataTable/Form、Redux Toolkit。引入新第三方套件前需評估：安全性（CSP、license）、Bundle 影響、i18n/RTL 支援、可與 Materia theme 共存。

## 2. UI/UX 行為

- [ ] **Workflow 守衛同步**：前端事件表單需檢查 line item、channel、reason、validUntil 等條件，與後端 `heuristicRequires*` 規則一致。
- [ ] **旗標欄位規則**：`deleted`、`enabled` 等欄位由後端控制；建立/更新表單不得將其當成必填輸入，軟刪除需呼叫專用 API（`:soft-delete`/`:restore`）。
- [ ] **Numbering 自動給號**：若表單未輸入單號，UI 應允許空值並顯示「由後端自動給號」提示；僅於 `_exists` API 回報衝突時提醒使用者。
- [ ] **共用元件**：地址快照（AddressSnapshot）、ExtAttr Panel、附件管理（DocumentLink）應使用 `shared/components` 內的共用版本，報價與訂單模組需呈現一致體驗。
- [ ] **Owner / DataScope Guard**：列表查詢、詳情頁按鈕、Workflow/Pipeline 操作需依 API 回傳的 `ownerId`、`canEdit`、`canTriggerEvent` 或 data-scope metadata 決定可見性；若後端拒絕操作需即時顯示 toast 並回復 UI。

## 3. 資料與 API 整合

- [ ] **RTK Query/Service 層**：新 API 呼叫統一透過 `app/shared/reducers` 或模組化的 RTK Query service，避免每個 component 直接 `fetch`。
- [ ] **型別對齊**：新增 DTO 型別（`app/shared/model`）需與後端 `*DTO` 同步，必要時以 `openapi-generator` 或 `ts-interface-builder` 重新產出。
- [ ] **錯誤顯示**：對於 `BadRequestAlertException`、`error.transitionMissing` 等後端訊息，前端需具備 i18n 映射與 toast/snackbar 呈現。

## 4. 文件與測試

- [ ] **Wireframe 更新**：UI 流程變更需同步 `docs/ui/quotation-ui-wireframe.md`，附最新畫面示意 / 操作流程。
- [ ] **Spec 更新**：相關模組規格（Phase 4/5 等）需新增「前端 UI 指引」章節，說明本次調整及相依 API。
- [ ] **React 測試**：新增或修改之關鍵 Component 需撰寫 Jest + RTL 測試；交互複雜時可加 Storybook或 Chromatic snapshot（若專案允許）。
- [ ] **AGENTS 同步**：若 UI 規範變更（例如新增共用元件策略、API 注解要求），同步更新 `AGENTS.MD` 供日後自動化代理參考。

---

> 本檢查清單需與每次前端 PR 一同檢閱；若有未完成項目，請在 PR 描述列出後續追蹤任務或 TODO，以免遺漏。
