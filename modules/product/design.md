# Product (商品管理) 設計文件

## 1. 模組概述
管理商品款式 (Item/SPU) 與庫存單位 (ItemSku)。採用 **SKU-first** 策略，所有交易 (報價/銷售/採購) 皆指向 SKU。支援多規格變體、多單位 (UOM) 與擴充屬性 (ExtAttr)。

## 2. UI 結構定義

### 2.1 主頁 (Workspace)

- **元件**: `ModuleWorkspace`
- **主要設定**:
  - `title`: "Product Management"
  - `entityName`: "Item & SKU"

**視圖 (Views)**:
1.  **Item List (款式視圖)**:
    - 顯示商品母體 (Style)。
    - 用於管理品牌、分類、ExtAttr 定義。
    - 點擊進入 Item Detail Overlay (管理變體)。
2.  **SKU List (SKU 視圖)**:
    - 扁平化顯示所有 SKU。
    - 欄位: SKU No, Name, UOM, Supply Mode, 啟用狀態。
    - 用於快速查找與總覽。

**延伸 Workspace**：
- `SkuWorkspace`：單一 workspace 針對 SKU 顯示篩選/表格/Drawer，配合 Item workspace 保持 Item-only 的卡片與 Detail，使用者可直接由 `/inventory/skus` 進入 SKU-first workflow。

**Sidebar**:
- **Presets**: All, Enabled, My Brands.
- **Filters**: Brand, Category, Item Kind (Product/Service/Material).

### 2.2 Item 詳情頁 Overlay

- **Header**: Item No, Name, Brand.
- **Tabs**:
    - **摘要**: 基本屬性 (Category, Description), ExtAttr.
    - **變體 (Variants)**: SKU 列表 (Grid with inline edit capable).
    - **附件**: 圖片/文件.

### 2.3 SKU 詳情頁 Overlay

- **Header**: SKU No, Name, 狀態 Badge.
- **Tabs**:
    - **基本**: UOM, Supply Mode, Barcodes.
    - **控制**: 銷售/採購/庫存設定 (Allow Sales, Maintain Stock).
    - **價格**: 關聯的 Price List 價格 (唯讀視圖).
    - **庫存**: 當前各倉庫存 (唯讀視圖).

### 2.4 商品建立流程 (Drawer)

- **Quick Create Drawer**:
    - Step 1: 基本資訊 (Name, Code, Brand).
    - Step 2: 屬性設定 (ExtAttr).
    - Step 3 (Optional): 產生變體 (SKU Gen) - 設定規格 (Color/Size) 並自動產生 SKU 列表。

## 3. 類型定義 (Frontend)

### 3.1 Item

```ts
export interface Item {
  id: string;
  itemNo: string;
  itemName: string;
  brand?: Brand; // Ref
  category?: Category; // Ref
  
  hasVariants: boolean;
  variantBasedOn?: 'NONE' | 'COLOR_SIZE' | 'EXT_ATTR';
  
  properties: Record<string, any>; // JSONB ExtAttr
  
  owner: Owner;
  version: number;
}
```

### 3.2 ItemSku

```ts
export interface ItemSku {
  id: string;
  skuNo: string;
  skuName: string; // T-Shirt Red L
  
  item: Item; // Parent Item
  
  // 核心設定
  defaultUom: Uom;
  itemKind: 'PRODUCT' | 'MATERIAL' | 'SERVICE' | 'VIRTUAL';
  supplyMode: 'MTS' | 'MTO' | 'ATO';
  
  // 開關
  enabled: boolean;
  allowSales: boolean;
  maintainStock: boolean;
  productBundle: boolean;
  
  // 數據
  barcode?: string; // Main barcode
  properties: Record<string, any>;
}
```

## 4. 特殊功能整合

1.  **ExtAttr (擴充屬性)**:
    - Item 與 SKU 皆需支援 `ExtAttrPanel`。
    - 欄位定義來自 `AttrDef` API。
2.  **SKU Generation**:
    - 根據選定的規格 (e.g. Color=[Red, Blue], Size=[S, M]) 自動產生 SKU 行。
    - 格式範例: `{ItemName}-{Color}-{Size}`。
3.  **UOM Conversion**:
    - 未來需支援多單位轉換 (Base UOM vs Sales UOM)。

## 5. 開發檢查清單

- [ ] **API Client**: `ItemService`, `ItemSkuService`.
- [ ] **Workspace**: 實作 Item/SKU 雙模式切換列表。
- [ ] **Components**:
    - `ItemPicker` / `SkuPicker` (供其他模組使用的選擇器).
    - `VariantGenerator` (變體產生 UI).
- [ ] **ExtAttr**: 整合 `ExtAttrPanel` 至詳情頁。

## 6. SKU 管理頁規格（SkuManagementPage）

- **目標**：提供大量 SKU 的快速檢視與單筆編輯，避免在 Item 編輯器裡塞滿欄位。
- **資訊架構**：
  1. **上方區塊**：篩選器（Item No, Owner, Supply Mode, Enabled）、計數（總筆數/啟用/已刪除）。
  2. **列表/表格**：
     - 主要欄位：SKU No, SKU 名稱, Owner, Supply Mode, Item Kind, Enabled, Allow Sales。
     - 工具欄：編輯、複製、刪除。
  3. **右側側邊**：可固定顯示選定 SKU 的詳細屬性卡（預設顯示 `extended` 設定）。
  4. **下方操作列**：新增 SKU 按鈕（跳出 Drawer）、批次啟用/停用等。
- **編輯 Drawer** (`SkuEditDrawer`)：
  - Fields: SKU No、SKU Name、Owner、UOM、行為旗標開關、供應策略、物料類別、Lead time、預設數量、保固/保存/損耗、客製化欄位。
  - Submit -> PUT `/api/item-skus/{id}`；create -> POST `/api/item-skus` with `owner`/`item`.
  - Drawer 內只更新 single SKU，Item 不需整包重送。

### 6.1 Advanced Filter Behavior（新增）

- Filter Hub 僅需在要篩選的值上點選，不需要「全部」按鈕；若沒有選任何供應策略或狀態，就代表不限該條件（backend 會拿掉對應 query）。
- 進階欄位用「套用」才會呼叫 API，未套用時會顯示 chips / `Draft` 狀態提示，chips 點 x 會清除該條件並自動 `fetch`。
- `fetchItemSkus` 查詢 payload 會包含 `keyword` / `ownerId` / `supplyMode` / `enabled` / `preset` five elements，preset 由 sidebar `productPresets` 驅動；沒有 Owner 值時不傳 `ownerId`，同樣 behaviour 會反映在 UI loading/empty state。

### 6.2 Quick Edit + Variant Flow（補充）

- `SkuManagementPage` 的 Table/Card 每行都有「編輯」與「複製」按鈕，開 Drawer 處理單筆 SKU；Drawer 內部保持原本的欄位集，有 toast 成功提示並刷新列表。
- Header 的「產生變體」則觸發 `VariantGeneratorDialog`，目前以 mock 模擬 3-step 流程，未來接上 `/api/item-skus/variants` 後可直接插入新的 SKU；變體 dialog 要同步顯示 Owner/Enabled/SupplyMode 等欄位，並可在 preview table 勾選產生的 SKU。
- **Owner Lookup 性能保障**：Drawer 與 advanced filter 皆使用 `OwnerLookupSelect`，它會把 `limit` 設為 20（可透過 props 調整）並搭配關鍵字查詢，因此即使 Owner 數量破千也不會一次塞滿下拉；如果場景需要更大範圍的 Owner 模糊查詢，可以同時利用 `keyword` 進行名稱關鍵字過濾，兩者互補。
- **API + ExtAttr 流程**：Drawer 會直接呼叫 `ProductService.createItemSku` / `updateItemSku`，創建時必須透過 `ItemLookupSelect` + `OwnerLookupSelect` 分別選定 Item 與 Owner，再把 `properties` 綁定 `ExtAttrPanel` 的輸入（資料來源為 `item-ext-attr-defs` / `item-sku-ext-attr-defs`），必要欄位如 `enabled`、`deleted` 由後端 `normalize*` 補值；儲存成功後刷新列表並更新卡片/Drawer 狀態，避免整包更新。
- `VariantGeneratorDialog` Step 1 改為直接使用真實的 `ItemSkuExtAttrDef`（包含 `code`、`label`、`requiredAttr`），Step 2 預覽會同時顯示每組 SKU 的 extAttr 值與狀態，Step 3 可把 `properties` 附在 payload 送到 API，未來 `/api/item-skus/variants` 的回傳也可直接延續這組資料。

### 6.3 Backend Query Contract（與 filter UI 對應）

- `/api/items` 與 `/api/item-skus` 都支援 `keyword`、`ownerId`、`enabled`、`supplyMode`、`preset` 等 query 參數，只要前端把選好的條件（Sidebar Preset + Advanced Panel）傳進去，後端就會透過 `ItemCriteria` / `ItemSkuCriteria` 轉成 `Specification` 再交給 `ItemQueryService` / `ItemSkuQueryService`。這兩支服務的 `buildSearchSpecification` 會像報價單的 `QuotationThreadQueryService` 一樣，把 `keyword` 轉成 `StringFilter`，再加上 `Item`/`ItemSku` 的 `skuNo`/`itemName`/`owner.name`/`itemGroup.itemGroupName`（和 `Item` 還包含 `brand`）等欄位的 `LIKE` 條件，確保關鍵字可以對名字與關聯資源進行模糊匹配。
- `ownerId` 透過 `ItemCriteria.getOwnerId()` 併入 `Specification`，等同於 `owner` 文字搜尋與 `Item`/`ItemSku` 的 `Owner` 下拉選擇器雙軌處理，靠 `keyword` 再把 `Owner.name` 撈出來；若 `keyword` 只填 Owner 名稱，仍會進入 `buildSearchSpecification` 所建立的 OR 子查詢。
- `ItemPreset` 枚舉由 Controller 端 `applyPreset` 把 `enabled` flag、`LOW_STOCK` 與 `MISSING_PRICELIST` 等預設狀態（目前只有 `ENABLED` / `DISABLED` 實作）作用在 Criteria；Advanced Filter Panel 的狀態 pills 只是同步該 Preset 的 UI 標籤，真正的資料過濾仍在 Controller/Specification 完成，因此即便選的是 `全部` 也會把 `preset=ALL` 送出，避免資料來源不一致。
- 這段 spec 也應該在開發文件（`docs/modules/product/ui-implementation-plan.md` / `ui-spec.md`）中留一份簡短摘要，讓未來其它 workspace 也知道 `keyword` 對應哪些欄位、`preset` 會如何套用、`ownerId` 與 `enabled` 是直接放進 Criteria 的 filter。

## 7. 變體產生器草案（VariantGeneratorDialog）

- **流程**：
  1. **Step 1：維度選擇**：列出 Item 可用的 ExtAttr/Variant Attributes (`item-ext-attr-defs`)，用 Tag/Select 勾選多個維度與值清單。
  2. **Step 2：SKU 預覽**：依據選定維度自動產生 SKU 組合預覽表（`SkuPreviewTable`），可勾選要產出的 SKU，並套用命名規則（如 `ItemNo-Color-Size`）。
  3. **Step 3：產生與回寫**：呼叫 POST `/api/item-skus/variants`（若尚未完成可先用 mock）取得 SKU 資料，將結果插入 `SkuManagementPage`（或頁面內 `SKU List`）。
- **資料需求**：
  - 需讀取 `item-sku-ext-attr-defs` 支援客製欄位輸入。
  - `VariantConfig` 內含 `{ attributeCode: string, values: string[] }`。
  - `SkuPreviewRow` 包含 `skuNo`, `skuName`, `extAttrs`, `supplyMode` 等預設。
- **UI 元件**：
  - `DimensionSelector`（List + multi-select chips）。
  - `SkuPreviewTable`（可快速調整 `Enabled`, `Owner`, `Supply Mode`）。
  - `VariantRulesPanel`（定義命名規則/前綴 suffix）。
