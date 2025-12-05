# Sales Order Module – Functional Spec

> 狀態：草稿，詳細需求仍可參考 `docs/flexora-salesorder-management-spec.md`、`docs/ui/customer/Customer Workspace SRS（完整系統規格）.md`。  
> 目標：集中定義 Sales Order（SO）與 Quote/Delivery/TL 的資料與行為，避免各檔案重複。

## 範圍

| 範疇 | 說明 |
| --- | --- |
| Sales Order 主檔 | `SalesOrder`、`SalesOrderItem`、Ext Attr、付款/交期、Owner |
| Workflow | 送審、核准、取消、回覆；歷史記錄 |
| 與 Quotation | 從 Quote 建立 SO、Thread / Revision Link、版本比對 |
| 與 Delivery/Invoice | 出貨/收款進度、庫存扣減 |
| Activity / Timeline | 備註、活動、流程紀錄 |

## 資料與關聯

- `SalesOrder (SO)` ↔ `SalesOrderItem` (1-n)
- `SalesOrder` ↔ `SalesOrderQuotationLink` (n-n via Link)
- `SalesOrder` ↔ `DeliveryNote` / `Invoice`（透過 Link 或外鍵）
- `SalesOrderStatusHistory` / `StateTransition`：記錄 workflow
- Ext Attr 定義檔：`SalesOrderExtAttrDef`

### 主要欄位（摘要）

| 類別 | 欄位 | 說明 |
| --- | --- | --- |
| 基本 | soNo, threadId, customerId, ownerId, status | Status: `DRAFT/IN_REVIEW/APPROVED/FULFILLED/CLOSED/CANCELLED` |
| 金額 | currency, exchangeRate, subtotal, taxTotal, grandTotal | 需對應 PriceList/Tax |
| 交期 | requestedDeliveryDate, promisedDate, shippingAddress, billingAddress | 與 Delivery 模組同步 |
| 行項 | skuId, uomId, qty, unitPrice, taxCode, discount | 可由 Quote 複製 |

## 操作流程（摘要）

1. 從 Quote 或直接建立 SO（Quick Create / Full Editor）。
2. 編輯行項、交期、客製欄位 → 儲存草稿。
3. 送審：`openWorkflow`（若 Workflow 啟用） → `approve/reject`。
4. 產生 Delivery Note、Invoice：透過 Link API / Buttons。
5. 完成 / 取消：更新狀態，寫入歷史。

## 與其他模組整合

| 模組 | 整合方式 |
| --- | --- |
| Quotation | 從 Quote 轉 SO（保留 Thread / Revision），同步狀態回報；可顯示來源版本 | 
| Delivery | 由 SO 產生 Delivery Note，推播剩餘可出貨數；Delivery 狀態回寫 SO | 
| Inventory | 依 SO Reservation 策略鎖定庫存或檢查可用量；取消時釋放 | 
| Invoice / Billing | 跟催開立 Invoice、紀錄收款進度；SO Detail 顯示對應 Invoice | 
| Activity / CRM | 在 Customer/Opportunity Detail 中顯示 SO 連結與狀態 | 

## Workflow / Timeline

- 事件定義：`SalesOrderEventDefResource`（E.g., SUBMIT, APPROVE, CANCEL）。
- 狀態轉換：`SalesOrderStateTransitionResource`。
- 歷史：`SalesOrderStatusHistoryResource`。
> 需在 UI 中顯示「查看流程圖」、「Timeline 版本切換」等功能。

## TODO

- [ ] 匯入現有流程圖、狀態圖。
- [ ] 完整欄位清單（含 Ext Attr、付款條件、物流資訊）。
- [ ] 梳理 SO ↔ Quote Link 流程（多版本、異動）。
- [ ] 描述與 Delivery/Invoice 的資料同步與異常處理（例如 Delivery 取消時回寫 SO）。  
- [ ] 補充與 Inventory Reservation、Pricing（PriceList）互動的欄位與 API。  
- [ ] 若需匯入/批次更新交期或行項，提供欄位與錯誤處理規格。  
