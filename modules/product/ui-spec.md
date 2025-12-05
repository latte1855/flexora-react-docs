# Product Module – UI Spec (草稿)

## 全域佈局

```
┌──────── Filter Hub ────────┐┌───────── Workspace ─────────┐
│Preset 切換 + Chips         ││操作列：快速建立 / 建立產品      │
│- 全部產品 ALL             ││View 切換：List / Pipeline*   │
│- 僅上架                    ││列表 / Pipeline 本體          │
│- 缺價目表                  ││右側 Detail（可折疊）          │
│- 客製（收藏的條件）        │└────────────────────────────┘
└────────────────────────────┘
```

- **Preset 行為**：點選後更新 `ownerScope + filters`，並在 Chips 顯示條件（同報價 Hub）。
- **進階篩選**：搜尋（支援中文 IME，Enter 或按「套用」才觸發）、ItemGroup、分類、Owner、狀態、PriceList、是否有 Variants。
- **操作列**：左側為 `快速建立產品`（Drawer，收集基本欄位）、右側黑底白字 `建立產品`（滿版編輯）。
- **Detail Panel**：點選列表行時於右側展開，顯示摘要與快速動作（啟用/停售、複製 SKU），與報價 Summary 一致。
- **Pipeline 視圖**：*待確認*；如無流程需求則隱藏切換器，TODO 已於 implementation plan 註記。

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

1. **Item Info 卡**：顯示 ItemGroup、品牌、分類、`hasVariants`（toggle）。這裡也顯示 PriceList 建議情況（例如「3 個價目表已設定」）。
2. **SKU 表格**：與 List 類似，但附加庫存狀態、條碼。  
   - 表格上方提供「新增 SKU」「匯入 CSV」「Variant Generator」。  
3. **SKU Drawer**：欄位包含：
   - 基本資訊：SKU No、名稱、描述、啟用狀態。
   - 資料屬性：UoM（Async Select）、稅別、包裝、條碼、預設倉庫。
   - PriceList 摘要：以 Accordion 或表格顯示各價目表的建議價、有效期，並提供「編輯價目表」連結。
   - Ext Attr：三欄網格（若該 SKU 有客製欄位）。
4. **Variant Flow**（僅 `hasVariants=true`）：
   - Step 1：選擇維度（顏色、尺寸…），設定維度值。
   - Step 2：系統預覽 SKU 組合，使用者可取消/重新命名。
   - Step 3：產生 SKU，進入表格；每個 SKU 還是可單獨編輯。
5. **Full Editor（建立產品）**：
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
