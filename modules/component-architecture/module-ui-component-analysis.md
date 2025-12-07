# 模組 UI Component 架構分析與拆解建議

> 版本：2025-12-06  
> 目的：從報價單模組（QuotesWorkspace）解構出可複用的 UI Component，讓後續模組（Sales Order、Customer 等）可以順利套用。

---

## 1. 報價單模組現況分析

### 1.1 核心檔案結構

```
flexora-react-ui/src/views/sales/quotes/
├── QuotesWorkspace.tsx      # 主頁面（2498 行）：管理所有狀態與邏輯
├── quotePresets.ts          # 預設篩選條件配置
└── components/
    ├── QuoteFilterSidebar.tsx   # 左側篩選面板（122 行）
    ├── QuoteHubList.tsx         # 列表視圖（214 行）
    ├── QuotePipeline.tsx        # Pipeline/Kanban 視圖（245 行）
    ├── QuoteDetail.tsx          # 詳情頁面（1772 行）
    ├── QuoteEditorPage.tsx      # 編輯器頁面（3013 行）
    ├── QuickCreateDrawer.tsx    # 快速建立抽屜
    ├── WorkflowDrawer.tsx       # 工作流程抽屜（204 行）
    └── ExtAttrPanel.tsx         # 擴充欄位面板
```

### 1.2 模組頁面結構（依使用者描述對應）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              PageContainer                              │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ [a] ModuleHeader: Workspace 名稱 + 右側功能按鈕（新增/工作流程） │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ [b] BackendStatus: 後端連線狀態/重新整理提示                     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┬──────────────────────────────────────────────────┐    │
│  │              │ [c-上] SearchFilterBar: 查詢條件區               │    │
│  │ [c-左]       ├──────────────────────────────────────────────────┤    │
│  │ FilterSidebar│                                                  │    │
│  │ (預設列表 +  │ [c-下] ContentView:                              │    │
│  │  自訂 Preset)│   • HubList（列表模式）                         │    │
│  │              │   • Pipeline（管道模式，可選）                   │    │
│  │              │                                                  │    │
│  └──────────────┴──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 詳情頁面（滿版）結構

```
┌─────────────────────────────────────────────────────────────────────────┐
│ [Header] 重點資訊（單號/客戶/金額）+ 右側功能按鈕（編輯/上下頁/新增返回）  │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────┬─────────────────────┐   │
│  │ [左上] 主要資訊區                          │ [右側]              │   │
│  │  • 基本欄位（客戶/聯絡人/有效期限等）      │  • 快速動作按鈕     │   │
│  │  • 客製化欄位 (ExtAttrPanel)               │  • 資訊卡片         │   │
│  ├────────────────────────────────────────────│  （狀態/Owner等）   │   │
│  │ [左下] Tabs 區域                           │                     │   │
│  │  • Overview（摘要）                        │                     │   │
│  │  • Line Items（行項）                      │                     │   │
│  │  • Pricing（計價）                         │                     │   │
│  │  • Workflow（流程）                        │                     │   │
│  │  • Attachments（附件）                     │                     │   │
│  │  • Related Records（關聯紀錄）             │                     │   │
│  └────────────────────────────────────────────┴─────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 可抽取的共用元件建議

### 2.1 頁面布局層

| 元件名稱 | 說明 | 目前位置 | 通用性 |
|---------|------|---------|-------|
| **ModuleWorkspace** | 模組主頁容器：包含 Header、Sidebar、內容區 | `QuotesWorkspace.tsx` 內含 | ⭐⭐⭐ 高 |
| **ModuleHeader** | 模組標頭：標題 + 右側按鈕組 | 內嵌於 QuotesWorkspace | ⭐⭐⭐ 高 |
| **FilterSidebar** | 左側預設篩選面板 | `QuoteFilterSidebar.tsx` | ⭐⭐⭐ 高 |
| **ContentSplitLayout** | 左 sidebar + 右內容的雙欄布局，並提供按鈕可隱藏/顯示左側 | 內嵌 | ⭐⭐⭐ 高 |

### 2.2 列表/視圖層

| 元件名稱 | 說明 | 目前位置 | 通用性 |
|---------|------|---------|-------|
| **ModuleHubList** | 資料列表（表格 + 分頁） | `QuoteHubList.tsx` | ⭐⭐⭐ 高 |
| **ModulePipeline** | Kanban 管道視圖（可選） | `QuotePipeline.tsx` | ⭐⭐ 中等 |
| **SearchFilterBar** | 查詢條件列（搜尋/進階篩選/chips） | 內嵌於 QuotesWorkspace | ⭐⭐⭐ 高 |
| **FilterChips** | 已套用篩選條件標籤 | 內嵌 | ⭐⭐⭐ 高 |

### 2.3 詳情頁層

| 元件名稱 | 說明 | 目前位置 | 通用性 |
|---------|------|---------|-------|
| **DetailPageLayout** | 滿版詳情頁佈局 | `QuoteDetail.tsx` 內含 | ⭐⭐⭐ 高 |
| **DetailHeader** | 詳情頁標頭：重點資訊 + 導覽按鈕 | 內嵌 | ⭐⭐⭐ 高 |
| **DetailMainSection** | 左側主要資訊+ExtAttr區塊 | 內嵌 | ⭐⭐⭐ 高 |
| **DetailTabs** | 下方 Tabs 容器 | 內嵌 | ⭐⭐⭐ 高 |
| **DetailSidebar** | 右側快速按鈕+卡片區 | 內嵌 | ⭐⭐⭐ 高 |

### 2.4 Drawer 層

| 元件名稱 | 說明 | 目前位置 | 通用性 |
|---------|------|---------|-------|
| **QuickCreateDrawer** | 快速建立資料抽屜 | `QuickCreateDrawer.tsx` | ⭐⭐ 需泛化 |
| **WorkflowDrawer** | 工作流程動作抽屜 | `WorkflowDrawer.tsx` | ⭐⭐⭐ 高 |
| **ModuleEditorPage** | 滿版編輯器頁面 | `QuoteEditorPage.tsx` | ⭐⭐ 需重構 |

### 2.5 共用功能元件（已部分抽取）

```
components/shared/quote/
├── LineSelectors.tsx       # SKU/UoM/稅別選擇器
├── OwnerLookupSelect.tsx   # Owner 查詢選擇
├── PaymentTermSelect.tsx   # 付款條件選擇
├── PriceBookSelectPanel.tsx # 價目表選擇面板
└── QuoteSelects.tsx        # 其他選擇器
```

### 2.6 已抽取或待泛化的共用元件

| 元件名稱 | 說明 | 建議放置/目前位置 | 狀態 |
|---------|------|-------------------|------|
| **ExtAttrPanel** | 擴充欄位面板 | `components/shared/ExtAttrPanel.tsx` | 待抽取/泛化 |
| **AddressSnapshotCard** | 地址快照卡片 | `components/shared/AddressSnapshotCard.tsx` | 待抽取/泛化 |
| **AttachmentGrid** | 附件網格 | `components/shared/AttachmentGrid.tsx` | 待抽取/泛化 |
| **CustomerLookupSelect** | 客戶查詢選擇 | `components/shared/CustomerLookupSelect.tsx` | 已抽取 |
| **ContactLookupSelect** | 聯絡人查詢選擇 | `components/shared/ContactLookupSelect.tsx` | 已抽取 |

---

## 3. 報價單特有邏輯（不適合泛化）

> [!IMPORTANT]  
> 以下為報價單模組專有功能，不建議抽取為共用元件：

1. **版本管理（Revision）**
   - 報價單有 Thread + Revision 的版本概念
   - 其他模組通常沒有版本追蹤需求

2. **報價單專屬 Preset**
   - `quotePresets.ts` 內的篩選邏輯為報價專用。
   - 後端查詢 API 需對應 key，目前做法是產生 `QuotationPreset` enum，並修改模組查詢的 API 套用。
   - 其他模組需自行定義 Preset 配置，並依循相同做法調整其後端查詢 API。

3. **Pipeline 狀態欄位**
   - 若模組具備工作流程 (Workflow)，Pipeline 的狀態欄位 (columns) 應從後端定義的 `state/event/transition` 實體動態產生。報價單模組即是依此機制生成其 Pipeline 視圖，並與對應的流程圖保持一致。
   - 非工作流程模組或需要客製化 Pipeline 狀態的模組，則需自行定義其狀態流程。

---

## 4. 建議拆解方向

### 4.1 第一階段：抽取基礎布局元件

```
components/module/ (新建目錄)
├── ModuleWorkspace.tsx      # 模組主頁容器
├── ModuleHeader.tsx         # 模組標頭
├── ModuleSidebar.tsx        # 左側篩選面板（泛化版）
├── ModuleContentLayout.tsx  # 雙欄內容布局
└── index.ts
```

### 4.2 第二階段：抽取列表/視圖元件

```
components/module/
├── ModuleHubList.tsx        # 通用列表元件
├── ModulePipeline.tsx       # 通用 Pipeline 元件（可選）
├── ModuleSearchBar.tsx      # 搜尋條件列
└── ModuleFilterChips.tsx    # 篩選 Chips
```

### 4.3 第三階段：抽取詳情頁元件

```
components/module/
├── ModuleDetailPage.tsx     # 滿版詳情頁容器
├── ModuleDetailHeader.tsx   # 詳情頁標頭
├── ModuleDetailTabs.tsx     # Tabs 容器
└── ModuleDetailSidebar.tsx  # 右側快速操作區
```

### 4.4 第四階段：抽取 Drawer 元件

```
components/module/
├── ModuleQuickCreateDrawer.tsx  # 通用快速建立
├── ModuleWorkflowDrawer.tsx     # 通用工作流程
└── ModuleEditorPage.tsx         # 通用滿版編輯器
```

---

## 5. 泛化設計建議

### 5.1 配置驅動設計

每個模組透過配置物件定義自己的：

```typescript
// 範例：ModuleConfig 介面
interface ModuleConfig {
  name: string                 // 模組名稱
  icon?: React.ReactNode       // 模組圖示
  presets: PresetConfig[]      // 預設篩選配置
  columns: ColumnConfig[]      // 列表欄位配置
  pipelineColumns?: PipelineColumnConfig[]  // Pipeline 欄位（可選）
  detailTabs: TabConfig[]      // 詳情頁 Tabs
  headerActions: ActionConfig[] // 標頭按鈕
  hasVersioning?: boolean      // 是否有版本概念（報價單專用）
}
```

### 5.2 Preset 預設篩選配置

```typescript
// 泛化版 Preset
interface ModulePresetConfig {
  key: string
  label: string
  preset: string          // API 預設代碼
  helper?: string        // 說明文字
  scopeHint?: string     // 範圍提示
  statusHint?: string    // 狀態提示
  icon?: React.ReactNode // 圖示
}

// 模組需實作的產生器函式
type ModulePresetsFactory = () => Record<string, ModulePresetConfig>
```

### 5.3 Slot/Plugin 架構

讓模組可以插入自訂區塊：

```typescript
interface ModuleWorkspaceProps<T> {
  config: ModuleConfig
  // 資料
  items: T[]
  loading: boolean
  // Slots
  renderItem?: (item: T) => React.ReactNode
  renderDetailContent?: (item: T) => React.ReactNode
  renderSidebarExtra?: () => React.ReactNode
  // Events
  onItemClick?: (item: T) => void
  onFilterChange?: (filters: FilterState) => void
}
```

---

## 6. 待討論問題

1. **元件目錄結構**
   - 建議放在 `components/workspace/`？
   - 是否需要依功能再細分子目錄？

2. **配置 vs 組合**
   - 偏好透過「配置物件」控制行為，還是透過「元件組合」？
   - 配置物件較一致，組合較靈活，建議混用。

3. **Pipeline 視圖**
   - 不是每個模組都需要 Pipeline（例如 Customer 可能不需要）
   - 建議作為可選功能，透過 config 開關

4. **Workflow 整合**
   - 是否所有模組都有 Workflow？
   - 建議 WorkflowDrawer 作為獨立元件，模組自行決定是否整合

5. **開發優先序**
   - 建議先完成 Sales Order 模組作為驗證
   - 再回頭調整共用元件

---

## 7. 下一步行動

> [!NOTE]
> 待確認上述設計方向後，將進入實作階段。

1. **建立開發分支**
   - 在 `flexora-react-ui` 建立 `feature/module-components` 分支

2. **撰寫正式 Implementation Plan**
   - 依討論結果撰寫詳細實作計劃

3. **開始抽取第一階段元件**
   - 從 `ModuleWorkspace` 開始

