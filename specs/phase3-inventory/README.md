# Phase 3 — 庫存管理（Inventory Management, IM）規格總覽

本文件是 **Flexora ERP 庫存管理（IM）模組** 的總覽說明與索引。  
詳細規格請依後續拆分文件（domain / workflow / costing / replenishment / API / UI / 測試…）查看。

> 完整版設計背景請參考：  
> 《庫存管理（IM）完整規格書 — 含成本估值設計》（原始長版文件）  
> 本目錄下的各檔案，則是將該規格拆解成「開發友善、Agent 友善」的子文件。

---

## 1. 模組目標與定位

### 1.1 模組目標

1. **統一管理所有實體庫存與可承諾庫存（ATP）**
   - 支援 B2B、B2C、接單生產（MTO）與一般備貨（MTS）
   - 能清楚回答：「現在每個倉、每個 SKU 還有多少？可以賣多少？」

2. **結合成本估值與 Cost Layer**
   - Tenant 預設使用 **加權平均（Moving Average）** 做會計過帳（COGS）
   - **SKU 層級可覆寫為 FIFO**
   - **所有入庫一律建立 Cost Layer**：同時可以跑 FIFO / AVG 報表，並做成本分析

3. **多倉與多維度庫存策略**
   - 每一個 SKU 可以有：
     - Global 預設策略
     - Per-SKU 覆寫
     - Per-Warehouse 覆寫
   - **庫存政策（安全庫存、最大庫存、補貨點）可分層設定**

4. **與採購 / 製造 / 銷售緊密聯動**
   - 銷售：Sales Order 建立 → 建立 / 調整 **InventoryReservation**
   - 採購：Purchase Order 收貨 → 產生入庫 **InventoryTransaction + CostLayer**
   - 製造：MO 投料 / 完工 → 影響原料與產成品庫存、成本分層

5. **為未來 WMS 擴充預留**
   - 目前以「Warehouse / Bin 基礎倉位」為主
   - 未來可往波次揀貨、條碼、貨格、序號管理擴充

---

## 2. 成本策略與估值（Executive Summary 壓縮版）

1. **Tenant 預設成本法**：加權平均（Moving Average）
2. **SKU 覆寫選項**：
   - `COSTING_AVG`（預設）
   - `COSTING_FIFO`
3. **Cost Layer 永久保存**
   - 每一筆入庫（PO 收貨、MO 入庫、調整調整…）建立一筆 `CostLayer`
   - `CostLayer` 最小粒度：`ItemSku × Warehouse × LayerId`
   - 報表可以選擇：
     - 用 AVG 視角看庫存成本
     - 用 FIFO 視角看庫存成本
4. **多倉成本隔離**
   - **加權平均與 CostLayer 以 `SKU × Warehouse` 維度管理**
   - 避免跨倉調撥時污染平均成本

> 詳細欄位與公式會在 `costing-rules.md` 中展開。

---

## 3. 主要 Domain 與概念（高階）

詳細欄位將在 `domain-model.md` 裡定義，這裡先列出「模組內的主角」：

- **Warehouse（倉庫）**
  - 代表實體或邏輯倉（例如：成品倉、原料倉、在途倉）

- **WarehouseBin（倉位，預留）**
  - 對應倉庫內更細部位置（預留 WMS 擴充）

- **InventoryBalance / StockByBin**
  - `ItemSku × Warehouse`（或 × Bin）的現有庫存數量
  - 包含 on-hand、allocated、available 等欄位

- **InventoryReservation（庫存預留）**
  - 對應 Sales Order / MO / 其他需求單的預留量
  - 區分：
    - **Hard Reservation**：鎖在特定倉/批次/Layer
    - **Soft Reservation**：僅鎖 SKU，在系統內保留

- **InventoryTransaction（庫存交易）**
  - 每一筆進出庫動作的紀錄（GR、GI、Transfer、Adjustment…）
  - 與 CostLayer 連動

- **CostLayer（成本層）**
  - 每一個入庫批次的成本記錄
  - FIFO / AVG 的基礎資料來源

- **ReplenishmentRequest / ReorderProposal（補貨建議）**
  - 依安全庫存 / 補貨點，自動產生「建議採購」或「建議製造」

- **InventoryPolicy（庫存策略）**
  - 每 SKU / 每 Warehouse 的安全庫存、補貨點、最大庫存設定

---

## 4. 流程與場景摘要（Overview）

### 4.1 庫存主要來源與去向

- **入庫（Increase）**
  - Purchase Receipt（PO 收貨）
  - Manufacturing Order 完工入庫
  - Inventory Adjustment（盤點調整）
  - Transfer In（調撥入）

- **出庫（Decrease）**
  - Sales Delivery / Delivery Note
  - Manufacturing Issue（MO 投料）
  - Inventory Adjustment（盤點調整）
  - Transfer Out（調撥出）

每一次入庫 → 產生 `InventoryTransaction` + `CostLayer`  
每一次出庫 → 產生 `InventoryTransaction`，並根據 AVG / FIFO 產生對應成本消耗。

### 4.2 需求與預留

- Sales Order 建立 → 建立 / 更新 `InventoryReservation`
- Reservation 會影響 UI 顯示的：
  - On-hand（實際庫存）
  - Reserved（已分配給訂單）
  - Available（可再承諾）

### 4.3 補貨建議（Replenishment）

- 系統定期或使用者手動執行「補貨計算」：
  - 以 `On-hand + On-order - Reserved` 與安全庫存 / 補貨點做比較
  - 輸出 `ReplenishmentRequest` 列表（建議採購 / 製造量）

---

## 5. 文件結構（本目錄規劃）

本目錄預計拆分為多個子文件（類似報價模組）：

```text
specs/phase3-inventory/
  README.md              ← 本檔：IM 總覽與索引
  domain-model.md        ← Entity / DTO 資料模型與關聯（Warehouse、Stock、Txn、Layer…）
  workflow-spec.md       ← 庫存相關流程與狀態（入庫 / 出庫 / 調撥 / 預留）
  costing-rules.md       ← 成本估值與 Cost Layer 規則（AVG / FIFO / SKU 覆寫）
  reservation-rules.md   ← 預留（Reservation）與可承諾庫存（ATP）邏輯
  replenishment-rules.md ← 補貨計算邏輯與建議產生（Reorder）
  api-spec.md            ← IM REST API（查庫存、異動、預留、補貨建議…）
  ui-spec.md             ← 前端 UI 規格（倉庫庫存總覽、庫存卡片、異動畫面…）
  error-codes.md         ← IM 模組專屬錯誤碼表
  test-strategy.md       ← 測試策略（單元 / 整合 / 場景）
````

> 現在先完成 `README.md` 作為入口，
> 後續可以依開發優先順序，分別展開 `domain-model`, `costing-rules`, `reservation-rules` 等。

---

## 6. 與其他模組的關聯

* **Phase 1 定價（Pricing）**

  * 成本估值不等同於銷售價格，但會成為毛利分析基礎
* **Phase 4 報價（Quotation）**

  * 報價可以查詢「當前庫存 / 可承諾量」作為參考
* **Sales Order / Delivery Note**

  * 出貨會減少庫存、消耗 CostLayer
* **Purchase Order / Receipt**

  * 收貨會增加庫存、建立 CostLayer
* **Manufacturing（未來）**

  * MO 投料與完工分別對應原料耗用與產成品入庫

---

## 7. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                       |
| ---- | ---------- | ----- | ------------------------ |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 3 庫存管理模組規格總覽初稿。 |

## 8. 合併來源與程式碼參照

- 本目錄已逐步整併自 `docs/migration/flexora-react/庫存管理 Inventory Management（IM）規格書.md`；如需查驗細部描述，請同步參考該備份檔案。
- 當 `costing-rules.md`、`workflow-spec.md` 或其他子檔與程式碼出現差異時，以 `flexora-react/src/main/java/com/asynctide/flexora/service/InventoryTransactionService.java`, `InventoryCostingService`, `InventorySettingService`, `CostLayerRepository` 等實作為權威，按需更新本目錄。
- 本 README 之後如需補充更詳細欄位表、API 路徑、流程圖，可繼續從 legacy doc 摘錄並於末尾附註 `Derived from flexora-react/docs/...`，確保追溯性。
