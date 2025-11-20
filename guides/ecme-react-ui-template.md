# Ecme React Template 開發參考

本文件整理 `flexora-react-ui` 所採用的 Ecme React Tailwind Admin Template Demo (`flexora/Ecme - React Tailwind Admin Template/TypeScript/demo`) 內容，方便團隊在修改正式 UI 時能快速對照 template 的結構、版型與現成頁面範例。除非另外指定使用 JHipster UI，本 chat 中的 UI 開發都以 `flexora-react-ui`（基於此 template）為主。

## 1. Ecme 模板一覽

| 項目 | 說明 |
| --- | --- |
| 側邊與頂部多種 layout | 支援 `collapsibleSide`、`stackedSide`、`topBarClassic`、`framelessSide`、`contentOverlay` 等六種布局，對應常見 ERP 管理介面（參考 `components/layouts/PostLoginLayout/PostLoginLayout.tsx`）。 |
| Theme + Darkmode + 設定面板 | `Theme` Provider 會讀 `theme.config`，搭配 `ThemeConfigurator` 允許即時切換主題、版面、方向與 dark/light（見 `components/template/Theme.tsx:1` 與 `components/template/ThemeConfigurator/ThemeConfigurator.tsx:1`）。 |
| 多國語系與 RTL 支援 | 透過 `locales` 目錄建立 `react-i18next` 與 `useLocale`，版面可切換 LTR/RTL，方向邏輯在 `utils/hooks/useDirection.ts` 與 `constants/theme.constant.ts`（例如 `DIR_LTR`、`DIR_RTL`）。 |
| 數百個 UI 原子元件 | `components/ui` 存放 Alert、Button、Table、DatePicker、Drawer、Toast、Form 等元件，且每項元件內建樣式與 hooks，可直接在 `views` 中引用。 |
| 雙模 mock/API 實作 | `app.config` 有 `enableMock`，結合 `services` 與 `mock` 模擬 REST，方便開發期快速測試（參考 `configs/app.config.ts:1` 以及 `mock/index.ts`）。 |

更多核心特性請參考 template README：`flexora/Ecme - React Tailwind Admin Template/TypeScript/demo/README.md:1`。

## 2. 啟動流程與 npm 指令

1. 安裝依賴：`npm install`（`flexora/Ecme - React Tailwind Admin Template/TypeScript/demo/package.json:1` 已定義完整依賴，包括 Vite、React 19、Tailwind 4.0、zustand、react-router 7 等）。
2. 本地開發：`npm run dev`，會啟動 Vite 開發伺服器，並在 `App` 中掛載 `Theme`、`BrowserRouter`、`AuthProvider` 與 `Layout`。
3. 其他常用指令：`npm run build`、`npm run preview`、`npm run lint` / `lint:fix`、`prettier` / `format`，可參照同一份 `package.json`。
4. API Mock：預設 `enableMock: true`，會在 `App.tsx:1` 動態 import `mock`。要串接真實後端，請先在 `configs/app.config.ts` 關閉此 flag，並調整 `services` 中的 axios 實作。

## 3. 核心程式架構

### 3.1 入口與 App 組裝

* Vite entry `src/main.tsx` 會渲染 `App`（`flexora/.../src/main.tsx:1`），而 `App.tsx` 依序包 `Theme`、`BrowserRouter`、`AuthProvider`、`Layout` 與 `Views`（`flexora/.../src/App.tsx:1`）。
* `Theme` 會初始化語系、方向、dark mode、預設 theme schema，主要設定保存在 `configs/theme.config.ts` 及 `store/themeStore.ts`（`themeStore` 透過 `zustand/persist` 保留使用者選擇）。

### 3.2 Routing 與 Guards

* `Views` 會用 `AllRoutes` 包一層 `Suspense`，避免 lazy component 的白屏（`views/Views.tsx:1`）。
* `AllRoutes` 由 `protectedRoutes` + `publicRoutes` 生成路由表，並透過 `ProtectedRoute`、`PublicRoute`、`AuthorityGuard` 與 `PageContainer` 包裹各頁面（`components/route/AllRoutes.tsx:1`, `configs/routes.config/routes.config.ts:1`）。
* 路由定義拆分為 `dashboardsRoute`、`conceptsRoute`、`uiComponentsRoute`、`authRoute` 等共七個檔案，可從 `configs/routes.config/index.ts` 追蹤每個 path 與 meta。

### 3.3 Layout 與 Page Container

* `Layout` 根據是否登入 (`useAuth`) 決定採 `PostLoginLayout` 還是 `PreLoginLayout`。登入後的 layout 會 lazy load 所需 layout type；版面切換會改變 `useThemeStore` 中的 `layout.type`（`components/layouts/PostLoginLayout/PostLoginLayout.tsx:1`）。
* `PageContainer` 負責內容水平/垂直間距、Header/Body/Footer 組件，自訂 container type（`default`、`gutterless`、`contained`）與 `pageBackgroundType`（`components/template/PageContainer.tsx:1`）。如需全域調整 can override `pageContainerReassemble` hook。

## 4. 主題、版型與狀態

* 主題架構由 `ThemeConfigurator` 控制面板、`useThemeSchema`（`utils/hooks/useThemeSchema.ts:1`）、`preset-theme-schema.config.ts` 以及 `themeStore` 緊密配合，透過 CSS vars（`mapTheme`）即時改變 `--primary` 等色票。
* `themeStore` 可更改 schema、mode、方向、panel 展開、layout type、side nav collapse；每個 setter 會呼叫 `set`，且採 `persist`，讓使用者偏好保留在 localStorage。
* `ThemeConfigurator` 也提供 `LayoutSwitcher`、`ModeSwitcher`、`DirectionSwitcher`、`ThemeSwitcher`，組合剪貼簿（`CopyButton`）以便複製設定連結。

## 5. UI 元件與頁面分類

* `components/ui` 是 Ecme 自帶的元件庫（Alert、Button、Card、Table、Stepper…），每個子資料夾都含 `index.ts` re-export，也支援 `hooks` / `toast` / `utils` 子集合，可直接在 `views` 中引用。
* `components/shared` 則提供 Grid、Loading、ConfirmDialog、DataTable、StrictModeDroppable 等 utility component。
* 頁面拆成：
  * `views/dashboards`：各種 dashboard 範例（Analytics、Ecommerce、Marketing、Project）。
  * `views/ui-components`：依照 common/data-display/feedback/forms/graph/navigation 等分類的元件展示。
  * `views/concepts`：設計理念、資料 visualization 示範。
  * `views/guide`：Ecme 撰寫的 Documentations、SharedComponentsDoc、UtilsDoc，用來展示使用方式與 API。
  * `views/auth` + `views/auth-demo`：示範登入、註冊、OAuth 等流程。
  * `views/others`：冰箱樣式頁（404、Pricing、Maintenance...）。
* `views/index.tsx` 暴露 `Views` 讓 Layout 直接掛載；多數頁面使用 `PageContainer` 與 `Header` 組合資料列表。

## 6. 服務、API 與 Mock

* 服務層集中在 `src/services`，依照 domain（`AuthService`, `DashboardService`, `ProductService` 等）拆分，內部透過 `axios` 實作（`services/axios`）並可替換成實際後端。
* `app.config.ts` 定義 `apiPrefix`（`/api`）、登入路徑、token persist 策略（localStorage/sessionStorage/cookie）、`enableMock`，可在正式串接時調整。
* `mock` 目錄啟動 `MockAdapter` 與 fake data（`mock/data`、`mock/fakeApi`），支援 `axios` mock，幫助開發與 UI showcase。
* `services` 與 `mock` 與 `AuthProvider` 整合，`AuthProvider` 的 `signIn`/`signUp`/`signOut` 都會呼叫對應 `AuthService` 並更新 `useSessionUser`、`useToken`。

## 7. 翻譯與方向設定

* `src/locales` 提供 i18n 設定（`locales/locales.ts` 與 `locales/lang` 內含 JSON），`useLocale` hook 會輸出目前 `locale` 與 `direction`、供 `ConfigProvider` 使用。
* `DirectionSwitcher` 與 `useDirection` hook 控制 `dir` 屬性與 `class`，搭配 `configs/theme.config` 設定 direction default。
* 所有導航（`navigation.config`）在 `SideNav` 內翻譯後再呈現，依據 `userAuthority` 顯示可用項目。

## 8. 自訂與延伸建議

1. **導航與路由同步**：`configs/navigation.config`（多個 `*.navigation.config.ts`）與 `configs/routes.config` 必須保持一致；新增頁面時先更新 `navigation`、再將 route 加入 `routes.config`、再於 `views` 建 `Page`。
2. **版型切換**：若想提供自訂 layout，可在 `themeStore` 加 `previousType`、或在 `PageContainer` 透過 `useLayout` hook 覆寫 `pageContainerReassemble`。
3. **Tables / Forms / Widgets**：建議先搜尋 `views/ui-components` 與 `components/ui` 的現成實作再複用，輔以 `mock` sample data 測試。
4. **引入與 override**：如果 `flexora-react-ui` 需要改動每頁 header、footer、breadcrumb，可 fork `components/template/PageContainer` 內的部分並在 `views` `meta` header 傳入自訂 element。
5. **整合真實後端**：正式開發時把 `app.config.enableMock` 關掉，利用 `configs/endpoint.config.ts` 統一 `apiPrefix`，在 `services` 裡面交換成與 `flexora-react` 對應的 API path。

## 9. 參考資源

* Ecme Demo 的內建 documentation 頁：`views/guide/Documentations/index.tsx` 與 `views/guide/SharedComponentsDoc` 提供 component metadata，可直接透過 template UI 查看使用範例。
* `Ecme - React Tailwind Admin Template/TypeScript/documentation/index.html` 為官方線上 guide 的 local copy，可以在開發時直接打開查看設計思想。
* 若需要 UI 範例，`flexora/Ecme - React Tailwind Admin Template/TypeScript/demo/src/views/guide/ChangeLog` 可幫助了解版本差異。

此份指南會是後續 `flexora-react-ui` UI 開發的單一參考來源，請在在修改或延伸模板時同步紀錄於 `docs/specs` 或 `docs/guides`（視變更類型而定）。
