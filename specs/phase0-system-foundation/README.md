# Phase 0 — System Foundation（系統基礎模組規格）

本文件描述 Flexora ERP 在 **Phase 0：System Foundation** 的需求與範圍，  
包含使用者與權限、Owner/Team 架構、系統共用元件（編號、錯誤格式、審計欄位…）等基礎能力。

> Phase 0 是之後所有模組（Product / Inventory / Quotation / Sales / Purchase…）的共同基礎。  
> 任何變更 Phase 0 的規則，都可能影響整個 ERP。

---

## 1. 範圍（Scope）

Phase 0 聚焦在「系統基礎」與「共用能力」，暫不處理複雜業務流程。

### 1.1 功能範圍

- 使用者與身份
  - User / Authority（Role）
  - 帳號啟用 / 停用
  - 基本登入 / 登出流程（由 JHipster 驗證機制提供）
- 組織結構與資料擁有者
  - Owner（公司 / 營運實體）
  - Department / Team（未來可擴充）
  - 後端資料實體與 Owner 關聯規則（每筆資料應屬於某 Owner）
- 系統共用機制
  - 編號/流水號服務（Numbering Service）
  - 全域錯誤回傳格式（BadRequestAlert、ErrorVM）
  - 共用審計欄位（createdBy / createdDate / lastModifiedBy / lastModifiedDate）
  - 軟刪除欄位（deleted / deletedAt / deletedBy）的統一規則
- 基礎設定（Configuration）
  - 系統參數（未來）
  - 時區 / 語系 / i18n 基礎（由 JHipster 支援）

### 1.2 不在本階段範圍內（Out of Scope）

- 任何具體業務流程（報價、銷售、庫存、採購…）
- 工作流程 / 狀態機的詳細實作（見 `/docs/architecture/WORKFLOW.md`）
- 複雜權限模型（DataScope、Owner/Department/Team 細緻資料權限）  
  > 🔸目前僅規劃 Owner / Team 結構，真正的資料權限策略可在後續 Phase 補上。

---

## 2. 相關專案位置

- 後端程式碼：
  - `../flexora-react/`
- 正式前端：
  - `../flexora-react-ui/`
- JHipster 內建前端（僅供測試與參考）：
  - `../flexora-react/src/main/webapp/`（實際路徑依專案而定）

---

## 3. 預期成果（Deliverables）

Phase 0 完成後，應達成：

1. 有一套可以正常登入 / 登出的後端系統。
2. 每一筆核心資料實體，都具備：
   - 審計欄位（建立/修改人與時間）
   - 軟刪除欄位（deleted / deletedAt / deletedBy）——若適用。
3. 至少完成：
   - User / Authority 設定與管理 API。
   - Owner 的資料結構與與主要業務實體的關聯方式（規格層次即可）。
   - Numbering Service 的設計原則（不一定要全部實作，但需有 spec）。
4. 全域錯誤格式一致：
   - Controller 透過 `BadRequestAlertException` 等機制回傳一致的錯誤格式。
   - 前後端可依錯誤碼（key）作 i18n 顯示。

---

## 4. 文件結構（Phase 0 內部）

本資料夾預計包含下列文件（可隨時擴充）：

```text
specs/phase0-system-foundation/
  README.md                     ← 本文件（Phase 0 總覽）
  domain-overview.md            ← Domain / Entity 概觀與欄位摘要
  api-spec.md                   ← Phase 0 相關 API 行為說明
  validation-and-rules.md       ← 驗證規則與防呆邏輯
  numbering-spec.md             ← 編號/流水號策略與實作規範
  error-handling-spec.md        ← 錯誤格式 / 統一回傳規格
  security-and-auth.md          ← 登入 / 權限 / Owner 關聯規範（簡要）
  test-strategy.md              ← 測試策略（單元 / 整合 / E2E 對應）
```

> ⚠ 一開始可以只有 `README.md`，其他檔案可視實作進度漸進式補齊。

---

## 5. 實作原則（Implementation Principles）

1. **以 JHipster 產生的基礎為起點，但不受限於預設模板**

   * 可以調整 Entity 與 API，但需更新本規格與 `docs/architecture/*`。

2. **所有共用能力都應有文件**

   * 例如 Numbering Service 的產號規則、錯誤格式對照表，都應在本 Phase 文件下有明確描述。

3. **與後續模組保持前向相容**

   * Phase 0 的設計會被後續模組大量依賴，須謹慎評估變更。

---

## 6. 變更控制（Change Control）

任何對 Phase 0 的變更必須：

1. 在 Git 中建立對應的變更記錄（commit message 建議前綴：`phase0:`）。
2. 若為重大架構 / 行為變更：

   * 更新 `../decisions/DECISION_LOG.md`。
   * 需要時新增 ADR 檔案。
3. 同步更新：

   * 本目錄中的對應文件（例如 `numbering-spec.md`、`error-handling-spec.md`）。
   * 若影響其他 Phase，須在對應 Phase 的 spec 中註明依賴。

---

## 7. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                            |
| ---- | ---------- | ----- | ----------------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 0 規格總覽初稿（README 骨架）。 |

## 8. 進一步參考

- `data-domain-guide.md`：彙整 Legacy `docs/migration/flexora-react/api/phase0-system-foundation.md` 的可交付條件、資料權限架構、系統設定、稅務與文件管理實作要點與程式路徑，建議以此為落地依據，之後再依步驟拆成更細的 spec（Owner/Document/Tax/Setting 專文）。
