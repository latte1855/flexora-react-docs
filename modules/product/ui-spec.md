# Product Module – UI Spec (草稿)

## 介面組成

| 區塊 | 說明 |
| --- | --- |
| **Filter Hub** | 與報價 Workspace 同版型：左側列出 Preset（全產品、僅上架、缺價目表等）＋進階篩選（關鍵字、ItemGroup、Owner、狀態、PriceList）。 |
| **主工作區** | 預設 List View；Pipeline 模式保留 TODO（若未來有「新品導入→上架→停售」流程再啟用）。 |
| **操作列** | 「快速建立產品」 Drawer（最小欄位）＋「建立產品」滿版編輯（含 SKU/Price 卡片）。按鈕位置沿用報價、客戶模組。 |

## List View

- 欄位：SKU/Item 名稱、ItemGroup/分類、UoM、稅別、狀態、最後更新者。
- 支援多選批次啟用/停售、標籤設定。
- Row 點擊 → 開啟右側 Detail / Drawer，內含 SKU 表格與 PriceList 摘要。

## Item Detail / 編輯畫面

1. **資訊卡**：ItemGroup、產品主檔欄位、`hasVariants` 開關。
2. **SKU 區塊**：表格 + Drawer。  
   - Drawer 內容：UoM、稅別、包裝、條碼、預設倉庫。  
   - 內嵌 PriceList preview（顯示各價目表建議價）。
3. **變形（Variants）設計**：若 `hasVariants=true`，提供維度選擇器與批次產生 SKU 流程。
4. **Ext Attr / 附註**：與報價/客戶模組一致的三欄網格輸入。

## PriceList / ItemPrice

- List 顯示所有 PriceList，可由產品模組進入或獨立入口。  
- 在 SKU Drawer 或 PriceList 詳細頁中維護 ItemPrice，支援匯入/複製、版本歷史。
- 若未來 PriceList 需要獨立模組，可在 README 中連結。

## Reference / TODO

- [ ] 將既有 wireframe（`docs/specs/phase3-inventory/product*.md`、`docs/ui/quotation-ui-wireframe.md`）搬遷至此。
|- [ ] Pipeline 模式需求確認（目前註記在 implementation plan）。
|- [ ] 價格預覽與報價模組互動詳細流程。
