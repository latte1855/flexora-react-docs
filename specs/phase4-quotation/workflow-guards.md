# Phase 4 — Workflow Guard 對照表（Guard Mapping）

依據 `docs/migration/flexora-react/flexora-quotation-management-spec.md` 與 `flexora-react` 的 workflow service 實作，整理各 Action 的 guard 與錯誤碼；若 guard 與程式碼有出入，應以以下 service 為最終依據並在欄位註記 `Updated per flexora-react/src/main/java/.../QuotationWorkflowService.java`.

| Action | Service Method | Guard Conditions | Error Key |
| --- | --- | --- | --- |
| SUBMIT | `QuotationWorkflowService.submitQuotation` | `status == DRAFT`, `customerId != null`, `revision.lines` not empty, `validUntil >= today`, `currencyCode != null`, `QuotationPricingService.verifyAmounts` | `quotation.invalid_state`, `quotation.customer_required`, `quotation.no_lines`, `quotation.invalid_valid_until`, `quotation.amount_mismatch`, `quotation.pricelist_no_price_found` |
| APPROVE | `QuotationWorkflowService.approveQuotation` | `status == SUBMITTED`, `validUntil >= today`, (optional `ROLE_SALES_MANAGER`) | `quotation.invalid_state`, `quotation.invalid_valid_until` |
| REJECT | `QuotationWorkflowService.reject` | `status == SUBMITTED`, `reason != null` | `quotation.invalid_state`, `quotation.reject_reason_required` |
| EXPIRE | `QuotationWorkflowService.expireQuotation` | `status == SUBMITTED`, `validUntil < today` | `quotation.invalid_state`, `quotation.invalid_valid_until` |
| CANCEL | `QuotationWorkflowService.cancel` | `status ∈ {DRAFT, SUBMITTED, APPROVED}`, not already `CANCELLED/EXPIRED/REJECTED`, not converted to SO | `quotation.invalid_state`, `quotation.already_converted_to_so` |
| CLONE | `QuotationWorkflowService.cloneQuotation` | source revision exists, `threadId` matches | `quotation.workflow.clone_guard_failed` |
| SET_ACTIVE | `QuotationWorkflowService.setActiveRevision` | revision belongs to thread, optionally `status == APPROVED` | `quotation.active_revision_must_be_approved`, `quotation.revision_thread_mismatch` |

> Guard implementations live in `QuotationWorkflowService` + `QuotationPricingService`; set this doc as checklist when new transitions are added.
