# Phase 4 — 報價模組測試策略（Quotation Test Strategy）

本文件定義 Flexora ERP **報價模組（Quotation）** 的測試策略與範圍，  
包含單元測試、整合測試、E2E/UI 測試建議，以及最低應覆蓋的情境清單。

> 目標：  
> - 確保 Quotation Domain / Workflow / API / Pricing 行為與 spec 一致  
> - 防止之後 refactor / 新功能破壞既有流程  
> - 提供 AI Agent / Codex 生成測試碼時的依據

---

## 1. 測試層級與範圍

報價模組的測試分為三層：

1. **單元測試（Unit Test）**
   - Domain / Service / Pricing / Guard
2. **整合測試（Integration Test）**
   - REST API + DB + Security + Workflow
3. **E2E / UI 測試（End-to-End / UI Test）**（可逐步導入）
   - `flexora-react-ui` 上的實際使用流程

---

## 2. 單元測試（Unit Test）

### 2.1 主要目標

- 驗證 **Domain 與 Service 邏輯**：
  - Workflow Guard（SUBMIT / APPROVE / CANCEL…）
  - Pricing 計算（Line / Revision）
  - Thread / Revision / Line 的基本行為

### 2.2 優先測試的類別（示意）

- `QuotationPricingService` 或類似計價服務
- `QuotationWorkflowService`（若有）或 `QuotationRevisionService` 裡的 Guard 方法
- `QuotationThreadService`（建立 / 刪除 / activeRevision 相關邏輯）

### 2.3 建議測試案例 — Pricing

以 `QuotationPricingService` 為例：

1. **行折扣（Line Discount）**
   - 有 `discountRate`、無 `discountAmount` → 正確計算 netUnitPrice / lineSubtotal
   - 無 `discountRate`、有 `discountAmount` → 正確反推 netUnitPrice
   - 兩者皆 0 → 不折扣

2. **含稅 / 未稅計算**
   - `taxIncluded = false` → `lineSubtotal * taxRate = taxAmount`，`lineTotal = subtotal + tax`
   - `taxIncluded = true` → correctly derive subtotal & tax from total

3. **整體折扣（Overall Discount）**
   - 先行折扣、再整體折扣 → 合計金額正確
   - 提供 `discountRate` → 正確計算 `discountAmount`
   - 提供 `discountAmount` → 可反算 `discountRate`（若規定）

4. **精度與四捨五入**
   - 特別測試邊界數值（例如 0.005, 0.004, 大數量）

5. **錯誤情境**
   - 單價缺失 → 拋出 `quotation.unit_price_missing` 或對應 Exception
   - 數量 <= 0 → `quotation.line_quantity_invalid`
   - 計算中發生除以零 / 其他異常 → `quotation.amount_calculation_failed`

### 2.4 建議測試案例 — Workflow Guard

以 `QuotationRevisionService`（或類似）為例：

1. **SUBMIT Guard**
   - 狀態 ≠ DRAFT → `quotation.invalid_state`
   - 無客戶 → `quotation.customer_required`
   - 無明細 → `quotation.no_lines`
   - 有效期限為 null → `quotation.invalid_valid_until`
   - 有效期限 < 今天 → `quotation.invalid_valid_until`
   - 所有條件都正確 → 狀態改為 SUBMITTED，無 Exception

2. **APPROVE Guard**
   - 狀態 ≠ SUBMITTED → `quotation.invalid_state`
   - 有效期限 < 今天 → `quotation.invalid_valid_until`
   - 角色權限不足（若 Service 層檢查） → 權限錯誤 / AccessDenied

3. **REJECT Guard**
   - 狀態 ≠ SUBMITTED → `quotation.invalid_state`
   - 無 reason → `quotation.reject_reason_required`
   - 正常情況 → 狀態改為 REJECTED

4. **EXPIRE Guard**
   - 狀態 ≠ SUBMITTED → `quotation.invalid_state`
   - validUntil >= today 時嘗試 EXPIRE → `quotation.workflow.expire_guard_failed`

5. **CANCEL Guard**
   - 狀態是 CANCELLED / EXPIRED / REJECTED → `quotation.invalid_state`
   - 已轉 Sales Order → `quotation.already_converted_to_so`
   - APPROVED 且無 reason（若必填） → 對應錯誤碼

6. **CLONE_REVISION**
   - 來源 Revision 不存在 → `quotation.revision_not_found`
   - 複製後新 Revision：
     - status = DRAFT
     - revisionIndex = max + 1
     - lines 內容正確複製

7. **SET_ACTIVE_REVISION**
   - Revision 不屬於 Thread → `quotation.revision_thread_mismatch`
   - 若要求必須 APPROVED 才能 active → 非 APPROVED → `quotation.active_revision_must_be_approved`

---

## 3. 整合測試（Integration Test）

### 3.1 主要目標

- 驗證 **REST API + 資料庫 + Security** 完整流程  
- 確保 API 規格（`api-spec.md`）與實作一致  
- 確保錯誤格式、錯誤碼、HTTP status 正確

### 3.2 工具與框架（預設）

- Spring Boot Test
- MockMvc 或 WebTestClient
- 測試用 DB（H2 / PostgreSQL Testcontainer）

### 3.3 典型測試案例

#### 3.3.1 建立 Thread + 初始 Revision

1. `POST /api/quotation-threads`
   - 傳入必要欄位（customerId, topic…）
   - 驗證：
     - 回傳 201
     - `threadNo` 不為空
     - DB 確實存在一筆 Thread

2. `POST /api/quotation-threads/{threadId}/revisions`
   - 建立 DRAFT 版本 + 一筆明細  
   - 驗證：
     - 狀態 = DRAFT
     - Lines 正確寫入 DB

#### 3.3.2 SUBMIT 正常流程

- 順序：
  1. 建立 Thread + DRAFT Revision
  2. 呼叫 `POST /api/quotation-revisions/{id}/submit`
  3. 驗證：
     - HTTP 200
     - 回傳 JSON 中 `status = SUBMITTED`
     - DB 中對應 Revision.status = SUBMITTED

#### 3.3.3 SUBMIT Guard 錯誤情境

- 無明細：
  1. 建立無 line 的 DRAFT Revision
  2. `POST /.../submit`
  3. 應得到：
     - HTTP 400
     - `errorKey = "quotation.no_lines"`

- 有效期限過期：
  1. validUntil = 昨天
  2. `POST /.../submit`
  3. 應得到：
     - HTTP 400
     - `errorKey = "quotation.invalid_valid_until"`

#### 3.3.4 APPROVE 流程

- 正常：
  1. DRAFT → SUBMITTED
  2. `POST /.../approve`
  3. 驗證：
     - 200 OK
     - status = APPROVED

- 無效狀態：
  1. 對 DRAFT 直接 `approve`
  2. -> 400 + `quotation.invalid_state`

#### 3.3.5 CANCEL / REJECT / EXPIRE

類似邏輯測試：

- 嘗試不合法狀態 → 400 + 對應錯誤碼
- 合法狀態 → 狀態改變、DB 更新成功

#### 3.3.6 CLONE Revision

- `POST /api/quotation-revisions/{id}/clone`
  - 回傳 201
  - 新 Revision：
    - 狀態 DRAFT
    - lines 數量與內容與來源一致（except id / revisionId）

#### 3.3.7 Workflow Info API

- `GET /api/quotation-revisions/{id}/workflow-info`
  - 在不同狀態下：
    - `availableActions` 是否符合 `workflow-spec.md`
    - `guards` 內容是否與當前資料相符（有/無明細、有/無有效期限）

---

## 4. E2E / UI 測試（End-to-End / UI）

> 此區視專案資源可分階段導入，可優先用手動測試清單，之後再自動化（Playwright / Cypress 等）。

### 4.1 建議工具

- Playwright / Cypress / WebdriverIO（擇一）
- 對 `flexora-react-ui` 做瀏覽器層級測試

### 4.2 必測使用者流程

1. **從 Hub 建立新報價**
   - 在 Quotation Hub 點「新增報價」
   - 選客戶、輸入主題 → 建立 Thread + 初始 Revision
   - 進入 Revision Editor 畫面

2. **編輯報價版本**
   - 新增 2 筆明細
   - 調整單價、折扣、數量
   - 確認畫面右側小計 / 稅額 / 總額顯示正確（前端試算）

3. **透過 Workflow Drawer 送出（SUBMIT）**
   - 若缺少有效期限，Drawer 顯示警告
   - 補上有效期限、再點「送出」
   - Toast 顯示成功，狀態 Badge 變為 SUBMITTED

4. **在 Pipeline 中查看 / 拖拉**
   - 開啟 Pipeline
   - 確認該 Thread 出現在 SUBMITTED 欄位
   - 將卡片拖到 APPROVED 欄
   - 預期 → 彈確認框 → 確認後狀態更新 / Badge 顏色改變

5. **作廢 / 拒絕 / 複製版本**
   - 在 Detail 中對某版本做 CANCEL / REJECT / CLONE
   - 確認：
     - 新版本出現在 Revision 列表
     - 原版本狀態正確顯示

---

## 5. 手動測試清單（可當 Smoke Test）

即使尚未導入自動化 UI 測試，也至少應在每次大改後跑一輪：

1. **建立 Thread + Revision**
2. 編輯明細（新增 / 刪除 / 調整金額）
3. SUBMIT（含失敗情境：沒明細 / 有效期限錯誤）
4. APPROVE / REJECT / CANCEL / EXPIRE
5. CLONE → 產生新 DRAFT 版本
6. Pipeline 拖拉狀態 → API / UI 同步
7. Workflow Drawer 的 Guard 提示是否如 spec 所述

---

## 6. 測試數據與 Fixture

### 6.1 測試專用資料建議

建立一組固定測試數據（可寫在 Liquibase / test SQL / TestDataInitializer）：

- Customers：
  - `CUST-001`: 一般客戶
- ItemSku：
  - `SKU-001`: 單價 50,000，稅別一般稅 5%
  - `SKU-002`: 單價 10,000，免稅
- PriceList：
  - `PL-DEFAULT-TWD`：對上述 SKU 給出預設單價
- Users：
  - `sales1`：ROLE_SALES_USER
  - `manager1`：ROLE_SALES_MANAGER

### 6.2 資料覆用

- 單元測試可自行建立 in-memory Entity
- 整合測試使用固定測試資料，以便斷言金額、狀態

---

## 7. 測試覆蓋目標

- Service 層（含 Workflow / Pricing）：
  - 行為重要的 public 方法覆蓋率達 **80%+**
- REST Controller 層：
  - 主要 API（Create / Update / Submit / Approve / Cancel / Clone / Workflow Info）  
    至少各有 1～2 個「成功」與「失敗」案例
- 錯誤碼：
  - `error-codes.md` 中的關鍵錯誤碼至少在測試中出現一次（成功觸發）

---

## 8. 與其他文件的關聯

- Domain：`domain-model.md`
- Workflow：`workflow-spec.md`
- API：`api-spec.md`
- UI：`ui-spec.md`
- Error Codes：`error-codes.md`
- Pricing：`pricing-rules.md`
- Phase 0 共用測試原則（若有）：`../phase0-system-foundation/*`

---

## 9. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立報價模組測試策略初稿。 |

