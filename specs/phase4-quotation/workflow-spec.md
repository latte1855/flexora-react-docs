# Phase 4 — 報價工作流程與狀態機規格（Quotation Workflow Specification）

本文件定義 **報價模組（Quotation）在 Flexora ERP 中的狀態、轉換、Guard（防呆條件）、  
以及前端 Workflow Drawer / 按鈕行為**，作為後端實作與前端 UI 的共同依據。

> Domain 模型請參考：`domain-model.md`  
> 共用 Workflow 設計原則請參考：`../../architecture/WORKFLOW.md`

---

## 1. Workflow 範圍與對象

本 Workflow 主要針對：

- `QuotationRevision`（單一報價版本）的狀態與操作  
- 部分狀態會回寫影響 `QuotationThread` 的「代表狀態」（例如 active revision）

**本文件不處理：**

- Sales Order 的 Workflow（在 Sales 模組 spec 中定義）
- 多模組跨單據的流程（如 SO → DN → Invoice）

---

## 2. 狀態列表（Statuses）

報價版本（`QuotationRevision.status`）預計使用以下狀態：

| 狀態代碼 | 說明 | 誰會看到 | 備註 |
|----------|------|----------|------|
| `DRAFT` | 草稿 | 內部 | 可自由修改，尚未對客戶送出 |
| `SUBMITTED` | 已送出 | 內部 | 已正式提供客戶參考，等待回覆 |
| `APPROVED` | 已核准 | 內部 | 客戶接受此版本，可轉銷售訂單 |
| `REJECTED` | 已拒絕 | 內部 | 客戶明確拒絕此版本（或內部棄用） |
| `EXPIRED` | 已逾期 | 內部 | 有效期限已過，自動或手動標記 |
| `CANCELLED` | 已作廢 | 內部 | 業務主動作廢，不再使用 |

> 前端 i18n 需提供對應文字與 Badge 樣式，例如：  
> `enum.quotationStatus.DRAFT=草稿` 等。

---

## 3. 狀態轉換（Transitions）

### 3.1 合法轉換表（概念）

```text
DRAFT      → SUBMITTED → APPROVED
   │             │          │
   │             │          └→ CANCELLED
   │             └→ REJECTED
   │             └→ EXPIRED
   └→ CANCELLED
```

也可以用表格形式：

| 從狀態       | 可轉換至      | 說明                |
| --------- | --------- | ----------------- |
| DRAFT     | SUBMITTED | 送出報價給客戶           |
| DRAFT     | CANCELLED | 草稿直接作廢            |
| SUBMITTED | APPROVED  | 客戶同意此版本           |
| SUBMITTED | REJECTED  | 客戶明確拒絕此版本         |
| SUBMITTED | EXPIRED   | 有效期限過期（可由排程或人工）   |
| SUBMITTED | CANCELLED | 業務決定作廢該版本         |
| APPROVED  | CANCELLED | 特殊情境（內部需要，且需紀錄原因） |

> 一旦狀態為 `CANCELLED` 或 `EXPIRED`，**不得再修改內容**，
> 也**不得再回到 DRAFT/SUBMITTED/APPROVED**，只能透過「複製新版本」延續。

---

## 4. 核心動作（Actions）

報價 Workflow 主要操作動作（Action）如下：

| 動作代碼                  | 說明                             | 來源狀態                         | 目標狀態            |
| --------------------- | ------------------------------ | ---------------------------- | --------------- |
| `SUBMIT`              | 送出報價                           | DRAFT                        | SUBMITTED       |
| `APPROVE`             | 標記為客戶已接受                       | SUBMITTED                    | APPROVED        |
| `REJECT`              | 標記為客戶拒絕                        | SUBMITTED                    | REJECTED        |
| `EXPIRE`              | 標記報價逾期                         | SUBMITTED                    | EXPIRED         |
| `CANCEL`              | 作廢報價版本                         | DRAFT / SUBMITTED / APPROVED | CANCELLED       |
| `CLONE_REVISION`      | 從現有版本複製出新版本                    | 任意                           | 新版 = DRAFT      |
| `SET_ACTIVE_REVISION` | 設定此版本為 Thread 的 activeRevision | APPROVED / SUBMITTED 等       | 同狀態（僅影響 Thread） |

> 實作上，每個 Action 通常對應一個後端 Service 方法 + REST API。
> 後端方法中會執行 Guard 檢查與狀態變更。

---

## 5. Guard（防呆條件）

以下 Guard 規則為 **後端 Service 的硬性規則**，
前端可做相同規則的預檢，但不可取代後端。

### 5.1 `SUBMIT`（草稿送出）

**來源狀態：** `DRAFT`
**目標狀態：** `SUBMITTED`

送出前必須通過以下 Guard：

1. **狀態檢查**

   * `status == DRAFT`
     否則 → `quotation.invalid_state`

2. **客戶必填**

   * `thread.customerId != null`
     否則 → `quotation.customer_required`

3. **至少一筆明細**

   * `revision.lines` 不得為空
     否則 → `quotation.no_lines`

4. **有效期限（ValidUntil）有效**

   * `validUntil != null`
   * `validUntil >= today`（可允許當天）
     否則 → `quotation.invalid_valid_until`

5. **金額與幣別**

   * `currencyCode != null`
   * 若有金額計算邏輯，需確認單價與數量非零、總額 >= 0 等

> 對應錯誤碼（建議）：
>
> * `quotation.invalid_state`
> * `quotation.customer_required`
> * `quotation.no_lines`
> * `quotation.invalid_valid_until`
> * `quotation.invalid_amount`（若需要）

---

### 5.2 `APPROVE`（核准）

**來源狀態：** `SUBMITTED`
**目標狀態：** `APPROVED`

Guard 規則：

1. 狀態檢查：`status == SUBMITTED`，否則 `quotation.invalid_state`
2. 有效期限：`validUntil != null && validUntil >= today`
3. 可選：

   * 若需要主管核准權限 → 檢查角色（`ROLE_SALES_MANAGER` 等）
   * 檢查是否已建立相同條件的訂單（避免重複）

---

### 5.3 `REJECT`（拒絕）

**來源狀態：** `SUBMITTED`
**目標狀態：** `REJECTED`

Guard 規則：

1. 狀態檢查：`status == SUBMITTED`
2. 建議需填「拒絕原因」：

   * 後端可要求傳入 `reason` 欄位，不可為空 → `quotation.reject_reason_required`

---

### 5.4 `EXPIRE`（逾期）

**來源狀態：** `SUBMITTED`
**目標狀態：** `EXPIRED`

Guard 規則：

1. 狀態檢查：`status == SUBMITTED`
2. `validUntil < today`

   * 若尚未逾期，通常不允許標記 EXPIRED（除非系統特別需求）

實作建議：

* 可由排程批次掃描 `validUntil < today AND status = SUBMITTED`，
  自動執行 `EXPIRE` 動作。

---

### 5.5 `CANCEL`（作廢）

**來源狀態：** `DRAFT` / `SUBMITTED` / `APPROVED`
**目標狀態：** `CANCELLED`

Guard 規則：

1. 狀態檢查：不得為 `CANCELLED` / `EXPIRED` / `REJECTED` 再作廢
   → 否則 `quotation.invalid_state`
2. 若狀態為 `APPROVED`：

   * 需檢查「是否已轉銷售訂單」：

     * 若已轉 SO，通常不允許作廢 → `quotation.already_converted_to_so`
   * 建議強制需要「作廢原因」

---

### 5.6 `CLONE_REVISION`（複製版本）

**來源狀態：** 任意
**目標：** 新的 `QuotationRevision`，狀態 = `DRAFT`，`revisionIndex` + 1

Guard 規則：

1. 原版本必須存在且未被硬刪除。
2. 若來源是已作廢/逾期版本，仍可複製（商業上合理），
   但可在 UI 顯示提醒。

包含的資料：

* 主檔欄位（有效期限可調整，建議預設 = 今天 + N 天）
* 明細行完整複製（含單價、折扣等）
* 狀態欄位 reset 為 `DRAFT`
* 金額欄位依實作方式決定是直接複製或重新計算

---

### 5.7 `SET_ACTIVE_REVISION`（設定主線代表版本）

**來源狀態：** 通常為 `APPROVED`（可視需求放寬）
**效果：** 更新 `QuotationThread.activeRevisionId` 指向此版本

Guard 規則：

1. Revision 必須屬於該 Thread。
2. 若系統規則要求只能將 `APPROVED` 設為 active：

   * `status == APPROVED`，否則 `quotation.active_revision_must_be_approved`

---

## 6. 前端 Workflow Drawer 與 Guard 提示

### 6.1 Workflow Drawer 角色

前端的「Workflow 抽屜 / 面板」主要功能：

* 顯示目前 Revision 狀態（Badge + 說明）
* 顯示可執行動作（按鈕群，例如 SUBMIT / APPROVE / CANCEL…）
* 顯示 Guard 提示（例如缺少有效期限、無明細）
* 快速接 Timeline / Pipeline action menu 的 Retry/Resend 預設（可直接帶入事件或重送 API）

### 6.2 後端建議回傳格式（概念）

後端可提供一個 Workflow 狀態 DTO，例如：

```json
{
  "revisionId": 123,
  "status": "DRAFT",
  "availableActions": ["SUBMIT", "CANCEL", "CLONE_REVISION"],
  "guards": {
    "requiresCustomer": true,
    "requiresValidUntil": true,
    "requiresAtLeastOneLine": true,
    "validUntilMissing": true,
    "validUntilExpired": false,
    "hasNoLines": true
  }
}
```

前端用法：

* 根據 `availableActions` 控制按鈕顯示與啟用/禁用
* 根據 `guards.*` 顯示提醒訊息或標示警告 Badge

### 6.3 Guard 提示行為（示意）

### 6.4 Timeline / Convert 補充

* Timeline 行動選單的 `Retry`/`Resend` 會提前帶入事件代碼、Reason、Note、Reference；前端可選擇直接 open Workflow Drawer 或直接重新呼叫事件 API 以補發。
* 若重新發送成功，需刷新 Timeline 與 Hub；若失敗則顯示錯誤訊息類似 backend `error` response。
* Convert Drawer 於確認轉單前會加入稅額 / 匯率摘要、每個稅別的金額與稅率提示，以及轉單後 Sales Order Link summary（包括已連結數量與金額），方便使用者立即檢查稅負和匯率影響。

* `requiresValidUntil = true` 且 `validUntilMissing = true`
  → Drawer 顯示警告：「請設定報價有效期限」
* `validUntilExpired = true`
  → 顯示：「有效期限已過，請重新設定或無法送出」
* `hasNoLines = true`
  → 顯示：「報價至少需要一筆明細」

> 這些提示 **不等於後端放行**，
> 真正送出時仍由後端 Guard 最終檢查。

---

## 7. 錯誤碼對應（與 error-handling-spec 串接）

報價 Workflow 常見錯誤碼建議如下：

| errorKey                                     | 意義               | 可能觸發動作                                      |
| -------------------------------------------- | ---------------- | ------------------------------------------- |
| `quotation.invalid_state`                    | 當前狀態不允許此操作       | SUBMIT / APPROVE / CANCEL / REJECT / EXPIRE |
| `quotation.no_lines`                         | 該版本沒有任何報價明細      | SUBMIT                                      |
| `quotation.invalid_valid_until`              | 有效期限未填或已過期       | SUBMIT / APPROVE                            |
| `quotation.customer_required`                | 報價未指定客戶          | SUBMIT                                      |
| `quotation.reject_reason_required`           | 拒絕時未提供原因         | REJECT                                      |
| `quotation.already_converted_to_so`          | 已轉銷售訂單，不可作廢      | CANCEL                                      |
| `quotation.active_revision_must_be_approved` | 僅允許核准版本設為 active | SET_ACTIVE_REVISION                         |

> 這些錯誤碼需同步整理到：
> `error-codes.md` 與全域 `error-handling-spec.md` 中的對應模組段落。

---

## 8. 狀態歷史與稽核（若實作）

若實作 `QuotationStatusHistory`，每次狀態變更時需記錄：

* `fromStatus`, `toStatus`
* `changedBy`, `changedAt`
* `reason`（例如作廢/拒絕原因）

前端可在報價詳情頁提供「狀態時間軸」顯示，例如：

* 2025-11-19 09:20：由 Jimmy 建立，狀態 = DRAFT
* 2025-11-19 10:05：由 Jimmy 送出，狀態：DRAFT → SUBMITTED
* 2025-11-20 15:30：由客戶確認，Jimmy 標記為 APPROVED

## 8.1 Convert Drawer 稅額與 SO 摘要

* Convert Drawer 在 APPROVED 狀態下應呈現所選行的稅別/稅額拆解與幣別說明，並在 UI 上同步顯示預計的 Sales Order 總額、稅額與行數，讓轉單前能快讀檢查稅負與匯率。
* 轉單完成後，Hub/Detail 頁的 Sales Order Links 卡片需顯示最新建立的 SO 訂單編號與日期，以及關聯類型（FULL/PARTIAL）。


---

## 9. 測試建議（Test Strategy 片段）

對 Workflow 的測試建議至少包含：

1. **單元測試（Service 層）**

   * 每個 Action（SUBMIT / APPROVE / …）：

     * 正常情境 → 應成功改變狀態
     * 違規情境 → 應拋出對應錯誤碼

2. **整合測試（REST + DB）**

   * 透過 REST API 呼叫對應操作，驗證：

     * HTTP 狀態碼
     * 回傳的錯誤結構與 errorKey
     * DB 中狀態、歷史紀錄是否正確更新

3. **E2E / UI 測試（未來）**

   * 在 UI 上嘗試：
   * Timeline Action Menu: 模擬重送事件/重新發送，確認 retry/resend spinner + toast/refresh。
   * Convert Drawer: 模擬多筆行項並檢查稅額/匯率摘要與 SO links 反映結果。

     * 空白明細送出 → 預期 UI 顯示錯誤，API 回 400 / `quotation.no_lines`
     * 有效期限在昨天送出 → UI 阻擋 + API Guard

---

## 10. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                  |
| ---- | ---------- | ----- | ------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立報價 Workflow 規格初稿。 |

