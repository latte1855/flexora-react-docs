# 資料權限管理 Domain 規格

**Spec-ID (MD5)**: `db778dff5da6e86ed6cd741a183fef22`  
**延伸閱讀**：[`docs/security/data-scope.md`](security/data-scope.md)（實作與用法指引）

## 變更摘要

- **統一以 Owner 控制主鍵與流水號**：`Owner.id` 為系統中 USER / DEPT / TEAM 的共同主鍵來源；其他實體以 **@MapsId** 依附 `Owner`，避免主鍵碰撞。
- **User 允許改結構**：`User` 不再自產主鍵，改為 `User -> @MapsId Owner`（`User.id = Owner.id`）。
- **部門階層定義明確**：

  - `parentHierarchy` 採 **包含自身（inclusive）** 的 materialized path，例：根部門( id=1 ) 為 `/1`，其子部門( id=3 ) 為 `/1/3`；尾碼 **包含該節點自己的 id**。
  - `depth` 採 **zero-based**：根節點深度為 `0`，其子節點為 `1`，以此類推。

- **成員關係採 B 方案（中介實體）**：`UserDepartment`、`UserTeam`，以便擴充屬性、稽核與唯一鍵控制。

---

## 實體與欄位

### 1) Owner（資料擁有者）

- `id` (PK, Long, **序號發號來源**)
- `ownerType` (enum: `USER`, `DEPT`, `TEAM`) **required**
- `name` (String, 255) **required**：顯示名稱（例如使用者姓名／部門名／團隊名）
- 推薦唯一鍵（視業務）：`(ownerType, name)`

> **一致性規則**：`Owner.ownerType` 必須與實際綁定的實體類型一致（例如被 `Department` 依附者其 `ownerType` 必須為 `DEPT`）。

### 2) Department（具階層）

- `id` (PK, Long) = **Owner.id**（`Department -> @MapsId Owner`）
- `deptName` (String, 255) **required**
- `parent` (self ManyToOne) 上層部門，可為 `null`（根節點）
- `parentHierarchy` (String, 1024) **包含自身** 的階層路徑，**以 `/` 分隔且以 `/` 起始**：

  - 根：`/1`
  - 子：`/1/3`
  - 子樹查詢以 `LIKE '/1/%'` 搭配等值比對 `= '/1'` 取得含自身與所有後代

- `depth` (Integer) **zero-based**：根=0，子=1，…
- `description` (String, 500)

> **索引建議**：對 `parentHierarchy` 建 B-Tree 索引（前綴查詢 `LIKE 'path/%'` 可走索引），另對 `(depth)`、`(deptName)` 依需求加索引。

### 3) Team（無階層）

- `id` (PK, Long) = **Owner.id**（`Team -> @MapsId Owner`）
- `teamName` (String, 255) **required**
- `description` (String, 500)

### 4) UserDepartment（中介：使用者－部門）

- `id` (PK, Long, 自增或序列)
- `user` (ManyToOne, required)
- `department` (ManyToOne, required)
- `roleInDept` (String, 30) 可空，例如 Manager/Member
- `assignedAt` (Instant)
- `expiredAt` (Instant)
- **唯一鍵**：`unique (user_id, department_id)`

> 可擴充：`isManager`、`note`、審計欄位等。

### 5) UserTeam（中介：使用者－團隊）

- `id` (PK, Long, 自增或序列)
- `user` (ManyToOne, required)
- `team` (ManyToOne, required)
- `roleInTeam` (String, 30)
- `assignedAt` (Instant)
- `expiredAt` (Instant)
- **唯一鍵**：`unique (user_id, team_id)`

---

## 共享主鍵（@MapsId）與建立順序

### 共識

- **唯一發號者**：`Owner.id`。
- **依附方向**：`User -> Owner`、`Department -> Owner`、`Team -> Owner`。
- **禁止** 在編輯 API 中更換依附對象（`user.owner`、`dept.owner`、`team.owner`），以免違反共享主鍵語意。

### 建立流程

1. 建立 `Owner(ownerType=USER, name=<userDisplayName>)` → 建立 `User`，`user.setOwner(owner)` → `User.id = owner.id`。
2. 建立 `Owner(ownerType=DEPT, name=<deptName>)` → 建立 `Department`，`dept.setOwner(owner)` → `Department.id = owner.id`。
3. 建立 `Owner(ownerType=TEAM, name=<teamName>)` → 建立 `Team`，`team.setOwner(owner)` → `Team.id = owner.id`。

> **刪除流程**：刪除依附者前，請先處理引用（例如 UserDepartment/UserTeam）以維持 FK 一致性。避免 Owner 直接被刪導致孤兒或違約束。

---

## 成員關係（B 方案）

- `UserDepartment` 與 `UserTeam` 以 **ManyToOne + 唯一鍵** 實作多對多。
- 能記錄角色、加入／到期時間、審計欄位，利於報表與權限策略變化。
- 查詢常用索引：

  - `user_department (user_id)`、`(department_id)`、`(user_id, department_id unique)`
  - `user_team (user_id)`、`(team_id)`、`(user_id, team_id unique)`

---

## 權限建議實作

- 各業務資料表新增 `owner` (ManyToOne, required) 指向 `Owner`。
- 判讀規則（舉例）：

  - **SELF**：`ownerType=USER && owner.id == currentUser.owner.id`
  - **DEPT**：`ownerType=DEPT && owner.id ∈ currentUser.deptOwnerIds`
  - **DEPT_AND_SUB**：同上再加入 `parentHierarchy` 以 prefix 匹配子樹
  - **TEAM**：`ownerType=TEAM && owner.id ∈ currentUser.teamOwnerIds`
  - **ALL/CUSTOM**：由角色策略模組另案定義

- 建議於登入時快取 `currentUser.deptOwnerIds`、`teamOwnerIds`，並以 `parentHierarchy` 做子樹展開。

---

## JDL（核心定義：Owner/Department/Team + 中介表）

> 注意：JHipster 生成器**不會自動修改**內建 `User` 的程式碼與 changelog。
> 下列 JDL 以 **意圖表達** `User -> @Id Owner`，產生後需**手動補丁** User 實體（見「User 手動調整」）。

```jdl

/**
 * 資料擁有者類型
 *
 * 值說明：
 * - USER：使用者
 * - DEPT：部門
 * - TEAM：團隊
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
enum OwnerType {
  /** 使用者 */
  USER("使用者"),
  /** 部門 */
  DEPT("部門"),
  /** 團隊 */
  TEAM("團隊")
}

/**
 * 資料擁有者（共同主鍵來源）
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
entity Owner {
  /** 類型（必填）：USER/DEPT/TEAM */
  ownerType OwnerType required,
  /** 顯示名稱（必填） */
  name String required maxlength(255)
}
dto Owner with mapstruct
service Owner with serviceClass
paginate Owner with pagination
filter Owner


/**
 * 部門（具階層）
 * parentHierarchy：包含自身（inclusive）的階層路徑，起始為 '/'，例如：
 *   根 id=1    => /1
 *   子 id=3    => /1/3
 * 子樹查詢可用： parent_hierarchy = '/1' OR parent_hierarchy LIKE '/1/%'
 * depth：zero-based（根=0，子=1）
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
entity Department {
  /** 部門名稱 */
  deptName String required maxlength(255),
  /** 包含自身的階層路徑，如：/1/3 */
  parentHierarchy String maxlength(1024),
  /** 深度（zero-based），根=0 */
  depth Integer,
  /** 說明 */
  description String maxlength(500)
}
dto Department with mapstruct
service Department with serviceClass
paginate Department with pagination
filter Department

/**
 * 團隊（無階層）
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
entity Team {
  /** 團隊名稱 */
  teamName String required maxlength(255),
  /** 說明 */
  description String maxlength(500)
}
dto Team with mapstruct
service Team with serviceClass
paginate Team with pagination
filter Team

/**
 * 中介：使用者－部門 關係
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
entity UserDepartment {
  /** 部門內角色（例：Manager/Member） */
  roleInDept String maxlength(30),
  /** 加入時間 */
  assignedAt Instant,
  /** 到期時間（可為空） */
  expiredAt Instant
}
dto UserDepartment with mapstruct
service UserDepartment with serviceClass
paginate UserDepartment with pagination
filter UserDepartment


/**
 * 中介：使用者－團隊 關係
 *
 * @author Chia-Ming Liu, <a href='mailto:latte1855@gmail.com'>latte1855@gmail.com</a>
 * @date 2025/09/16
 */
entity UserTeam {
  /** 團隊內角色（例：Manager/Member） */
  roleInTeam String maxlength(30),
  /** 加入時間 */
  assignedAt Instant,
  /** 到期時間（可為空） */
  expiredAt Instant
}
dto UserTeam with mapstruct
service UserTeam with serviceClass
paginate UserTeam with pagination
filter UserTeam

/** 共享主鍵：全部依附 Owner（@MapsId） */
relationship OneToOne {
  Department{owner(name)} to @Id Owner,
  Team{owner(name)}       to @Id Owner,

}

/** 部門階層（自關聯） */
relationship ManyToOne {
  Department{parent(deptName)} to Department
}

/** 中介關係（多對多 via 中介） */
relationship ManyToOne {
  UserDepartment{user(login)}          to User with builtInEntity,
  UserDepartment{department(deptName)} to Department,

  UserTeam{user(login)}                to User with builtInEntity,
  UserTeam{team(teamName)}             to Team
}

```

---

## User 手動調整（生成後必做）

> 目標：讓 `User.id = Owner.id`（`User -> @MapsId Owner`），移除 User 的主鍵自動生成。

1. **修改 `User.java`**

   - 將 `@Id @GeneratedValue(...) Long id;` 改為：

     ```java
     @Id
     private Long id;

     @OneToOne(optional = false, fetch = FetchType.LAZY)
     @MapsId
     @JoinColumn(name = "id")
     private Owner owner;

     ```

   - 其他程式碼中如有使用 `user.setId(...)` 則移除／改為以 `Owner` 建立流程取得 id。

2. **資料庫變更（Liquibase）**

   - `jhi_user` 的 `id` 欄位取消序列／自增預設值。
   - 新增 `FOREIGN KEY (id) REFERENCES owner(id)`；確保 `id` 仍為 `PRIMARY KEY`。
   - 若現有資料需轉換：

     - 先為每一筆 `jhi_user` 建立對應 `owner`（`ownerType=USER`, `name=<displayName>`），把 `owner.id` 設為原 `user.id`，再移除 `user` 的序列設定。
     - 或採 **新系統空表** 直接佈署（建議）。

3. **建立流程調整**

   - 建立使用者時：先建 `Owner(USER)` → 再建 `User` 並 `user.setOwner(owner)`。
   - 編輯頁／API **不可更換** `user.owner`。

---

## 驗證與查詢範式

- **部門子樹**（含自身）：

  ```
  WHERE d.parent_hierarchy = :path_self
     OR d.parent_hierarchy LIKE CONCAT(:path_self, '/%')
  ```

- **使用者可讀部門**：

  - 先由 `UserDepartment` 撈該使用者直屬部門 ownerIds；
  - 若角色策略為 `DEPT_AND_SUB`，展開為各部門 `parentHierarchy` 的前綴集合。

- **唯一鍵保護**：

  - `user_department(user_id, department_id)`、`user_team(user_id, team_id)`。

---

## 風險與注意

- 由於 `User` 需改為 `@MapsId`，**JHipster 生成器不會自動更改**此部分，請務必完成「User 手動調整」。
- 既有資料遷移需周延規劃；若先行無資料的新環境最簡單。
- 請於 Service 層嚴格校驗 `Owner.ownerType` 與依附實體的一致性。
