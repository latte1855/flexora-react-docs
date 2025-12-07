# Flexora ERP — Decision Log（決策紀錄）

本文件用於記錄 Flexora ERP 開發過程中的 **重大架構、技術、流程、資料模型、前後端設計** 等重要決策。

> 所有重大決策都應在此留存紀錄，並視情況建立對應的 ADR 檔案  
> （`/docs/decisions/adr/ADR-xxxx-title.md`）。  
> 若決策會影響 specs / API / front-end / data model，也需同步更新其他文件。

---

# 0. 決策紀錄原則

- 本文件採「倒序方式（最新在最前）」。
- 每條紀錄必須包含：
  - 日期
  - 決策標題
  - 決策摘要
  - 理由 / 背景
  - 影響範圍（後端、前端、架構、DB、文件…）
  - 是否需要 ADR（若需要則附連結）

---

# 1. 決策列表（Decision Entries）

以下為目前已確認的初始決策：

---

## 2025-12-07 — 模組 UI Component 架構設計

**摘要：**  
從報價單模組解構可複用 UI Component，建立 `components/workspace/` 目錄，採用**配置驅動 + 組合**混合設計模式。

**決策內容：**
1. **驗證策略**：先用 Sales Order 模組驗證新元件，不先動報價單；驗證完成後再重構報價單
2. **元件目錄**：放置於 `components/workspace/`
3. **設計模式**：配置物件控制通用行為 + React children 組合保留彈性
4. **文件要求**：需建立元件使用指南供其他 Agent 使用

**背景：**
- 報價單模組已實作完整 UI 結構（Workspace + Sidebar + List/Pipeline + Detail）
- 後續模組（Sales Order、Customer 等）需複用相同結構
- 報價單有版本管理（Thread + Revision）為專有邏輯，不適合泛化

**影響：**
- 新增 `flexora-react-ui/src/components/workspace/` 目錄
- 新增 `docs/ui/components/workspace-components.md` 元件使用指南
- 後續模組開發應參考此指南

**相關文件：**
- 分析文件：`docs/modules/component-architecture/module-ui-component-analysis.md`
- 元件使用指南：`docs/ui/components/workspace-components.md`

**ADR：**  
*不需 ADR（屬 UI 元件組織方式，可漸進調整）*

**補充決策（Phase 1-4 完成後）：**

1. **詳情頁寬度規範**
   - 右側 sidebar 必須使用 `w-full lg:w-80`
   - 與報價單詳情頁保持一致，避免左欄超出容器寬度

2. **Overlay Header 按鈕標準**
   - 必須包含：上一筆、下一筆、快速建立、關閉
   - 上下頁按鈕使用 `variant="plain"` + disabled 狀態控制
   - 快速建立使用 `variant="solid"` + 黑底白字樣式

3. **快速操作區塊樣式規範**
   - 標題：`text-xs text-gray-500 mb-2`
   - 按鈕容器：`flex flex-wrap gap-2`
   - 按鈕：`size="sm" variant="default"`

4. **擴展性設計**
   - ✅ Tabs 可由各模組自由增加，不需修改元件
   - ✅ Sidebar 可由各模組新增自訂卡片，不需修改元件
   - ❌ 關閉按鈕不應放在 sidebar 底部（應在 Overlay header）

**驗證結果：**
- Phase 1-4 已在 Sales Order 模組驗證完成
- Commit: `d25b701` - feat(workspace): add reusable workspace components (Phase 1-4)
- 14 個檔案，1544 行新增



---

## 2025-02 — 採用 Code-first 而非 API-first（OpenAPI Generator 停用）

**摘要：**  
ERP 後端採用 JHipster 8.11.0 的 **Code-first 模式**，即 Controller / DTO / Service 為真實 API 定義，OpenAPI 僅做為自動產生的文件。

**原因：**
- API-first 會降低開發速度並造成文件雙軌（YAML 與程式碼分離）
- JHipster 本身對 Code-first 支援最佳（含 DTO、Pageable、Error 物件）
- Swagger UI 與 `/v3/api-docs` 已能反射出完整 API

**影響：**
- 停用 openapi-generator（不再產生 Java/TS Stubs）
- 所有 API 調整 → 以 Controller 為準 → 再回寫文件
- OpenAPI YAML 僅作快照 → 放在 `/docs/api-spec/openapi/`

**ADR：**  
*不需 ADR（屬高層架構風格、已確立不可逆）*

---

## 2025-02 — 前端正式 UI 採用 Ecme React Admin Template（非 JHipster React）

**摘要：**  
正式 ERP 前端採用 **Ecme React Tailwind Admin Template**，  
JHipster 自動產生的 React 僅作為 **後端測試 UI**。

**原因：**
- Ecme 提供更符合企業 ERP 的 UI 結構、元件、風格
- Tailwind 更適合長期擴充
- JHipster React UI 適合作為 CRUD 驗證工具，不適合作為產品前端

**影響：**
- 正式 UI 放在 `flexora-react-ui/`  
- JHipster UI（flexora-react 內建 React）不得修改產品功能  
- 若 API 變更 → 需同步更新 specs 與正式前端

**ADR：**  
*不需 ADR（產品前端策略）*

---

## 2025-02 — 文件獨立成專用 Repo（flexora/docs）

**摘要：**  
原本散落在後端 repo 的規格與文件移至獨立 repo：`flexora/docs`。

**原因：**
- 前後端需同一份文檔來源（SSOT）
- 與程式碼解耦 → 文件變更不影響 build
- 可跨專案（backend / frontend）共用同一套規格

**影響：**
- 部分後端 docs 資料夾被移除 → 改放此 repo
- Codex / Agent 在 workspace 中可同時讀取 docs / backend / frontend
- 所有功能、架構、API 變更 → 必須回寫 `docs/`

**ADR：**  
*建議未來加入 ADR（包含文件維護策略與版本規範）*

---

# 2. 未來預計會加入的決策主題（草稿）

以下為在開發中預期會形成正式決策的主題，可視開發情況逐步補上：

- 狀態機（Workflow）採用固定 Enum + 資料表（StatusDef）模式  
- ERP 模組邊界（Product、Inventory、Quotation、Sales Order…）  
- ExtAttr（自訂欄位）策略  
- 驗證規則處理方式（後端 Bean Validation 為主）  
- API 錯誤回傳格式統一（BadRequestAlert + ErrorVM 標準）  
- 資料模型命名原則（xxxNo、xxxCode、xxxId…）  
- 前端狀態管理（Redux Toolkit? Zustand? TanStack Query?）  
- LLM / Codex / Agent 介入哪些階段（文件同步、CRUD 自動產生…）  

這些主題應隨時間逐一寫入正式決策段落。

---

# 3. ADR（Architecture Decision Records）

若某個決策需要更深入的背景、比較、取捨、替代方案，  
請在 `/docs/decisions/adr/` 新增 ADR，例如：

```

ADR-0001-code-first-api.md
ADR-0002-form-validation-strategy.md
ADR-0003-status-machine-design.md

```

ADR 文件通常包含：

- 背景  
- 問題  
- 決策  
- 後果 / 影響  
- 替代方案  

---

# 4. 貢獻規則（簡要）

1. 新增重大決策 → 必須加入本文件  
2. 若影響架構 → 建議同時建立 ADR  
3. 若影響功能 → 同步更新 `/docs/specs/`  
4. 若影響 API → 同步更新 `/docs/api-spec/`  
5. Commit message 建議使用英文，例如：  
   `docs(decision): adopt code-first api strategy`

---

# 5. 結語

本 Decision Log 是 ERP 開發過程中「所有不可逆決策」的紀錄點。  
它能讓整個團隊（含後端、前端、文件、Agent）在同一基準上協作，  
並確保系統架構的演進可被追蹤、可被理解、可被解釋。

