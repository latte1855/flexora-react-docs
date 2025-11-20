# 出貨與客退 Legacy 規格摘要

摘自 `docs/migration/flexora-react/flexora-deliverynote-management-spec.md`，並對照 `flexora-react/src/main/java/com/asynctide/flexora/domain/DeliveryNote*`、`CustomerReturn`、`InventoryTransaction` 等實作。

## 1. 流程概覽

| 單據 | 流程 | 核心動作 |
| --- | --- | --- |
| `DeliveryNote` | DRAFT → CONFIRMED → (PARTIALLY_)SHIPPED / CANCELLED | 出貨時觸發 IM `ISSUE`、釋放 Reservation、更新 SO 出貨量，並產生稅額快照 |
| `CustomerReturn` | DRAFT → RECEIVED → INSPECTED → (RESTOCKED / SCRAPPED) → CLOSED | 收貨時生成 IM `RECEIPT`（初進 QUARANTINE），IQC 後決定 RESTOCK 或 SCRAP，並回寫 SO 退貨量 |

## 2. 重要規則

- **Enum**：`discount_type`（NONE/AMOUNT/RATE）、`serial_mode`（LINE/LIST）、`delivery_note.status_code` 等為 `VARCHAR`。
- **數值/時間**：與全域開發規範一致（UTC `TIMESTAMP`、金額 `DECIMAL(19,4)`、quantity `DECIMAL(19,6)`）。
- **資料表**：Header/Line + tax tables + ExtAttr (ExtAttrDef/Value) + DB 驅動狀態四表 + 轉單/序號/附件。
- **稅計算**：與報價 / 銷售共用 `tax_code`/`tax_rate_line`，DN/CRN 含行稅與單頭稅，稅率/折扣採 0–1 表示。

## 3. 參考

1. 快照欄位請確保 `serial_mode` 是否為 `LINE` (單一序號) 或 `LIST` (多序號)；若為後者，利用 `*_item_serial` 表記錄每個序號。
2. 轉單與稅資訊需與 `SalesOrder`、`Quotation`、`InventoryTransaction` 同步，避免資料不一致。
3. 若需查看完整欄位表、ER 圖或測試案例，請直接打開原始 legacy doc。
