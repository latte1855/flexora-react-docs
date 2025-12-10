# [模組名稱] 設計文件

## 1. 模組概述
<!-- 說明模組的功能、與其他模組的關聯 -->

## 2. UI 結構定義

### 2.1 主頁（Workspace）
<!-- 描述主頁面結構，使用的共用元件 -->

- **元件**: `ModuleWorkspace`
- **主要設定**:
  - `title`: "[標題]"
  - `icon`: [IconName]
  - `entityName`: "[實體名稱]"

**程式碼範例**:
```tsx
<ModuleWorkspace
  title="[標題]"
  icon={[IconName]}
  entityName="[實體名稱]"
  // ...其他 props
/>
```

### 2.2 詳情頁 Overlay
<!-- 取代 Dialog，使用側邊彈出或覆蓋層 -->
<!-- 定義結構、必要按鈕 -->

- **使用情境**: 快速查看詳情，非完整編輯
- **結構**:
  - Header: 標題、狀態、操作按鈕 (Edit, Close)
  - Content: 唯讀資訊展示 (DescriptionList)
  - Footer: (Optional)

**程式碼範例**:
```tsx
<Drawer/Overlay ...>
  <DetailHeader ... />
  <DetailContent ... />
</Drawer/Overlay>
```

### 2.3 完整編輯頁
<!-- 完整表單頁面 -->

- **路徑**: `/modules/[module]/[id]/edit`, `/modules/[module]/new`
- **結構**:
  - `FormContainer`
  - `FormSection`
  - `FormActions` (Save, Cancel)

**程式碼範例**:
```tsx
<FormContainer onSubmit={...}>
  <FormSection title="基本資訊">
    <TextField name="code" label="編號" />
    <TextField name="name" label="名稱" />
  </FormSection>
  <FormActions>
    <Button type="submit">儲存</Button>
    <Button onClick={onCancel}>取消</Button>
  </FormActions>
</FormContainer>
```

## 3. 類型定義
<!-- 定義核心類型，並對應後端 DTO -->

### 3.1 核心介面

```ts
export interface [Entity] {
  id: string;
  code: string;
  // ...
  createdAt: string;
  updatedAt: string;
}
```

### 3.2 DTO 對應

| 前端欄位 | 後端 DTO 欄位 | 類型 | 備註 |
|:---------|:--------------|:-----|:-----|
| id       | id            | string | UUID |
| code     | code          | string |      |

## 4. Workflow 整合
<!-- 是否需要 Workflow，如何與 ModuleWorkflowDrawer 整合 -->

- **支援 Workflow**: Yes/No
- **Workflow 類型**: `[WorkflowType]`

**程式碼範例**:
```tsx
<ModuleWorkflowDrawer
  open={drawerOpen}
  onClose={closeDrawer}
  entityId={id}
  entityType="[EntityType]"
/>
```

## 5. 特殊需求
<!-- 記錄與標準模式的差異、特殊邏輯 -->

## 6. 開發檢查清單

- [ ] 定義核心類型 (Types)
- [ ] 實作 API Client (Service)
- [ ] 建立 List Page (Workspace)
- [ ] 建立 Detail Overlay
- [ ] 建立 Editor Page (Create/Edit)
- [ ] 整合 Workflow (如有)
- [ ] 單元測試 / 整合測試
