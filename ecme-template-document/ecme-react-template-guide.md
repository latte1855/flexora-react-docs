# Ecme React Tailwind Admin Template — 開發者操作手冊（強化擴充版）

> 適用：Flexora ERP 前端、企業管理後台、React + Tailwind 工程  
> 目的：提供可長期維運（10 年以上）的完整 UI、樣式、元件設計手冊  
> 技術：React 18/19、TypeScript、Vite、TailwindCSS  
> 文檔版本：2025.02  

---

# 目錄
1. [簡介](#簡介)
2. [專案架構](#專案架構)
3. [安裝與初始化](#安裝與初始化)
4. [佈局與主題（Layout & Theme）](#佈局與主題layout--theme)
5. [樣式與設計系統（Tailwind / Styling）](#樣式與設計系統tailwind--styling)
6. [元件系統（Component System）](#元件系統component-system)
7. [表單與驗證](#表單與驗證)
8. [資料表格（DataTable）](#資料表格datatable)
9. [圖表與 Dashboard Widgets](#圖表與-dashboard-widgets)
10. [國際化（i18n / RTL）](#國際化i18n--rtl)
11. [Flexora ERP 導入策略](#flexora-erp-導入策略)
12. [專案檔案架構建議](#專案檔案架構建議)

---

# 簡介
Ecme React Template 是一套以 **React + Tailwind + Vite** 為核心的管理後台模板，支援高度客製、暗黑模式、可延展設計系統、企業級元件組合。

適用場景包括：ERP、CRM、儀表板、B2B 系統、SaaS 控制台。

在 Flexora ERP 的大量 UI、一致化、模組化需求下，該模板是理想的基礎 UI 層。

---

# 專案架構

典型 Ecme 專案結構如下（依版本微調）：

```
src/
 ├─ app/               # Router、Providers、App Shell
 ├─ components/        # 全局共用元件
 ├─ features/          # 以功能模組拆分（推薦用法）
 ├─ layouts/           # Layout 系統（Sidebar / Header）
 ├─ pages/             # 頁面級組件
 ├─ styles/            # 全域樣式、theme、variables
 ├─ utils/             # 通用工具
 └─ assets/            # 靜態資源
```

Flexora 建議採：  
**features/ (模組化 UI) + components/（UI 底層）+ layouts/（Layout Shell）**

---

# 安裝與初始化

## 套件安裝
```
npm install
# 或
yarn install
```

## 啟動開發模式
```
npm run dev
```

## 構建生產環境
```
npm run build
```

## Tailwind 啟用（重要）
`tailwind.config.js` 必須包含：

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

# 佈局與主題（Layout & Theme）

Ecme 提供下列 Layout 能力：

- 固定側邊欄（Sidebar）
- 折疊側邊欄（Collapsed）
- 上方導覽（Top Navigation）
- 多 Header 風格
- Sticky Header / Sticky Sidebar
- 暗黑模式（Dark Mode）
- 支援 RTL（從右至左）

## 主題系統能力
- 以 CSS Variables 提供品牌色管理  
- 全局色系可延伸於 Tailwind
- 可高度客製化：按鈕樣式、佈局間距、字級系統等

---

# 樣式與設計系統（Tailwind & Styling）

> **此章為核心強化點，適用於大型 ERP 長期維運**

---

# 1. Tailwind 設計原則

### A. Utility-First
優先使用 Tailwind utility classes，不撰寫多餘自訂 CSS。

範例：

```
<div class="flex items-center justify-between p-4 bg-white shadow">
```

### B. 避免寫 SCSS
ERP 長期維運中，SCSS 容易造成不可控的全域污染。

### C. 建立 Design System
建議建立：

- 色彩系統 Colors
- 排版系統 Typography
- 間距 Spacing
- 邊框/圓角 Radius
- 組件外觀規範（Buttons、Cards、Forms）

---

# 2. Tailwind 主題擴充

在 `tailwind.config.js` 追加：

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

# 3. 全局 CSS Variables

`src/styles/theme.css`

```css
:root {
  --color-primary: 59 130 246;
  --color-danger: 239 68 68;
  --color-warning: 234 179 8;
}
```

**Tailwind 使用方式：**

```
bg-[rgb(var(--color-primary))]
```

---

# 4. ERP 組件樣式模式（Component Styling Pattern）

Flexora ERP 建議採用：

## Button 樣式模式

### API
```
<Button variant="primary" size="md">儲存</Button>
```

### 實作
```ts
const variants = {
  primary: "bg-primary-600 hover:bg-primary-700 text-white",
  secondary: "bg-gray-200 hover:bg-gray-300 text-gray-900",
};

const sizes = {
  md: "px-4 py-2 text-sm rounded",
  lg: "px-6 py-3 text-base rounded-lg",
};
```

---

# 元件系統（Component System）

> Ecme 提供大量元件，此章將強化并加上 ERP 規劃建議。

---

# 1. 基礎元件（Base Components）

| 元件 | 用途 |
|------|------|
| Button | Primary / Secondary / Danger |
| Input | 文本輸入 |
| NumberInput | 數字欄位 |
| Select | 單選 / 遠端搜尋 |
| Checkbox | 多選 |
| Radio | 單選 |
| Toggle | 開關 |
| Tag | 審核狀態、標籤 |
| Badge | 告示 |
| Tooltip | 提示 |
| Modal | 彈窗 |
| Drawer | 右側滑出 |
| Alert | 訊息通知 |
| Spinner | Loading |

ERP 實務大量使用於：
- 審核流程按鈕
- 註銷作業 modal
- 列表行動作面板

---

# 2. 複合元件（Composite Components）

## 表格 Table / DataTable
包含：

- 排序（Sort）
- 篩選（Filter）
- 分頁（Pagination）
- 批次選取（Bulk Select）
- 工具列（Toolbar）
- 行動作欄（Action Column）
- 標籤（Status Tag）
- Sticky Header

ERP 應用：  
講習試卷維護、專責人員清冊、標籤查詢

---

## 表單 Form 系列
- FormContainer
- FormItem（label + error）
- InputGroup
- DatePicker
- Select / AsyncSelect
- NumberInput
- TextArea
- Switch

---

## Layout Components
- Sidebar
- Topbar
- Footer
- PageContainer
- Breadcrumb

ERP 會套用於所有模組。

---

# 表單與驗證

建議採用：

- React Hook Form
- Zod（或 Yup）

範例：

```ts
const schema = z.object({
  name: z.string().min(1, "必填"),
  quantity: z.number().positive(),
});
```

```ts
const { register, handleSubmit, formState: { errors } } = 
  useForm({ resolver: zodResolver(schema) });
```

ERP 中適用於：  
EAP102 試卷維護、專責人員註銷原因、標籤查詢條件。

---

# 資料表格（DataTable）

常用功能：

- 排序  
- 搜尋與篩選  
- 分頁  
- 多選（批次操作）  
- Action Buttons  
- Row Status Tag  
- Sticky Header  

ERP 強烈建議統一 DataTable 樣式與 API：

```tsx
<Table
  data={rows}
  columns={columns}
  selectable
  pagination
  toolbar={<BatchActions />}
  onRowClick={...}
/>
```

---

# 圖表與 Dashboard Widgets

支持：

- Line / Bar / Area
- Pie / Doughnut
- KPI Card
- Trend Card
- Statistic Widget

ERP 用途：
- 標籤申請量趨勢  
- 註銷統計  
- 當月案件進度  

---

# 國際化（i18n / RTL）

建議使用：

- i18next
- 多語系 JSON
- 支援 RTL（阿拉伯系）

```
/locales/zh/common.json
/locales/en/common.json
```

---

# Flexora ERP 導入策略

1. 建立 `src/ui/` 做為 Ecme Template Shell  
2. 自 Template 複製 Layout / Base Components  
3. Tailwind 統一主題色（primary / success / danger）  
4. 全專案統一定義 Button / Input / Table / Modal  
5. 建議加入 Storybook 做為 UI Library  
6. 所有模組都必須使用同一套 UI

---

# 專案檔案架構建議

```
src/
 ├─ ui/
 │   ├─ components/
 │   ├─ layouts/
 │   ├─ theme/
 │   ├─ styles/
 │   └─ hooks/
 ├─ modules/
 │   ├─ exam-paper/
 │   ├─ staff-revoke/
 │   ├─ label-query/
 │   └─ ...
 └─ app/
```

---

# END