# Phase 3 — 庫存管理測試策略（Inventory Test Strategy）

本文件定義 Flexora ERP **庫存管理（Inventory Management, IM）模組** 的測試策略與範圍，  
涵蓋：

- 單元測試（Unit Test）
- 整合測試（Integration Test）
- E2E / UI 測試建議
- 關鍵情境與錯誤碼覆蓋

目標：

1. 確保 **成本（Costing）、庫存餘量（Balance）、預留（Reservation）、補貨（Replenishment）** 行為符合規格  
2. 防止 refactor / 新功能破壞既有流程  
3. 提供 AI Agent / Codex 產生測試碼時的依據  

> 相關規格：  
> - `README.md`  
> - `domain-model.md`  
> - `costing-rules.md`  
> - `reservation-rules.md`  
> - `replenishment-rules.md`  
> - `api-spec.md`  
> - `ui-spec.md`  
> - `error-codes.md`  

---

## 1. 測試層級與目標

### 1.1 單元測試（Unit Test）

重點驗證：

- 成本計算服務（CostingService / InventoryTransactionService 中的成本邏輯）
- 庫存餘量更新邏輯（InventoryBalance 更新）
- 預留與 ATP 計算（InventoryReservationService）
- 補貨計算（ReplenishmentService）

### 1.2 整合測試（Integration Test）

重點驗證：

- REST API + DB + Security + Transaction 一致性  
- 入庫 / 出庫 / 調整 / 調撥 → Balance / CostLayer / Transaction 關聯正確  
- Reservation API 與 SO/MO 集成點  
- 補貨運算 API 輸出與資料狀態一致  

### 1.3 E2E / UI 測試（End-to-End / UI）

重點驗證：

- 在 `flexora-react-ui` 上的實際操作流程  
- 使用者在畫面上的行為是否能「準確觸發後端邏輯」  
- 關鍵視覺提示（低於安全庫存、超賣、補貨 Badge…）有正確顯示  

---

## 2. 單元測試（Unit Test）

### 2.1 建議測試類別

- `CostingService`（或同角色類別）：  
  - 處理 AVG / FIFO 計算、CostLayer 建立/消耗  
- `InventoryTransactionService`：  
  - 負責入庫（GR）、出庫（GI）、調整（ADJ）、調撥（TRANSFER）  
- `InventoryBalanceService`：  
  - 更新 onHand / reserved / available / onOrder  
- `InventoryReservationService`：  
  - 建立 / 更新 / 解除 Reservation 與 ATP 邏輯  
- `ReplenishmentService`：  
  - 依 `InventoryPolicy` 與 Balance 計算補貨建議  

### 2.2 成本計算（Costing）單元測試案例

#### 2.2.1 AVG 成本

1. **單次入庫更新平均成本**

- 前置：
  - oldQty=100, oldAvgCost=50
  - 新入庫：grQty=20, grUnitCost=60
- 驗證：
  - newAvgCost = (100*50 + 20*60) / 120 = 52
  - Balance.avgUnitCost 正確更新

2. **連續多次入庫**

- 模擬多次 GR，確認平均成本隨筆數動態調整且無精度異常。

3. **出庫成本（COGS, AVG）**

- 使用 currentAvgCost 計算：
  - 出庫 30 → COGS = 30 * avgCost
- 驗證：
  - Transaction.totalCost 正確
  - avgCost 不因出庫變化

4. **除以零防呆**

- oldQty=0, grQty=0 → 應拋出 `inventory.cost.moving_avg_zero_division` 或避免建立異常層。

#### 2.2.2 FIFO 成本

1. **單層消耗**

- costLayer：remainingQty=10, unitCost=100
- GI qty=6 →  
  - 應消耗 layer 1 的 6，剩 4
  - COGS = 6 * 100

2. **跨層消耗**

- L1: remaining=10 @100  
- L2: remaining=5  @110  
- GI qty=12 →  
  - L1: 10 全吃  
  - L2: 2 吃  
  - COGS = 10*100 + 2*110 = 1220  
  - L1.remaining=0, L2.remaining=3

3. **無層可用**

- 所有 remainingQty=0 或無層存在  
- 出庫 / 調撥 → 拋 `inventory.cost.no_layer_to_consume`

4. **調撥成本**

- 來源倉 FIFO 消耗多層 → 計算平均移出成本  
- 目的倉建立新層，unitCost = consumedTotalCost / qty  

### 2.3 庫存餘量更新（Balance）單元測試案例

1. **入庫更新 onHand**

- 異動前 onHand=50  
- GR qty=20 → onHand=70

2. **出庫更新 onHand**

- onHand=50  
- GI qty=30 → onHand=20  
- 若不允許負庫存，GI qty>onHand → 拋 `inventory.negative_stock_not_allowed`

3. **預留更新 reserved / available**

- reserved 開始前：onHand=100, reserved=20  
- 建立 Reservation: qty=30 → reserved=50, available=50  
- 解除 Reservation: qty=10 → reserved=40, available=60

### 2.4 Reservation / ATP 單元測試案例

1. **建立 Reservation（SO）**

- available ≥ requested → 建立成功  
- 驗證：
  - Reservation.status=OPEN
  - Balance.reservedQty 增加
  - Balance.available=onHand-reserved

2. **可用量不足**

- available < requested  
- 若 Tenant 禁止超賣 → 拋 `inventory.reservation.insufficient_available`

3. **解除 Reservation**

- OPEN → RELEASED  
- reservedQty 減少，available 增加

4. **CLOSED Reservation 不可再操作**

- 對 CLOSED 重新 release → `inventory.reservation.invalid_status`

### 2.5 補貨計算（Replenishment）單元測試案例

依 `replenishment-rules.md`：

1. **FIXED 策略**

- maxStock=200, onHand=50, onOrder=10  
- reorderPoint=40, available=30 ≤ ROP  
- 建議量：200 - (50+10) = 140

2. **安全庫存強制補貨**

- available=15, safetyStock=20 → 即使 onOrder>0 仍需補貨  
- reason=BELOW_SAFETY

3. **LAST_X_DAYS 策略**

- 過去 30 天消耗 120 → avgDaily=4  
- leadTime=10, buffer=3 → suggested=4*13=52

4. **EOQ 資料不足**

- 缺少 S/H 常數 → 拋 `inventory.replenishment.insufficient_data` 或回退 FIXED（依決策）

---

## 3. 整合測試（Integration Test）

### 3.1 工具與基礎

- Spring Boot Test + Testcontainers 或 H2（建議 Postgres Testcontainers）
- MockMvc 或 WebTestClient
- 啟用 Security（JWT），使用測試用帳號：

  - `inventoryUser`（ROLE_INVENTORY_USER）
  - `inventoryManager`（ROLE_INVENTORY_MANAGER）

### 3.2 入庫 / 出庫整合流程

#### 3.2.1 入庫（GR）整合測試

流程：

1. 預建 Warehouse + ItemSku + InventoryBalance
2. 呼叫對應入庫 API（實務上可能是 PO Receipt API，而非直接 inventory API）
3. 驗證：
   - InventoryTransaction 被建立
   - InventoryBalance.onHand 正確增加
   - CostLayer 建立一筆，remainingQty=receivedQty
   - AVG / FIFO 資料一致（依 SKU 成本法）

#### 3.2.2 出庫（GI）整合測試

1. 流程：

   - 先建立數筆 CostLayer + Balance
   - 呼叫出庫 API（例如 DeliveryNote 發佈）
   - 驗證：
     - onHand 減少
     - InventoryTransaction 正確記錄 before/after
     - FIFO 消耗對應剩餘層 / AVG 僅使用 avgCost

2. 無庫存時：

   - 預設不允許負庫存 → 應得到 400 + `inventory.negative_stock_not_allowed`

### 3.3 調整 / 調撥整合測試

#### 3.3.1 調整（Adjustment）

- `POST /api/inventory-transactions/adjustment`：

  - qty > 0 → ADJ_IN + CostLayer  
  - qty < 0 → ADJ_OUT（須檢查成本與 onHand）

- 驗證：
  - TransactionType 正確
  - Balance 更新正確
  - 成本計算符合 `costing-rules.md`

#### 3.3.2 調撥（Transfer）

- `POST /api/inventory-transactions/transfer`：

  - 來源倉 TR_OUT
  - 目的倉 TR_IN

- 驗證：
  - 兩筆 Transaction 都寫入
  - 兩倉 Balance 分別更新
  - 目的倉 CostLayer 成本 = 來源消耗成本平均值  
  - 來源倉 FIFO 層 remainingQty 正確調整

### 3.4 Reservation / ATP 整合測試

#### 3.4.1 SO 預留同步（sync）

- 模擬建立 `SalesOrderLine`（可使用測試 stub 或 fixture）
- 呼叫：

  - `POST /api/inventory-reservations/so-line/{id}/sync`

- 驗證：

  - 產生 Reservation 記錄
  - Balance.reservedQty 與 available 正確更新
  - 若 available 不足 → 400 + `inventory.reservation.insufficient_available`

#### 3.4.2 SO 取消解除預留（release）

- 呼叫：

  - `POST /api/inventory-reservations/so-line/{id}/release`

- 驗證：

  - Reservation.status = RELEASED
  - reservedQty 減少，available 增加

#### 3.4.3 Reservation Rebuild

- `POST /api/inventory-reservations/rebuild`  
- 先造一些錯誤資料，再看 rebuild 是否能重新同步為正確 Reserved 數量。

### 3.5 補貨計算與建議整合測試

#### 3.5.1 補貨運算（Run）

- 呼叫：

  - `POST /api/inventory/replenishments/run`（dryRun=true 與 false 都測）

- 驗證：

  - dryRun=true → 不建 `ReplenishmentRequest`，僅回傳建議
  - dryRun=false → 建立 `ReplenishmentRequest`，並可用 `GET /api/replenishment-requests` 查到

#### 3.5.2 補貨建議確認 / 取消

- `POST /api/replenishment-requests/{id}/confirm`
- `POST /api/replenishment-requests/{id}/cancel`

驗證：

- status 正確更新（OPEN → CONFIRMED / CANCELLED）
- 權限不足時 → 403 / `inventory.permission.denied`

---

## 4. E2E / UI 測試（End-to-End / UI）

> 可逐步導入（Playwright / Cypress 等），  
> 若尚未導入自動化，可先用此清單作為手動 Smoke Test。

### 4.1 建議工具

- Playwright / Cypress（擇一）
- 針對 `flexora-react-ui` 的以下畫面：

  - Warehouse List  
  - Inventory Overview  
  - Item Inventory Detail  
  - Inventory Transactions  
  - Adjust/Transfer Dialog  
  - Replenishment List  

### 4.2 必測流程（高優先）

1. **倉庫管理**

   - 新增倉庫 → 出現在所有倉庫下拉  
   - 停用倉庫 → 試圖對該倉庫操作調整/調撥 → 顯示錯誤 / 禁止

2. **庫存總覽**

   - 查詢指定 SKU + 倉庫 → onHand/reserved/available 顯示正確  
   - 低於安全庫存 → 顯示紅色 Badge  
   - 低於補貨點 → 顯示橘色 Badge  

3. **庫存明細（Item Inventory Detail）**

   - 由總覽點「查看明細」進入  
   - 各倉庫數量與總覽一致  
   - 下方交易列表可查看最近異動

4. **調整 / 調撥表單**

   - 輸入不合法數值（0 / 負數） → 前端防呆  
   - 調撥來源=目的 → 顯示錯誤  
   - 調整後回到總覽畫面，數量有更新

5. **預留檢視**

   - 點某 SKU → 進入「預留」Tab  
   - 顯示每筆 Reservation 的來源 SO/MO  
   - 手動解除預留（若開放）後，available 立刻更新

6. **補貨建議**

   - 執行補貨試算（dryRun） → 畫面顯示預計建議  
   - 執行實際補貨 → ReplenishmentRequest List 出現記錄  
   - 對某建議做「確認 / 取消」，狀態 Badge 有變更

---

## 5. 測試資料與 Fixture 設計

### 5.1 基本測試資料

建議預先準備以下測試資料（可經由 Liquibase / TestDataInitializer）：

- Warehouse：
  - `FG-TW`（成品倉）
  - `RM-TW`（原料倉）
- ItemSku：
  - `SKU-AVG-01`（成本法：AVG）
  - `SKU-FIFO-01`（成本法：FIFO）
- InventoryPolicy:
  - safetyStock / reorderPoint / maxStock / strategy 各有代表性設定
- 使用者：
  - `inventoryUser`（ROLE_INVENTORY_USER）
  - `inventoryManager`（ROLE_INVENTORY_MANAGER）

### 5.2 測試資料重用原則

- 單元測試可自行建立 in-memory Entity，不必依賴 DB  
- 整合測試與 E2E → 重用同一份固定測試資料，  
  利於斷言 onHand / reserved / available / 建議量的結果。

---

## 6. 覆蓋率與驗證目標

### 6.1 程式碼覆蓋率（建議）

- Service 層（Costing / Transaction / Reservation / Replenishment）：
  - **80%+** line coverage
- Controller 層：
  - 各主 API 至少有：
    - 1 個成功案例（200/201）
    - 1 個失敗案例（400/403/404）

### 6.2 錯誤碼覆蓋

- `error-codes.md` 中的主要錯誤碼（特別是 `inventory.cost.*`, `inventory.reservation.*`, `inventory.replenishment.*`）  
  至少各有一個測試可以確定被觸發，避免成為「死碼」。

---

## 7. 與其他文件的關聯

本測試策略與以下文件緊密對應：

- `domain-model.md`（實體欄位）  
- `costing-rules.md`（AVG / FIFO / CostLayer 規則）  
- `reservation-rules.md`（預留與 ATP 規則）  
- `replenishment-rules.md`（補貨計算規則）  
- `api-spec.md`（REST 介面）  
- `ui-spec.md`（畫面行為）  
- `error-codes.md`（錯誤碼）  

任何上述文件變更時，  
需同步檢查測試案例是否需要新增 / 調整。

---

## 8. 版本與歷程（History）

| 版本 | 日期       | 編輯者 | 說明 |
|------|------------|--------|------|
| v0.1 | 2025-11-20 | Jimmy  | 建立庫存管理測試策略初稿。 |

