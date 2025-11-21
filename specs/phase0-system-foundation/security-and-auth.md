# Flexora ERP — 安全與認證規格（Security & Authentication）
Phase 0 — System Foundation

本文件說明 Flexora ERP 在 Phase 0 的 **登入 / 認證 / 授權** 設計與規範，  
目前基於 JHipster 8.11.0 的 Spring Security + JWT 架構，並預留與 Owner / Team 資料模型整合的空間。

> 目標：  
> - 清楚描述目前實作的登入與授權機制  
> - 定義 User / Authority 的使用規則  
> - 為未來 DataScope / Owner-based Permission 打基礎

---

## 1. 安全架構總覽（High-Level Security Overview）

Flexora ERP 採用：

- **認證（Authentication）**：JWT（JSON Web Token）
- **授權（Authorization）**：Spring Security 基於 Authority / Role
- **使用者與權限資料**：由 JHipster 基礎 User / Authority 表結合自訂 Owner / Team 模型

### 1.1 流程簡述

1. 使用者由前端（`flexora-react-ui` 或 JHipster 內建前端）提交登入資訊。
2. 後端驗證帳號密碼無誤後：
   - 產生 JWT token，回傳給前端。
3. 前端將 JWT 存放於適當位置（通常是 memory / localStorage，以實作為準）。
4. 後續 API 呼叫一律於 HTTP Header 夾帶 `Authorization: Bearer <token>`。
5. 後端透過 JWT Filter 驗證 Token，有效則建立 SecurityContext（含 User / Authorities）。

---

## 2. 認證（Authentication）

### 2.1 登入 API

- 路徑：`POST /api/authenticate`（JHipster 預設）
- Request 格式（示意）：

```json
{
  "username": "admin",
  "password": "admin",
  "rememberMe": true
}
```

`rememberMe = true` 時後端會改用 `token-validity-in-seconds-for-remember-me` 的設定延長 JWT 壽命；`false` 則採一般有效時間。

* Response 格式（示意）：

```json
{
  "id_token": "<JWT_TOKEN>"
}
```

> ⚠ 實際欄位名稱依 JHipster 設定，通常為 `id_token`（Flexora 目前採用）或 `access_token`。

### 2.2 JWT Tokens

* 內容至少包含：

  * `sub`（login / user id）
  * `auth`（Authorities / Roles）
  * 過期時間（exp）
* 後端會使用簽名金鑰驗證 Token 真偽與有效期限。

### 2.3 Token 有效期限與刷新

* Phase 0 可先使用 JHipster 預設設定。
* 若未來支援 Refresh Token 或短/長效 Token，需更新本文件並建立 ADR。

### 2.4 前端串接（flexora-react-ui）

1. 正式 UI 的登入頁提供 `username`、`password` 與「記住我」（`rememberMe`）欄位。  
   - 勾選記住我 → JWT 寫入 `localStorage`，頁面重新整理仍保留。  
   - 未勾選 → JWT 僅存在 `sessionStorage`，關閉分頁即移除。
2. 成功取得 JWT 後，前端立即呼叫 `GET /api/account` 以 `AdminUserDTO` 同步 `authority`、姓名、Email 與頭像，作為 Route / Menu 權限的唯一資料來源。
3. axios request interceptor 會依序從 `localStorage`、`sessionStorage`、Cookie 取得 Token，並自動加上 `Authorization: Bearer <token>`。
4. UI 會啟動 Websocket Tracker（見 6.4）以回報活動訊息，方便後端監控同時在線的帳號。

---

## 3. 授權（Authorization）

### 3.1 Authority / Role

JHipster 預設使用：

* `jhi_user`
* `jhi_authority`
* `jhi_user_authority`

Flexora ERP 中，Authority 代表一組「角色」或「權限」，例如：

* `ROLE_ADMIN`
* `ROLE_USER`
* `ROLE_SALES_MANAGER`
* `ROLE_INVENTORY_MANAGER`

### 3.2 使用方式

在 Controller / Service 端，可透過以下方式限制權限：

```java
@PreAuthorize("hasAuthority('ROLE_ADMIN')")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    // ...
}
```

或：

```java
@PreAuthorize("hasAnyAuthority('ROLE_ADMIN', 'ROLE_SALES_MANAGER')")
public ResponseEntity<QuotationDTO> createQuotation(...) {
    // ...
}
```

### 3.3 規範

1. **權限名稱一律使用 `ROLE_` 前綴**。
2. Controller 方法應清楚標註所需權限（`@PreAuthorize` 或全域設定）。
3. 更細緻的行為控制（例如：只能操作自己 Owner 的資料）未來透過 DataScope / Owner 機制補上。

---

## 4. 使用者模型（User Model）

### 4.1 User 實體

基本欄位可參考 Phase 0 `domain-overview.md`，
此處補充與安全相關的部分：

| 欄位          | 說明               |
| ----------- | ---------------- |
| login       | 登入帳號（唯一，通常不分大小寫） |
| password    | 加密後的密碼（BCrypt 等） |
| activated   | 帳號是否啟用           |
| authorities | 多對多關聯 Authority  |
| langKey     | 語系設定             |

**規則：**

* 密碼一律以安全演算法（BCrypt）加密，禁止明文存放。
* `activated = false` 的帳號不得登入。
* 重設密碼流程可視需求在後續 Phase 補充。

---

## 5. Owner / Team 與安全的關係（Phase 0 規劃）

> ⚠ Phase 0 不會完整實作 DataScope / Owner-based Permission，
> 但資料模型需預先設計，以利未來擴充。

### 5.1 基本構想

* **Owner** 代表公司的資料範圍。
* User 通常隸屬於特定 Owner（或多個 Owner，視往後需求）。
* 未來若有 DataScope：

  * 判斷使用者是否有權讀寫某 Owner 的資料。

### 5.2 目前（Phase 0）策略

* 安全判斷以角色（Authority）為主。
* Owner / Team 僅作為 Domain 模型存在，尚未套用到所有 Query。

> 當資料權限（DataScope）實際導入時，本文件需要擴充一章節
> 說明：Owner-based Access Control 的具體實作方式。

---

## 6. 前端安全考量（flexora-react-ui）

### 6.1 Token 儲存位置

* 目前策略：

  * 預設存於 `sessionStorage`（瀏覽器分頁關閉即登出）。
  * 勾選 rememberMe 時改寫入 `localStorage`，同時在 `localStorage` 標記目前的儲存策略，後續登入沿用。
* 若未來有更嚴謹需求（避免 XSS），可考慮：

  * HttpOnly Cookie + CSRF 策略

### 6.2 路由保護

* 依據 User 的 Authorities 決定：

  * 哪些 Route 可顯示
  * 哪些功能按鈕（如刪除 / 審核）可以使用

範例（概念層級）：

```tsx
if (!user?.authorities.includes('ROLE_ADMIN')) {
  return <ForbiddenPage />;
}
```

### 6.3 不可相信前端

* 前端權限控制主要是 UX 層級的，避免無權限使用者看到不該看到的按鈕。
* 後端才是最終的安全閘，必須在所有關鍵操作端點驗證權限。

### 6.4 Websocket 活動追蹤

* `flexora-react-ui/src/services/WebsocketTrackerService.ts` 會在下列情境連線至 `/websocket/tracker`：
  1. 成功登入並取得 JWT。
  2. 頁面重新整理後偵測到 JWT 與登入狀態仍有效。
* Token 會附加在 query string（`?access_token=...`），後端即可識別使用者並將其加入 tracker。
* 登出或 JWT 無效時會自動斷線與清理資源，確保後端不保留幽靈連線。

---

## 7. REST API 安全規範

### 7.1 HTTP 方法與資源保護

* `GET /api/**`：需登入（一般使用者）
* `POST /api/**`：依據模組需求設定權限
* `PUT /api/**`：依據模組需求設定權限
* `DELETE /api/**`：通常限制較高權限（例如：`ROLE_ADMIN` 或專屬角色）

### 7.2 常見錯誤碼與行為

* 無 Token 或 Token 無效 → HTTP 401（`error.unauthorized`）
* Token 有效但權限不足 → HTTP 403（`error.accessDenied`）

對應前端行為：

* 401 → 導向登入頁 / 提示重新登入
* 403 → 顯示「沒有權限」訊息

---

## 8. 安全相關配置文件（Configuration）

在後端專案 `flexora-react` 中，安全相關設定通常位於：

* `SecurityConfiguration.java`
* `JwtSecurityConfigurer.java`（或類似名稱）
* `application-*.yml` 中 JWT/安全設定（如 secret key、token 有效時間）

**要求：**

* secret key 不得硬編在程式碼中（應由環境變數/設定檔管理）
* Production 與 Dev 的 JWT 設定應分開管理。

---

## 9. 日誌與審計（Logging & Audit）— 與安全的關聯

* 所有登入失敗 / 權限拒絕（403）事件應被記錄於 Log。
* 重要操作（如刪除、核准、作廢）：

  * 應透過審計欄位（`createdBy`, `lastModifiedBy`）或獨立 Log 實體紀錄。
* 若未來導入「操作紀錄 / Audit Trail」模組，需要補充專門 spec。

---

## 10. 未來擴充方向（Roadmap）

以下為安全相關的潛在擴充項目，Phase 0 不一定會實作：

* DataScope：

  * Owner / Department / Team 維度的資料存取控制
* Multi-tenancy：

  * 不同 Company/Client 的資料隔離
* OAuth2 / SSO 整合：

  * 與 AD、Google、Azure AD 等整合
* 2FA / MFA：

  * 多因子驗證機制

每一項一旦開始設計與實作，需：

1. 更新本 `security-and-auth.md`
2. 在 `decisions/DECISION_LOG.md` 增加對應紀錄
3. 視需要建立 ADR 檔案（例如：`ADR-00xx-security-strategy.md`）

---

## 11. 版本與歷程（History）

| 版本   | 日期         | 編輯者   | 說明                               |
| ---- | ---------- | ----- | -------------------------------- |
| v0.1 | 2025-11-19 | Jimmy | 建立 Phase 0 Security & Auth 規格初版。 |
