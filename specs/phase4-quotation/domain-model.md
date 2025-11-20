# Phase 4 — 報價管理 Domain Model 規格（Quotation Domain Model）

本文件定義 Flexora ERP 報價模組（Phase 4）中的 **主要 Entity / DTO / 關聯與欄位規格（概念層級）**。  
實際實作時，若 Entity 欄位有差異，需同步回寫本文件。

> 本文件偏向「業務＋資料模型」視角，  
> 具體 API 行為請參考同目錄的 `api-spec.md`，  
> Workflow 與 Guard 請參考 `workflow-spec.md`。

---

## 1. 報價 Domain 高階總覽

主要 Domain 物件如下：

- **QuotationThread**
  - 一次「客戶詢價 / 專案報價」的主線
  - 代表一個「商機 / Case」，其下可以有多個報價版本（Revision）
- **QuotationRevision**
  - 報價的具體版本，包含有效期限、整體折扣、總金額等
- **QuotationLine**
  - 報價明細行，對應某個 `ItemSku`、數量、單價、稅別、折扣等
- **QuotationStatusDef / QuotationStatus（Enum）**
  - 報價狀態定義（草稿、已送出、已核准、已逾期、已作廢…）
- **QuotationStatusHistory（預留）**
  - 狀態變化歷程，用於稽核與時間軸顯示（如之後需要）

---

## 2. QuotationThread（報價主線）

### 2.1 概念說明

`QuotationThread` 代表「一次詢價或專案」，其下可產生多次不同條件的報價版本（Revision），  
例如：

- 客戶第一次詢價 → 建立 Thread + Revision v1
- 客戶要求調整條件 → 在同一個 Thread 下再產生 Revision v2
- 最後客戶接受 v3 → 由該 Revision 轉成 Sales Order

### 2.2 主要欄位（概念版）

> 以下為「欄位概念」，實際型別與名稱需與後端 Entity 對應。

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| id | Long | Y | 主鍵 |
| threadNo | String | Y | 報價主線編號（可選擇是否與 Revision 共用編號策略） |
| ownerId | Long | Y | 所屬 Owner（公司 / 實體） |
| customerId | Long | Y | 客戶 ID（對應 Customer 模組） |
| topic / subject | String | Y | 報價主題（如專案名稱、用途） |
| description | String | N | 補充說明 |
| status | Enum / String | Y | 主線目前狀態（可由最後一次 Revision 狀態推導，或獨立定義） |
| activeRevisionId | Long | N | 目前「主要版本」之 Revision ID（例如客戶最後確認的版本） |
| salesOwnerId | Long | N | 負責業務（User） |
| salesTeamId | Long | N | 所屬業務團隊（Team） |
| createdBy / createdDate | - | Y | 審計欄位 |
| lastModifiedBy / lastModifiedDate | - | Y | 審計欄位 |
| deleted / deletedAt / deletedBy | - | N | 軟刪除欄位（若適用） |

> 是否需要 `threadNo` 可再思考：  
> - A. Thread 與 Revision 共用一組編號（例如：`QTN-20251119-00001` 專指某一版）  
> - B. Thread 有自己的號碼，Revision 有獨立版本號（需在 `numbering-spec` or 報價 spec 裡決策）  
> 一旦決定，請在 `DECISION_LOG.md` 記錄。

---

## 3. QuotationRevision（報價版本）

### 3.1 概念說明

`QuotationRevision` 代表在某一 Thread 下的一個具體報價版本：  
包含：

- 有效期限（ValidUntil）
- 幣別、稅別、總金額
- 付款條件、交期、Incoterm 等條款
- 多筆明細行（QuotationLine）

### 3.2 主要欄位（概念版）

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| id | Long | Y | 主鍵 |
| threadId | Long | Y | 所屬 QuotationThread |
| revisionNo | String | Y | 報價版本編號（例如 QTN-20251119-00001-REV01，或純序號） |
| revisionIndex | Integer | Y | 版本序號（1, 2, 3…） |
| status | Enum / String | Y | 此版本狀態（DRAFT / SUBMITTED / APPROVED / EXPIRED / CANCELLED…） |
| validFrom | LocalDate | N | 報價生效日起（可選） |
| validUntil | LocalDate | Y | 報價有效期（Workflow Guard 會檢查） |
| currencyCode | String | Y | 幣別（對應 Currency 模組） |
| taxIncluded | Boolean | Y | 單價是否含稅 |
| paymentTermId | Long | N | 付款條件（PaymentTermDef） |
| priceListId | Long | N | 參考的 PriceList ID（若有） |
| subtotalAmount | BigDecimal | N | 稅前小計（可由 Line 計算） |
| taxAmount | BigDecimal | N | 稅額（可由 Line 計算） |
| totalAmount | BigDecimal | N | 含稅總金額（可由 Line 計算） |
| discountRate | BigDecimal | N | 整體折扣率（例如 0.05 = 5%） |
| discountAmount | BigDecimal | N | 整體折扣金額 |
| notesInternal | String | N | 內部備註（客戶看不到） |
| notesExternal | String | N | 對客戶顯示的備註（報價單上會列出） |
| createdBy / createdDate | - | Y | 審計欄位 |
| lastModifiedBy / lastModifiedDate | - | Y | 審計欄位 |
| deleted / deletedAt / deletedBy | - | N | 軟刪除欄位（若適用） |

> 金額欄位可視實作決定是否全部持久化，或部分以 runtime 計算。  
> 若決定不存某些衍生欄位，需在本文件註明。

---

## 4. QuotationLine（報價明細）

### 4.1 概念說明

`QuotationLine` 代表單一品項的報價明細：

- 指向某個 `ItemSku`
- 有自己的數量、單價、折扣、稅別等
- 可額外包含描述、備註、行號順序

### 4.2 主要欄位（概念版）

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| id | Long | Y | 主鍵 |
| revisionId | Long | Y | 所屬 QuotationRevision |
| lineNo | Integer | Y | 行號（1, 2, 3…，用於排序與 UI 顯示） |
| itemSkuId | Long | Y | SKU ID（對應 Phase 1 Product 模組） |
| itemCode | String | N | 冗餘存放的品號（避免 SKU 刪除影響歷史報價） |
| itemName | String | N | 冗餘存放的品名 / 規格簡述 |
| quantity | BigDecimal | Y | 報價數量 |
| uomCode | String | Y | 單位（對應 UoM 模組） |
| unitPrice | BigDecimal | Y | 單價（未稅 / 含稅依 `taxIncluded` 而定） |
| discountRate | BigDecimal | N | 單行折扣率 |
| discountAmount | BigDecimal | N | 單行折扣金額 |
| netUnitPrice | BigDecimal | N | 折扣後單價（可存，可算） |
| lineSubtotal | BigDecimal | N | 單行小計（不含稅） |
| taxType | Enum / String | N | 稅別（一般稅、零稅率、免稅…） |
| taxRate | BigDecimal | N | 稅率（例如 5%） |
| taxAmount | BigDecimal | N | 單行稅額 |
| lineTotal | BigDecimal | N | 單行含稅總額 |
| remark | String | N | 行備註 |
| sortOrder | Integer | N | 額外排序欄位（若與 lineNo 分離） |

> `itemCode` / `itemName` 是否冗餘存放是重要決策：  
> - 若要確保歷史報價不受 SKU 改名影響 → 建議冗餘存放當時資訊  
> - 若不冗餘，報價單顯示會依當前 SKU 名稱變動  
> 一旦決策，請寫入 `DECISION_LOG.md`。

---

## 5. 報價狀態模型（Quotation Status Model）

### 5.1 QuotationStatus（Enum / 字串）

報價 Version 的狀態（`QuotationRevision.status`）建議使用 Enum（再映射為 DB String）：

- `DRAFT`：草稿中，可自由修改
- `SUBMITTED`：已送出給客戶，待回覆
- `APPROVED`：客戶確認接受此版本
- `REJECTED`：客戶明確拒絕（可選）
- `EXPIRED`：已逾有效期限（ValidUntil 之後）
- `CANCELLED`：由業務或系統端作廢

> 詳細狀態轉換與 Guard 條件會在 `workflow-spec.md` 定義。

### 5.2 QuotationStatusDef（資料表，若需要）

若需要更彈性（可設定顯示名稱、排序、顏色等），可新增：

- `quotation_status_def` 表  
  包含：

| 欄位 | 說明 |
|------|------|
| id | 主鍵 |
| code | 狀態代碼（對應 Enum 或作為 Enum 替代） |
| name | 顯示名稱（例如「草稿」、「已送出」） |
| description | 說明 |
| orderNo | 顯示順序 |
| styleClass | 建議 CSS 樣式（例如 badge 顏色，前端可參考使用） |

> 若採用此方式，則 `QuotationRevision.status` 可以存 code（String），  
> 並由程式 or Join 取得對應顯示資訊。

---

## 6. 狀態歷史（QuotationStatusHistory）— 預留

### 6.1 概念說明

若需詳細記錄報價狀態變化的歷程與誰在何時操作，可設計：

`QuotationStatusHistory`（或 `QuotationRevisionStatusHistory`）

### 6.2 主要欄位（概念版）

| 欄位 | 類型 | 說明 |
|------|------|------|
| id | Long | 主鍵 |
| revisionId | Long | 所屬 Revision |
| fromStatus | String | 原狀態 |
| toStatus | String | 新狀態 |
| changedBy | String / Long | 操作人（User） |
| changedAt | Instant | 變更時間 |
| reason | String | 變更原因（例如作廢理由） |

> 是否在 Phase 4 就實作，視實際需求而定。  
> 若未實作，建議在 `workflow-spec.md` 的「審計」章節說明未納入原因。

---

## 7. DTO 設計建議（簡要）

### 7.1 Thread DTO

`QuotationThreadDTO` 建議包含：

- `id`
- `threadNo`
- `ownerId`
- `customerId` + `customerName`（冗餘顯示用）
- `topic`
- `status`
- `activeRevisionId`
- `activeRevisionStatus`
- `activeRevisionTotalAmount`
- 審計欄位

> 列表頁可只用 ThreadDTO 顯示主資訊，點入後再載入 Revisions。

### 7.2 Revision DTO

`QuotationRevisionDTO` 建議包含：

- 所有主檔欄位（validUntil, currency, totals, notes…）
- 重點關聯資訊（如 priceListName, paymentTermName）
- 一個 `lines` 欄位（`List<QuotationLineDTO>`），  
  或在 API 層採「主檔 + 明細分開載入」，視 UI 需求而定。

### 7.3 Line DTO

`QuotationLineDTO` 包含：

- 欄位接近 Entity 欄位
- 前端 UI 必填欄位應在 Phase 4 `ui-spec.md` 中另行標示

---

## 8. 與其他模組的關聯（再次整理）

- **Customer（CRM 模組）**
  - `QuotationThread.customerId`
- **Product / Pricing（Phase 1）**
  - `QuotationLine.itemSkuId`, `uomCode`
  - `QuotationRevision.priceListId`（可選）
- **Owner / Team / User（Phase 0）**
  - `QuotationThread.ownerId`
  - `QuotationThread.salesOwnerId`
  - `QuotationThread.salesTeamId`
- **Sales Order（未來）**
  - SalesOrder 需能記錄來源：
    - `sourceQuotationThreadId`
    - `sourceQuotationRevisionId`

---

## 9. 待決策 / TODO

以下事項在真正實作前需要決策，建議列入 `DECISION_LOG.md` 或 ADR：

1. **Thread 與 Revision 的編號策略**
   - 是否 Thread 與 Revision 共用一組 `quotationNo`
   - 是否為 Thread 產編號，Revision 用純版本號（例如 v1, v2, v3）
2. **金額欄位是否完全持久化**
   - `subtotalAmount`, `taxAmount`, `totalAmount` 等  
   - 是否每次改明細時同步更新，或 runtime 計算
3. **Line 冗餘資訊程度**
   - 是否冗餘存放當時的 `itemCode`, `itemName`, `uomName` 等  
   - 若冗餘，可保留歷史報價樣貌不受商品變更影響
4. **StatusHistory 是否在 Phase 4 就納入**
   - 若延後，需標記在 spec

---

## 10. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立報價模組 Domain Model 初稿。 |

