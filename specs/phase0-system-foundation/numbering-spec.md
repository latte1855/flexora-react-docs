# Flexora ERP — 編號 / 流水號規格（Numbering Service Specification）
Phase 0 — System Foundation

本文件定義 Flexora ERP 的 **統一編號/流水號（Numbering）策略與規格**，  
包括編號規則、格式、避免碰撞（collision）策略、遞增邏輯、Owner 隔離（若需要）等。

> 所有業務模組（Quotation / SalesOrder / Purchase / Warehouse / …）  
> 都會使用此編號邏輯，因此必須穩定、可靠、可擴充。

---

# 1. 編號服務（Numbering Service）概述

Flexora ERP 採用獨立的 **NumberingService**（或 NumberSequenceService）管理所有流水號。  
其特性：

- 單一負責編號（Single Responsibility）
- 不依賴 UI 或業務層邏輯
- 可支援多模組、多 Owner、多格式的編號
- 必須具備併發安全（Concurrent Safe）

---

# 2. 編號的適用對象（何時需要編號）

下列類型的資料 **必須使用編號服務產生 No**：

| 模組 | 需產生編號 |
|------|-------------|
| Quotation | `quotationNo` |
| Sales Order | `salesOrderNo` |
| Delivery Note | `deliveryNoteNo` |
| Purchase Order | `purchaseOrderNo` |
| Vendor Return | `returnNo` |
| Inventory Transaction | `txnNo`（可選） |
| Warehouse Document | `whDocNo`（若需要） |

> ⚠「No（可讀編號）」與「ID（主鍵）」必須分開。  
> ID = 業務無關  
> No = 業務識別、給人看的、可搜尋的

---

# 3. 編號格式規範（Format Rules）

Flexora ERP 所有編號須符合以下統一原則：

## 3.1 一般格式（建議）

```

<Prefix>-<YYYYMMDD>-<Seq>

```

範例：

- `QTN-20251119-00001`
- `SO-20250201-00012`
- `PO-20250315-00007`

## 3.2 元件拆解

| 元件 | 說明 |
|------|------|
| Prefix | 模組代碼，例如 QTN、SO、PO |
| YYYYMMDD | 當天日期（或 YYYYMM，依需求） |
| Seq | 當日（或當月）遞增序號（零補齊） |

> 若有 Owner、多公司需求，可擴充為：  
> `<OwnerCode>-<Prefix>-<YYYYMMDD>-<Seq>`

---

# 4. 序號（Seq）的管理策略

序號是整個編號規則中最重要的部分。

## 4.1 Seq 與日期 / Owner 維度

Flexora 目前規劃：

- **預設以「日期 + Prefix」為序號維度**
- 之後若有多公司需求，可再加上 Owner 維度

一般來說：

```

( Prefix + Date ) → seq

```

例：

| Prefix | Date | Seq |
|--------|------|------|
| QTN | 2025-11-19 | 1 |
| QTN | 2025-11-19 | 2 |
| QTN | 2025-11-20 | 1（重新開始） |

## 4.2 序號的持久化（Persistence）

為避免並發衝突，序號**必須儲存在資料庫**，不可僅依照查詢最大值生成。

建議建立：

### `number_sequence`（示例 Schema）

| 欄位 | 說明 |
|------|------|
| id | 主鍵 |
| prefix | 模組代碼 |
| date_str | 例如 `20251119` |
| owner_id |（可選） |
| seq | 下一次要用的序號 |
| updated_at | 最後更新時間 |

主鍵建議：

```

unique(prefix, date_str, owner_id)

```

## 4.3 遞增邏輯（atomic increment）

### 必須達成：

- **並發安全（Atomic）**
- **避免跳號（除非明確允許）**
- **避免取到同一個 seq**

建議實作：

- 使用 `SELECT ... FOR UPDATE`
- 或使用 PostgreSQL 原生序列（需搭配 prefix/date partition）

---

# 5. Numbering Service 設計

### 5.1 介面（Interface）

```java
public interface NumberingService {
    /**
     * 產生編號，如 QTN-20251119-00001
     */
    String generateNumber(String prefix);
}
```

可擴充版本：

```java
String generateNumber(String prefix, Long ownerId);
String generateNumber(String prefix, LocalDate date);
```

---

# 6. 完整流程範例

以下展示編號產生的完整流程：

### Step 1：取得 prefix

例如報價：

```java
String prefix = "QTN";
```

### Step 2：組出今天的 date_key

例如 `20251119`

```java
String dateKey = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);
```

### Step 3：在 number_sequence 表查詢/鎖定

伺服器執行：

```
SELECT * FROM number_sequence 
WHERE prefix = 'QTN' AND date_str = '20251119'
FOR UPDATE;
```

### Step 4：若無紀錄，新增 seq = 1

若有紀錄：

```
seq = seq + 1
```

### Step 5：組出編號

```
QTN-20251119-00001
```

---

# 7. 併發控制（Concurrency Control）

必須確保：

* 多個使用者「同時新增單據」時不會拿到相同編號
* 不會跳號（例如從 1 跳到 3，但沒有 2）

建議：

* **必須使用資料庫鎖（FOR UPDATE）**
  或
* 修改 seq 欄位時採用 **atomic update**

---

# 8. 錯誤處理規範（Error Handling）

Numbering Service 發生錯誤時，應拋出統一錯誤：

```
error.numbering.failed
```

並讓前端顯示：

* 「編號產生失敗，請稍後再試」
* 避免暴露 DB 細節訊息

---

# 9. 擴充性（Extensibility）

本編號策略預留以下擴充能力：

* Owner 隔離（跨公司獨立序號）
* 月序號（YYYYMM + Seq）
* 客戶自訂格式（例如 `SO/<year>/<seq>`）
* Entity-based prefix （例如 `prefix = entity.getClass().getSimpleName()`）

---

# 10. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                    |
| ---- | ---------- | ----- | --------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Flexora ERP 編號策略初版 |

