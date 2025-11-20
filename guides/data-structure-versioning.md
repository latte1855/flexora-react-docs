# Flexora ERP 開發規範 — 時間、數值、流程、JaVers（Legacy Summary）

本檔整理 `docs/migration/flexora-react/Flexora ERP 開發規範 – 資料結構與版本管理（含 JaVers）.md` 內容（已備份），協助快速回顧時間/數值、流程表、Enum 選擇、審計與 JaVers 建議。實際實作仍以 `flexora-react` 程式碼的 Entity/Service 與 Liquibase migration 為主，並以此檔作為補充說明。

## 1. 時間與數值

- 所有事件時間欄位使用 `Instant`（UTC），僅日期才用 `LocalDate`；禁止用 `String` 存時間。
- 金額/成本 `precision=19, scale=4`；稅率/折扣 `scale=6`；計算 BigDecimal 使用 `RoundingMode.HALF_UP`。
- API 傳輸建議以字串呈現數值（避免 JS 浮點），前端顯示時可依時區轉換。

## 2. 流程治理（State/Event/Transition/History）

- 所有流程採「四表」架構：`*_status_def`、`*_event_def`、`*_state_transition`、`*_status_history`。
- 各表可儲存 metadata（JSONB），並對應 VO 類別；`state_transition` 需提供 guard_expression 與 action_handler，保證服務層可依約定讀表。
- 新增/變更狀態時，僅需更新資料表；若需額外副作用才改程式。

## 3. 其他規則

- Enum vs Lookup：若需後台維護可放 lookup table；單純狀態可預先定義 Enum（前端/後端同步）。
- 審計基底 `AbstractAuditingEntity` 提供 `createdBy`, `createdDate`, `lastModifiedBy`, `lastModifiedDate`，所有 Entity 建議繼承。
- 軟刪除欄位 `deleted`, `deletedAt`, `deletedBy` 為可刪 Entity 必備；Repository 查詢建議預設加 `deleted=false`。
- JaVers：若需版本紀錄，可在 DTO 上加 `@JaversSpringDataAuditable` 並設定 `JaVers` config，方便 diff 與審計。

## 4. 下一步

將上述規範與 `docs/architecture/DATA_MODEL.md`、`docs/specs/phase0-system-foundation/validation-and-rules.md` 結合，在新增 Entity / Guard 時同步檢查。在遇到模糊規則時，先查 `flexora-react` 中的 Domain 或 Liquibase，再回寫本指南。
