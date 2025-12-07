# Workspace 元件使用指南

> 版本：2025-12-07  
> 目錄：`flexora-react-ui/src/components/workspace/`

---

## 概述

本目錄提供模組主頁（Workspace）的共用 UI 元件，從報價單模組解構而來。所有模組（Sales Order、Customer、Inventory 等）應使用這些元件以保持一致的 UI 結構。

---

## 元件清單

### 主頁元件

| 元件 | 說明 | 狀態 |
|-----|------|------|
| `ModuleWorkspace` | 模組主頁容器 | ✅ 可用 |
| `ModuleHeader` | 模組標頭（標題 + 按鈕） | ✅ 可用 |
| `ModuleSidebar` | 左側篩選面板 | ✅ 可用 |
| `ModuleContentLayout` | 雙欄布局（sidebar + 內容） | ✅ 可用 |

### 詳情頁元件

| 元件 | 說明 | 狀態 |
|-----|------|------|
| `ModuleDetailContainer` | 詳情頁外層容器（雙欄布局） | ✅ 可用 |
| `ModuleDetailHeader` | 詳情頁摘要標頭 | ✅ 可用 |
| `ModuleDetailTabs` | Tab 切換框架 | ✅ 可用 |

---

## ⚠️ 重要注意事項

> [!IMPORTANT]
> 以下是使用這些元件時的關鍵注意事項，請務必遵守

### 1. 詳情頁寬度設定

- **右側 sidebar 寬度**：必須使用 `w-full lg:w-80`
- **原因**：與報價單詳情頁一致，避免左欄超出容器寬度
- **錯誤示範**：`lg:w-64 xl:w-72`（會導致左欄過寬）

### 2. Overlay Header 按鈕配置

詳情頁 Overlay 的 header 必須包含以下按鈕（從左到右）：

1. **上一筆** - `variant="plain"` + `disabled={!canNavigatePrev}`
2. **下一筆** - `variant="plain"` + `disabled={!canNavigateNext}`
3. **快速建立** - `variant="solid"` + 黑底白字樣式
4. **關閉** - `variant="plain"`

### 3. 快速操作區塊樣式

- **標題**：`text-xs text-gray-500 mb-2`
- **按鈕容器**：`flex flex-wrap gap-2`
- **按鈕**：`size="sm" variant="default"`
- **圖示大小**：`h-3.5 w-3.5`

### 4. Tabs 擴展性

✅ **各模組可自由增加 Tabs 數量**，不需修改 `ModuleDetailTabs` 元件

```tsx
// 範例：不同模組可有不同數量的 tabs
const tabs = [
  { value: 'overview', label: '摘要', content: <Overview /> },
  { value: 'items', label: '行項', content: <Items /> },
  { value: 'workflow', label: '流程', content: <Workflow /> },
  // 可繼續新增更多 tabs...
]
```

### 5. Sidebar 卡片擴展

✅ **各模組可在 sidebar 中新增自訂卡片**，不需修改元件

```tsx
const sidebar = (
  <div className="w-full space-y-4 lg:w-80">
    {/* 快速操作（必要） */}
    <QuickActionsCard />
    
    {/* 模組專屬卡片（可選） */}
    <CustomInfoCard />
    <StatisticsCard />
  </div>
)
```

### 6. 關閉按鈕位置

❌ **不要在 sidebar 底部放置關閉按鈕**  
✅ **關閉按鈕應在 Overlay header 右上角**



---

## 使用方式

### 基本用法

```tsx
import {
  ModuleWorkspace,
  ModuleHeader,
  ModuleSidebar,
  ModuleContentLayout,
} from '@/components/workspace'

const MyModuleWorkspace = () => {
  const [activePreset, setActivePreset] = useState('all')
  
  return (
    <ModuleWorkspace
      moduleTitle="銷售訂單"
      headerActions={
        <>
          <Button>新增</Button>
          <Button>匯出</Button>
        </>
      }
    >
      <ModuleContentLayout
        sidebar={
          <ModuleSidebar
            presets={myModulePresets}
            activeKey={activePreset}
            onPresetChange={setActivePreset}
          />
        }
      >
        {/* 模組內容 */}
        <MyModuleList />
      </ModuleContentLayout>
    </ModuleWorkspace>
  )
}
```

---

## 模組專用配置

每個模組需定義自己的 Preset 配置檔：

```tsx
// myModulePresets.ts
export const myModulePresets = {
  all: {
    key: 'all',
    label: '全部',
    preset: 'ALL',
    helper: '顯示所有資料',
  },
  my_open: {
    key: 'my_open',
    label: '我負責的開啟案件',
    preset: 'MY_OPEN',
    helper: '只顯示自己負責且未結案',
  },
  // ... 依模組需求新增
}
```

---

## Props 說明

### ModuleWorkspace

| Prop | 類型 | 必填 | 說明 |
|------|-----|------|------|
| `moduleTitle` | `string` | ✅ | 模組標題 |
| `headerActions` | `ReactNode` | - | 右側按鈕區 |
| `children` | `ReactNode` | ✅ | 內容區 |
| `loading` | `boolean` | - | 載入中狀態 |
| `error` | `string` | - | 錯誤訊息 |
| `onRetry` | `() => void` | - | 重試回呼 |

### ModuleSidebar

| Prop | 類型 | 必填 | 說明 |
|------|-----|------|------|
| `presets` | `SidebarPreset[]` | ✅ | 預設篩選列表 |
| `activeKey` | `string` | ✅ | 當前選中 |
| `onPresetChange` | `(key) => void` | ✅ | 切換回呼 |
| `width` | `number` | - | 寬度，預設 240 |
| `collapsible` | `boolean` | - | 可收合 |

---

## 設計原則

1. **配置驅動**：模組行為透過配置物件控制，減少重複程式碼
2. **組合優先**：使用 React children 組合，保留彈性
3. **報價單為參考**：以報價單模組為 UI 標準，確保一致性
4. **向下相容**：新元件不影響現有報價單功能

---

## 相關文件

- [模組架構分析](file:///Users/takk/Documents/開花國際/繁花計畫/flexora/docs/modules/component-architecture/module-ui-component-analysis.md)
- [決策記錄](file:///Users/takk/Documents/開花國際/繁花計畫/flexora/docs/decisions/DECISION_LOG.md)
- [報價單 UI 實作計劃](file:///Users/takk/Documents/開花國際/繁花計畫/flexora/docs/ui/quotation-ui-implementation-plan.md)

---

## 詳情頁元件（Phase 3）

### 元件清單

| 元件 | 說明 | 狀態 |
|-----|------|------|
| `ModuleDetailContainer` | 詳情頁外層容器（雙欄布局） | ✅ 可用 |
| `ModuleDetailHeader` | 詳情頁摘要標頭 | ✅ 可用 |
| `ModuleDetailTabs` | Tab 切換框架 | ✅ 可用 |

### 詳情頁 Overlay 用法

詳情頁 Overlay 需放在模組 Workspace 中，並包含以下按鈕：

```tsx
// 在 Overlay header 中加入按鈕
<div className="flex flex-wrap items-center gap-2">
    <Button size="sm" variant="plain" disabled={!canNavigatePrev} onClick={() => handleNavigate('prev')}>
        上一筆
    </Button>
    <Button size="sm" variant="plain" disabled={!canNavigateNext} onClick={() => handleNavigate('next')}>
        下一筆
    </Button>
    <Button size="sm" variant="solid" className="gap-1 px-4 text-white bg-gray-900 hover:bg-gray-800" 
            icon={<Sparkles className="h-4 w-4" />} onClick={openQuickCreate}>
        快速建立
    </Button>
    <Button size="sm" variant="plain" onClick={handleClose}>
        關閉
    </Button>
</div>
```

### 快速操作區塊

快速操作區塊放在 `ModuleDetailContainer` 的 `sidebar` 屬性中：

```tsx
const sidebar = (
    <div className="w-full space-y-4 lg:w-80">
        {/* 快速操作 */}
        <div className="rounded-2xl border border-gray-100 bg-white/70 p-4 shadow-sm dark:border-gray-700 dark:bg-gray-900/40">
            <div className="text-xs text-gray-500 mb-2">快速操作</div>
            <div className="flex flex-wrap gap-2">
                <Button size="sm" variant="default" className="gap-1" icon={<GitBranch className="h-3.5 w-3.5" />}>
                    查看流程圖
                </Button>
                <Button size="sm" variant="default">編輯</Button>
                <Button size="sm" variant="default">觸發流程</Button>
            </div>
        </div>

        {/* 其他模組可繼續增加卡片 */}
        <div className="rounded-2xl border border-gray-100 bg-white/70 p-4 text-sm shadow-sm dark:border-gray-700 dark:bg-gray-900/40">
            <div className="text-xs font-semibold text-gray-500 mb-3">自訂卡片標題</div>
            <div className="space-y-3">
                {/* 卡片內容：欄位、grid、統計數據等 */}
            </div>
        </div>

        {/* 統計卡片（虛線邊框） */}
        <div className="rounded-2xl border border-dashed border-gray-200/70 bg-gray-50/70 p-3 text-sm shadow-sm dark:border-gray-700 dark:bg-gray-900/40">
            <div className="text-xs text-gray-500">附件數量</div>
            <div className="text-lg font-semibold text-primary">5</div>
        </div>
    </div>
)

<ModuleDetailContainer sidebar={sidebar}>
    <ModuleDetailHeader fields={headerFields} />
    <ModuleDetailTabs tabs={tabs} value={activeTab} onChange={setActiveTab} />
</ModuleDetailContainer>
```

### 卡片樣式規範

| 卡片類型 | 樣式 class |
|---------|-----------|
| 一般卡片 | `rounded-2xl border border-gray-100 bg-white/70 p-4 shadow-sm dark:border-gray-700 dark:bg-gray-900/40` |
| 統計卡片 | `rounded-2xl border border-dashed border-gray-200/70 bg-gray-50/70 p-3 text-sm shadow-sm ...` |
| 標題樣式 | `text-xs text-gray-500 mb-2` 或 `text-xs font-semibold text-gray-500 mb-3` |

### 按鈕樣式規範

| 位置 | size | variant | 說明 |
|-----|------|---------|------|
| Overlay header | `sm` | `plain` | 上一筆/下一筆/關閉 |
| Overlay header | `sm` | `solid` | 快速建立（黑底白字） |
| 快速操作區塊 | `sm` | `default` | 所有操作按鈕 |

---

## Drawer 元件規範

### Drawer 寬度標準

| Drawer 類型 | 寬度 | 說明 |
|-----------|------|------|
| 工作流程 Drawer | `420px` | 用於觸發工作流程事件（WorkflowDrawer） |
| 快速建立 Drawer | `640px` | 用於快速建立單據（QuickCreateDrawer） |
| 一般資訊 Drawer | `480px` | 用於顯示詳細資訊或簡單表單 |

### Drawer 配置範例

```tsx
// 工作流程 Drawer
<Drawer
  closable
  isOpen={open}
  placement="right"
  width={420}
  title="Workflow 動作"
  shouldCloseOnOverlayClick={!submitting}
  onClose={onClose}
>
  {/* 內容 */}
</Drawer>

// 快速建立 Drawer
<Drawer
  closable
  isOpen={open}
  placement="right"
  width={640}
  title="快速建立"
  shouldCloseOnOverlayClick={!submitting}
  onClose={onClose}
>
  {/* 多步驟表單內容 */}
</Drawer>
```

### 快速建立 Drawer 設計指南

> [!NOTE]
> QuickCreateDrawer 不泛化為共用元件，各模組應根據業務需求自行實作

**設計原則**：
1. **寬度**: 固定 `640px`
2. **步驟導航**: 使用報價單樣式的可點選步驟精靈
   ```tsx
   <div className="flex gap-3 mb-6">
       <button
           type="button"
           className={`flex-1 rounded-md border px-3 py-2 text-sm transition-colors ${
               step === 0
                   ? 'border-primary bg-primary/10 text-primary'
                   : 'border-gray-200 hover:border-primary'
           }`}
           onClick={() => setStep(0)}
       >
           Step 1 · 基本資訊
       </button>
       {/* 更多步驟... */}
   </div>
   ```
3. **表單佈局**: 使用 `space-y-4` 間距
4. **欄位標籤**: `text-xs text-gray-500` + 必填標記 `*`
5. **按鈕區**: 底部固定，使用 `justify-between` 佈局
6. **按鈕邏輯**:
   - 第一步：左側「取消」，右側「下一步」
   - 中間步：左側「上一步」，右側「下一步」
   - 最後步：左側「上一步」，右側「建立草稿」
   ```tsx
   <div className="mt-6 flex justify-between">
       {step === 0 ? (
           <Button variant="plain" onClick={handleClose}>取消</Button>
       ) : (
           <Button variant="plain" onClick={() => setStep(step - 1)}>上一步</Button>
       )}
       
       {step === lastStep ? (
           <Button variant="solid" onClick={handleCreate}>建立草稿</Button>
       ) : (
           <Button variant="solid" onClick={() => setStep(step + 1)}>下一步</Button>
       )}
   </div>
   ```
7. **載入狀態**: 提交時禁用所有輸入並顯示 loading

**參考實作**：
- 報價單快速建立: `views/sales/quotes/components/QuickCreateDrawer.tsx`
- 包含：基本資訊 → 行項 → 預覽 三步驟
- 支援模式：建立 / 修訂 / 編輯草稿

---

## 詳情頁 Overlay 規範

詳情頁 Overlay 是點選列表項目後顯示的全螢幕詳情預覽頁面。

### 容器結構

```tsx
{/* 背景遮罩 */}
<div className="fixed inset-0 z-40 flex flex-col bg-black/40 backdrop-blur-sm">
    {/* 內容容器 */}
    <div className="relative flex-1 overflow-y-auto bg-white shadow-2xl dark:bg-gray-950">
        {/* Header - 固定在頂部 */}
        <div className="sticky top-0 z-10 flex flex-wrap items-center justify-between gap-2 border-b border-gray-200 bg-white/95 px-6 py-4 backdrop-blur dark:border-gray-800 dark:bg-gray-900/90">
            {/* 左側：標題區 */}
            <div>
                <div className="text-xs text-gray-500">檢視中</div>
                <div className="text-lg font-semibold">{recordTitle}</div>
            </div>
            {/* 右側：按鈕區 */}
            <div className="flex flex-wrap items-center gap-2">
                <Button size="sm" variant="plain" onClick={navigatePrev}>上一筆</Button>
                <Button size="sm" variant="plain" onClick={navigateNext}>下一筆</Button>
                <Button size="sm" variant="solid" onClick={quickCreate}>快速建立</Button>
                <Button size="sm" variant="plain" onClick={close}>關閉</Button>
            </div>
        </div>
        
        {/* 主要內容區 */}
        <div className="px-6 py-5">
            {/* 詳情內容，使用 ModuleDetailContainer 或自訂結構 */}
        </div>
    </div>
</div>
```

### 設計規範

| 屬性 | 規範值 |
|-----|--------|
| z-index | `z-40`（Overlay），`z-10`（Header） |
| 背景遮罩 | `bg-black/40 backdrop-blur-sm` |
| Header padding | `px-6 py-4` |
| 內容區 padding | `px-6 py-5` |
| 按鈕大小 | `size="sm"` |
| 導航按鈕 | `variant="plain"` |
| 主要操作按鈕 | `variant="solid"` |

### Header 按鈕順序

1. **上一筆** - 導航到上一個項目（disabled 時灰顯）
2. **下一筆** - 導航到下一個項目（disabled 時灰顯）
3. **快速建立**（可選）- 快速新增相關記錄
4. **關閉** - 關閉 Overlay 返回列表

**參考實作**：
- 報價單詳情: `views/sales/quotes/components/QuoteDetail.tsx`
- 銷售訂單詳情: `views/sales/sales-orders/components/SalesOrderDetail.tsx`

---

## 完整編輯頁規範

完整編輯頁是進行複雜表單編輯的全螢幕頁面（如報價單編輯器）。

### 容器結構

```tsx
{/* 全螢幕 Overlay */}
<div className="fixed inset-0 z-50 flex flex-col bg-black/40 backdrop-blur-sm">
    <div className="relative flex-1 overflow-hidden">
        <div className="absolute inset-0 overflow-y-auto bg-white dark:bg-gray-950">
            
            {/* Header - 固定在頂部 */}
            <div className="sticky top-0 z-10 border-b border-gray-200 bg-white/90 px-6 py-4 backdrop-blur dark:border-gray-800 dark:bg-gray-900/90">
                <div className="flex flex-wrap items-center justify-between gap-3">
                    {/* 左側：標題 */}
                    <div>
                        <div className="text-xs text-gray-500">{modeLabel}</div>
                        <h2 className="text-xl font-semibold">{pageTitle}</h2>
                    </div>
                    {/* 右側：操作按鈕 */}
                    <div className="flex items-center gap-2">
                        <Button variant="plain" onClick={onCancel}>取消</Button>
                        <Button variant="solid" onClick={onSave}>儲存</Button>
                    </div>
                </div>
                {/* 載入提示（可選） */}
                {loading && (
                    <div className="mt-3 flex items-center gap-2 text-sm text-gray-500">
                        <Spinner size={14} />
                        正在載入資料…
                    </div>
                )}
            </div>
            
            {/* 表單內容區 */}
            <div className="px-6 py-6 space-y-6">
                {/* 使用 grid 佈局 */}
                <div className="grid gap-6 lg:grid-cols-[2fr,1fr]">
                    {/* 主要表單區（左側） */}
                    <div className="space-y-6">
                        <Card className="p-5 space-y-6">
                            {/* 表單欄位 */}
                        </Card>
                    </div>
                    
                    {/* 側邊欄（右側） */}
                    <div className="space-y-6">
                        {/* 預覽、計價摘要等 */}
                    </div>
                </div>
            </div>
            
        </div>
    </div>
</div>
```

### 設計規範

| 屬性 | 規範值 |
|-----|--------|
| z-index | `z-50`（高於詳情頁 Overlay） |
| 主區塊佈局 | `grid gap-6 lg:grid-cols-[2fr,1fr]` |
| 表單卡片 | `Card className="p-5 space-y-6"` |
| 欄位間距 | `space-y-4` 或 `gap-4` |
| 欄位標籤 | `text-xs text-gray-500 mb-1` |
| 必填標記 | `<span className="ml-1 text-red-500">*</span>` |

### 欄位標籤元件

```tsx
const FieldLabel = ({
    children,
    required,
    action,
}: {
    children: React.ReactNode
    required?: boolean
    action?: React.ReactNode
}) => (
    <div className="mb-1 flex items-center justify-between text-xs font-medium text-gray-500">
        <span>
            {children}
            {required && <span className="ml-1 text-red-500">*</span>}
        </span>
        {action}
    </div>
)
```

### 表單欄位佈局

使用 12 欄 Grid 系統：

```tsx
<div className="grid gap-4 lg:grid-cols-12">
    <div className="lg:col-span-4">
        <FieldLabel required>客戶</FieldLabel>
        <Select ... />
    </div>
    <div className="lg:col-span-4">
        <FieldLabel>聯絡人</FieldLabel>
        <ContactSelect ... />
    </div>
    <div className="lg:col-span-4">
        <FieldLabel>Owner</FieldLabel>
        <OwnerSelect ... />
    </div>
</div>
```

**參考實作**：
- 報價單編輯器: `views/sales/quotes/components/QuoteEditorPage.tsx`

---

## 頁面類型總覽

| 頁面類型 | z-index | 用途 | 觸發方式 |
|---------|---------|------|---------|
| **Workspace** | - | 模組主頁（列表/Pipeline） | 選單導航 |
| **詳情頁 Overlay** | z-40 | 預覽/查看詳情 | 點選列表項目 |
| **完整編輯頁** | z-50 | 複雜表單編輯 | 點擊「編輯」或「建立」 |
| **Drawer** | z-40 | 簡易操作（工作流程/快速建立） | 按鈕觸發 |
| **Dialog** | z-50 | 確認/警告對話框 | 操作觸發 |

