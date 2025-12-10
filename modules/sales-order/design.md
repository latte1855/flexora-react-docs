# Sales Order（銷售訂單）設計文件

## 1. 模組概述
管理客戶的正式訂單，支援從報價單轉入或直接建立。包含訂單的建立、審核、履約狀態追蹤（與出貨、發票模組連動）。

## 2. UI 結構定義

### 2.1 主頁（Workspace）

- **元件**: `ModuleWorkspace`
- **主要設定**:
  - `title`: "Sales Order Workspace"
  - `icon`: (UI顯示用)
  - `entityName`: "Sales Order"

**功能區塊**:
- **Filter Hub**: 左側 Sidebar，提供預設篩選（我的草稿、待審核、本月出貨等）。
- **Advanced Filter**:上方進階篩選（關鍵字、客戶、狀態）。
- **View Switcher**: 列表模式 (List) / 管道模式 (Pipeline)。

**程式碼範例**:
```tsx
<ModuleWorkspace
  moduleTitle="Sales Order Workspace"
  headerExtra={<HeaderActions />}
>
  <ModuleSidebar
    groups={salesOrderSidebarGroups}
    presets={salesOrderPresets}
    activeKey={activeFilter}
    onPresetChange={handlePresetChange}
  />
  <SalesOrderList /> {/* or PipelineView */}
  <DetailOverlay />
</ModuleWorkspace>
```

### 2.2 詳情頁 Overlay

- **使用情境**: 快速查看訂單詳情、審核操作、查看關聯紀錄。
- **結構**:
  - **Header**: 訂單編號、狀態 Badge、操作按鈕 (Edit, Close, Navigate Prev/Next)。
  - **Content**: 
    - Summary Card (Key Info)
    - Line Items (Table)
    - Workflow History
    - Related Documents (Delivery Notes, Invoices)

**程式碼範例**:
```tsx
<div className="fixed inset-0 z-40 ...">
  <DetailHeader 
    title={order.orderNo} 
    status={order.status}
    onClose={onClose}
  />
  <DetailContent order={order} />
</div>
```

### 2.3 完整編輯頁

- **路徑**: `/modules/sales-orders/[id]/edit`, `/modules/sales-orders/new`
- **結構**: 與報價單編輯頁類似，採用 `FormContainer` 佈局。
  - **基本資訊**: 客戶、幣別、匯率、訂單日期、期望交期。
  - **財務資訊**: 價格表、付款條件、Incoterm。
  - **行項編輯**: 支援 SKU 選擇、數量、單價、折扣、稅別、預計出貨日。
  - **地址資訊**: Billing/Shipping Address (Snapshot)。
  - **擴充屬性**: ExtAttr (JSONB properties)。

## 3. 類型定義

### 3.1 核心介面 (Frontend)

```ts
// 對應後端 SalesOrderDTO
export interface SalesOrder {
  id: string;
  orderNo: string; // 訂單號
  currencyCode: string; // ISO 4217
  exchangeRate?: number;
  orderDate: string; // YYYY-MM-DD
  requestedDeliveryDate?: string; // YYYY-MM-DD
  
  // 金額相關
  subtotal: number;
  taxTotal: number;
  grandTotal: number;
  
  // 狀態
  status: SalesOrderStatusDef; // { code, name, category }
  fulfillmentStatus?: 'OPEN' | 'PARTIAL' | 'FULFILLED' | 'CANCELLED';
  
  // 關聯物件
  customer: CustomerDTO; // { id, name, ... }
  owner: OwnerDTO;
  priceList?: PriceListDTO;
  contact?: ContactDTO;
  
  // 地址 Snapshots (JSONB)
  billingAddress?: AddressSnapshot;
  shippingAddress?: AddressSnapshot;
  
  // 行項 (通常在 Detail API 回傳)
  items?: SalesOrderItem[];
}
```

### 3.2 DTO 對應參考

| 前端欄位 | 後端 DTO 欄位 | 類型 | 備註 |
|:---------|:--------------|:-----|:-----|
| id | id | Long (string in JSON) | |
| orderNo | orderNo | String | |
| customer | customer | Object | 包含 id, name |
| billingAddress | billingAddress | Object (JSONB) | 地址快照 |
| properties | properties | String (JSON) | ExtAttr 存於此 |

## 4. Workflow 整合

- **支援 Workflow**: Yes
- **Workflow 類型**: `SALES_ORDER_APPROVAL` (假設)
- **整合點**:
  - Workspace Header: "流程圖" 按鈕開啟 Workflow Drawer 查看定義。
  - Detail Overlay: 顯示當前狀態，並提供 "送審"、"核准"、"退回" 按鈕（觸發 Workflow Action）。

**程式碼範例**:
```tsx
<ModuleWorkflowDrawer
  open={workflowOpen}
  entityId={order.id}
  entityType="SALES_ORDER"
  onClose={() => setWorkflowOpen(false)}
/>
```

## 5. 特殊需求

1.  **地址快照**: 訂單成立當下需複製客戶地址為快照 (AddressSnapshot)，後續客戶地址變更不應影響已成立訂單。
2.  **庫存預留 (Reservation)**: 訂單確認後可能觸發庫存預留邏輯 (ReservationStrategy)。
3.  **來源追溯**: 若由報價單轉入，需記錄 `originatingQuotationThreadId`。

## 6. 開發檢查清單

- [ ] **類型定義**: 建立 `src/types/sales-order.ts` 完整定義 DTO。
- [ ] **API Client**: 建立 `SalesOrderService` 連接後端 API。
- [ ] **Workspace**:確認篩選與分頁功能與後端 Search API 對接。
- [ ] **Quick Create**: 實作快速建立 Drawer，支援基本欄位與客戶選擇。
- [ ] **Full Editor**: 實作完整編輯頁，處理行項增刪改。
- [ ] **Detail Overlay**: 完善詳情頁展示，包含行項列表與 Workflow 操作。
- [ ] **Workflow**: 整合 `ModuleWorkflowDrawer` 顯示審核歷程。
