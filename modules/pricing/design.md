# Pricing (定價管理) 設計文件

## 1. 模組概述
管理價目表 (Price List) 與商品價格 (Item Price)。支援多幣別、分通路 (Channel)、各期間 (Valid From/To) 與數量階梯 (Min Qty) 定價。

## 2. UI 結構定義

### 2.1 主頁 (Workspace)

- **元件**: `ModuleWorkspace`
- **主要設定**:
  - `title`: "Pricing Workspace"
  - `entityName`: "Price List"

**視圖 (Views)**:
1.  **Price List View**:
    - 顯示所有價目表。
    - 欄位: Code, Name, Currency, Channel, Type (Sales/Purchase), Valid Period, Status.
    - 支援設為預設 (Default) 的快速切換。

**Sidebar**:
- **Presets**: All, Active, Sales, Purchase.
- **Filters**: Currency, Channel, Owner.

### 2.2 價目表詳情頁 Overlay

- **Header**: Price List Name, Currency Badge, Status Toggle.
- **Tabs**:
    - **基本設定**: Code, Period, Channel, Price Type (Tax Included/Excluded).
    - **價格明細 (Items)**: SKU 價格列表。
        - **Toolbar**: Add Item, Import CSV (Future).
        - **Grid Columns**: SKU No, Name, UOM, Min Qty, **Unit Price**, Valid From/To.
        - **Inline Edit**: 支援直接修改價格與效期。

## 3. 類型定義 (Frontend)

### 3.1 PriceList

```ts
export interface PriceList {
  id: string;
  priceListCode: string;
  priceListName: string;
  
  priceType: 'TAX_INCLUSIVE' | 'TAX_EXCLUSIVE';
  priceListType: 'SALES' | 'PURCHASE';
  
  currencyCode: string; // ISO 4217: TWD, USD
  channelCode?: string; // B2B, RETAIL
  
  validFrom?: string;
  validTo?: string;
  
  isDefault: boolean;
  enabled: boolean;
  
  owner: Owner;
}
```

### 3.2 ItemPrice

```ts
export interface ItemPrice {
  id: string;
  priceListId: string;
  skuId: string;
  sku?: ItemSku; // Expand
  
  unitPrice: number;
  minQty: number; // For tier pricing, default 0 or 1
  
  uom?: Uom; // Optional override
  taxCode?: TaxCode; // Optional override
  
  validFrom?: string;
  validTo?: string;
}
```

## 4. 特殊功能整合

1.  **預設價目表邏輯**:
    - 系統應限制同一 Owner + Currency + Channel 下只能有一個 Default Price List。
    - 前端在切換 `isDefault` 時應提示會覆蓋舊的預設值。
2.  **時效驗證**:
    - `ItemPrice` 的效期應在 `PriceList` 的效期範圍內 (雖然不必強制，但建議 Warning)。
    - 若同一 SKU 在同一 Price List 有多筆價格，需確保 `minQty` 或 `validPeriod` 不重疊 (Backend guard)。
3.  **數量階梯 (Tier Pricing)**:
    - 支援同一 SKU 多筆價格，區別在於 `minQty` (例如 100個以上單價較低)。

## 5. 開發檢查清單

- [ ] **API Client**: `PriceListService`, `ItemPriceService`.
- [ ] **Workspace**: 實作 Price List 列表。
- [ ] **Editor**: 實作 `ItemPrice` 的 CRUD Grid。
- [ ] **Selector**: `PriceListSelect` (供報價/訂單使用)。
