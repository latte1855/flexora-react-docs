# `ecme-react-template-guide.md`

# **Ecme React Tailwind Admin Template — 開發者操作手冊（加強擴充版）**

> 適用：Flexora ERP 前端、企業管理後台、React + Tailwind 工程
> 作者：內部強化版（基於官方文件進行重組＋補完）
> 目的：提供工程師在大型 ERP 系統中可長期維運的完整 UI 樣式與元件指南
> 版本：2025.02 編修

---

# **目錄**

1. [簡介](#簡介)
2. [專案架構](#專案架構)
3. [安裝與初始化](#安裝與初始化)
4. [佈局與主題（Layout & Theme）](#佈局與主題layout--theme)
5. [樣式與設計系統（Tailwind / Styling）](#樣式與設計系統tailwind--styling) ⭐ **重點**
6. [元件系統（Component System）](#元件系統component-system) ⭐ **重點強化**
7. [表單與驗證](#表單與驗證)
8. [資料表格（DataTable）](#資料表格datatable)
9. [圖表與 Dashboard Widgets](#圖表與-dashboard-widgets)
10. [國際化（i18n）與 RTL](#國際化i18n與-rtl)
11. [最佳整合策略（套用於 Flexora ERP）](#最佳整合策略套用於-flexora-erp)
12. [檔案結構建議（專案內部）](#檔案結構建議專案內部)

---

# **簡介**

Ecme React Template 是一套高度模組化、可擴充的 React + Tailwind + Vite 的前端管理模板，官方設計為 “企業級 Admin Template”。
其優點：

* **現代化技術堆疊**：React 18/19、TypeScript、Vite、TailwindCSS
* **超輕量，可高度客製化**
* **大量 UI 組件可直接複用（Form / Table / Drawer / Chart / Modal / Dashboard Widgets）**
* **完整的暗黑模式（Dark Mode）與主題擴充能力**

⚡ **對 Flexora ERP 特別適合**
ERP 需要大量頁面一致化、元件重複使用、樣式統一、長期維護 → Ecme 的設計邏輯非常契合。

---

# **專案架構**

以下為典型 Ecme React Template 結構（實際依版本微調）：

```
src/
 ├─ app/                     # App Shell, Router, Providers
 ├─ components/             # 全站共用 UI 元件
 ├─ features/               # 功能模組 (建議依 ERP 模組拆分)
 ├─ layouts/                # Layout 系統（Sidebar / Navbar / Footer）
 ├─ pages/                  # Page 級頁面
 ├─ styles/                 # Tailwind, Variables, Theme 設定
 ├─ utils/                  # Helper, 工具
 └─ assets/                 # 圖片、icons、靜態資源
```

ERP 大型專案建議採用 features/ 架構（domain-driven UI）。

---

# **安裝與初始化**

### **1. 安裝相依**

```
npm install
# 或
yarn install
```

### **2. 本地啟動**

```
npm run dev
```

### **3. 生產建置**

```
npm run build
```

### **4. Tailwind 啟用（重要）**

`tailwind.config.js` 基本包含：

```js
module.exports = {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: '#4f46e5',
        secondary: '#64748b',
      }
    },
  },
  plugins: [],
}
```

---

# **佈局與主題（Layout & Theme）**

### **Ecme 提供以下 Layout 特性：**

* 固定側邊欄（Sidebar）
* 折疊側邊欄（Collapsed Sidebar）
* 上方導覽（Top Nav Mode）
* 多種 Header 風格
* 多種 Sidebar Skin
* Sticky Header / Sticky Sidebar
* 暗黑模式支援（Dark Mode）

### **主題系統能力**

* 可用 CSS Variables 實作企業主題色（brand color）
* Tailwind 擴充 primary / secondary / gray / error / success …
* 可支援 ERP 多專案共用主題套件

### **Dark Mode**

* Tailwind 原生 dark class 切換
* Ecme 已提供全局 Theme Context，可直接呼叫 `toggleTheme()`

---

# **樣式與設計系統（Tailwind & Styling）**

> ⭐ **此章為本文件最重要的強化區！**
> **你使用 Tailwind Admin Template → 樣式一致性是 ERP 可維運 10 年的關鍵。**

---

## **1. Tailwind 設計策略（適用大型 ERP）**

### **（1）全站採用 Utility-First，不再寫大量 SCSS**

Tailwind 目標：減少 CSS 汙染、降低樣式分散問題。

範例：

```
<div class="flex items-center justify-between p-4 bg-white shadow">
```

ERP 需要大量複雜頁面 → **建議建立「UI Pattern Library」**（後面會列出框架）。

---

## **2. Tailwind 主題擴充（色彩、間距、邊框）**

建議統一定義 ERP 主題色：

```js
extend: {
  colors: {
    primary: {
      DEFAULT: '#3B82F6',
      50: '#EFF6FF',
      100: '#DBEAFE',
      500: '#3B82F6',
      600: '#2563EB',
    },
    success: '#10b981',
    danger: '#ef4444',
  },
  borderRadius: {
    xl: '1rem',
  }
}
```

---

## **3. 全局 CSS 變數（品牌色）**

`src/styles/theme.css`

```css
:root {
  --color-primary: 59 130 246;
  --color-danger: 239 68 68;
}
```

Tailwind 使用：

```html
<button class="bg-[rgb(var(--color-primary))] text-white">...</button>
```

---

## **4. Component-Level Styling Pattern 建議**

ERP 巨大系統中的 UI 元件會爆量，建議：

### **A. 建立統一 Component API 樣式**

例如 Button：

```
<Button variant="primary" size="md">儲存</Button>
```

Tailwind 在內部：

```
const variants = {
  primary: "bg-primary-500 hover:bg-primary-600 text-white",
  secondary: "bg-gray-200 hover:bg-gray-300 text-gray-900",
}

const sizes = {
  md: "px-4 py-2 text-sm rounded",
  lg: "px-6 py-3 text-base rounded-lg",
}
```

---

# **元件系統（Component System）**

> ⭐ **這是 Flexora ERP 最重要章節之一**
> 下方為擴充版，並不只官方提供，**是可長期維護的元件規範**。

---

## **元件分類**

### **1. 基礎元件（Base Components）**

| 元件       | 說明                       | ERP 用途            |
| -------- | ------------------------ | ----------------- |
| Button   | 多種 variant + size        | 表單提交／操作按鈕         |
| Input    | Text / Number / Password | 編輯欄位              |
| Select   | 單選、下拉、遠端搜尋               | 證書下拉、客戶下拉         |
| Checkbox | 單選、多選                    | 多筆操作、權限           |
| Toggle   | 開關                       | Active / Inactive |
| Modal    | 動作確認、編輯小視窗               | 註銷原因、警告           |
| Drawer   | 側邊編輯區                    | 工作流程、右側詳情         |
| Tabs     | 多區段頁籤                    | 基本資料／明細           |

---

### **2. 複合元件（Composite Components）**

#### **Table（DataTable）**

* 排序
* 篩選
* 分頁
* 全選／多選
* 工具列（批次操作）
* 狀態 Tag 標籤（Status Tag）
* 行內動作（View/Edit/Delete）

**ERP 例子：**

* 講習試卷維護列表
* 專責人員清冊
* 標籤查詢結果

---

#### **Form（整合元件）**

包含：

| 元件          | 說明        |
| ----------- | --------- |
| FormItem    | 標籤 + 錯誤訊息 |
| InputGroup  | 多欄位置字元    |
| DatePicker  | 日期        |
| NumberInput | 僅數字       |
| TextArea    | 多行敘述      |

---

### **3. 功能元件（Feature Components）**

| 元件                   | 說明             |
| -------------------- | -------------- |
| Notification / Toast | 成功／失敗／警告通知     |
| Spinner / Loader     | 請求加載           |
| Breadcrumb           | 頁面導覽路徑         |
| PageHeader           | 統一主視覺（標題／工具按鈕） |
| ConfirmDialog        | 二次確認           |

ERP 中非常常用，例如「開始組卷」「提交審核」「註銷」。

---

### **4. Layout Components**

* Sidebar（含折疊）
* Navbar（ToolBar）
* Footer
* PageContainer

ERP 多模組 → Layout 必須統一。

---

# **表單與驗證**

### **建議採用：React Hook Form + Zod（或 Yup）**

範例：

```tsx
const schema = z.object({
  name: z.string().min(1, "必填"),
  quantity: z.number().positive(),
})
```

表單：

```tsx
const { register, handleSubmit, formState: { errors } } = useForm({ resolver: zodResolver(schema) });
```

ERP 實務需求：

* 有大量必填／日期／數字／跨欄位邏輯（如 EAP102 試卷維護）。

---

# **資料表格（DataTable）**

功能：

* 排序
* 篩選
* Sticky Header
* 多選、批次 Select
* Action Column
* Tag 標籤（審核狀態）
* 表格 Toolbar（批次操作）

ERP 建議樣式：

```tsx
<Table
  columns={columns}
  data={rows}
  selectable
  onSelectionChange={...}
  toolbar={<ExamPaperBatchActions />}
  pagination
/>
```

---

# **圖表與 Dashboard Widgets**

官方提供：

* Line Chart
* Bar Chart
* Doughnut / Pie
* Status Card
* KPI Card
* Trend Card

ERP 中可使用於：

* 標籤使用統計
* 近七日審核數量
* 人員註銷分布
* 耗材庫存趨勢

---

# **國際化（i18n）與 RTL**

如需：

* 中文、英文、越南文
  → Ecme 可接：i18next

檔案示例：

```
/locales/zh/common.json
/locales/en/common.json
```

---

# **最佳整合策略（套用於 Flexora ERP）**

### **1. 建立 UI Shell 模組（專案共用）**

```
src/ui/
   components/
   layouts/
   theme/
   styles/
```

### **2. 由 Ecme Template 複製 Layout / Theme / Base Component**

→ 這樣 ERP 所有模組一致使用相同 UI。

### **3. 建立 Storybook（強烈建議）**

長期維運 10 年 → UI 必須標準化。

### **4. 所有 ERP 模組使用同一 DataTable / Form / Modal**

### **5. 預先定義 ERP 顏色**

| 類別  | 色彩         |
| --- | ---------- |
| 審核中 | yellow-500 |
| 通過  | green-600  |
| 退回  | red-500    |
| 已註銷 | gray-500   |

---

# **檔案結構建議（專案內部）**

```
src/
 ├─ ui/
 │   ├─ components/
 │   ├─ hooks/
 │   ├─ layouts/
 │   ├─ styles/
 │   └─ theme/
 ├─ modules/
 │   ├─ exam-paper/
 │   ├─ staff-revoke/
 │   ├─ label-query/
 │   └─ ...
 └─ app/
```

---

# **END**