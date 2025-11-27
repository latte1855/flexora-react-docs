# 🧩 **STEP 3 — Codex Customer Module 開發 Checklist（終端可複製版本）**

> **目標：在不依賴後端 API 完成的情況下，先讓 UI 全面跑得起來，之後再逐段串接 Spring Boot API。**

---

# ✅ **A. 前端建立 Customer 模組結構（React + ECME + TanStack Table）**

### **A.1 建立目錄結構**

在 `/src/app/modules/crm/customers/` 建立：

```
customers/
 ├─ index.tsx
 ├─ CustomerListPage.tsx
 ├─ CustomerTable.tsx
 ├─ CustomerSidebar.tsx
 ├─ CustomerFilterBar.tsx
 ├─ CustomerDrawer.tsx
 ├─ CustomerDetailPage/
 │    ├─ index.tsx
 │    ├─ CustomerDetailHeader.tsx
 │    ├─ CustomerDetailTabs.tsx
 │    ├─ tabs/
 │         ├─ TabGeneral.tsx
 │         ├─ TabContacts.tsx
 │         ├─ TabActivity.tsx
 │         ├─ TabQuotations.tsx
 │         ├─ TabOpportunities.tsx
 │         ├─ TabOrders.tsx
 │         ├─ TabAttachments.tsx
 ├─ mock/
 │    ├─ mock-customers.ts
 │    ├─ mock-activity.ts
 ├─ api/
      ├─ customer.api.ts
      ├─ activity.api.ts
```

---

# ✅ **B. Mock API（無後端時 UI 先能操作）**

### B.1 `mock-customers.ts`

建立至少 20 筆假資料：

* id
* customerName
* customerNo
* segment (VIP/A/B/C)
* lastActivityAt
* nextFollowUpDate
* nextFollowUpNote
* phone
* email
* industry
* owner
* healthStatus（🟢 🟡 🔴）→ 根據 lastActivityAt 自動算

### B.2 `mock-activity.ts`

每個 customer 要有 3～10 筆 Activity：

* type
* content
* createdAt
* createdBy

### B.3 Mock delay

使用：

```ts
await new Promise(r => setTimeout(r, 400))
```

---

# ✅ **C. Sidebar（CustomerSidebar.tsx）**

### 清單邏輯：

按鈕點擊 → 觸發 callback → 更新 `preset` 狀態

Side presets：

```
MY_OWNED
RECENT_CREATED
RECENT_ACTIVITY
NEED_FOLLOWUP
ALL
VIP
DORMANT
HIGH_POTENTIAL
```

### UI：

* 使用 `<Menu>`（ECME）
* 選取項目有 active 狀態
* 支援 badge（顯示數量）

---

# ✅ **D. Filter Bar（CustomerFilterBar.tsx）**

包含：

* 搜尋框（debounce 300ms）
* 下拉選單：

  * Owner
  * Segment
  * Health status
  * Follow-up status
  * Industry

更新語法：

```ts
setFilters({
  ...filters,
  keyword: value,
})
```

---

# ✅ **E. TanStack Table（CustomerTable.tsx）**

### E.1 欄位

* Select checkbox
* Name + Code
* Segment badge (VIP/A/B/C)
* Owner
* Health status
* Next Follow-up
* Phone
* Email
* Industry
* Actions（View / More）

### E.2 功能

* Sorting
* Filtering
* Column visibility
* Column reorder
* Row selection
* Infinite scroll / pagination（先用 pagination）

---

# ✅ **F. Drawer（CustomerDrawer.tsx）**

觸發：點擊表格列時開啟：

### 區塊：

#### 1) Header

* 名稱
* Segment
* Owner

#### 2) 基本資料

* 代號
* 電話
* Email
* Industry
* Website

#### 3) Follow-up

* DatePicker
* TextArea
* Save / Clear 按鈕
  → 操作更新 local state（無後端時）
* 串接 API：`GET /api/customers/{id}/follow-up` 讀取、`PUT /api/customers/{id}/follow-up` 寫入（僅 nextFollowUpDate / nextFollowUpNote）

#### 4) Recent Activity

* 列出前 5 筆
* 「查看更多活動」

#### 5) Quick Actions

按鈕：

* 建立報價
* 建立商機
* 建立訂單

→ 先以 toast 顯示 “尚未串接 API”。

---

# ✅ **G. Customer Detail Page**

### G.1 `CustomerDetailHeader.tsx`

包含：

* 返回列表
* 客戶名稱
* Segment badge
* Owner
* Health status
* 整合 Follow-up 區塊（可編輯）

---

### G.2 Tabs 內容（/tabs/*.tsx）

以下全部先用 mock 資料：

---

## TabGeneral

表單欄位：

* customerNo
* customerName
* segment
* owner
* industry
* employees
* annualRevenue
* phone
* email
* website
* fax
* description

---

## TabContacts

欄位：

* contactName
* title
* phone
* email
* isPrimary

支援：

* 新增
* 編輯
* 刪除

先全部用 local state。

---

## TabActivity

使用 timeline 或 list 風格：

* type icon
* createdAt
* createdBy
* content

新增活動的 form：

* type
* content
* createdAt（自動）

---

## TabQuotations / TabOpportunities / TabOrders

列表欄位：

* Document No
* Created At
* Amount
* Status

先 mock。

---

## TabAttachments

支援：

* 上傳
* 預覽
* 刪除（local 模式）

不需要後端即可展示 UI。

---

# ✅ **H. Customer API（customer.api.ts）**

建立 placeholder 方法：

```
getCustomers(filters, preset)
getCustomerSummary(id)
updateFollowUp(id, data)
getActivities(customerId)
addActivity(customerId, payload)
```

初期全部呼叫 mock 資料。

未來串後端時：

* 替換 baseURL
* 把 mock 改成 axios

---

# ✅ **I. 後端任務（Spring Boot + JPA）**

以下給 Codex 作為「後端實作時再處理」：

---

## I.1 Customer Entity 增加欄位

```java
Instant lastActivityAt;
LocalDate nextFollowUpDate;
String nextFollowUpNote;
String segment;
String healthStatus; // 可選：計算欄位
```

---

## I.2 新增 ActivityLog Entity

```java
@Entity
class ActivityLog {
    @Id Long id;
    @ManyToOne Customer customer;
    String type;
    String content;
    Instant createdAt;
    String createdBy;
}
```

---

## I.3 Repository Query（必要）

### 最近新增：

```sql
created_at >= now() - interval '30 days'
```

### 冷凍客戶（Dormant）：

```
last_activity_at < now() - interval '90 days'
```

### 最近互動：

```
last_activity_at >= now() - interval '30 days'
```

### 需要跟進：

```
next_follow_up_date <= current_date + 7 days
```

---

# 🧪 **J. 測試任務（Codex 可產）**

前端：

* Render sidebar lists
* Filter keyword
* Filter by segment
* Drawer opens
* Drawer updates follow-up
* Activity adds correctly
* Tab navigation works

後端（之後再加）：

* 篩選條件 QueryDSL 單元測試
* CustomerRepository integration test
* ActivityLogRepository integration test

---

# 📦 **K. 可交付成果（最終）**

Codex 開完後前端會有：

* 完整 Customer Workspace
* 左側清單可互動
* 可複雜過濾
* 可分頁
* 可 Drawer 編輯
* 可 Detail Page
* 使用 mock API 開發
* 後端準備好時可直接接線

完全符合你的 ERP 模組標準。

---
