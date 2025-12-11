# Product Module – Design Debt & Clarifications

> 建立日期：2025-12-09  
> 狀態：待討論

本文件記錄 Product 模組在現有 JDL/後端設計中發現的設計債務與澄清事項。

---

## 1. Product Bundle 與 Variant 的耦合問題

### 現狀

根據 `jdl/04-product.jdl` (lines 73-90) 的**註解描述**：

```
productBundle = true → 強制 hasVariants = true
```

### 查證結果 (2025-12-09)

經查證實際程式碼，**此限制只存在於 JDL 文件註解，並未實際強制執行**：

| 檢查項目 | 結果 |
|----------|------|
| Liquibase changelog (`20251002062526_added_entity_ItemSku.xml`) | ❌ 無 check constraint |
| `ItemSkuService.java` | ❌ 無業務驗證邏輯 |
| 資料庫 `item_sku` 表 | `product_bundle` 和 Item 的 `has_variants` 為獨立欄位 |

**結論**：JDL 註解是原作者的「設計意圖」，但實際並未在資料庫或服務層強制。前端 UI 可自由設計，不受此限制約束。

### 原始問題分析（備參）

原作者的設計意圖是將「綑綁商品」視為「變體容器」，但這混淆了兩個不同的概念：

| 概念 | 用途 | 實際需求 |
|------|------|----------|
| **變體商品** (hasVariants=true) | 同款商品不同規格（顏色/尺寸） | SKU 層管理庫存，主檔不可直接銷售 |
| **綑綁商品** (productBundle=true) | 多個獨立商品組成套組 | 組成品管理庫存，套組本身可銷售 |

**實務場景：**

- ✅ **固定套組**：「禮盒 A = 洗髮精 + 護髮素」，無需變體，直接銷售
- ✅ **可選套組**：「自選禮盒 = 任選 3 樣商品」，需要變體機制
- ❌ **目前限制**：無法建立「固定套組」，因為 `productBundle=true` 強制 `hasVariants=true`

### 建議修正方案

**解耦 `productBundle` 和 `hasVariants`：**

| 場景 | productBundle | hasVariants | maintainStock | allowSales | 說明 |
|------|---------------|-------------|---------------|------------|------|
| 一般商品 | false | false | true | true | 標準單品 |
| 變體商品（主檔）| false | true | false | false | 需銷售 SKU |
| 固定套組 | true | false | false | true | 可直接銷售 |
| 可配置套組 | true | true | false | true | 可選配件 |

**保留的規則：**
1. `productBundle = true` → `maintainStock = false`（庫存由組成品管理）
2. `hasVariants = true` 且非 Bundle → 主檔 `allowSales = false`

### 行動項目

- [ ] **後端**：修改 JDL 移除 `productBundle → hasVariants=true` 的強制約束
- [ ] **驗證邏輯**：保留 `productBundle → maintainStock=false` 規則
- [ ] **前端**：UI 先按解耦後的邏輯設計，等後端調整後啟用

---

## 2. Variant 機制的現有限制

根據 JDL 分析，變體機制有以下限制：

### 2.1 變體來源（VariantBasedOn）

目前只支援單一值 `ITEM_ATTRIBUTE`，代表變體只能基於產品屬性產生。

```java
enum VariantBasedOn {
    ITEM_ATTRIBUTE // 唯一值
}
```

**影響**：無法支援以下變體類型：
- BOM 變體（組合不同物料）
- UoM 變體（不同計量單位的包裝）

### 2.2 變體屬性鏈結

變體必須依循完整鏈結：

```
Item → ItemAttribute → ItemAttributeValue → ItemVariantAttribute
```

- 變體屬性值必須先在 `ItemAttributeValue` 定義
- 不能在變體建立時自由輸入新屬性值

### 2.3 後端 API 缺口

Variant Generator UI 需要的 API 尚未實作：

- `POST /api/item-skus/variants` - 批次產生 SKU
- `GET /api/items/{id}/variant-dimensions` - 取得可用維度

---

## 3. 相關文件

- [spec.md](./spec.md) - 功能規格
- [ui-spec.md](./ui-spec.md) - UI 規格
- [api-spec.md](./api-spec.md) - API 規格

---

### 4. 最近 frontend 處理記錄

> 2025/12/14 更新：`fix/lint-errors` branch 已被清理並移除，後續更動會直接回推 `main-github`。

- 完整清理 `Contact`/`Organization`/`Campaign`/`Leads` 的表單/列表元件，補齊型別、搜尋欄位，讓 UI 可直接套用報價式的 preset + filter UX。
- 調整 `SKU Workspace` 及 Variant Generator/Drawer 的 props 排序與 hook 依賴；`ProductWorkspace`、`QuoteDetail`、`QuoteEditor` 加入 `useMemo/useCallback` 避免 lint 警告，`QuickCreateDrawer` 的 preview payload 也補充型別。
- 把 `Button_new` 的 variant 定義抽到 `buttonVariants.ts`，減少 Fast Refresh 警告；同時補上 `buttonVariants` 的 import 確保 `Button` 只輸出 component。
- `main-github` 已整合上述修改，並透過 `npm run lint` 驗證；若需要進一步紀錄，可在此段落補入屆時的測試/PR 連結。
