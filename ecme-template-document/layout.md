# Ecme React Tailwind Admin Template — 頁面佈局（Layout）規範手冊  
版本：2025.02  
作者：AI Internal Doc Generator  
用途：Flexora ERP 全系統頁面 Layout（Sidebar / Header / Page Shell）統一規範  
技術：React、TailwindCSS、Ecme Template  

---

# 目錄
1. [Layout 設計目標](#layout-設計目標)
2. [Layout 基本結構](#layout-基本結構)
3. [Sidebar（側邊欄）](#sidebar側邊欄)
4. [Topbar（頂部導覽列）](#topbar頂部導覽列)
5. [Page Container（主內容框架）](#page-container主內容框架)
6. [PageHeader（頁面標題區）](#pageheader頁面標題區)
7. [Breadcrumb（麵包屑）](#breadcrumb麵包屑)
8. [通知區（Notifications）](#通知區notifications)
9. [主題切換（Theme / Dark Mode）](#主題切換theme--dark-mode)
10. [響應式行為（Responsive Layout）](#響應式行為responsive-layout)
11. [權限與側邊欄菜單（Menu + RBAC）](#權限與側邊欄菜單menu--rbac)
12. [Layout 程式範例](#layout-程式範例)
13. [Flexora ERP 專用規範](#flexora-erp-專用規範)

---

# Layout 設計目標

Flexora ERP 的 Layout 模式需達到：

- **一致的外觀與結構**  
- **模組化**（所有功能頁共享同一 Layout）  
- **可擴充**（不同模組可插入工具列）  
- **支援暗黑模式**  
- **支援 RBAC（Role-Based Access Control）**  
- **支援大型 ERP 導覽**（多階層 Menu、固定 Header）  
- **非常適合資訊密集的後台系統**  

---

# Layout 基本結構

ERP Layout 基本骨架如下：

```

<AppShell>
 ├── Sidebar         // 左側導覽
 ├── Topbar          // 頂部導航列
 └── PageContainer   // 主內容區（含 Breadcrumb、PageHeader、PageContent）
```

目錄結構：

```
src/ui/layouts/
 ├── AppShell.tsx
 ├── Sidebar/
 │    ├── Sidebar.tsx
 │    ├── SidebarItem.tsx
 │    └── MenuConfig.ts
 ├── Topbar/
 │    ├── Topbar.tsx
 │    └── UserMenu.tsx
 ├── PageContainer/
 │    ├── PageContainer.tsx
 │    ├── PageHeader.tsx
 │    └── Breadcrumb.tsx
 └── Theme/
      └── ThemeToggle.tsx
```

---

# Sidebar（側邊欄）

Sidebar 是 ERP 的導航核心，要支援：

* 多階層選單（支援 group 及子選單）
* 折疊（Collapse）
* 固定於左側（sticky left）
* 滾動（overflow-y-auto）
* 權限過濾（RBAC）
* Icon + Label
* 展開動畫（可選）

## Sidebar UI 樣式

```tsx
<div className="w-64 bg-white border-r border-gray-200 dark:bg-gray-900 dark:border-gray-700">
```

## 選單項目示例

```ts
export const menu: MenuItem[] = [
  {
    key: 'dashboard',
    label: '首頁儀表板',
    icon: HomeIcon,
    path: '/dashboard',
  },
  {
    key: 'inventory',
    label: '庫存管理',
    icon: BoxIcon,
    children: [
      { key: 'stock-list', label: '庫存查詢', path: '/inventory/stock' },
      { key: 'replenish', label: '補貨需求', path: '/inventory/replenish' },
    ],
  },
];
```

## 子選單展開樣式

```
pl-10 py-2 text-sm hover:bg-gray-100 dark:hover:bg-gray-800
```

---

# Topbar（頂部導覽列）

Topbar 為頁面上方固定區：

包含功能：

* 左側：Sidebar 展開/收起按鈕（Mobile）
* 右側：使用者頭像、UserMenu、通知鈴鐺、主題切換 Dark Mode
* 主題色背景（預設白色）

UI 樣式：

```tsx
<div className="h-14 px-4 border-b bg-white flex items-center justify-between 
                dark:bg-gray-900 dark:border-gray-700">
```

UserMenu：

* 個人設定
* 登出
* 語言切換（如需）
* 顯示當前使用者名稱（例如：王小明 / 管理者）

---

# Page Container（主內容框架）

PageContainer 是頁面內容的最外層：

### 標準樣式：

```tsx
<div className="p-6 bg-gray-50 min-h-screen dark:bg-gray-800">
```

PageContainer 包含：

1. **Breadcrumb**（可選）
2. **PageHeader（頁面標題 + 工具列）**
3. **PageContent**（主要內容）

---

# PageHeader（頁面標題區）

包含：

* 頁面標題（title）
* 頁面副標題（description，可選）
* 工具按鈕列（右側）

範例：

```tsx
<PageHeader
  title="講習試卷維護"
  description="管理試卷基本資料、題目與審核流程"
  actions={
    <Button variant="primary" size="sm">新增試卷</Button>
  }
/>
```

UI 樣式：

```
flex justify-between items-center mb-6
```

---

# Breadcrumb（麵包屑）

表示使用者在系統中的位置。

使用方式：

```tsx
<Breadcrumb items={[
  { label: "系統管理", path: "/admin" },
  { label: "講習試卷維護" }
]}/>
```

樣式：

```
text-sm text-gray-500 mb-2
```

Breadcrumb 不宜過長，建議最多 3 層。

---

# 通知區（Notifications）

Topbar 中可加入：

* 🔔 Notification Bell
* 系統訊息
* 審核提醒
* 庫存不足提醒

樣式建議：

```
relative cursor-pointer text-gray-600 hover:text-gray-900
```

通知面板：

```
absolute right-0 mt-2 w-80 bg-white shadow-lg rounded p-4
```

---

# 主題切換（Theme / Dark Mode）

Ecme Template 已內建 Dark Mode，Flexora 需要：

* 按鈕切換（ThemeToggle）
* 儲存在 localStorage
* Tailwind `dark:` 切換類別

ThemeToggle：

```tsx
export function ThemeToggle() {
  return (
    <button onClick={toggleTheme} className="p-2 rounded hover:bg-gray-100">
      {theme === 'dark' ? <MoonIcon /> : <SunIcon />}
    </button>
  );
}
```

---

# 魯棒且一致的 Dark Mode 样式示例

```
bg-white dark:bg-gray-900
text-gray-900 dark:text-gray-200
border-gray-200 dark:border-gray-700
```

---

# 響應式行為（Responsive Layout）

在 Mobile 時：

| 區塊            | 行為                    |
| ------------- | --------------------- |
| Sidebar       | 隱藏，點擊按鈕展開             |
| Topbar        | 保留（提供入口）              |
| PageContainer | padding 減少（p-4 → p-2） |

Sidebar 手機版：

```tsx
<div className="fixed inset-0 bg-black/50 lg:hidden" />
<div className="fixed left-0 w-64 bg-white h-full shadow-lg lg:hidden">
```

---

# 權限與側邊欄菜單（Menu + RBAC）

Flexora ERP 含大量模組 → 必須 RBAC。

Menu 設定範例：

```ts
{
  key: 'exam',
  label: '講習試卷維護',
  path: '/exam',
  roles: ['ADMIN', 'INSTRUCTOR']
}
```

權限過濾：

```ts
const permittedMenu = menu.filter(m => hasRole(user, m.roles));
```

子選單也需過濾。

---

# Layout 程式範例

完整骨架 AppShell：

```tsx
export function AppShell() {
  const { collapsed } = useSidebar();

  return (
    <div className="flex h-screen bg-gray-50 dark:bg-gray-800">
      <Sidebar collapsed={collapsed} />

      <div className="flex flex-col flex-1">
        <Topbar />
        
        <PageContainer>
          <Outlet /> {/* React Router */}
        </PageContainer>
      </div>
    </div>
  );
}
```

---

# Flexora ERP 專用規範

以下為 Flexora 的專案規定：

## ✔ Sidebar

* 寬度固定：`w-64`
* 折疊後：`w-20`
* 所有 Icon 必須一致大小（24px）
* 子選單縮排 16px（`pl-10`）
* 選單 active 使用 `bg-primary-50 text-primary-700`

---

## ✔ PageHeader

* 字體：`text-xl font-semibold`
* 工具列靠右
* PageHeader 下必有 `mb-6`

---

## ✔ PageContainer

```
max-w-full p-6 md:p-8
```

背景色統一：

```
bg-gray-50 dark:bg-gray-800
```

---

## ✔ Breadcrumb

* 最多 3 層
* 色系：`text-sm text-gray-500`
* 最後一段不能點擊

---

## ✔ 文字規範

* 主標題：text-xl
* 區段標題：text-lg
* 一般文字：text-base
* 表格：text-sm

---

## ✔ Theme (Dark/Light)

* 所有背景、字色都必須支援 dark
* 禁止硬編碼如 `text-black`，一律用 gray 系

---

## ✔ UI 階層（Z-index）

| 元件      | Z-index |
| ------- | ------- |
| Modal   | 50      |
| Drawer  | 40      |
| Topbar  | 30      |
| Sidebar | 20      |

---

# END
