# Ecme React Tailwind Admin Template — 表單（Form）規範手冊  
版本：2025.02  
作者：AI Internal Doc Generator  
用途：Flexora ERP 全系統表單（新增／編輯／查詢）的統一規範  
技術：React Hook Form、Zod、TailwindCSS、Ecme Template  

---

# 目錄
1. [表單設計目標](#表單設計目標)
2. [整體結構（Form Architecture）](#整體結構form-architecture)
3. [表單容器 FormContainer](#表單容器-formcontainer)
4. [欄位容器 FormItem](#欄位容器-formitem)
5. [欄位元件 Field Components](#欄位元件-field-components)
6. [欄位驗證（Validation）](#欄位驗證validation)
7. [跨欄位驗證（Cross-field Validation）](#跨欄位驗證cross-field-validation)
8. [表單版面（Layout）](#表單版面layout)
9. [錯誤訊息與提示規範](#錯誤訊息與提示規範)
10. [表單動作區（Footer / Action Buttons）](#表單動作區footer--action-buttons)
11. [查詢表單（Search Forms）](#查詢表單search-forms)
12. [常用表單設計模式（ERP 實務）](#常用表單設計模式erp-實務)
13. [完整 CRUD 表單範例](#完整-crud-表單範例)
14. [最佳實務與反模式](#最佳實務與反模式)

---

# 表單設計目標

Flexora ERP 中，表單是最常見的操作畫面。表單規範必須滿足：

- 可適用於「新增、編輯、查詢、審核、註銷、設定」等場景  
- 統一樣式（label、欄位、錯誤訊息、寬度、間距）  
- 統一行為（驗證、disable、loading、送出）  
- 易於擴充（客製欄位、佈局變更）  
- 使用 TypeScript + React Hook Form + Zod 驗證  

---

# 整體結構（Form Architecture）

所有表單採用以下架構：

```

<FormContainer>
  <FormItem label="..." error="...">
    <Input ... />
  </FormItem>

<FormItem ... />

  <FormFooter>
    <Button>取消</Button>
    <Button>儲存</Button>
  </FormFooter>
</FormContainer>
```

表單包含：

| 區塊               | 說明                         |
| ---------------- | -------------------------- |
| FormContainer    | 負責 React Hook Form context |
| FormItem         | label + input + 錯誤訊息       |
| Field Components | Input, Select, DatePicker… |
| FormFooter       | 按鈕群組（取消／儲存／送審）             |

---

# 表單容器 FormContainer

負責：

* 注入 `react-hook-form`
* 處理 submit 行為
* 鎖定 loading 狀態

範例：

```tsx
<FormContainer methods={methods} onSubmit={handleSubmit(onSubmit)}>
  {children}
</FormContainer>
```

### Props

| Prop      | 說明             |
| --------- | -------------- |
| methods   | useForm() 回傳物件 |
| onSubmit  | 成功送出的事件        |
| className | 自訂樣式           |

---

# 欄位容器 FormItem

所有欄位 UI 必須統一使用 FormItem。

```tsx
<FormItem label="證書號碼" error={errors.certNo?.message}>
  <Input {...register("certNo")} />
</FormItem>
```

## 樣式規範

FormItem 包含：

* **Label**：`text-sm font-medium text-gray-700 mb-1`
* **Field**：內嵌 Input/Select/DatePicker
* **Error**：`text-red-600 text-xs mt-1`

---

# 欄位元件 Field Components

以下為 Flexora ERP 需統一的欄位元件：

| 元件               | 用途                 |
| ---------------- | ------------------ |
| Input            | 一般文字輸入             |
| NumberInput      | 數字                 |
| Select           | 下拉選單               |
| AsyncSelect      | 遠端搜尋（廠商、證書等）       |
| Checkbox / Radio | 單選／多選              |
| Switch           | 開關                 |
| DatePicker       | 日期選擇               |
| TextArea         | 多行文字               |
| InputGroup       | 輸入群組（日期區間、數字 + 單位） |

---

## Input

```tsx
<Input
  {...register("name")}
  placeholder="請輸入名稱"
/>
```

Tailwind：

```
border border-gray-300 rounded px-3 py-2 focus:ring-primary-600
```

---

## Select / AsyncSelect

### 一般 Select：

```tsx
<Select
  options={certificateOptions}
  {...register("certId")}
/>
```

### AsyncSelect（遠端搜尋）：

```tsx
<AsyncSelect
  loadOptions={queryCertificates}
  onChange={val => setValue("certId", val.value)}
/>
```

---

## DatePicker

ERP 必須支援：

* 單日
* 區間
* 禁用未來／過去日期

```tsx
<DatePicker
  selected={value}
  onChange={date => setValue("applyDate", date)}
/>
```

---

# 欄位驗證（Validation）

Flexora ERP 統一使用：

* **React Hook Form** 提供表單管理
* **Zod**（推薦）或 Yup 進行驗證

Zod Schema：

```ts
const schema = z.object({
  name: z.string().min(1, "名稱為必填欄位"),
  quantity: z.number().int().positive("數量必須大於 0"),
  email: z.string().email("Email 格式不正確"),
});
```

綁定：

```tsx
const methods = useForm({
  resolver: zodResolver(schema),
  mode: "onBlur",
});
```

---

# 跨欄位驗證（Cross-field Validation）

常見範例：

* 起始日 < 結束日
* 數量 A 必須等於 B 的合計
* 科目分布總和 = 試卷題數
* 廠商 = 證書所屬廠商

Zod Cross-field：

```ts
const schema = z.object({
  startDate: z.date(),
  endDate: z.date(),
}).refine(data => data.endDate >= data.startDate, {
  message: "結束日期不可早於開始日期",
  path: ["endDate"],
});
```

---

# 表單版面（Layout）

Flexora ERP 表單採用以下 Layout：

## 1 欄版面

```tsx
<div className="grid grid-cols-1 gap-4">
```

## 2 欄（最常用）

```tsx
<div className="grid grid-cols-1 md:grid-cols-2 gap-6">
```

## 3 欄（大量欄位）

```tsx
<div className="grid grid-cols-1 md:grid-cols-3 gap-6">
```

## 區段標題

```tsx
<h3 className="text-sm font-semibold text-gray-900 border-l-4 border-primary-600 pl-2">
  基本資料
</h3>
```

---

# 錯誤訊息與提示規範

所有錯誤訊息（Validation Error）格式統一：

* 文字顏色：`text-red-600`
* 字級：`text-xs`
* 行數：單行，不折行
* 位置：欄位下方

範例：

```tsx
<p className="text-xs text-red-600 mt-1">{errors.name?.message}</p>
```

---

# 表單動作區（Footer / Action Buttons）

統一採右側排列：

```tsx
<div className="flex justify-end space-x-2 py-4">
  <Button variant="ghost">取消</Button>
  <Button variant="primary" loading={loading}>儲存</Button>
</div>
```

ERP 常見按鈕：

* 儲存
* 清除
* 送審
* 退回
* 關閉

危險動作需套用 variant="danger"。

---

# 查詢表單（Search Forms）

放在列表上方（DataTable 工具列）。

### 布局規範：

```
grid grid-cols-1 md:grid-cols-4 gap-4
```

每欄大約：

* Label
* Input
* Select
* Date Range

範例：

```tsx
<div className="grid grid-cols-1 md:grid-cols-4 gap-4">
  <FormItem label="廠商">
    <AsyncSelect loadOptions={queryVendors} />
  </FormItem>
  <FormItem label="證書">
    <Select options={certOptions} />
  </FormItem>
  <FormItem label="標籤流水號">
    <Input placeholder="末 X 碼模糊查詢" />
  </FormItem>
  <FormItem label="狀態">
    <Select options={statusOptions} />
  </FormItem>
</div>
```

---

# 常用表單設計模式（ERP 實務）

## 1. 「基本資料 + 明細」兩段式

適用：

* 試卷維護
* 出貨單
* 標籤申請

Layout：

```
基本資料區（Form）
明細表格（DataTable）
按鈕（新增、刪除）
```

---

## 2. Modal 型表單（小型輸入）

適用：

* 註銷原因
* 編輯單筆資料
* 類型設定

範例：

```tsx
<Modal title="註銷原因" ...>
  <FormItem label="原因">
    <Select options={cancelReasons} />
  </FormItem>
  
  <FormFooter>
    <Button variant="ghost">取消</Button>
    <Button variant="danger">確認註銷</Button>
  </FormFooter>
</Modal>
```

---

## 3. Drawer 型表單（右側詳細資料）

適用：

* 審核畫面
* 詳細資料瀏覽
* 工作流程

```tsx
<Drawer open={open}>
  <PageHeader title="試卷明細" />
  <FormContainer ...>
    ...
  </FormContainer>
</Drawer>
```

---

# 完整 CRUD 表單範例

以「講習試卷維護」為例：

```tsx
export function ExamPaperForm({ defaultValues, mode }) {
  const methods = useForm({
    resolver: zodResolver(schema),
    defaultValues,
  });

  const onSubmit = async data => {
    await saveExamPaper(data);
    toast.success("儲存成功");
  };

  return (
    <FormContainer methods={methods} onSubmit={methods.handleSubmit(onSubmit)}>
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        
        <FormItem label="試卷名稱" error={errors.name?.message}>
          <Input {...register("name")} />
        </FormItem>

        <FormItem label="科目" error={errors.subjectId?.message}>
          <Select options={subjectOptions} {...register("subjectId")} />
        </FormItem>

        <FormItem label="難易度" error={errors.level?.message}>
          <Select options={difficultyOptions} {...register("level")} />
        </FormItem>

        <FormItem label="題數" error={errors.totalQuestions?.message}>
          <NumberInput {...register("totalQuestions", { valueAsNumber: true })} />
        </FormItem>
      </div>

      <FormFooter>
        <Button variant="ghost">取消</Button>
        <Button variant="primary" type="submit">儲存</Button>
      </FormFooter>
    </FormContainer>
  );
}
```

---

# 最佳實務與反模式

## ✔ 最佳實務

* 所有驗證寫在 Zod，不寫在畫面上
* 不要在每個欄位手動寫錯誤訊息 → 統一用 FormItem
* 表單 Layout 統一 spacing、grid
* 非必要請不要用「自訂 CSS」，用 Tailwind
* 禁止 inline-style
* 表單內的按鈕要放在 FormFooter
* API error 統一顯示於：

  * toast.error
  * 或 toggle 在 FormContainer 最上方

---

## ❌ 反模式（禁止）

* 每個欄位自己寫 `<label>`（必須使用 FormItem）
* 錯誤訊息用 alert/exclamation icon
* Input 大小不統一（欄位高度不一致）
* FormItem 間距隨意
* 表單沒有 Zod 驗證（不能只靠前端 HTML）
* 在表單內寫 CSS style=“…”
* 驗證寫在 onSubmit 裡

---

# END
