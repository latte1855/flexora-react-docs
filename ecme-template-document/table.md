# Ecme React Tailwind Admin Template — 表格（DataTable）規範手冊  
版本：2025.02  
作者：AI Internal Doc Generator  
用途：定義 Flexora ERP 全系統表格（列表頁）的統一行為與樣式  
技術：React、TypeScript、TailwindCSS、Ecme UI Template  

---

# 目錄
1. [設計目標](#設計目標)  
2. [核心能力總覽](#核心能力總覽)  
3. [API 設計與基本使用](#api-設計與基本使用)  
4. [欄位定義（Columns）](#欄位定義columns)  
5. [排序（Sorting）](#排序sorting)  
6. [搜尋與篩選（Filter / Search）](#搜尋與篩選filter--search)  
7. [分頁（Pagination）](#分頁pagination)  
8. [列選取與批次操作（Row Selection & Bulk Actions）](#列選取與批次操作row-selection--bulk-actions)  
9. [行內操作與動作欄（Row Actions）](#行內操作與動作欄row-actions)  
10. [狀態顯示與 Tag](#狀態顯示與-tag)  
11. [響應式（Responsive）](#響應式responsive)  
12. [載入中／空狀態／錯誤狀態](#載入中空狀態錯誤狀態)  
13. [效能與最佳實務](#效能與最佳實務)  
14. [Flexora ERP 樣式與互動規範](#flexora-erp-樣式與互動規範)  

---

# 設計目標

Flexora ERP 的 DataTable 是所有「列表頁」的共同基礎，必須滿足：

- 可支援 **大量資料**（上萬筆，分頁載入）
- 具備 **排序、篩選、搜尋、分頁** 標準能力
- 支援 **批次選取與批次操作**
- 行為一致：所有模組（講習試卷維護、專責人員、標籤查詢…）**看起來與操作都要一致**
- 方便二次客製（加入自訂欄位、操作欄、Tag 等）

---

# 核心能力總覽

DataTable 應至少具備以下功能：

- **排序（Sorting）**：依欄位升冪／降冪切換
- **搜尋與篩選（Filter/Search）**：可依條件查詢
- **分頁（Pagination）**：前端／後端分頁皆可
- **列選取（Row Selection）**：單選、多選，全選
- **批次操作（Bulk Actions）**：刪除、送審、匯出等
- **行內操作（Row Actions）**：檢視、編輯、複製、註銷
- **狀態顯示（Status Tag）**
- **載入中、空狀態、錯誤顯示**

---

# API 設計與基本使用

我們為 DataTable 定義一個統一的介面（TypeScript）：

```ts
export type ColumnDef<T> = {
  key: keyof T | string;
  title: string;
  width?: string | number;
  align?: 'left' | 'center' | 'right';
  sortable?: boolean;
  render?: (value: any, record: T, index: number) => React.ReactNode;
};

export type DataTableProps<T> = {
  columns: ColumnDef<T>[];
  data: T[];
  loading?: boolean;
  rowKey?: (record: T) => string | number;
  selectable?: boolean;
  onSelectionChange?: (selected: T[]) => void;
  pagination?: {
    page: number;
    pageSize: number;
    total: number;
    onChange: (page: number, pageSize: number) => void;
  };
  onRowClick?: (record: T) => void;
  toolbar?: React.ReactNode;
  className?: string;
  emptyText?: string;
};
```

基本使用：

```tsx
<DataTable
  columns={columns}
  data={rows}
  loading={loading}
  rowKey={row => row.id}
  selectable
  onSelectionChange={setSelectedRows}
  pagination={{
    page,
    pageSize,
    total,
    onChange: handlePageChange,
  }}
  toolbar={<ExamPaperTableToolbar selected={selectedRows} />}
/>
```

---

# 欄位定義（Columns）

欄位定義統一使用 `ColumnDef<T>`：

```ts
const columns: ColumnDef<ExamPaper>[] = [
  {
    key: 'examNo',
    title: '試卷代號',
    sortable: true,
    render: (value, record) => (
      <span className="font-mono text-sm text-primary-700">
        {value}
      </span>
    ),
  },
  {
    key: 'subjectName',
    title: '科目',
    sortable: true,
  },
  {
    key: 'status',
    title: '狀態',
    render: (_, record) => <StatusTag status={record.status} />,
  },
  {
    key: 'actions',
    title: '操作',
    align: 'right',
    render: (_, record) => <ExamPaperRowActions record={record} />,
  },
];
```

欄位命名原則：

* `title`：UI 顯示名稱（支援 i18n）
* `key`：對應資料欄位，或特別標記如 `"actions"`
* `render`：用於自訂顯示（Tag、按鈕、連結等）

---

# 排序（Sorting）

DataTable 支援：

* 點擊欄位標題 → 切換排序狀態：`none → asc → desc`
* `sortable: true` 的欄位才啟用排序圖示與行為

欄位標題樣式：

```tsx
<button
  className="flex items-center space-x-1 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
  onClick={() => onSortChange(column.key)}
>
  <span>{column.title}</span>
  {sortState.key === column.key && sortState.direction === 'asc' && <IconChevronUp />}
  {sortState.key === column.key && sortState.direction === 'desc' && <IconChevronDown />}
</button>
```

後端排序建議流程：

1. 使用 `sortKey` + `sortDirection` 傳給後端 API
2. 後端使用 QueryDSL / Specification 排序
3. 回傳資料與 total，同步更新表格

---

# 搜尋與篩選（Filter / Search）

DataTable 本身只負責顯示結果；搜尋與篩選通常透過：

* 表格上方的 **搜尋列 / 篩選表單**
* 後端 API 提供 `filter` / `query` 參數

範例（上方工具列）：

```tsx
function ExamPaperTableToolbar({ selected }: { selected: ExamPaper[] }) {
  const [keyword, setKeyword] = useState('');

  const onSearchSubmit = () => {
    // 觸發上層的查詢邏輯
  };

  return (
    <div className="flex items-center justify-between mb-3">
      <div className="flex items-center space-x-2">
        <input
          className="border border-gray-300 rounded px-2 py-1 text-sm"
          placeholder="搜尋試卷名稱 / 代號"
          value={keyword}
          onChange={e => setKeyword(e.target.value)}
        />
        <Button size="sm" variant="secondary" onClick={onSearchSubmit}>
          查詢
        </Button>
      </div>
      <div className="flex items-center space-x-2">
        {/* 批次操作按鈕 */}
        <ExamPaperBulkActions selected={selected} />
      </div>
    </div>
  );
}
```

篩選類型建議：

* 關鍵字（文字搜尋）
* 下拉式（狀態、科目、廠商等）
* 日期區間（起始日 / 結束日）
* 數值範圍（分數、次數）

---

# 分頁（Pagination）

ERP 常見需求為「後端分頁」：

* 前端組裝：`page`, `pageSize`, `sort`, `filters`
* 後端回應：`content`, `totalElements`, `totalPages`

分頁元件可統一：

```tsx
<Pagination
  page={page}
  pageSize={pageSize}
  total={total}
  onChange={(newPage, newPageSize) => {
    setPage(newPage);
    setPageSize(newPageSize);
    fetchData({ page: newPage, pageSize: newPageSize });
  }}
/>
```

UI 規範：

* 分頁控制放在表格底部右側
* 顯示目前筆數區間，例如：「顯示第 1–20 筆，共 132 筆」

---

# 列選取與批次操作（Row Selection & Bulk Actions）

DataTable 必須提供：

* 每列前方 checkbox
* 表頭 checkbox（全選 / 全不選）
* 批次操作工具列（根據已選取筆數動態顯示）

表頭部分：

```tsx
<th className="px-4 py-2">
  <input
    type="checkbox"
    checked={isAllCurrentPageSelected}
    onChange={toggleSelectAllCurrentPage}
  />
</th>
```

資料列：

```tsx
<td className="px-4 py-2">
  <input
    type="checkbox"
    checked={selectedIds.includes(row.id)}
    onChange={() => toggleRowSelection(row)}
  />
</td>
```

批次操作工具列（Toolbar）：

```tsx
function ExamPaperBulkActions({ selected }: { selected: ExamPaper[] }) {
  if (selected.length === 0) {
    return null;
  }

  return (
    <div className="flex items-center space-x-2 bg-amber-50 border border-amber-200 px-3 py-1 rounded">
      <span className="text-xs text-amber-800">
        已選取 {selected.length} 筆
      </span>
      <Button size="xs" variant="primary">批次送審</Button>
      <Button size="xs" variant="danger">批次刪除</Button>
      <Button size="xs" variant="ghost">清除選取</Button>
    </div>
  );
}
```

---

# 行內操作與動作欄（Row Actions）

每列最後一欄通常為操作欄位：

* 檢視（View）
* 編輯（Edit）
* 註銷（Cancel / Revoke）
* 更多操作（More … 下拉）

範例：

```tsx
function ExamPaperRowActions({ record }: { record: ExamPaper }) {
  return (
    <div className="flex justify-end space-x-1">
      <Button size="xs" variant="ghost" onClick={() => openPreview(record)}>
        預覽
      </Button>
      <Button size="xs" variant="ghost" onClick={() => openEdit(record)}>
        編輯
      </Button>
      <Button size="xs" variant="ghost" onClick={() => openRevoke(record)}>
        註銷
      </Button>
    </div>
  );
}
```

注意：

* 若操作過多，可整合為「更多」的 Dropdown
* 危險操作（刪除／註銷）需彈出 ConfirmDialog

---

# 狀態顯示與 Tag

ERP 中各種狀態應使用統一 Tag 元件，避免自行拼 Tailwind。

範例 `StatusTag`：

```tsx
type Status = 'DRAFT' | 'SUBMITTED' | 'APPROVED' | 'REJECTED' | 'CANCELLED';

const statusStyleMap: Record<Status, string> = {
  DRAFT: 'bg-gray-100 text-gray-700',
  SUBMITTED: 'bg-amber-100 text-amber-700',
  APPROVED: 'bg-emerald-100 text-emerald-700',
  REJECTED: 'bg-red-100 text-red-700',
  CANCELLED: 'bg-slate-200 text-slate-700',
};

export function StatusTag({ status }: { status: Status }) {
  return (
    <span
      className={`inline-flex items-center px-2 py-0.5 rounded text-xs font-medium ${statusStyleMap[status]}`}
    >
      {translateStatus(status)}
    </span>
  );
}
```

在表格中使用：

```ts
{
  key: 'status',
  title: '狀態',
  render: (_, record) => <StatusTag status={record.status} />,
}
```

---

# 響應式（Responsive）

DataTable 在小螢幕（平板／手機）建議行為：

* 隱藏不重要欄位（透過 `hidden md:table-cell`）
* 改為「卡片式表格」呈現（small breakpoint）

欄位隱藏範例：

```tsx
<td className="px-4 py-2 hidden md:table-cell">
  {record.certificateNo}
</td>
```

---

# 載入中／空狀態／錯誤狀態

## 載入中（Loading）

* 顯示 Skeleton 或 Spinner
* 禁用操作按鈕

```tsx
{loading ? (
  <TableSkeleton rows={10} columns={columns.length} />
) : (
  <ActualTable ... />
)}
```

## 空狀態（Empty）

* 顯示友善訊息：

  * 「目前尚無資料」
  * 「請調整查詢條件後重試」

```tsx
{!loading && data.length === 0 && (
  <div className="py-10 text-center text-sm text-gray-500">
    尚無符合條件的資料
  </div>
)}
```

## 錯誤狀態

* 顯示錯誤訊息 + 重新整理按鈕

```tsx
{error && (
  <div className="py-4 px-3 bg-red-50 border border-red-200 text-sm text-red-700 flex items-center justify-between">
    <span>查詢資料時發生錯誤：{errorMessage}</span>
    <Button size="xs" variant="ghost" onClick={reload}>重新整理</Button>
  </div>
)}
```

---

# 效能與最佳實務

* 大量資料必須使用「後端分頁」
* 避免在 `render` 中執行重計算，可提前在 hook 中處理好
* 如果表格很大，可使用 `React.memo` 或虛擬捲動（virtualized list）
* 盡可能避免在每個 cell 內新建匿名函數（用 `useCallback`）

---

# Flexora ERP 樣式與互動規範

統一規範如下：

1. **列表頁都使用 DataTable，不自行實作 table**
2. **所有狀態欄位都使用 `StatusTag` 元件**
3. **表頭高度與字體統一：**

   * `text-xs font-medium text-gray-500 uppercase tracking-wider`
4. **列高度統一：**

   * `py-2 text-sm`
5. **Hover 效果：**

   * 列 hover 時背景為 `bg-gray-50`
6. **操作欄對齊方式：**

   * 一律 `text-right`
7. **行點擊 vs. Checkbox：**

   * 點擊列（Row）開啟詳情
   * 點擊 Checkbox 專職選取，不觸發 row click
8. **危險操作必須 ConfirmDialog**
9. **批次操作與單筆操作的文案與行為需一致**

---

# END