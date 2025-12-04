# Contact Module – UI / Interaction Spec

> 狀態：骨架，供彙整既有客戶模組的聯絡人畫面定義。

## 主要畫面

- **Contact Workspace**：搜尋 / 篩選 / 分頁，支援篩選 owner、客戶、標籤。
- **Contact Drawer**：新建 / 編輯，需支援 Ext Attr、Auto Assign owner。
- **Follow-up Drawer / Detail**：在 Customer 模組中嵌入的聯絡人清單。
- **Async Select**：報價/訂單的 owner/contact 選單（關鍵字 + 權限過濾）。

## 互動要點

- 鍵盤輸入須支援中文 IME，避免重複發 request。
- 搜尋結果顯示 contact fullName + 客戶名稱，hover 顯示 email/phone。
- Drawer 儲存後應可回填至呼叫端並提示成功訊息。
- Ext Attr 需彈性配置行數（三欄網格）。

## TODO

- [ ] 從 `docs/ui/customer/Customer Workspace Wireframe.md` 擷取相關 wireframe。
- [ ] 標示 Drawer/Detail 的欄位群組、驗證規則。
- [ ] 訂定 Async Select 的 loading、空狀態、錯誤訊息。
