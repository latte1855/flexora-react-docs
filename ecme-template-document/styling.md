# 📄 styling.md

## **Ecme React Tailwind Admin Template — 樣式與主題規範（Flexora ERP 擴充版）**

版本：2025.02
作者：AI Internal Doc Generator
用途：提供 Flexora ERP 統一的 UI 樣式、色彩、字級、間距、主題模式規範
技術：TailwindCSS、CSS Variables、React

---

# 目錄

1. [設計系統概述](#設計系統概述)
2. [色彩系統（Colors）](#色彩系統colors)
3. [字級與排版（Typography）](#字級與排版typography)
4. [間距（Spacing）](#間距spacing)
5. [邊框與圓角（Border / Radius）](#邊框與圓角border--radius)
6. [陰影（Shadow）](#陰影shadow)
7. [暗黑模式（Dark Mode）](#暗黑模式dark-mode)
8. [CSS Variables（主題變數）](#css-variables主題變數)
9. [Tailwind 設定範例](#tailwind-設定範例)
10. [企業級 ERP 樣式規範建議](#企業級-erp-樣式規範建議)

---

# 設計系統概述

Flexora ERP 採用：

* **Tailwind Utility-first**
* **CSS Variables 作為主題擴充**
* **不可散亂撰寫自訂 CSS（除非在 variables 或 theme 下）**
* **所有元件 UI 都從統一設計系統衍生**

此規範可確保：

* 長期維運（10 年）
* 主題可調整（多品牌、多客戶）
* 暗黑模式可擴充
* 統一外觀（Button、Input、Table…）

---

# 色彩系統（Colors）

Flexora ERP 色彩分為：

## **1. 主色（Primary Colors）**

```css
--color-primary: 59 130 246;   /* #3B82F6 */
--color-primary-dark: 37 99 235;  /* #2563EB */
```

Tailwind 映射：

```
bg-primary-500
bg-primary-600
text-primary-600
```

---

## **2. 功能色（Semantic Colors）**

| 類型      | 顏色      | 用途       |
| ------- | ------- | -------- |
| Success | #059669 | 審核通過、完成  |
| Warning | #eab308 | 警示、提醒、草稿 |
| Danger  | #dc2626 | 錯誤、退回、取消 |
| Info    | #0284c7 | 一般通知     |

---

## **3. 灰階（Grayscale）**

ERP 大量使用灰階（表格、背景、邊線）

| Token    | 顏色      | 用途        |
| -------- | ------- | --------- |
| gray-50  | #f9fafb | 頁面背景      |
| gray-100 | #f3f4f6 | 表格 header |
| gray-200 | #e5e7eb | 邊框        |
| gray-700 | #374151 | 次要文字      |
| gray-900 | #111827 | 標題文字      |

---

# 字級與排版（Typography）

ERP 頁面資訊量大，但必須保持扎實的閱讀階層。

## 字級（Font Size）

| Token     | Pixel   | 用途            |
| --------- | ------- | ------------- |
| text-xs   | 12px    | 標籤、Tag        |
| text-sm   | 14px    | 表格、一般說明       |
| text-base | 16px    | 表單欄位、主要文字     |
| text-lg   | 18px    | Section Title |
| text-xl   | 20–24px | Page Title    |

---

## 字重（Font Weight）

```
font-normal
font-medium
font-semibold
```

ERP 規範：

* Page Title → `font-semibold`
* Table Header → `font-medium`
* Button → `font-medium`

---

# 間距（Spacing）

統一間距可讓系統介面一致：

### ERP 標準間距單位（Spacing Scale）

| Token | PX   | 用途          |
| ----- | ---- | ----------- |
| p-2   | 8px  | 小型區塊        |
| p-3   | 12px | 表格內部間距      |
| p-4   | 16px | 卡片內距        |
| p-5   | 20px | 表單大間距       |
| p-6   | 24px | Page Header |

### Grid 間距

```
gap-2 (8px) → 控件群組
gap-4 (16px) → 表單區塊
gap-6 (24px) → 主內容分段
```

---

# 邊框與圓角（Border / Radius）

ERP 需要清楚的邊線層級，避免資訊過度擁擠。

### 邊框（Border Color）

```
border-gray-200
border-gray-300
```

### 圓角（Border Radius）

| Token      | PX   | 元件           |
| ---------- | ---- | ------------ |
| rounded    | 4px  | Input/Button |
| rounded-md | 6px  | Modal、Card   |
| rounded-lg | 8px  | Drawer       |
| rounded-xl | 12px | 大卡片          |

---

# 陰影（Shadow）

ERP 採低階陰影（避免太浮誇）。

```
shadow-sm
shadow
shadow-md
```

建議：

* Card → `shadow`
* Modal → `shadow-lg`
* Dropdown → `shadow-md`

---

# 暗黑模式（Dark Mode）

Ecme 支援：

* `class` 模式切換
* Tailwind dark: 前綴
* CSS Variables 於暗色主題重新定義

## 暗黑模式示例

```
bg-white dark:bg-gray-800
text-gray-800 dark:text-gray-200
border-gray-200 dark:border-gray-700
```

## 主題切換程式碼

```tsx
document.documentElement.classList.toggle("dark")
```

---

# CSS Variables（主題變數）

ERP 必須支援未來：

* 客製化品牌色
* 多主題切換
* 白標（White-label）系統

建立 variables：

`src/styles/theme.css`

```css
:root {
  --color-primary: 59 130 246;
  --color-success: 16 185 129;
  --color-danger: 239 68 68;
  --color-warning: 234 179 8;

  --radius-md: 6px;
  --radius-lg: 12px;
}
```

---

# Tailwind 設定範例

`tailwind.config.js`

```js
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          100: '#dbeafe',
          500: '#3b82f6',
          600: '#2563eb',
        },
        danger: '#dc2626',
        success: '#059669',
        warning: '#eab308',
      },
      borderRadius: {
        md: 'var(--radius-md)',
        lg: 'var(--radius-lg)',
      },
      spacing: {
        4.5: '18px',
      },
    },
  },
}
```

---

# 企業級 ERP 樣式規範建議

以下為 Flexora ERP 全專案通用規範：

## **1. 按鈕樣式統一**

| 動作    | variant   |
| ----- | --------- |
| 儲存    | primary   |
| 送審    | primary   |
| 刪除／註銷 | danger    |
| 查詢    | secondary |
| 清除    | ghost     |

---

## **2. 表格（DataTable）規範**

* 表頭：`bg-gray-50 text-gray-700 text-sm font-medium`
* 列分隔線：`border-gray-200`
* 行 hover：`hover:bg-gray-50`
* 狀態欄使用 Tag（顏色須固定）

---

## **3. 表單（Form）規範**

* 表單標題：`text-lg font-semibold`
* 欄位標籤：`text-sm text-gray-700 font-medium`
* 欄位間距：`mb-4`
* 錯誤訊息：`text-red-600 text-xs`

---

## **4. Modal 規範**

* 寬度：預設 480px、可配置
* padding：`p-6`
* Footer：右側 Button 群組

---

## **5. Drawer 規範**

* 寬度：360px / 480px
* 用於：明細、審核流程
* Header：`p-4 border-b`

---

## **6. 狀態色系（統一）**

| 狀態  | 樣式                              |
| --- | ------------------------------- |
| 已送審 | `bg-yellow-100 text-yellow-700` |
| 已通過 | `bg-green-100 text-green-700`   |
| 已退回 | `bg-red-100 text-red-700`       |
| 已註銷 | `bg-gray-200 text-gray-600`     |

---

# END
