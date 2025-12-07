# Product Module – UI Spec (草稿)

## 全域佈局（沿用 Quote Workspace 樣式）

```
┌──────── Filter Hub ────────┐┌───────── Workspace ─────────┐
│Preset 切換 + Chips         ││操作列：快速建立 / 建立產品      │
│- 全部產品 ALL             ││View 切換：List / Pipeline*   │
│- 僅上架                    ││列表 / Pipeline 本體          │
│- 缺價目表                  ││右側 Detail（可折疊）          │
│- 客製（收藏的條件）        │└────────────────────────────┘
└────────────────────────────┘
```

- **套版原則**：本模組必須與 `/sales/quotes` 完全一致的操作體驗。Hub 可收合、右側有固定的 Filter Toolbar + List/Pipeline 切換，左右欄距離維持 sidebar 280px + 16px gap，不得額外加 padding。所有按鈕、Segment、字體、圓角以報價單為準。
- **Preset 行為**：點選後更新 `ownerScope + filters`，並在 Chips 顯示條件。
- **進階篩選**：搜尋（支援中文 IME，按「套用」或 Enter 才查詢）、ItemGroup、分類、Owner、狀態、PriceList、是否有 Variants。
- **操作列**：左側 `Segment (列表/Pipeline)`、中間 `流程圖`（GitBranch icon），右側依序為黑底 `快速建立產品`（Sparkles icon，Drawer 寬 640px）與主色 `建立產品`（FilePlus icon，滿版 Editor）。
- **Detail Panel**：點選列時開啟「滿版 Overlay」，排版與 QuoteDetail 相同：大卡片包裹左右欄、右側為「快速操作 + 統計卡」，所有按鈕尺寸、排序、icon 完全比照報價單。
- **Pipeline 視圖**：*待確認*；如無流程需求則隱藏切換器，TODO 已於 implementation plan 註記。
- **Wireframe**：  
  ```
  [Filter Hub 260px] | [Main List 2/3] | [Detail Panel 1/3]
  └─操作列 (Quick Create / 建立產品) 置於 Workspace 上方
  └─List Row -> Drawer (基本 + SKU + PriceList)
  ```
  > Figma 草稿：`https://figma.com/file/TBD/flexora-product?node-id=hub`（TODO：待設計完成後更新）。
  > PriceList Drawer Wireframe：`https://figma.com/file/TBD?node-id=pricelist-drawer`。

## List View

| 欄位 | 說明 | 互動 |
| --- | --- | --- |
| Item / SKU 名稱 | 顯示 Item 名稱 + SKU 編號；若 `hasVariants`，以 Badge 標示 | 點擊 → Detail Panel |
| ItemGroup / 分類 | 兩行顯示（Group 上面、Category 下面） | 可排序/篩選 |
| UoM / 稅別 | 顯示預設 UoM、稅代碼 | Drawer 中可編輯 |
| Status | 標示 `草稿 / 審核中 / 上架 / 停售` | 直接顯示，批次操作可改變 |
| Last Modified | 使用者 + 時間 | 依最新更新排序 |

- **批次操作**：Checkbox 選取後，可批次「啟用/停售」、「移動至 ItemGroup」、「同步價目表」。
- **Row Action**：Hover 顯示「編輯」、「複製 SKU」、「查看價目」。
- **空狀態**：提供「建立產品」CTA 與導向文案（與報價一致）。

## Detail / Drawer / Full Editor

1. **Detail Overlay 標頭**：左側「檢視中 + Item 名稱 / code + Tag + Status Badge」，右側按鈕依序為「上一筆 / 下一筆 / 流程圖 / 快速建立產品 / 關閉」，尺寸、icon、顏色與 QuoteDetail 相同（黑底 Sparkles、plain 上下筆）。
2. **左右欄**：外層使用預設 `<Card>`（無額外圓角/padding），內層小卡沿用 QuoteDetail：左邊為主資訊 + ExtAttr + Tabs，右邊為「快速操作」(GitBranch、建立新版本、啟動 Workflow、查看 SKU/PriceList) 與「統計資訊」。卡片圓角、陰影、間距需完全一致。
3. **Item Info 卡**：顯示 ItemGroup、Owner、Channel、Default PriceList、更新時間、`hasVariants`。描述放在卡片底部。
4. **SKU 表格**：與 List 類似，但附加庫存狀態、條碼。  
   - 表格上方提供「新增 SKU」「匯入 CSV」「Variant Generator」。  
5. **SKU Drawer**：欄位包含：
   - 基本資訊：SKU No、名稱、描述、啟用狀態。
   - 資料屬性：UoM（Async Select）、稅別、包裝、條碼、預設倉庫。
   - PriceList 摘要：以 Accordion 或表格顯示各價目表的建議價、有效期，並提供「編輯價目表」連結。
   - Ext Attr：三欄網格（若該 SKU 有客製欄位）。
6. **Variant Flow**（僅 `hasVariants=true`）：
   - Step 1：選擇維度（顏色、尺寸…），設定維度值。
   - Step 2：系統預覽 SKU 組合，使用者可取消/重新命名。
   - Step 3：產生 SKU，進入表格；每個 SKU 還是可單獨編輯。
7. **Detail 卡片格式**：欄位卡使用 `rounded-2xl border border-gray-100 bg-white/70 p-4 shadow-sm`，欄位值加 `font-semibold`；快速操作 / 統計卡標題使用 `text-xs text-gray-500`、內容 `text-sm`，不得改變 padding 以維持與報價單一致。
8. **Full Editor（建立產品）**：
   - 版面沿用報價整單編輯：左側主資訊卡 + SKU 區塊，右側為 PriceList 摘要/操作紀錄。
   - Page Footer：固定顯示「取消 / 儲存 / 送審」，若流程未啟用則只顯示 儲存。

## PriceList / ItemPrice 互動

- **在 SKU Drawer**：顯示一個簡易表格：
  | 價目表 | 建議價 | 幣別 | 狀態 | 有效期 |
  - 點「編輯」跳至 PriceList 頁面（或打開子 Drawer）。
  - 若缺少價目表，顯示警示訊息（同報價缺價目提示）。
- **PriceList 主頁**（可獨立 / 由產品連結維護）：
  - Hub + List 佈局，與產品一致。
  - 支援一次對多個 SKU 批次調價或匯入。
- **審批提示**：若 PriceList 有審批流程，在 Drawer/Detail 中顯示「待審」「已核准」標籤。

## TODO / 待補

- [ ] 產出 wireframe 截圖並嵌入（可引用 Figma 或原型圖）。
- [ ] Pipeline 模式需求確認（例如新品導入流程）；若取消需更新 README。
- [ ] PriceList 子 Drawer / 全頁畫面細部欄位（含匯入流程）。
- [ ] Variant Generator 的錯誤處理（重複 SKU、缺維度值）。
- [ ] 與報價、庫存共用的 Async Select 元件在畫面中的置入示例。  
- [ ] 提供 PriceList 缺漏提示的 wireframe（例如 Detail Panel 的提醒）。  
