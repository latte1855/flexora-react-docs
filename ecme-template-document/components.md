# 📄 components.md

## **Ecme React Tailwind Admin Template – 元件手冊（Flexora ERP 擴充版）**

版本：2025.02
作者：AI Internal Doc Generator
適用：企業後台／長期 ERP 系統（10+ 年維運）

---

# 目錄

1. [元件設計原則](#元件設計原則)
2. [元件分類總表](#元件分類總表)
3. [基礎元件（Base Components）](#基礎元件base-components)

   * Button
   * Input
   * Select / AsyncSelect
   * Checkbox / Radio / Switch
   * Tag / Badge
   * Tooltip
   * Spinner / Skeleton
4. [容器與排版元件](#容器與排版元件)

   * Card
   * SectionHeader / PageHeader
   * Divider
5. [表單元件（Form Components）](#表單元件form-components)

   * FormContainer
   * FormItem
   * InputGroup
   * DatePicker
6. [互動元件（Interactive Components）](#互動元件interactive-components)

   * Modal
   * Drawer
   * Dropdown
   * ConfirmDialog
7. [資料顯示（Data Display）](#資料顯示data-display)

   * Table / DataTable
   * List / Description List
   * Status Tag
8. [通知與回饋（Feedback）](#通知與回饋feedback)

   * Toast / Notification
   * Alert
9. [Dashboard Widgets](#dashboard-widgets)
10. [ERP 樣式規範建議](#erp-樣式規範建議)
11. [元件命名規範](#元件命名規範)

---

# 元件設計原則

Flexora ERP 所有元件遵守以下原則：

### **1. API 一致性**

所有元件需具備一組清晰且一致的 props 命名，例如：

| 類別 | 統一 Prop 名稱                          |
| -- | ----------------------------------- |
| 事件 | `onChange` / `onClick` / `onSubmit` |
| 樣式 | `variant` / `size` / `className`    |
| 狀態 | `disabled` / `loading` / `readOnly` |

---

### **2. Tailwind Utility-First**

元件外觀不直接寫 CSS class，透過 **variant + size** 決定外觀。

---

### **3. 支援企業級要求**

* 支援暗黑模式
* 支援國際化（i18n）
* 支援可及性（ARIA）
* 樣式、邏輯分離
* 可作為 UI Design System 核心

---

# 元件分類總表

| 類別               | 元件                                              |
| ---------------- | ----------------------------------------------- |
| 基礎 (Base)        | Button, Input, Select, Checkbox, Switch, Tag    |
| 容器 (Layout)      | Card, Divider, PageHeader                       |
| 表單 (Form)        | FormContainer, FormItem, InputGroup, DatePicker |
| 互動 (Interactive) | Modal, Drawer, Dialog, Dropdown                 |
| 資料顯示             | Table, List, DescriptionList                    |
| 回饋 (Feedback)    | Toast, Alert, Spinner                           |
| Dashboard        | KPI Card, Trend Chart, Mini Stat                |

---

# 基礎元件（Base Components）

---

## **Button 按鈕**

### Props

```ts
type ButtonProps = {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  disabled?: boolean;
  icon?: ReactNode;
  onClick?: () => void;
};
```

### 樣式規範

| variant   | Tailwind 範例                                      |
| --------- | ------------------------------------------------ |
| primary   | `bg-primary-600 hover:bg-primary-700 text-white` |
| secondary | `bg-gray-200 hover:bg-gray-300 text-gray-900`    |
| danger    | `bg-red-600 hover:bg-red-700 text-white`         |
| ghost     | `bg-transparent hover:bg-gray-100`               |

### 使用示例

```tsx
<Button variant="primary" size="md">儲存</Button>
<Button variant="danger" size="sm">刪除</Button>
```

---

## **Input / TextField**

### Props

```ts
{
  value?: string;
  onChange?: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
  error?: string;
}
```

### Tailwind

```
border border-gray-300 rounded px-3 py-2 focus:ring-primary-600
```

---

## **Select / AsyncSelect**

適用：

* 下拉選單
* AJAX 搜尋（廠商列表、證書下拉等）

### Props

```ts
{
  options: { label: string; value: string }[];
  async?: boolean;
  loadOptions?: (q: string) => Promise<Option[]>;
}
```

---

## Checkbox / Radio / Switch

### 核心設計

* 支援 Forms
* 支援多選（Checkbox.Group）
* Switch 可用於啟用／停用狀態

---

## Tag / Badge

ERP 中極度常用（例如審核狀態）

範例：

```
<span class="px-2 py-1 bg-green-100 text-green-700 rounded">已通過</span>
```

---

# 容器與排版元件

---

## Card

外層容器元件（適用 Dashboard、資訊卡）

```
<div class="bg-white shadow rounded p-4">
```

---

## SectionHeader / PageHeader

ERP 必備標題元件：

```
<PageHeader
  title="講習試卷維護"
  description="設定試卷基本資料與組卷管理"
  actions={<Button>新增</Button>}
/>
```

---

## Divider

```
<hr class="border-gray-200 my-4" />
```

---

# 表單元件（Form Components）

---

## FormContainer

負責整體 layout：

```
<FormContainer onSubmit={handleSubmit(onSubmit)}>
```

---

## FormItem（含 label + error）

```
<FormItem label="名稱" error={errors.name?.message}>
  <Input {...register("name")} />
</FormItem>
```

---

## InputGroup（輸入群組）

適合：

* 日期 + 時間
* 區間查詢
* 數字 + 單位（例：kg, pcs）

```
<div class="flex space-x-2">
```

---

## DatePicker

建議使用 `react-datepicker` 或 `shadcn/date-picker`：

```
<DatePicker selected={value} onChange={setValue} />
```

---

# 互動元件（Interactive Components）

---

## Modal

### Props

```ts
{
  open: boolean;
  title?: string;
  onClose: () => void;
  footer?: ReactNode;
}
```

### 使用範例

```
<Modal open={open} title="註銷原因" footer={footer}>
  <Select options={reasons}/>
</Modal>
```

---

## Drawer

右側滑出（適用流程審核、明細瀏覽）：

```
<Drawer open={showDetail} onClose={close}/>
```

---

## ConfirmDialog

ERP 必備（刪除、送審、退回、註銷）

```
<ConfirmDialog 
  open={show}
  title="確認送審？"
  onConfirm={submit}
/>
```

---

# 資料顯示（Data Display）

---

# Table / DataTable（ERP 最重要）

功能：

* 排序 Sort
* 搜尋 Filter
* Sticky Header
* 分頁 Pagination
* Row Actions
* 批次選取 Bulk Select
* 標籤 Tag 欄位

### 建議 API

```
<DataTable
  columns={columns}
  data={data}
  selectable
  onSelectionChange={setSelected}
  pagination
  toolbar={<BatchActions />}
/>
```

---

## List / Description List

```
<DescriptionList
  items={[
    { label: "證書號碼", value: data.certNo },
    { label: "廠商名稱", value: data.vendorName },
  ]}
/>
```

---

# 通知與回饋（Feedback）

---

## Toast / Notification

標準：

* success
* error
* warning
* info

```
toast.success("新增成功");
```

---

## Alert

```
<div class="bg-yellow-50 border-l-4 border-yellow-500 p-4">
```

---

## Spinner / Skeleton

```
<Spinner size="md" />
```

Skeleton 用於頁面進場：

```
<div class="animate-pulse bg-gray-200 h-6 rounded"></div>
```

---

# Dashboard Widgets

提供：

* KPI Card
* Trend Line
* Mini Chart
* Statistic Block

Flexora ERP 使用情境：

* 當月標籤發放量
* 審核案件統計
* 庫存異常警示

---

# ERP 樣式規範建議

以下為 Flexora 推薦規範：

### **1. 按鈕色系**

| 動作    | 顏色        |
| ----- | --------- |
| 儲存    | primary   |
| 送審    | primary   |
| 退回／註銷 | danger    |
| 詳細    | secondary |

---

### **2. 狀態色系**

| 狀態  | 顏色         |
| --- | ---------- |
| 已送審 | yellow-500 |
| 已通過 | green-600  |
| 已退回 | red-600    |
| 已註銷 | gray-500   |

---

# 元件命名規範

| 類型          | 命名                                |
| ----------- | --------------------------------- |
| Button      | `Button`, `PrimaryButton`（避免 Btn） |
| Modal       | `Modal`, `ConfirmModal`           |
| Table       | `DataTable`                       |
| Page Header | `PageHeader`                      |
| Tag         | `StatusTag`                       |
| Drawer      | `DetailDrawer`                    |

---

# END

（本文件可直接存為：`docs/ui/components.md`）

---