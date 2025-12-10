# Product UI Implementation Plan

> 建立日期：2025-12-09  
> 狀態：進行中

本文件規劃 Product 模組 UI 的實作藍圖，考量現有後端限制和設計債務。

---

## 現有實作狀態

### ✅ 已完成

| 元件 | 說明 | 檔案 |
|------|------|------|
| ProductWorkspace | 清單頁（Card/Table 切換、Item/SKU 視圖） | `ProductWorkspace.tsx` |
| ProductDetail | 詳情 Overlay（API 整合） | `ProductDetail.tsx` |
| ProductEditorPage | 滿版編輯器（Item + SKU inline editing） | `ProductEditorPage.tsx` |
| ProductEditorWrapper | 編輯器資料載入層 | `ProductEditorWrapper.tsx` |
| ProductQuickCreateDrawer | 快速建立 Drawer | `ProductQuickCreateDrawer.tsx` |

### 🚧 進行中 / 待完成

| 元件 | 狀態 | 依賴 |
|------|------|------|
| Variant Generator UI | 待後端 API | `/api/item-skus/variants` |
| Pipeline View | 待實作 | - |
| 實際 API 儲存 | Mock 中 | 後端 Transaction API |

---

## Phase 1：MVP（目前）

**目標**：基本 CRUD 功能，使用現有 API

### Item / SKU Workspace 架構

目前 plan 將 Product 模組拆成兩條 workspace：
- `ProductWorkspace` / `/inventory/products`：專注 Item（款式）資料、Item Detail/Editor 以及 Item 層級的擴充欄位與統計。  
- `SkuWorkspace` / `/inventory/skus`：SKU-first layout（`SkuManagementPage`、Drawer、Variant Generator），直接支援 SKU 查詢與操作。  

Item 詳情頁的「管理所有 SKU」入口與 Workspace header 中的快捷按鈕會指向 `SkuWorkspace`，讓使用者不需要先進入 Item 就能查 SKU。  

### Navigation 更新

- Inventory / Sales 的側邊攔現在額外提供 `SKUs` 項目（icon：Grid），分別對應 `/inventory/skus` 與 `/sales/skus`，讓使用者可直接進入 SKU workspace。  

### Item 管理

```
ProductWorkspace
├── Item 列表（Card / Table）
├── SKU 列表（Card / Table）
├── Preset 篩選（全部 / 已上架 / 停售 / 有變體）
└── 進階篩選面板

ProductDetail (Overlay)
├── 摘要 Tab
├── SKU 列表 Tab
├── 價目表 Tab（待 API）
└── 附件 Tab（待實作）

ProductEditorPage (滿版)
├── Sticky Header（返回/狀態/操作）
├── 基本資訊 Card
├── SKU 列表 Card（展開式 inline 編輯）
└── 統計資訊 Sidebar
```

### 現有流程

1. **建立產品**：清單頁「建立產品」→ 編輯器（新建模式）→ 儲存 → 返回 Detail
2. **編輯產品**：清單頁點選商品 → Detail Overlay → 編輯按鈕 → 編輯器 → 儲存 → 返回 Detail
3. **快速建立**：清單頁「快速建立」→ Drawer → 儲存 → 刷新列表

### SKU Workspace 快捷與整合

新建立的 `SkuWorkspace`（`/inventory/skus`）已按照報價單 workspace 的視覺語彙重建，包含：

- **Filter Hub + advanced panel**：左側 `ModuleSidebar` 使用 `productPresets`，可切換多個 preset，中央 advanced panel 則提供關鍵字、Owner Search、供應策略與 Enabled 狀態 pills。所有欄位需點選「套用」才能觸發查詢，並可透過 chips 或「清除」按鈕針對單一條件復位；API 查詢時會帶入 `keyword`/`ownerId`/`supplyMode`/`enabled` 以及 preset 參數，確保與後端 `ItemSkuCriteria`/`ItemPreset` 保持一致；關鍵字搜尋需自定義 Criteria（類似 `QuotationThreadCriteria` 加入 `search` 字段並在 `QuotationThreadQueryService.createSpecification` 中以 `Specification` 合併 `skuNo`/`ThreadName`/`Owner`等欄位），以免 `keyword=` 沒法對應資料。
- **雙視圖與狀態提示**：列表支援 Table / Card 模式切換，顯示 `SKU No`/`Owner`/`Supply Mode`/`Enabled` 等欄位，查詢中顯示 spinner，無結果時呈現空狀態，API 錯誤會顯示 warning alert 並提供重試按鈕。
- **快速編輯 & 複製機制**：每筆 SKU 可從 List/Card 上啟動 `SkuEditDrawer`，只編輯單一 SKU 的編號、名稱、Owner、供應策略、旗標、Lead time、預設銷售數量，儲存前驗證必要欄位，成功後 toast 顯示更新訊息並同步列表，不會再在 Item 編輯頁做全量儲存。
- **變體產生器**：Header button `產生變體` 呼叫 `VariantGeneratorDialog`（目前 mock）；Dialog 用戶選取維度 → 預覽 SKU 組合 → 產出 SKU，後續會串 `/api/item-skus/variants`，並將結果加入 SKU Workspace，保持單一 workflow 供 Item/Variant 分離操作。

上述內容已同步更新在 `docs/modules/product/design.md` 的排版（Filter Hub/advanced panel、List / Card 兩欄在畫面中的擺放），而 `SkuEditDrawer` 未來會延伸支援由 `ItemExtAttrDef` / `ItemSkuExtAttrDef` API 定義的額外欄位，讓 `properties` 或 `extAttrs` 可以動態 render。

為了讓快速編輯能夠真正落地，Drawer 會直接連到 `ProductService` 的 `createItemSku`/`updateItemSku`，編輯時攜帶原始 `item.id` 與 `owner.id`，新增時則搭配 `ItemLookupSelect`（搜尋 ItemNo / ItemName）與 `OwnerLookupSelect`（limit=20、可自訂限制）選出關聯；輸出 payload 則會把 `properties` 經 `stringifyExtAttrValues` 包成 JSON，必要欄位如 `enabled`/`deleted`/`allowSales`/`maintainStock` 由後端 `normalize*`/`applyDefaultFlags` 補值，避免 NotNull 驗證失敗。儲存完成後顯示 toast、重新呼叫 `fetchItemSkus` 並把最新資料套回 Table/Card。

此外，變體產生器在 Step 1 改用 `ItemSkuExtAttrDef` 當作可選維度，Step 2 預覽會同時呈現維度與狀態，Step 3 結果可附帶 `properties`，未來直接串 `/api/item-skus/variants` 回傳的清單；整體 Owner 判斷也會支援名稱關鍵字與 lookup 雙軌查詢，在 URL 與 UI 上保持 `preset` + `search` + `owner` 的同步狀態，讓使用者可以用分享連結複製目前畫面。

### 變體功能同步規劃

- 在 Phase 1 內同時啟動變體 UI 規劃，讓 SKU 管理頁能支援「選取維度 → 預覽組合 → 生成 SKU」的流程，UI 與 API 規格可同步討論；即使後端 `/api/item-skus/variants` 尚未完成，也可先用 mock data 驗證畫面流程與欄位。  
- 這段規劃會包括 `VariantGeneratorDialog` 的三步驟（維度選擇/組合預覽/結果）、`DimensionSelector` 與 `SkuPreviewTable` 等共用元件，並延伸至 SKU 管理頁的快速操作列（「勾選生成」、「導入模板」等）。  
- 若變體 API 最後延至 Phase 2，再視實際情況把這部分視為 Phase 1 的延伸項目，但目前先同步設計與資料結構，盡量提前消化前端需求。
- 目前已經在 `SkuManagementPage` 裡放上「產生變體」按鈕，會呼叫 `VariantGeneratorDialog` 處理 mock 預覽並提示要送出的 SKU，後續可直接在該 Drawer 內觸發 `/api/item-skus/variants` 並將結果注入 SKU 清單。

---

## Phase 2：變體功能

**依賴**：後端 Variant API + Design Debt 解決

### 設計考量

根據 [design-debt.md](./design-debt.md)，變體功能需考量：

1. **解耦 Bundle 與 Variant**：UI 先按解耦邏輯設計
2. **VariantBasedOn 限制**：目前只支援 ITEM_ATTRIBUTE

### Variant Generator 流程

```
Step 1: 選擇維度
┌─────────────────────────────┐
│ 維度名稱        可選值       │
│ ☑ Color        [Red][Blue] │
│ ☑ Size         [S][M][L]   │
│ ☐ Material                 │
└─────────────────────────────┘

Step 2: 預覽組合
┌─────────────────────────────┐
│ SKU 編號          勾選      │
│ SKU-001-RED-S     ☑        │
│ SKU-001-RED-M     ☑        │
│ SKU-001-RED-L     ☑        │
│ SKU-001-BLUE-S    ☐        │
│ ...                         │
└─────────────────────────────┘

Step 3: 產生 SKU
→ 呼叫 POST /api/item-skus/variants
→ 返回 SKU 列表
```

### UI 元件規劃

| 元件 | 說明 | 位置 |
|------|------|------|
| VariantGeneratorDialog | 變體產生精靈（3 步驟） | ProductEditorPage 內 |
| DimensionSelector | 維度選擇器（樹狀） | Step 1 |
| SkuPreviewTable | SKU 預覽表格（可勾選） | Step 2 |

---

## Phase 3：Bundle 功能

**依賴**：後端 JDL 修正 + Bundle Component API

### 目標

支援「固定套組」商品建立，無需強制 `hasVariants=true`

### Bundle 編輯器

```
Bundle Editor
├── 基本資訊（同 Item）
├── 組成品列表
│   ├── Item Picker（搜尋選擇）
│   ├── 數量欄位
│   └── 排序拖拉
└── 定價設定（套組價 vs 組成品合計）
```

---

## Phase 4：進階功能

| 功能 | 說明 | 優先級 |
|------|------|--------|
| Pipeline View | Kanban 式 Item 狀態看板 | P2 |
| 批次匯入 | CSV 匯入 Item/SKU | P2 |
| 價目表整合 | PriceList 快速設定 | P2 |
| 審批流程 | Item 上架審批 | P3 |

---

## 技術備註

### API 整合狀態

| API | 狀態 | 用途 |
|-----|------|------|
| `GET /api/items` | ✅ 已整合 | 列表 |
| `GET /api/items/{id}` | ✅ 已整合 | 詳情 |
| `GET /api/item-skus` | ✅ 已整合 | SKU 列表 |
| `POST /api/items` | 🚧 Mock | 建立 |
| `PUT /api/items/{id}` | 🚧 Mock | 更新 |
| `DELETE /api/items/{id}` | 🚧 Mock | 刪除 |
| `POST /api/item-skus/variants` | ❌ 未實作 | 變體產生 |

### 檔案結構

```
src/views/product/
├── ProductWorkspace.tsx       # 清單主頁
├── productPresets.ts          # Preset 定義
├── components/
│   ├── ProductDetail.tsx      # 詳情 Overlay
│   ├── ProductEditorPage.tsx  # 滿版編輯器
│   ├── ProductEditorWrapper.tsx # 編輯器 Wrapper
│   └── ProductQuickCreateDrawer.tsx # 快速建立
└── (future)
    ├── VariantGeneratorDialog.tsx  # 變體產生器
    └── BundleEditor.tsx            # 套組編輯器
```
