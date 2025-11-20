# Flexora ERP 開發規範 – 資料結構與版本管理（含 JaVers）

## 目錄

1. [時間處理](#時間處理)
2. [數值處理](#數值處理)
3. [流程設計（Status/Event/Transition/History）](#流程設計statuseventtransitionhistory)
4. [流程 Metadata VO（JSONB 映射）](#流程-metadata-vojsonb-映射)
5. [表單地址處理（JSONB + VO）](#表單地址處理jsonb--vo)
6. [Enum vs Lookup Table](#enum-vs-lookup-table)
7. [審計基底類別（AuditingEntity）](#審計基底類別auditingentity)
8. [軟刪除（Soft Delete）](#軟刪除soft-delete)
9. [JaVers（資料版本管理）](#javers資料版本管理)
10. [其他開發規則](#其他開發規則)
11. [落地建議](#落地建議)

---

## 時間處理

- **事件時間（發生點、建立/修改時間）**

  - 使用 **`Instant (UTC)`** 儲存，DB 欄位為 `TIMESTAMP`（一律轉換成 UTC 存）。
  - 前端顯示時依使用者時區（預設 Asia/Taipei）轉換。

- **僅日期（無時區語意）**

  - 使用 **`LocalDate`**（DB 用 `DATE`）。
  - 例：生日、合約起迄日、有效期限。

- **需要保留時區規則（少數情境）**

  - 如「每週一 09:00 Asia/Taipei 的排程」才用 `ZonedDateTime`。

- **禁止**
  - 用 `String` 存日期或時間。

---

## 數值處理

- **金額、數量、稅率、折扣** ➜ **`BigDecimal`**

  - 金額：`precision=19, scale=4`
  - 稅率/折扣：`scale=6`
  - **計算必須指定 `RoundingMode`**（建議 `HALF_UP`）。

- **純計數/天數/件數** ➜ `Integer`/`Long`。

- **前端 API 傳輸** ➜ 金額/比率以 **字串**（如 `"100.00"`）避免 JS 浮點誤差。

---

## 流程設計（Status/Event/Transition/History）

- **規範**：所有業務流程必須採用 **表驅動設計**，不可僅依 enum 實作。
- **必備四表**

  - `*_status_def`：狀態定義（含名稱、是否終結、metadata）
  - `*_event_def`：事件定義（含名稱、payload schema、metadata）
  - `*_state_transition`：狀態轉換（from→event→to，含 guard_expression、action_handler、metadata）
  - `*_status_history`：狀態歷史紀錄

- **metadata 欄位**

  - 一律使用 `JSONB`，並對應 VO 類別，而非 Map。
  - 範例：
    ```json
    {
      "uiColor": "#EF4444",
      "badge": "danger",
      "editable": false,
      "role": ["PURCH_MGR"]
    }
    ```

- **程式修改規則**

  - 新增/刪除狀態、事件、轉換 → 改表即可，不需重發程式。
  - 新增全新副作用行為（ActionHandler） → 才需要改程式並實作新處理器。

- **快取與驗證**
  - 啟動時讀取四表至快取，並做完整性檢查（是否有孤兒、閉環等）。
  - 提供「重新載入流程定義」管理功能，可即時套用變更。

---

## 流程 Metadata VO（JSONB 映射）

所有流程相關表格的 `metadata (JSONB)` 必須對應固定 VO 類別，避免使用 `Map<String,Object>`。

### 狀態定義 (`*_status_def.metadata`)

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class StatusMetadataVO {

  private String uiColor; // 狀態顏色，如 #EF4444
  private String badge; // 標籤樣式，如 "danger"
  private Boolean editable; // 是否允許編輯表單
  private List<String> role; // 可見角色清單
}

```

### 事件定義 (`*_event_def.metadata`)

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class EventMetadataVO {

  private String icon; // UI icon 名稱
  private List<String> role; // 哪些角色可觸發
  private Boolean confirm; // 是否需要二次確認
}

```

### 狀態轉換 (`*_state_transition.metadata`)

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TransitionMetadataVO {

  private String buttonLabel; // 按鈕顯示文字
  private Boolean visible; // 是否顯示按鈕
  private String category; // 分組分類
}

```

### Entity 中使用範例

```java
@Column(columnDefinition = "jsonb")
@Type(JsonType.class)
private StatusMetadataVO metadata;

```

---

## 表單地址處理（JSONB + VO）

- **規範**

  - 單據上的地址一律使用 **JSONB 欄位** 儲存，**不開獨立資料表**。
  - Entity 層定義為 **VO（值物件類別）**，非 Entity。
  - Flexora 既有 VO：`com.asynctide.flexora.domain.vo.AddressSnapshot`，並透過 `AddressSnapshotConverter` 自動轉換 JSONB。

- **範例**

  ```java
  @Data
  public class AddressVO {

    private String country;
    private String state;
    private String city;
    private String postalCode;
    private String line1;
    private String line2;
  }

  @Entity
  public class PurchaseOrder {

    @Column(columnDefinition = "jsonb")
    @Type(JsonType.class)
    private AddressVO billingAddress;

    @Column(columnDefinition = "jsonb")
    @Type(JsonType.class)
    private AddressVO shippingAddress;
  }

  ```

* **MapStruct Deep Copy 規範**

  - VO 欄位在 Entity ↔ DTO 轉換時，必須 **deep copy**，避免修改 DTO 時同步影響到 Entity → 造成非預期自動更新。
  - 建議實作 `@AfterMapping` 或自訂 Mapper，確保 `new AddressVO(...)`。

* **範例**

  ```java
  @Mapper(componentModel = "spring")
  public interface AddressMapper {
    AddressVO toDto(AddressVO vo);
    AddressVO toEntity(AddressVO vo);

    @AfterMapping
    default void deepCopy(@MappingTarget AddressVO target, AddressVO source) {
      if (source != null) {
        target.setCountry(source.getCountry());
        target.setState(source.getState());
        target.setCity(source.getCity());
        target.setPostalCode(source.getPostalCode());
        target.setLine1(source.getLine1());
        target.setLine2(source.getLine2());
      }
    }
  }

  ```

## Properties / ExtAttr JSON 規範

- `properties`（或同語意欄位）一律以 JSONB 儲存，自訂欄位值需為「JSON 物件」。
- 後端在寫入前會以 Jackson 驗證字串是否為合法 JSON 物件，若為空字串則自動正規化為 `{}`，避免後續讀取噴錯。
- API 若收到非 JSON 物件（例如純字串、陣列或格式錯誤）即回傳 400，errorKey=`invalidJson`，caller 必須修正再送。

---

## Enum vs Lookup Table

- **Enum 使用時機**

  - 僅用於「程式中長期穩定不變的常數」，如錯誤碼、幣別代碼常數、ActionHandler key。
  - 不再用來描述「流程狀態/事件」。

- **Lookup Table 使用時機**

  - 需要後台維護、可增刪、需````語或排序的字典資料。

---

## 審計基底類別（AuditingEntity）

- 所有業務實體 **繼承 `AbstractAuditingEntity`**，自動帶入：

  - `createdBy`, `createdDate (Instant)`
  - `lastModifiedBy`, `lastModifiedDate (Instant)`

- 建議再加：

  - `@Version`（樂觀鎖）
  - 軟刪欄位：`deleted (Boolean)`, `deletedAt (Instant)`, `deletedBy (String)`

- **Spring 設定**：

  - `@EnableJpaAuditing`
  - 提供 `AuditorAware<String>`，從登入帳號取得使用者資訊。

---

## 軟刪除（Soft Delete）

- **欄位設計**

  - `deleted`, `deletedAt`, `deletedBy`。

- **JPA 實作**

  - `@SQLDelete` + `@Where(clause="deleted=false")` 或全域 Filter。

- **唯一鍵**

  - 必須包含 `deleted=false` 條件。

- **查詢效能**

  - 建 `(deleted, 業務鍵)` 覆合索引。

- **清理策略**

  - 依保留期批次清理或分割區 DROP。

---

## JaVers（資料版本管理）

（同原始規範，無變動，僅調整排版）

---

## 其他開發規則

- **ID 與唯一鍵**

  - 主鍵建議 `BIGINT` 或 `UUID`。
  - 唯一鍵要納入 `deleted=false`。
  - **本系統不使用 `tenant_id` 欄位**。

    - 未來若要多租，將採「多 App 部署 + 獨立資料庫」實現，而非單 DB 混租。

- **欄位命名**

  - 金額 → `xxxAmount`
  - 數量 → `xxxQty`
  - 稅率/折扣 → `xxxRate` / `xxxPct`
  - 時間點 → `occurredAt` (Instant)
  - 日期 → `validFrom`, `validTo` (LocalDate)

- **API 傳輸格式**

  - Instant → ISO8601 UTC
  - LocalDate → `YYYY-MM-DD`
  - BigDecimal → 字串

- **併發控制**

  - 加 `@Version`，前端遇 409 要提示「資料已被更新，請重新載入」。

- **事件記錄**

  - 需對外同步時用 **Domain Event / Outbox**，避免業務交易與外部通訊綁在一起。

---

## 落地建議

1. 所有實體繼承 `AbstractAuditingEntity`，再加軟刪欄位。
2. 在核心業務表啟用 JaVers。
3. 設定 JaVers `commitProperties` 帶 ownerId, ip。
4. 建 DB 分割區策略，保留 18 個月版本。
5. 前端統一處理時間與數值格式。
6. 流程設計一律表驅動，不使用 enum。
7. 表單地址一律 JSONB + VO，並確保 MapStruct deep copy。
8. 流程 metadata 一律 JSONB + VO，並確保 MapStruct deep copy。

---
