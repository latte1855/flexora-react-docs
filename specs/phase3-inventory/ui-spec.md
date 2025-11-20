# Phase 3 — 庫存管理 UI 規格（Inventory UI Specification）

本文件定義 Flexora ERP 在 `flexora-react-ui` 中  
**庫存管理（Inventory Management, IM）模組** 的畫面結構與行為，包括：

- 倉庫 / 庫存總覽畫面
- 單一 SKU 多倉庫分布與明細
- 庫存交易查詢
- 調整 / 調撥 UI
- 預留（Reservation）檢視與操作入口
- 補貨建議清單

> API 介面請參考：`api-spec.md`  
> Domain 與欄位請參考：`domain-model.md`  
> 成本規則：`costing-rules.md`  
> 預留 / 補貨：`reservation-rules.md`, `replenishment-rules.md`

---

## 0. 前端技術背景

- 正式前端專案：`flexora-react-ui`（Ecme React Tailwind Admin）
- JHipster 內建 React UI（在 `flexora-react`）：
  - 給後端開發人員測 API 用
  - 可作為欄位 / DTO 的參考樣本
  - 實際給使用者使用的畫面，以 `flexora-react-ui` 為準

---

## 1. 功能地圖（Screens Overview）

庫存模組 UI 主要包含以下畫面：

1. **Warehouse List** — 倉庫列表（維護倉庫主檔）
2. **Inventory Overview** — 庫存總覽（按 SKU × Warehouse）
3. **Item Inventory Detail** — 單一料號庫存詳情（含多倉分布）
4. **Inventory Transactions** — 庫存交易明細查詢
5. **Adjustments & Transfers** — 庫存調整 / 調撥操作畫面
6. **Reservations View** — 預留（Reservation）檢視畫面
7. **Replenishment Suggestions** — 補貨建議清單

> 所有畫面均使用一致的 Layout：左側主選單 + 上方 Breadcrumb + 卡片式內容。

---

## 2. Warehouse List（倉庫列表）

### 2.1 目的

- 維護各個倉庫的基本資料（倉庫代碼、名稱、類型、啟用狀態）
- 提供庫存總覽畫面的篩選來源（倉庫下拉）

### 2.2 UI 元素

- 頁首：
  - 標題：`倉庫管理`
  - 按鈕：
    - `新增倉庫`
- 查詢條件：
  - 倉庫代碼（文字模糊搜尋）
  - 倉庫名稱（文字模糊搜尋）
  - 類型（下拉：FINISHED_GOODS / RAW_MATERIAL / IN_TRANSIT…）
  - 是否啟用（Switch / Select：全部 / 僅啟用）
- 資料表欄位：
  - 倉庫代碼（warehouseCode）
  - 倉庫名稱（warehouseName）
  - 類型（type）
  - 虛擬倉（isVirtual：以 Tag 顯示）
  - 是否啟用（active Badge）
  - 備註（remark，超長顯示 Tooltip）
  - 操作：
    - 編輯
    - 停用 / 啟用

### 2.3 主要互動行為

- 新增 / 編輯倉庫：
  - Modal 或獨立畫面表單
  - 欄位對應 `Warehouse` Domain
  - 禁止任意修改 `warehouseCode`（若已被使用）
- 停用倉庫：
  - 若該倉庫仍有 On-hand 庫存 → 顯示警告 / 禁止停用

---

## 3. Inventory Overview（庫存總覽畫面）

### 3.1 目的

- 讓使用者快速掌握每一個 SKU 在各倉庫的庫存狀態：
  - On-hand / Reserved / Available / On-order
  - 是否低於安全庫存 / 補貨點

### 3.2 API 對應

- `GET /api/inventory-balances`（支援各種過濾）

### 3.3 UI 元素

- 頁首：
  - 標題：`庫存總覽`
- 查詢條件區：
  - 料號（ItemSku）：Autocomplete（可輸入 code / name）
  - 倉庫（Warehouse）：下拉（可多選）
  - 低於安全庫存（CheckBox：僅顯示低於 safetyStock）
  - 低於補貨點（CheckBox：僅顯示低於 reorderPoint）
  - 只看有庫存（onHand > 0）
- 資料表欄位建議：

| 欄位 | 說明 |
|------|------|
| SKU | 顯示 code + name |
| Warehouse | 倉庫代碼＋名稱 |
| On-hand | 現有庫存 |
| Reserved | 預留量 |
| Available | 可承諾庫存 |
| On-order | 在途量（PO+MO） |
| Safety Stock | 安全庫存 |
| Reorder Point | 補貨點 |
| Max Stock | 最大庫存 |
| 狀態指示 | Badge：低於安全庫存 / 低於補貨點 / 正常 |

- 操作欄：
  - `查看明細`（連到 Item Inventory Detail）
  - `查看交易`（直接帶入 SKU+Warehouse 到 Transaction 畫面）

### 3.4 視覺提示

- 若 `availableQty < 0` → 以紅色高亮整列或欄位（代表超賣）
- 若 `availableQty < safetyStock` → 顯示紅色 Badge：`低於安全庫存`
- 若 `availableQty <= reorderPoint` → 顯示橘色 Badge：`達補貨點`

---

## 4. Item Inventory Detail（單一料號庫存詳情）

### 4.1 目的

- 針對單一 SKU，檢視「各倉庫」的分布與趨勢
- 深入了解某個 SKU 是否需要調撥 / 補貨

### 4.2 導入方式

- 由 Inventory Overview 點選某 SKU 的「查看明細」
- 也可從 Product / Item SKU 詳情頁提供「庫存」Tab

### 4.3 版面結構建議

- 頂部：SKU 基本資訊卡片：
  - Code / Name / Spec
  - ItemType（PURCHASED / MANUFACTURED）
  - Cost 視角（AVG / FIFO）摘要（可選）
- 中間：多倉庫分布表格：

| 倉庫 | On-hand | Reserved | Available | On-order | Safety / ROP / Max | 補貨狀態 |

- 底部（可選 Tab）：
  - 最新庫存交易（最近 20 筆）
  - CostLayer（若 SKU 使用 FIFO 或需看平均成本歷史）

---

## 5. Inventory Transactions（庫存交易查詢）

### 5.1 目的

- 提供類似「總帳明細」的檢視，  
  讓使用者針對 SKU/倉庫/期間查詢所有出入庫紀錄。

### 5.2 API 對應

- `GET /api/inventory-transactions`

### 5.3 UI 元素

- 查詢條件：
  - SKU（Autocomplete）
  - 倉庫
  - 交易類型（多選：GR / GI / TR_IN / TR_OUT / ADJ_IN / ADJ_OUT / MO_IN / MO_OUT…）
  - 來源單據類型（PO / SO / DN / MO / ADJ）
  - 日期區間（起訖）
- 表格欄位：

| 欄位 | 說明 |
|------|------|
| 日期時間 | createdDate |
| SKU | itemSkuCode + name |
| Warehouse | 倉庫 |
| Type | 交易類型（顯示對應中文：收貨/出庫/調撥…） |
| Source | 來源（PO-9001, SO-1002, DN-2001…） |
| Qty | 數量（正負顯示） |
| Before / After | 異動前後庫存 |
| Unit Cost | 單位成本 |
| Total Cost | 總成成本 |
| Layer | CostLayer id（可連到成本層畫面） |

---

## 6. Adjustments & Transfers（調整 / 調撥畫面）

### 6.1 調整（Adjustment）UI

#### 6.1.1 目的

- 提供盤點差異 / 零星修正的入口，不需透過 PO / DN。

#### 6.1.2 API 對應

- `POST /api/inventory-transactions/adjustment`

#### 6.1.3 表單欄位

- SKU（Autocomplete，必填）
- 倉庫（Select，必填）
- 異動數量：
  - 正數 → 增加庫存（ADJ_IN）
  - 負數 → 減少庫存（ADJ_OUT）
- 調整類型（Select）：
  - STOCKTAKE（盤點）
  - CORRECTION（修正）
  - DAMAGE（損壞）
- 單位成本（unitCostOverride）：
  - 僅在 ADJ_IN 時可輸入
  - 若未填 → 使用 AVG / 預設策略
- 備註（reason）

#### 6.1.4 前端驗證

- SKU / 倉庫必填
- 數量不可為 0
- ADJ_OUT 且 `|qty| > onHand`：
  - 提醒使用者（可允許產生負庫存，依 Tenant 設定）
- ADJ_IN 且未輸入成本，但 Tenant 要求 → 顯示「請輸入單位成本」

---

### 6.2 調撥（Transfer）UI

#### 6.2.1 目的

- 在「倉庫間」調整庫存，常見情境：
  - 總公司 → 分公司
  - 成品倉 → 寄售倉

#### 6.2.2 API 對應

- `POST /api/inventory-transactions/transfer`

#### 6.2.3 表單欄位

- SKU（必填）
- 來源倉（sourceWarehouse，必填）
- 目的倉（targetWarehouse，必填，且不可與來源倉相同）
- 數量（必填）
- 備註（reason）

#### 6.2.4 前端驗證

- 來源與目的倉不可相同
- 數量 > 0
- 若 FIFO / AVG 都會檢查來源倉 onHand 是否足夠：
  - 不足 → 顯示警告（可禁止或允許負庫存，依 Tenant 設定）

---

## 7. Reservations View（預留檢視畫面）

### 7.1 目的

- 讓使用者可以看到某 SKU / 倉庫的「預留明細」，  
  知道庫存被哪些 SO / MO 佔用。

### 7.2 API 對應

- `GET /api/inventory-reservations`

### 7.3 UI 元素

可作為以下兩種呈現：

1. 獨立畫面：`庫存預留查詢`
2. 子畫面 Tab：在 Item Inventory Detail 底部的「預留」Tab

表格欄位：

| 欄位 | 說明 |
|------|------|
| SKU | itemSkuCode + name |
| Warehouse | 倉庫 |
| Source | 來源類型 + 單據（SO-xxx / MO-xxx） |
| reservedQty | 預留數量 |
| reservationType | HARD / SOFT |
| status | OPEN / RELEASED / CLOSED |
| createdDate | 建立時間 |
| releasedDate | 解除時間 |

操作（可能視權限開放）：

- `查看來源單`（連到 SO / MO 畫面）
- （選擇性）`手動解除預留`（呼叫 release API）

---

## 8. Replenishment Suggestions（補貨建議清單）

### 8.1 目的

- 將 `replenishment-rules.md` 計算出的建議結果可視化：
  - 哪些 SKU 缺貨、建議補多少
  - 可一鍵轉為採購請購 PR 或 MO

### 8.2 API 對應

- `POST /api/inventory/replenishments/run`（執行計算，dry-run 或真正寫入）
- `GET /api/replenishment-requests`（查詢已產生建議）

### 8.3 畫面 1：補貨運算執行畫面（可選）

- 「執行補貨計算」區塊：
  - 倉庫（多選）
  - SKU 範圍（可選）
  - 試算（dryRun）checkbox
  - 按鈕：`執行補貨計算`
- 下方顯示本次運算結果（不論 dryRun 或實際寫入）：
  - SKU / 倉庫
  - onHand / reserved / available / onOrder
  - Policy（safetyStock / ROP / maxStock）
  - 建議補貨量（suggestedQty）
  - 原因（reason Badge：LOW_STOCK / BELOW_SAFETY / BELOW_ROP…）
  - 策略（FIXED / EOQ / LAST_X_DAYS…）

### 8.4 畫面 2：補貨建議列表（ReplenishmentRequest List）

- 查詢條件：
  - 倉庫
  - SKU
  - 狀態（OPEN / CONFIRMED / CANCELLED）
  - 建立日期區間
- 表格欄位：

| 欄位 | 說明 |
|------|------|
| SKU | itemSkuCode + name |
| Warehouse | 倉庫 |
| suggestedQty | 建議補貨量 |
| reason | 產生原因 |
| status | OPEN / CONFIRMED / CANCELLED |
| createdDate | 建立時間 |

- 操作：
  - `轉為採購申請 PR`（若 SKU 為 PURCHASED）
  - `轉為製令 MO`（若 SKU 為 MANUFACTURED）
  - `標記為已處理（CONFIRMED）`
  - `取消（CANCELLED）`

> 實際 PR/MO 轉單邏輯由採購 / 製造模組實作，  
> 該動作需呼叫對應模組 API，並將 `replenishmentRequestId` 存為來源。

---

## 9. UX 細節與提醒

### 9.1 數字 / 狀態視覺

- 金額、數量採同一格式（3 位小數 or 0 位，依 Tenant 設定）
- 使用 Badge / Tag 區分：
  - 低於安全庫存（紅）
  - 低於補貨點（橘）
  - 正常（綠）

### 9.2 異常狀況

- `availableQty < 0`：
  - 在所有畫面統一以紅色醒目顯示
  - 提供 Tooltip：「可承諾庫存為負數，代表已超賣或資料需檢查」
- 調整、調撥數量過大：
  - 顯示確認對話框（含 Before / After 預覽）

---

## 10. 與其他模組 UI 的關聯

- **報價（Quotation）/ SO / DN 畫面**：
  - 商品選擇時可打開「庫存查詢」側邊抽屜（Drawer），  
    使用 `Inventory Overview` 的縮小版，快速顯示某 SKU 的可用庫存。
- **Product / Item 詳情頁**：
  - 加入「庫存」Tab，直接嵌入 `Item Inventory Detail` 畫面。

---

## 11. 文件同步規則

當以下情況發生時，必須同步更新本 `ui-spec.md`：

- API 結構（尤其是 InventoryBalance / Reservation / Replenishment DTO）變更
- 新增 / 移除庫存相關畫面
- 調整 Status / Badge 規則（如安全庫存判斷邏輯）

並檢查是否需要同步更新：

- `api-spec.md`
- `reservation-rules.md`
- `replenishment-rules.md`
- `error-codes.md`
- `test-strategy.md`

---

## 12. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-19 | Jimmy  | 建立庫存管理 UI 規格初稿。 |
