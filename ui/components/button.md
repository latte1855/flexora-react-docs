# Button 元件使用指南

**目錄**: `flexora-react-ui/src/components/ui/Button`

標準按鈕元件，支援多種樣式變體、尺寸、形狀與載入狀態。

## Props 列表

| Prop | 類型 | 預設 | 說明 |
|------|-----|------|------|
| `variant` | `'solid' \| 'twoTone' \| 'plain' \| 'default'` | `'default'` | 樣式變體 |
| `size` | `'xs' \| 'sm' \| 'md' \| 'lg'` | `'md'` | 尺寸 |
| `shape` | `'round' \| 'circle' \| 'none'` | `'round'` | 形狀（圓角） |
| `icon` | `ReactNode` | - | 圖示 |
| `loading` | `boolean` | `false` | 載入中狀態（顯示 Spinner） |
| `block` | `boolean` | `false` | 是否佔滿父容器寬度 |
| `disabled` | `boolean` | `false` | 禁用狀態 |
| `active` | `boolean` | `false` | 啟用/按壓狀態 |
| `iconAlignment` | `'start' \| 'end'` | `'start'` | 圖示位置 |

## 樣式變體 (Variant)

### Solid (實心)
主要操作按鈕，通常用於 "建立"、"儲存"、"送審"。
```tsx
<Button variant="solid">Primary Action</Button>
<Button variant="solid" disabled>Disabled</Button>
```

### TwoTone (雙色)
次要操作，或需顯眼但不需強調的按鈕。
```tsx
<Button variant="twoTone">Secondary Action</Button>
```

### Default (預設/白底)
一般操作按鈕，有邊框。
```tsx
<Button variant="default">Default Action</Button>
```

### Plain (無框)
低強調按鈕，通常用於 "取消"、"關閉"、導航按鈕。
```tsx
<Button variant="plain">Cancel</Button>
```

## 使用情境範例

### 1. 表單操作區
```tsx
<div className="flex justify-end gap-2">
    <Button variant="plain" onClick={onCancel}>取消</Button>
    <Button variant="solid" loading={submitting} onClick={onSubmit}>
        儲存
    </Button>
</div>
```

### 2. 工具列按鈕
```tsx
<div className="flex gap-2">
    <Button size="sm" icon={<Plus />} variant="solid">新增</Button>
    <Button size="sm" icon={<Download />} variant="twoTone">匯出</Button>
</div>
```

### 3. Icon Button
```tsx
<Button shape="circle" variant="plain" icon={<X />} />
```
