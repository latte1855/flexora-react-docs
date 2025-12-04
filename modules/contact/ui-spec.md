# Contact Module – UI Spec

> 聯絡人畫面將與 Customer Workspace 密切整合，以下為新版 UI 行為。

## Contact Workspace

- **Filter Hub**：Owner、Customer、標籤、狀態、城市。預設條件：`我的聯絡人`、`5 天內未跟進` 等。  
- **列表**：欄位（姓名、Customer、職稱、Email、Phone、Owner、狀態、最近活動）。  
- **操作列**：`快速建立聯絡人` Drawer、`匯入`、`匯出`。  
- **Detail Panel**：顯示聯絡人摘要（姓名、客戶、聯絡方式、備註）、關聯活動、Ext Attr。

## Drawer / 編輯流程

1. **基本資訊**：姓/名、顯示名稱、客戶、職稱、部門、Owner。  
2. **聯絡方式**：Email、Mobile、Phone、Assistant、首選語言。  
3. **地址 / 偏好**：Mailing Address、時區。  
4. **Ext Attr**：三欄網格。  
5. **標記**：`isPrimary`（主聯絡人）、標籤。  
6. 儲存後可選擇「繼續建立下一位」。

## 與 Customer Detail 的整合

- 在 Customer Detail 的 Contacts Tab 內嵌 Contact 列表，支援搜尋、新增、設定主聯絡人。  
- 在 Customer Drawer 中提供「立即新增聯絡人」子 Drawer。  
- Detail Panel 需顯示活動紀錄、近期郵件（若有）。

## Async Select

- 通用元件 `ContactLookupSelect`：支援 keyword + customerId + ownerScope。  
- 顯示：`FullName (Customer)`，次行顯示 Email / Phone。  
- Loading / Error 狀態需顯眼，支援中文輸入（一樣需 Debounce & IME fix）。

## TODO

- [ ] 將最新 wireframe / Figma 圖貼入。  
- [ ] 列出欄位驗證（Email、手機格式、必填規則）。  
- [ ] 規劃匯入/匯出 (CSV) 流程與 UI 提示。  
- [ ] 若要顯示活動/郵件整合，需在此標註 API 需求。
