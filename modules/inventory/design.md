# Inventory（庫存管理）設計文件

## 1. 模組概述
管理商品 SKU (Stock Keeping Unit) 及其庫存狀態。支援多倉庫管理、庫存調整、調撥與盤點功能。

## 2. UI 結構定義

### 2.1 主頁（Workspace）

- **元件**: `ModuleWorkspace`
- **主要設定**:
  - `title`: "Inventory Workspace"
  - `entityName`: "SKU"

**功能區塊**:
- **Filter Hub**: 篩選庫存不足、安全庫存預警、呆滯品等。
- **View**: 
  - **SKU List**: 顯示 SKU 基本資料與總庫存。
  - **Stock Availability**: 顯示各倉庫庫存分佈。

**程式碼範例**:
```tsx
<ModuleWorkspace
  moduleTitle="Inventory Workspace"
  headerExtra={<InventoryHeaderActions />}
>
  <ModuleSidebar
    groups={inventorySidebarGroups}
    presets={inventoryPresets}
  />
  <SkuList />
  <SkuDetailOverlay />
</ModuleWorkspace>
```

### 2.2 SKU 詳情頁 Overlay

- **結構**:
  - **Header**: SKU No、名稱、圖片、總庫存量 Badge。
  - **Tabs**:
    - **摘要**: 基本屬性 (Item Kind, Supply Mode, UOM)。
    - **庫存**: 各倉庫現有量、預留量、可用量。
    - **異動紀錄**: Inventory Transaction History。
    - **價格**: 銷售/採購價格資訊。

**程式碼範例**:
```tsx
<Drawer title={sku.skuNo} ...>
  <DetailHeader title={sku.skuName}>
     <Badge>{sku.totalStock} {sku.defaultUom.code}</Badge>
  </DetailHeader>
  <Tabs>
    {/* Summary, Stock, Transactions */}
  </Tabs>
</Drawer>
```

### 2.3 庫存調整 (Inventory Adjustment)

- **使用情境**: 盤盈虧、破損報廢、雜項收發。
- **UI**: 
  - 獨立的 `AdjustmentDrawer` 或 Modal。
  - 選擇倉庫、調整原因 (Reason Code)。
  - 行項: 選擇 SKU、調整數量 (+/-)、備註。

**程式碼範例**:
```tsx
<Drawer title="庫存調整">
  <Form>
    <Select name="warehouse" label="倉庫" />
    <Select name="reason" label="調整原因" />
    <AdjustmentItemsTable />
  </Form>
</Drawer>
```

## 3. 類型定義

### 3.1 核心介面 (SKU)

```ts
// 對應後端 ItemSkuDTO
export interface ItemSku {
  id: string;
  skuNo: string;
  skuName: string;
  
  // 屬性
  itemKind: 'PRODUCT' | 'MATERIAL' | 'SERVICE';
  supplyMode: 'MTS' | 'MTO';
  enabled: boolean;
  productBundle: boolean;
  
  // 庫存設定
  maintainStock: boolean;
  allowNegativeStock: boolean;
  
  // 預設值
  defaultUom: UomDTO;
  defaultSalesQuantity?: number;
  
  // 關聯
  owner: OwnerDTO;
  item: ItemDTO; // 母商品
}
```

### 3.2 庫存交易介面

```ts
export interface InventoryTransaction {
  id: string;
  transactionType: 'RECEIPT' | 'ISSUE' | 'ADJUSTMENT' | 'TRANSFER';
  skuId: string;
  quantity: number;
  warehouseId: string;
  occurredAt: string;
}
```

## 4. 特殊功能整合

1.  **多單位轉換 (UOM)**: UI 顯示庫存時需標註單位，交易時若使用非基本單位需進行換算（目前 DTO 僅見 `defaultUom`，需確認是否支援多單位）。
2.  **預留與可用量 (ATP)**: 詳情頁需區分 `On Hand` (現有) 與 `Available` (可用 = 現有 - 預留)。
3.  **批號/序號管理**: 若 SKU 啟用批號管理，交易時需強制輸入 Batch No。
4.  **成本與估值 (Costing & Valuation)**:
    - 系統支援移動平均 (Moving Average) 與 FIFO。
    - 入庫 (Receipt) 時將建立 Cost Layer。
    - UI 需在庫存詳情頁顯示預估成本 (Unit Cost)，但需注意權限控管 (僅授權人員可見)。

## 5. 開發檢查清單

- [ ] **API Client**: 建立 `InventoryService` 連接 SKU 與 Stock API。
- [ ] **Workspace**: 實作 SKU 列表視圖。
- [ ] **Detail Overlay**: 實作 SKU 詳情與庫存分佈展示。
- [ ] **Stock Adjustment**: 實作庫存調整 UI (前端驗證數量邏輯)。
- [ ] **Image Handling**: SKU 圖片上傳與顯示（DTO 未見 Image 欄位，可能使用 `properties` 或獨立附件）。
