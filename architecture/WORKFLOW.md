# Flexora ERP — 工作流程與狀態機設計（Workflow & State Model）

本文件說明 Flexora ERP 中 **工作流程（Workflow）與狀態（Status）** 的設計原則、實作模式、  
以及前後端協作方式（含 Workflow Drawer / Guard Rule）。

> 本文件屬「架構級」說明，細節由各模組在 `/docs/specs/<module>/` 中補充。

---

# 1. 設計目標

Flexora ERP 的 Workflow 設計目標：

1. **狀態清楚、可追蹤**：每筆單據的生命週期需明確定義（例如：Draft → Submitted → Approved → Closed）。
2. **防呆完整**：不允許非法狀態轉換（例如：已作廢不能再修改）。
3. **前後端一致**：前端 Workflow UI（例如 Drawer / Button 狀態）與後端 Guard Rule 完全對齊。
4. **未來可擴充為真正的狀態機**（如 Spring Statemachine）但目前以 **「狀態 + Guard + Service 邏輯」** 為主。

---

# 2. 核心概念（Concepts）

### 2.1 Status（狀態）

- 每個模組都有自己的狀態集合，例如：
  - 報價：`QuotationStatusDef`
  - 銷售訂單：`SalesOrderStatusDef`
  - 出貨單：`DeliveryNoteStatusDef`
- 狀態通常包含：
  - **技術代碼（code）**：例如 `DRAFT`, `CONFIRMED`, `CANCELLED`
  - **顯示名稱（name）**：例如「草稿」、「已確認」、「已作廢」
  - **排序 / 權重**：例如 `sequence`

### 2.2 Transition（轉換）

- 狀態之間允許的「流向」：
  - 例如：`DRAFT` → `SUBMITTED` → `APPROVED` → `CLOSED`
- 目前以 **Service 邏輯 + Enum + Guard** 實作，不一定有獨立資料表。

### 2.3 Guard（防呆條件）

- 在狀態轉換前要通過的條件檢查：
  - 欄位是否填寫完整
  - Workflow 必要欄位（如 ValidUntil）是否有效
  - 是否已存在衝突紀錄
  - 是否符合角色 / 權限條件

### 2.4 Action（動作）

- 使用者在 UI 上觸發的操作（如：送出、核准、作廢），會對應到：
  - 一個後端 Service 方法，或
  - 一個「狀態轉換 + Side-effect」（如寫入 Log、產生後續單據）

---

# 3. Workflow 設計原則

1. **單向流程、可控回退**
   - 預設流程應是單向（不允許任意跳躍）。
   - 若要回退（例如從 `APPROVED` 回到 `DRAFT`），必須明確設計「回退動作」，並記錄原因。

2. **狀態數量保持精簡**
   - 避免過度細分（例如將「待審核中」拆成太多子狀態）。
   - 每個狀態要能清楚回答：「這個單據現在可以做什麼、不能做什麼？」

3. **狀態轉換必須有 Guard**
   - 絕不可直接在 Controller 修改狀態。
   - 必須透過 Service 方法進行轉換，並包含完整檢查。

4. **前端只負責呈現「可選動作」，不自行判斷商業規則**
   - 前端可根據後端提供的「可執行動作」清單顯示按鈕。
   - 商業邏輯一律在後端檢查。

---

# 4. 典型範例：Quotation Workflow（報價流程）

以「報價」為例，常見狀態如下：

- `DRAFT`：草稿，可自由修改。
- `SUBMITTED`：已送出，等待客戶回覆或內部確認。
- `APPROVED`：已核准，客戶接受條件。
- `EXPIRED`：已逾期（ValidUntil 過期）。
- `CANCELLED`：已作廢，不可再修改。

### 4.1 合法狀態轉換（示意）

```text
DRAFT      → SUBMITTED → APPROVED → (轉為 Sales Order 或 CLOSE)
   │               │
   └──────────────→ CANCELLED

SUBMITTED → EXPIRED（時間逾期）
```

### 4.2 Guard 條件示意

* `DRAFT → SUBMITTED`

  * 必填欄位是否完整（客戶、有效期限、主要條件…）
  * 是否存在至少一筆報價明細
* `SUBMITTED → APPROVED`

  * 是否已確認最終價格與條件
  * 權限是否允許（特定角色才能核准）
* `SUBMITTED → CANCELLED`

  * 需記錄原因（例如：客戶放棄）
* `SUBMITTED → EXPIRED`

  * 系統或排程根據 ValidUntil 自動變更狀態

---

# 5. 後端實作模式（Backend Workflow Pattern）

目前 Flexora ERP 工作流程的實作方式如下：

### 5.1 Status 欄位

* 每個主檔（如 `QuotationThread`, `SalesOrder`）有一個狀態欄位：

  * Enum（例如 `QuotationStatus`），或
  * String（搭配 `StatusDef` 資料表）

### 5.2 Service 方法格式（建議）

```java
@Transactional
public QuotationThread submit(Long id) {
    QuotationThread thread = findByIdOrThrow(id);

    // 1. Guard 檢查
    validateSubmitGuard(thread);

    // 2. 狀態轉換
    thread.setStatus(QuotationStatus.SUBMITTED);

    // 3. Side-effect（紀錄 Log / 事件 / 通知）
    quotationLogService.logStatusChange(thread, QuotationStatus.SUBMITTED);

    return thread;
}
```

### 5.3 Guard 實作（推薦獨立方法）

```java
private void validateSubmitGuard(QuotationThread thread) {
    if (thread.getStatus() != QuotationStatus.DRAFT) {
        throw new BadRequestAlertException("quotation.invalid_state", "quotationThread", "invalidState");
    }

    if (thread.getLines().isEmpty()) {
        throw new BadRequestAlertException("quotation.no_lines", "quotationThread", "noLines");
    }

    if (thread.getValidUntil() == null || thread.getValidUntil().isBefore(LocalDate.now())) {
        throw new BadRequestAlertException("quotation.invalid_valid_until", "quotationThread", "invalidValidUntil");
    }
}
```

> ✅ 所有 Guard 失敗都應回傳可預期的錯誤碼，讓前端能以 i18n 顯示友善訊息。

---

# 6. 前端 Workflow 呈現（Drawer / Buttons）

前端（`flexora-react-ui`）的 Workflow 通常會以：

* **Drawer / Side Panel**（例如「報價工作流程」抽屜）
* **Toolbar Button**（例如「送出」、「核准」、「作廢」按鈕）
* **狀態 Badge**（顯示目前狀態）

### 6.1 後端提供的資訊（建議）

後端可提供一個 **「目前可執行動作清單」**，例如：

```json
{
  "id": 123,
  "status": "DRAFT",
  "availableActions": [
    "SUBMIT",
    "DELETE"
  ],
  "guards": {
    "requiresValidUntil": true,
    "requiresAtLeastOneLine": true
  }
}
```

前端依照：

* `availableActions` 決定顯示哪些按鈕
* `guards` 顯示提示訊息（例如有效期限未填）

### 6.2 前端行為原則

1. **不在前端重寫業務邏輯**

   * 可做基本 UI 層驗證（必填欄位、格式…）
   * 不做複雜條件判斷（例如「客戶信用額度」檢查）。

2. **將錯誤交由後端判斷**

   * 若後端拋出 `BadRequestAlertException` 帶錯誤碼，前端以 i18n 顯示。

---

# 7. Workflow 與資料庫的關係

目前 Workflow 與 DB 的關係：

* Status 主要存在於：

  * 單據主檔（例如 `quotation_thread.status`）
  * Status 定義表（`quotation_status_def`）負責描述與排序
* 狀態歷程（History）建議另外存表：

  * `quotation_status_history`
  * `sales_order_status_history`

用途：

* 稽核（誰在何時做了什麼）
* 顯示 timeline（時間軸）

---

# 8. 未來：導入正式狀態機（Optional）

未來若導入如 **Spring Statemachine** 或其他狀態機框架：

* 現有 Workflow 設計已可自然映射：

  * State = Status
  * Event/Action = Service 方法
  * Guard = Guard 方法
* 導入時建議：

  * 不破壞現有 API 介面
  * 僅於內部替換實作方式（從 if/else 改為 statemachine）

---

# 9. 文檔與實作同步規則

每次新增或修改 Workflow：

1. 必須更新：

   * 本文件（若為架構級調整）
   * 該模組的 spec（例如 `/docs/specs/phase4-quotation/` 裡的流程章節）
2. 若調整狀態名稱 / 定義：

   * 更新 DB對應的 `StatusDef` 表。
   * 更新前端 i18n。
3. 若調整 Guard 條件：

   * 更新後端 Service Guard 方法。
   * 更新 spec 中「防呆條件」列表。

---

# 10. 結語

Workflow 是 Flexora ERP 核心中的核心。
只要涉及「狀態」與「動作」，就應該回到本文件與對應 specs 中檢查是否一致。

* 想知道「這張單到底可以做什麼、不可以做什麼」 → 看 Workflow。
* 想知道「為什麼系統不讓使用者按某個按鈕」 → 看 Guard。
* 想知道「這條規則為什麼存在」 → 查看 `DECISION_LOG.md` 或 ADR。

未來 Workflow 演進時，本文件會持續更新。
