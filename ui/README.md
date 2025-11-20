# 前端文件目錄規劃

> 目的：集中管理 Flexora ERP React UI 相關文件，讓 wireframe、設計規範、檢查清單與模組筆記有固定位置。此區資料與 `docs/api/*.md`、`AGENTS.MD` 互相參照，變更時務必同步更新。

## 1. 目錄結構

```
docs/ui/
├── README.md                  # 本文件，說明目錄用途與命名規則
├── quotation-ui-wireframe.md  # Phase 4 報價模組 UI 流程、wireframe
├── quotation-ui-implementation-plan.md # 報價 UI 實作藍圖（畫面列表、資料流程、任務）
├── react-ui-checklist.md      # React 開發檢查清單（所有模組共用）
├── sales-order-ui-notes.md    # (預留) Phase 5 Sales Order UI 維護筆記
└── components/
    └── README.md              # (預留) 共用元件設計與 Storybook 連結
```

- `quotation-ui-wireframe.md`：保留最新畫面雛型、操作流程、Pipeline 配色、Drawer 內容。後續若有 Sales Order 或其他模組，可複製同樣結構命名為 `<module>-ui-wireframe.md`。
- `react-ui-checklist.md`：跨模組檢查清單，涵蓋 theme、Workflow 守衛、DataScope Guard、共用元件、文件更新要求，每次前端 PR 需參照。
- `sales-order-ui-notes.md`、`components/README.md` 目前為預留檔名，方便之後新增檔案時遵循格式（若尚未使用，可先保留空檔或由後續任務補上）。

## 2. 命名與維護規則

1. **模組檔案命名**：`<phase>_<module>_ui_wireframe.md` 或 `<module>-ui-notes.md`。必要時可附帶版本，例如 `phase5_sales-order_ui_wireframe.md`。若需任務藍圖，可使用 `<module>-ui-implementation-plan.md`。
2. **共用資料夾**：`components/` 儲存共用元件規格、範例 props、UI token；`patterns/`（需要時再建立）可記錄操作流程或 UX 規則。
3. **文件更新流程**：新增/調整 UI 行為時，需同時更新對應模組的 wireframe 與 `react-ui-checklist.md`（若新增檢查項目），並在相關 spec (`docs/api/phase*.md`) 內引用。
4. **與 AGENTS.MD 對應**：本目錄定位為前端 UI 權威參考，AGENTS 會引用本檔；若結構有變更，記得同步更新 AGENTS。

## 3. 後續擴充建議

- `docs/ui/sales-order-ui-wireframe.md`：銷售訂單 Phase 5 UI 設計完成後，將內容移至獨立檔案。
- `docs/ui/components/address-card.md` 等檔案：紀錄共用元件的屬性、事件與 Storybook 連結，方便其他模組沿用。
- `docs/ui/themes.md`：若未來需要自訂 Materia theme token 或顏色表，可在此集中說明。

> 若需加入額外資源（Figma 連結、Screenshots），請在各檔案的附錄記錄 URL 或路徑，保持 repo 乾淨。\*\*\*
