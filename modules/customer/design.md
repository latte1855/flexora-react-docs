# Customer（客戶管理）設計文件

## 1. 模組概述
管理客戶基本資料，作為銷售流程的基礎核心。包含客戶主檔、聯絡人管理、地址管理，並作為訂單與報價的資料來源。

## 2. UI 結構定義

### 2.1 主頁（Workspace）

- **元件**: `ModuleWorkspace` (建議從 `PageContainer` 遷移)
- **主要設定**:
  - `title`: "Customer Workspace"
  - `entityName`: "Customer"

**功能區塊**:
- **Filter Hub**: 左側 Sidebar，提供預設篩選（我的客戶、潛在客戶、30天未跟進等）。
- **Advanced Filter**: 進階篩選（關鍵字、行業別、健康狀態）。
- **View**: 主要為 List View，可選 Kanban View (依 Pipeline/Status)。

**程式碼範例**:
```tsx
<ModuleWorkspace
  moduleTitle="Customer Workspace"
  headerExtra={<CustomerHeaderActions />}
>
  <ModuleSidebar
    groups={customerSidebarGroups}
    presets={customerPresets}
  />
  <CustomerTable />
  <CustomerDetailOverlay />
</ModuleWorkspace>
```

### 2.2 詳情頁 Overlay

- **結構**:
  - **Header**: 客戶名稱、編號、健康度 Badge、Owner Avatar。
  - **Tabs**:
    - **摘要 (Summary)**: 基本資料、Next Follow-up、財務資訊、最近活動時間軸。
    - **聯絡人 (Contacts)**: 關聯聯絡人列表 (CRUD)。
    - **地址 (Addresses)**: 帳單/送貨地址管理。
    - **相關單據**: 報價單、訂單歷史連結。

**程式碼範例**:
```tsx
<Drawer title={customer.name} ...>
  <Tabs defaultValue="summary">
    <Tabs.List>
      <Tabs.Tab value="summary">摘要</Tabs.Tab>
      <Tabs.Tab value="contacts">聯絡人</Tabs.Tab>
      <Tabs.Tab value="addresses">地址</Tabs.Tab>
    </Tabs.List>
    <Tabs.Panel value="summary">
      <CustomerSummary customer={customer} />
    </Tabs.Panel>
    {/* ... other panels */}
  </Tabs>
</Drawer>
```

### 2.3 完整編輯頁

- **使用情境**: 建立新客戶或完整編輯所有資訊。
- **結構**:
  - **基本資訊**: 名稱、統編(Tax ID)、行業別、電話/Email/網站。
  - **歸屬管理**: Owner、Segment (分級)、Tag。
  - **財務設定**: 預設付款條件、幣別、信用額度。
  - **地址資訊**: Billing Address / Shipping Address (預設)。

## 3. 類型定義

### 3.1 核心介面

```ts
// 對應後端 CustomerDTO
export interface Customer {
  id: string;
  customerNo?: string;
  customerName: string;
  
  // 聯絡資訊
  phone?: string;
  email?: string;
  website?: string;
  
  // 營運資訊
  industry?: string;
  annualRevenue?: number;
  employees?: number;
  
  // 管理資訊
  owner: OwnerDTO;
  segment?: string; // VIP, A, B, C
  healthStatus?: string; // HEALTHY, RISK, CHURNED
  
  // 跟進資訊
  lastActivityAt?: string;
  nextFollowUpDate?: string;
  nextFollowUpNote?: string;
  
  // 預設地址 (物件)
  billingAddress?: AddressDTO;
  shippingAddress?: AddressDTO;
  
  // 擴充屬性
  properties?: Record<string, any>;
}

// 聯絡人 (通常是獨立 DTO，但在 UI 上常與 Customer 併同顯示)
export interface ContactDTO {
  id: string;
  firstName: string;
  lastName: string;
  email?: string;
  phone?: string;
  isPrimary: boolean;
}
```

### 3.2 DTO 對應參考

| 前端欄位 | 後端 DTO 欄位 | 類型 | 備註 |
|:---------|:--------------|:-----|:-----|
| customerName | customerName | String | |
| healthStatus | healthStatus | String | 列舉值 |
| billingAddress | billingAddress | Object (AddressDTO) | 實體關聯或 Embedded |
| properties | properties | String (JSON) | ExtAttr 存於此 |

## 4. 特殊功能整合

1.  **聯絡人管理**: 
    - 需支援多聯絡人 (Master-Detail)。
    - 可標記 "Primary Contact" (主要聯絡人)。
    - UI 需提供 "新增聯絡人" 子 Drawer 或 Modal。

2.  **地址管理**:
    - DTO 顯示 `billingAddress` 與 `shippingAddress` 為單一欄位。若需支援多地址，需確認後端是否支援 Address Book 模式。
    - 編輯時需支援由 Google Maps API 或郵遞區號查找 (如有)。

3.  **跟進 (Follow-up)**:
    - 類似 `CustomerDrawer` 中的實作，支援設定 `nextFollowUpDate` 與 `nextFollowUpNote`，並即時更新列表狀態。

## 5. 開發檢查清單

- [ ] **遷移 Workspace**: 將 `CustomerWorkspace` 重構為使用 `ModuleWorkspace` 與 `ModuleSidebar`。
- [ ] **Detail Overlay**: 擴充目前的 Drawer，加入 Tabs 支援聯絡人與地址管理。
- [ ] **Contact UI**: 實作聯絡人列表與編輯元件。
- [ ] **Address UI**: 實作地址表單元件 (可重用於 Sales Order)。
- [ ] **API**: 完善 `CustomerService`，支援跟進狀態更新。
