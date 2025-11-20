# 共用元件設計筆記

> 資料夾內元件已於 2025-11-16 首波導入 Quotation Hub（詳情頁 + Quick Drawer）。以下為目前可用的三個元件說明，後續如有新增請更新本檔案。

## AddressCard

| 項目     | 說明                                                                                            |
| -------- | ----------------------------------------------------------------------------------------------- |
| 檔案     | `app/shared/components/address-card.tsx`                                                        |
| 用途     | 顯示帳單/送貨地址摘要。支援 companyName、line1/fullAddress、recipientName/Phone。               |
| Props    | `titleKey`（i18n key）、`address`（AddressSnapshot 物件或 JS 物件）、`placeholderKey`（可選）。 |
| 互動     | 無狀態；若 `address` 為 `null` 會顯示 placeholder。                                             |
| 佈局     | 內建 border/padding；可放入 Bootstrap grid。                                                    |
| 使用範例 | Quotation Detail Page、Quick Create Drawer（Step 3）。                                          |

## ExtAttrPanel

| 項目     | 說明                                                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 檔案     | `app/shared/components/ext-attr-panel.tsx`                                                                                      |
| 用途     | 顯示/編輯 Quotation Revision ExtAttr。根據 `dataType` 自動選擇輸入類型。                                                        |
| Props    | `definitions: IQuotationRevisionExtAttrDef[]`、`values: Record<string,string>`、`onChange(code,value)`、`disabled`、`columns`。 |
| 特性     | 支援 INT/DECIMAL/DATE/BOOL/JSON/STRING；Boolean 會顯示 Select；`disabled` 時仍顯示值。                                          |
| 注意     | `columns` 預設 2，可調整以適應 Drawer/Detail。                                                                                  |
| 使用範例 | Quick Drawer Step 2（可編輯）、Detail Page（唯讀）。                                                                            |

## AttachmentList

| 項目     | 說明                                                                       |
| -------- | -------------------------------------------------------------------------- |
| 檔案     | `app/shared/components/attachment-list.tsx`                                |
| 用途     | 顯示 `DocumentLink` 列表（檔名、關聯類型、檔案類型、連結時間、是否主要）。 |
| Props    | `attachments: IDocumentLink[]`、`emptyKey`（可選）。                       |
| 佈局     | 採 `<ul>` 列表，內建 border-bottom 分隔。                                  |
| 互動     | 無狀態；未來可延伸加入下載/預覽按鈕。                                      |
| 使用範例 | Quotation Detail Page 附件區。                                             |

---

### 待辦 / 後續

1. Storybook：補 AddressCard / ExtAttrPanel / AttachmentList 的展示案例（含暗色主題）。
2. Attachment 行為：一旦 Document Service Ready，可加入「下載／預覽」按鈕或 CTA。
3. ExtAttrPanel：未來需支援多語 label、字數限制、validation 提示。
