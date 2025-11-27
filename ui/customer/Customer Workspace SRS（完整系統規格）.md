# **1. 模組目的與範圍（Scope）**

Customer Workspace 是 Flexora CRM 的核心頁面，提供：

* 客戶清單管理
* 左側快速過濾（Smart Lists）
* 客戶詳細資訊查看
* 客戶快速編輯 Drawer
* 互動狀態（Activity / 將來可擴充成 ActivityLog）
* 跟進管理（Follow-up）
* VIP / 分級顯示
* 支援將來的商機 / 報價 / 訂單整合

本模組在 Phase 1 完成基本 CRUD 與 UI 操作，後端 API 可後補。

前端對應路徑：`/sales/organizations`（左側導航 Sales → Organizations）

---

# **2. 新增資料欄位（後端 Domain）**

### Customer Entity 必加（Version: Phase 1.1）

| 欄位                 | 類型          | 說明                                                    |
| ------------------ | ----------- | ----------------------------------------------------- |
| `lastActivityAt`   | Instant     | 最後互動時間（ActivityLog 自動更新）                              |
| `nextFollowUpDate` | LocalDate   | 下一次跟進日期                                               |
| `nextFollowUpNote` | String(500) | 跟進說明                                                  |
| `segment`          | String(20)  | VIP/A/B/C 分級                                          |
| `healthStatus`     | String(20)  | GREEN / YELLOW / RED（依 lastActivityAt 自動計算，不一定需要存 DB） |

---

# **3. 新增 Entity：ActivityLog（Phase 1.2，可先 UI mock）**

```
ActivityLog {
  id: Long
  customer: Customer
  type: CALL / EMAIL / MEETING / NOTE / QUOTATION_SENT / ORDER_CREATED
  content: String(2000)
  createdAt: Instant
  createdBy: User
}
```

ActivityLog 讓：

* 最近互動（lastActivityAt）
* 冷凍客戶
* 客戶健康度
* 活動流

都能準確運作。

---

# **4. 左側快速過濾清單（Smart Lists）**

左側固定兩大區塊：

---

## **4.1 我的清單（My Lists）**

| 清單名稱   | 條件                          | SQL/QueryDSL                             |
| ------ | --------------------------- | ---------------------------------------- |
| 我負責的客戶 | ownerId = currentUser       | `c.owner.id = :userId`                   |
| 最近新增   | created_at ≥ 30 天           | `c.createdAt >= now - 30d`               |
| 最近有互動  | lastActivityAt ≥ 30 天       | `c.lastActivityAt >= now - 30d`          |
| 需要跟進   | nextFollowUpDate ≤ 今天 + 7 天 | `c.nextFollowUpDate <= currentDate + 7d` |

---

## **4.2 共享清單（Shared Lists）**

| 清單名稱     | 條件                          |
| -------- | --------------------------- |
| 全部客戶     | deleted = false             |
| VIP      | segment = 'VIP'             |
| 冷凍客戶     | lastActivityAt < now - 90 天 |
| 高潛力（有商機） | 將來用 Opportunity Count       |

> 後端 `GET /api/customers?preset=...` 會套用相同條件，其中 `preset` 對應 `CustomerPreset` enum（`MY_OWNED`、`RECENT_CREATED`、`RECENT_ACTIVITY`、`NEED_FOLLOWUP`、`VIP`、`DORMANT`、`ALL`）。前端只需傳入代碼即可由後端統一定義條件，再視需要疊加其他 criteria。

---

# **5. 客戶列表 List View**

### 使用：TanStack Table（ECME Admin 內建 Table 樣式）

---

## **5.1 Table 欄位定義**

| 欄位        | 顯示        | 說明                      |
| --------- | --------- | ----------------------- |
| 客戶名稱 + 代號 | 主欄位       | 兩行顯示：Name + Code        |
| Segment   | Badge     | VIP / A / B / C         |
| Owner     | 使用者頭像     | 顯示負責業務                  |
| 最近互動      | 日期 + 健康燈號 | lastActivityAt          |
| 跟進日期      | 日期 + 小筆記  | nextFollowUpDate / Note |
| 電話        |           |                         |
| Email     |           |                         |
| 行業別       |           |                         |
| 動作        | Button    | View / Edit / More      |

---

## **5.2 健康度判斷（HealthStatus）**

以「距離最近互動的天數 (`daysSinceLastActivity`)」為基準：

| Health | 條件                                             | 顯示 |
| ------ | ---------------------------------------------- | -- |
| GREEN  | `daysSinceLastActivity <= 30`                  | 🟢 |
| YELLOW | `30 < daysSinceLastActivity <= 60`             | 🟡 |
| RED    | `daysSinceLastActivity > 60` 或 `lastActivityAt` 為空 | 🔴 |

> `daysSinceLastActivity = 今天 - lastActivityAt（LocalDate）`  
> 計算邏輯需與後端 `Customer.healthStatus` 演算法一致，ActivityLog 新增/更新後應即時回寫。

---

## **5.3 Follow-up 判斷**

| 狀態          | 條件                          |
| ----------- | --------------------------- |
| Overdue（紅）  | nextFollowUpDate < 今天       |
| Upcoming（黃） | nextFollowUpDate ≤ 今天 + 7 天 |
| OK（綠）       | 其他                          |

---

# **6. 客戶快速預覽 Drawer（右側）**

事件：點選列表任意一列 → Drawer 出現。

### Drawer 區塊：

---

## **6.1 Header**

* 客戶名稱
* Segment Badge
* Owner
* 編輯按鈕（進入 Detail Page）

---

## **6.2 基本資訊**

* 客戶代號
* 電話
* Email
* 網站
* 行業別

---

## **6.3 Follow-up 區塊**

* 日期（DatePicker）
* 說明（Textarea，2行）
* 儲存 / 清除按鈕

> 使用 `GET/PUT /api/customers/{id}/follow-up`，Drawer 與 Detail Page 右側 Follow-up 面板共用同一個 state。

---

## **6.4 最近活動（Recent Activity）**

顯示最近 5 筆 ActivityLog：

```
11/23 · 通話：討論報價 QTN-001
11/18 · 備註：詢問付款時間
...
```

按鈕：`＋ 新增活動`（暫用 Drawer form）

> UI 透過 `GET /api/customers/{id}/summary` 一次取得基本資料、Follow-up 與 `recentActivities`，避免多次呼叫。

---

## **6.5 快速動作**

* 建立報價
* 建立商機
* 建立訂單
* 查看 Detail Page

*(這些按鈕可先不連後端，保持 UI 動線)*

---

# **7. 詳細頁 Detail Page（Tab 化）**

URL: `/sales/organizations/:id`

Tab：

1. 基本資料（General）
2. 聯絡人（Contacts）
3. 互動紀錄（ActivityLog）
4. 報價（Quotations）
5. 商機（Opportunities）
6. 訂單（Orders）
7. 附件（Documents）

採 ECME card-style + Tab。

---

# **8. 操作規格**

---

## **8.1 新增客戶**

Modal 或 Full page：

* Name（required）
* No（required, unique）
* Owner（required）
* 基本欄位
* FollowUp 可選填

---

## **8.2 編輯客戶**

* Drawer 可編簡單欄位
* Detail Page 可編全部欄位
* 自動重新整理列表

---

## **8.3 刪除**

軟刪除（deleted=true）

---

# **9. API Spec（假設版，Codex 後續可分析後端程式碼更新）**

---

## **9.1 客戶清單 API**

```
GET /api/customers
Query Parameters:
  keyword
  ownerId
  segment
  minLastActivityAt
  maxLastActivityAt
  nextFollowUpBefore
  createdAfter
  preset=MY_OWNED|RECENT_CREATED|RECENT_ACTIVITY|NEED_FOLLOWUP|VIP|DORMANT|ALL
  page / size / sort
```

* `preset` 套用 Smart List 預設條件，後端仍可接受額外 criteria 疊加。
* 回傳 `X-Total-Count`/`Link` 分頁標頭與 `CustomerDTO[]`。

---

## **9.2 Drawer API**

```
GET /api/customers/{id}/summary
```

資料包含：

```jsonc
{
  "customer": { /* CustomerDTO 基本欄位 */ },
  "followUp": {
    "nextFollowUpDate": "2025-11-25",
    "nextFollowUpNote": "追蹤報價 QTN-001"
  },
  "healthStatus": "GREEN",
  "recentActivities": [
    {
      "id": 9801,
      "logType": "CALL",
      "content": "討論報價 QTN-001",
      "occurredAt": "2025-11-23T08:20:00Z",
      "createdBy": "angela"
    }
  ]
}
```

> `recentActivities` 預設取最新 5 筆（後端可用 `limit 5` 或 `findTop5ByCustomerIdOrderByOccurredAtDesc`），供 Drawer 顯示。

---

## **9.3 Follow-up API**

```
GET /api/customers/{id}/follow-up
Response:
{
  "nextFollowUpDate": "2025-11-25",
  "nextFollowUpNote": "追蹤報價 QTN-001"
}
```

```
PUT /api/customers/{id}/follow-up
Request/Response Body:
{
  "nextFollowUpDate": "2025-11-28",
  "nextFollowUpNote": "改以新版報價跟進"
}
```

---

## **9.4 ActivityLog API**

```
GET  /api/customers/{id}/activities?page=0&size=20
POST /api/customers/{id}/activities
```

* `GET`：依客戶取得活動列表，預設以 `occurredAt desc` 排序，Detail Page Tab 使用。
* `POST`：建立活動後需刷新 `Customer.lastActivityAt`/`healthStatus`。

---

# **10. 權限**

| 操作         | 權限                                    |
| ---------- | ------------------------------------- |
| 查看列表       | DataScope (Owner / Dept / Team / All) |
| 新增         | CRM_CUSTOMER_CREATE                   |
| 編輯         | CRM_CUSTOMER_UPDATE                   |
| 刪除         | CRM_CUSTOMER_DELETE                   |
| 查看報價/商機/訂單 | 依對應模組權限                               |

---

# **11. 前端技術規格（ECME + TanStack）**

* 使用 ECME Admin Layout
* TanStack Table 作為主要表格
* Drawer from ECME（或自訂）
* 全部模擬 API → 未來可切換 axios
* 日期使用 Day.js
* 表格需支援 Column visibility + reorder

---

# **12. Mock Data 規格（前端可先做）**

Customer：

```
{
 id: 1001,
 customerName: "富達科技",
 customerNo: "CUST-001",
 segment: "VIP",
 lastActivityAt: "2025-11-20T10:30:00Z",
 nextFollowUpDate: "2025-11-25",
 nextFollowUpNote: "追蹤報價 QTN-001",
 owner: { id: 3, name: "王大明" },
 ...
}
```

ActivityLog：

```
{
 type: "CALL",
 content: "電話詢問交期",
 createdAt: "2025-11-21T09:00:00Z"
}
```

---
