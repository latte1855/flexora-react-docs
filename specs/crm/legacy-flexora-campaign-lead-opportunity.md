# CRM (Campaign / Lead / Opportunity) Legacy 規格摘要

依據 `docs/migration/flexora-react/flexora-campaign-lead-opportunity-spec.md`，此文闡述 Campaign、Lead、Opportunity 的核心模型與資料欄位。在實作上請以 `flexora-react/src/main/java/com/asynctide/flexora/domain/crm`（如 `Campaign`, `Lead`, `Opportunity`）以及對應的 `service`/`resource` 為準。

## 1. 模組定位

- **Campaign**：行銷活動，記錄花費/成效來源與狀態四表，支援 metadata `properties` JSONB。
- **Lead**：潛在客戶/聯絡人，可獨立存在或歸屬 `Customer`，保留 email/phone/metadata。
- **Opportunity**：商機，連結 Lead 與 Campaign，並延伸報價/估價資訊；可附 `pipeline_stage`、`estimated_amount` 等欄位。

## 2. 共通規則

- 所有 `*_no` 採 `partial unique (WHERE deleted=false)`；時間/數值規格（UTC TIMESTAMP、`DECIMAL(19,4)`、百分比0-1）與 `phase0` 文件一致。
- ExtAttr 採 `*AttrDef / *AttrValue`（定義表 + 值表）設計，方便跨模組擴充。
- DB 驅動狀態機：每張單據配有 `status_def / event_def / state_transition / status_history` 表，metadata 以 JSONB 保存。

## 3. 下一步

- 如需更完整的欄位表格或 ER 圖，請開啟原始 legacy doc。
- 所有欄位調整應同步記錄於 `docs/architecture/DATA_MODEL.md` 與 `docs/specs/phase0-system-foundation/domain-overview.md`（Owner/User 關聯）。
