# UI 元件與規範文檔

本目錄包含 Flexora 前端 UI 元件的使用指南與開發規範。

## 📚 核心文檔

### 1. Workspace 元件 (New 🚀)
[Workspace Components Guide](./workspace-components.md)
- 包含 `ModuleWorkspace`, `ModuleSidebar` 等模組主頁元件的使用說明。
- 定義了 Overlay 與 Drawer 的設計規範。

### 2. Button 元件
[Button Component Guide](./button.md)
- 詳解按鈕的 `variant`, `size`, `shape` 與使用規範。

### 3. 類型定義指南
[Frontend Type Definition Guide](../types/type-definition-guide.md)
- 前端 TypeScript Interface 定義的最佳實踐。
- 如何對應後端 DTO。

---

## 📦 共用業務元件

以下為各模組通用的業務邏輯元件說明。部分元件目由報價模組 (`views/sales/quotes`) 孵化中，未來將遷移至 `src/components/shared`。

### AddressCard

| 項目     | 說明                                                                                            |
| -------- | ----------------------------------------------------------------------------------------------- |
| 檔案     | 待實作/遷移 (預計: `src/components/shared/AddressCard.tsx`)                                      |
| 用途     | 顯示帳單/送貨地址摘要。支援 companyName、line1/fullAddress、recipientName/Phone。               |
| Props    | `titleKey`（i18n key）、`address`（AddressSnapshot 物件或 JS 物件）、`placeholderKey`（可選）。 |
| 互動     | 無狀態；若 `address` 為 `null` 會顯示 placeholder。                                             |
| 佈局     | 內建 border/padding；可放入 Bootstrap grid。                                                    |
| 使用範例 | Quotation Detail Page、Quick Create Drawer（Step 3）。                                          |

### ExtAttrPanel

| 項目     | 說明                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 檔案     | `src/views/sales/quotes/components/ExtAttrPanel.tsx` (需遷移至 shared)                                                          |
| 用途     | 顯示/編輯 Quotation Revision ExtAttr。根據 `dataType` 自動選擇輸入類型。                                                        |
| Props    | `definitions: IQuotationRevisionExtAttrDef[]`、`values: Record<string,string>`、`onChange(code,value)`、`disabled`、`columns`。 |
| 特性     | 支援 INT/DECIMAL/DATE/BOOL/JSON/STRING；Boolean 會顯示 Select；`disabled` 時仍顯示值。                                          |
| 注意     | `columns` 預設 2，可調整以適應 Drawer/Detail。                                                                                  |
| 使用範例 | Quick Drawer Step 2（可編輯）、Detail Page（唯讀）。                                                                            |

### AttachmentList

| 項目     | 說明                                                                       |
| -------- | -------------------------------------------------------------------------- |
| 檔案     | 待實作/遷移 (預計: `src/components/shared/AttachmentList.tsx`)                                |
| 用途     | 顯示 `DocumentLink` 列表（檔名、關聯類型、檔案類型、連結時間、是否主要）。 |
| Props    | `attachments: IDocumentLink[]`、`emptyKey`（可選）。                       |
| 佈局     | 採 `<ul>` 列表，內建 border-bottom 分隔。                                  |
| 互動     | 無狀態；未來可延伸加入下載/預覽按鈕。                                      |
| 使用範例 | Quotation Detail Page 附件區。                                             |

### ContactLookupSelect

| 項目     | 說明                                                                                                                           |
| -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 檔案     | `src/components/shared/ContactLookupSelect.tsx`                                                                                |
| 用途     | 依照所選客戶呼叫 `/api/contacts/lookup` 取得聯絡人下拉選項（含 Owner 可視範圍過濾），提供 Quick Drawer / Full Editor 共用。 |
| Props    | `customerId?: string`、`value: ContactOption | null`、`onChange(option)`、`placeholder`（可選）、`isDisabled`、`className`。      |
| 特性     | 內建 AsyncSelect、支援 keyword 搜尋、會快取 server 回傳結果並在客戶切換時清空；無客戶時自動 disable 並提示「請先選擇客戶」。 |
| 互動     | onChange 回傳 `{ value, label, email?, phone? }`；父層可決定是否將 label 寫回 `contactName`。                                   |
| 使用範例 | QuickCreateDrawer 基本資訊區、QuoteEditorPage 左上基本欄位。                                                                   |

## 🔗 相關資源
- [模組設計範本](../../modules/module-design-template.md)
- [Quotation UI Refactor Notes](./quotation-ui-redesign-plan.md)
