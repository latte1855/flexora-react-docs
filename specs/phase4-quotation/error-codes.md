# Phase 4 — 報價模組錯誤碼表（Quotation Error Codes）
本文件列出 Flexora ERP **報價模組（Quotation）** 所有錯誤碼（errorKey），  
提供給：

- 後端開發  
- 前端 UI / i18n  
- Codex / Agent 自動產生程式  
- 文件維護  

作為唯一權威（Single Source of Truth）。

> ⚠️ 注意：  
> 這些錯誤碼必須同時在：  
> - 後端 Exception / Guard 中使用  
> - 前端 i18n 檔（如 `zh-TW.json`、`en.json`）新增對應譯文  
> - 與 `error-handling-spec.md` 保持同步

---

# 1. 全域錯誤碼（報價模組常用）

| errorKey | 說明 | 何時觸發 | 對應 HTTP |
|----------|------|-----------|------------|
| `quotation.invalid_state` | 當前狀態不允許執行該操作 | 各種 Workflow 動作 | 400 |
| `quotation.no_lines` | 該版本沒有任何報價明細 | SUBMIT | 400 |
| `quotation.invalid_valid_until` | 有效期限未填或已過期 | SUBMIT / APPROVE | 400 |
| `quotation.customer_required` | 報價缺少客戶 | SUBMIT | 400 |
| `quotation.reject_reason_required` | 拒絕動作缺少理由 | REJECT | 400 |
| `quotation.already_converted_to_so` | 已轉為銷售訂單，不可作廢 | CANCEL | 400 |
| `quotation.active_revision_must_be_approved` | 只有 APPROVED 可設定為 active version | SET_ACTIVE_REVISION | 400 |

---

# 2. Thread（主線）相關錯誤碼

| errorKey | 說明 | 何時觸發 |
|----------|------|-----------|
| `quotation.thread_not_found` | 查無此報價主線 | 查詢 / 更新 / 刪除 |
| `quotation.thread_cannot_delete_with_approved_revision` | 主線含有已核准版本，不可刪除 | DELETE Thread |
| `quotation.thread_customer_required` | 建立 Thread 時未指定客戶 | POST / PUT Thread |
| `quotation.thread_invalid_owner` | ownerId 無效或使用者無權使用 | 建立 / 更新 Thread |

> Thread 的錯誤碼命名建議使用 `quotation.thread_xxx` 形式，  
> 以免與 Revision 衝突。

---

# 3. Revision（報價版本）相關錯誤碼

| errorKey | 說明 | 何時觸發 |
|----------|------|-----------|
| `quotation.revision_not_found` | 查無此報價版本 | GET / PUT / DELETE Revision |
| `quotation.revision_cannot_delete` | 此版本不可直接刪除（非 DRAFT） | DELETE Revision |
| `quotation.revision_invalid_state` | 此版本狀態不允許此操作（Alias to invalid_state） | 各 Workflow 動作 |
| `quotation.revision_thread_mismatch` | revision 不屬於指定的 thread | 任何跨 thread 操作 |
| `quotation.revision_clone_failed` | 複製版本時發生錯誤 | CLONE |
| `quotation.revision_amount_invalid` | 金額不正確（計價異常） | 計算流程 / 驗證 |

> 若該錯誤僅發生在 Revision 內部，建議使用 `revision_` 頭  
> 若屬跨版本或 Workflow 則用全域的 `quotation.`。

---

# 4. Line（報價明細）相關錯誤碼

| errorKey | 說明 |
|----------|------|
| `quotation.line_item_required` | 明細缺少 itemSkuId |
| `quotation.line_quantity_invalid` | 數量不能小於或等於零 |
| `quotation.line_unit_price_invalid` | 單價不能為負數 |
| `quotation.line_tax_type_invalid` | 無效的稅別 |
| `quotation.line_invalid` | 整行資料不完整（前端可整行標紅） |

> 這些錯誤多半由前端先攔住，但後端仍可能在 DTO / Service 層做最終檢查。

---

# 5. Workflow（工作流程）相關錯誤碼

Workflow 錯誤大多屬於「狀態不符」或「條件不足」，  
以下為專屬代碼（部分與前述重疊，但此表更聚焦於流程邏輯）。

| errorKey | 描述 | 動作 |
|----------|------|--------|
| `quotation.workflow.submit_guard_failed` | SUBMIT Guard 未通過（有效期限/明細/客戶） | SUBMIT |
| `quotation.workflow.approve_guard_failed` | APPROVE Guard 未通過（有效期限無效） | APPROVE |
| `quotation.workflow.reject_guard_failed` | 拒絕時理由未輸入 | REJECT |
| `quotation.workflow.expire_guard_failed` | 尚未逾期，不能標記為 EXPIRED | EXPIRE |
| `quotation.workflow.cancel_guard_failed` | 不允許從該狀態 CANCEL | CANCEL |
| `quotation.workflow.clone_guard_failed` | 無法從來源版本複製（資料不完整或 thread 錯誤） | CLONE |

> 前端若遇到 `quotation.workflow.*` 錯誤，  
> 建議顯示「(這個動作的前置條件未達成)」＋錯誤內容。

---

# 6. 計價 / 金額相關錯誤碼（與 Phase 1 價格模組連動）

報價版本中包含計價（subtotal / tax / total）流程，  
若計算異常或資料錯誤，會回傳以下代碼：

| errorKey | 描述 |
|----------|------|
| `quotation.amount_calculation_failed` | 計算過程失敗 |
| `quotation.amount_subtotal_mismatch` | 小計不一致（前端與後端計算不同） |
| `quotation.amount_tax_mismatch` | 稅額不一致 |
| `quotation.amount_total_mismatch` | 總金額不一致 |

> 後端可選擇：  
> - 以後端計算覆寫前端（較安全）  
> - 或要求前端修正明細欄位（較嚴格）

---

# 7. API Body / 欄位驗證錯誤（DTO Validation）

源自 Bean Validation（`@NotNull`, `@Size`, `@Positive`…）：

| errorKey / message | 說明 |
|---------------------|--------|
| `error.validation` | 通用欄位驗證錯誤（會含 fieldErrors） |
| `NotNull` | 例如 `validUntil` 未填 |
| `NotBlank` | 例如 topic 未填 |
| `Positive` | 數量/價格需為正數 |
| `Size` | 長度不符 |

> 在前端 mapping 時，  
> - 若是 `error.validation` → 對應欄位顯示  
> - 若是 `quotation.xxx` → 顯示 Toast 或 Alert

---

# 8. 權限錯誤（Security）

報價模組也可能被 Security 阻擋，相關錯誤碼為 Phase 0 全域：

| errorKey | 說明 |
|----------|------|
| `error.unauthorized` | 未登入 / Token 失效（401） |
| `error.accessDenied` | 已登入但權限不足（403） |
| `error.forbidden` | 等同於 access denied |

> 報價模組的「核准」通常需要較高權限（例如 `ROLE_SALES_MANAGER`）。  
> 若後端新增自訂權限，例如 `quotation.approve`，  
> 也應在此文件新增相關錯誤碼。

---

# 9. 未來可能新增的錯誤碼（預留）

若報價模組擴充計價 / 稅務 / 附件 / 匯率等功能，可能增加：

- `quotation.exchange_rate_missing`
- `quotation.tax_code_invalid`
- `quotation.attachment_required`
- `quotation.workflow.invalid_transition`
- `quotation.pricelist_not_applicable`
- `quotation.inventory_reservation_failed`

> 一旦新增請立即補充此文件，並同步更新 i18n。

---

# 10. 錯誤碼命名規則總結

- **模組範圍錯誤** → `quotation.xxx`
- **Thread 專屬錯誤** → `quotation.thread_xxx`
- **Revision 專屬錯誤** → `quotation.revision_xxx`
- **Line 專屬錯誤** → `quotation.line_xxx`
- **Workflow Guard 錯誤** → `quotation.workflow.xxx`
- **計價錯誤** → `quotation.amount_xxx`
- 保留與 Phase 0 全域錯誤碼一致性（`error.xxx` 用於更通用錯誤）

---

# 11. 與其他文件的關聯

- 報價 Workflow：`workflow-spec.md`
- 報價 Domain：`domain-model.md`
- 報價 API：`api-spec.md`
- 全域錯誤規格：`../phase0-system-foundation/error-handling-spec.md`
- 驗證規範：`../phase0-system-foundation/validation-and-rules.md`

---

# 12. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立報價模組錯誤碼表初稿。 |
