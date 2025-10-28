# 🧩 MonoClean Specification Template
> Version: 2025-10  
> Target: Codex Auto-Maintenance  
> Project: MonoClean (Modular Monolith, Clean Architecture)

---

## 1. 系統整體規格（Global Specification）

### 1.1 系統概述（System Overview）

- **專案名稱**：MonoClean  
- **架構模式**：Modular Monolith（Clean Architecture + DDD + CQRS）  
- **技術棧**：
  - Backend：.NET 8 / EF Core / Dapper / MediatR / PostgreSQL / Redis / Hangfire / Seq
  - Frontend：React + TypeScript + Ant Design Pro（i18n + Axios）
  - Container：Docker Compose（Postgres, Redis, Seq, WebApi）
- **設計原則**：
  - 每個模組皆獨立 Domain / Application / Infrastructure / Presentation
  - 採用 CQRS + Domain Events + Outbox Pattern
  - 所有 API 回傳格式統一採 `Result<T>` + `Error`
  - 採用強型別 ID（StronglyTypedId）與 Value Objects
  - 全域遵循 PostgreSQL 命名規範（`snake_case`、小寫）

---

### 1.2 模組清單（Modules Overview）

| 模組名稱 | 功能說明 | 狀態 |
|-----------|-----------|------|
| Identity | 使用者、登入、權限、節點架構 | ✅ 已完成 |
| Directory | 組織 / 部門 / 成員結構管理 | 🟡 開發中 |
| AccessControl | Permission and Resource Control (Node-based) | 🔲 Not yet developed (Planning) |
| Finance | 應收應付、月帳、群組結算 | 🟡 設計中 |
| Subscription | 訂閱方案與方案權限控制 | 🟡 設計中 |
| Audit | 系統審計與操作紀錄 | ⏳ 計畫中 |

---

## 2. 模組規格（Modules Specification）

每個模組皆應有 `/src/Modules/{ModuleName}/SPEC.md`，  
遵循以下模板格式（Codex 自動維護）。

---

### 🧩 Module: Identity

#### 2.1 模組目的（Purpose）
負責使用者身分、登入註冊、Email 驗證、OAuth 整合、角色與權限授權。

#### 2.2 Domain Entities
| 實體 | 說明 | 關聯 |
|------|------|------|
| User | 系統使用者 | User→Roles（多對多） |
| Role | 角色群組 | Role→Permissions（多對多） |
| Permission | 操作權限 | 綁定 Node 層級 |
| Node / NodeRelation | 層級節點 | ancestor / descendant / depth |
| RefreshToken | 聚合根 | User 一對多 |
| EmailVerification | 驗證信聚合 | User 一對多 |

#### 2.3 Use Cases
| 用例 | 輸入 | 輸出 | 描述 |
|------|------|------|------|
| RegisterUser | Email, Password, DisplayName | UserId | 建立使用者並寄送驗證信 |
| VerifyEmail | Token | Result | 驗證 Email 並啟用帳號 |
| Login | Email, Password | Access/Refresh Token | 登入並產生 Token |
| RefreshToken | Token | Access Token | 重新簽發 Access Token |
| RequestEmailVerification | Email | void | 重新寄送驗證信 |
| GetPermissions | UserId | Permission[] | 查詢權限集合 |

#### 2.4 API
| Method | Path | 功能 | 權限 |
|---------|------|------|------|
| POST | `/auth/register` | 註冊 | Public |
| POST | `/auth/login` | 登入 | Public |
| POST | `/auth/refresh` | 更新 Token | Public |
| POST | `/auth/verify-email` | 驗證信 | Public |
| GET | `/auth/me` | 取得登入者資料 | Authorized |
| GET | `/permissions` | 查詢權限 | Authorized |

#### 2.5 權限模型
- 範例權限：
  - `identity.users.create`
  - `identity.users.read`
  - `identity.roles.manage`
- 權限來源：Role → Permission → Node
- 權限繼承：支援 Node 層級繼承（祖先節點權限傳遞給子節點）

#### 2.6 外部整合
| 整合對象 | 說明 |
|------------|------|
| SMTP | Email 驗證信 |
| Google OAuth | 外部登入 |
| Hangfire | 定期清理過期 RefreshToken |

---

### 🧩 Module: Directory

#### 2.1 模組目的
維護公司 / 組織 / 部門 / 成員結構，支援階層與指派角色。

#### 2.2 預期 Entities
`Company`, `Department`, `Membership`, `Invitation`, `Position`

#### 2.3 Use Cases
- 建立公司 / 部門  
- 邀請新成員  
- 指派角色或節點  
- 啟用 / 停用部門  
- 取得樹狀層級結構  

#### 2.4 API（預期）
- `POST /directory/companies`
- `GET /directory/companies/{id}/members`
- `POST /directory/invitations`

#### 2.5 權限模型
- `directory.companies.manage`
- `directory.members.invite`
- `directory.departments.read`

---

### 🧩 Module: AccessControl（開發中）

> 🟡 狀態：開發中。
> 目標是成為全系統授權核心，負責 Node-based 授權、角色群組與資源權限分配。

#### 2.1 模組目的
集中管理授權邏輯與資源權限，提供統一的 Policy Provider 供其他模組呼叫。

#### 2.2 預期 Domain Entities
| 實體 | 說明 |
|------|------|
| AccessPolicy | 定義資源與動作（如 Create / Read / Update / Delete） |
| Resource | 被保護資源（Node、Endpoint、功能代號） |
| RoleGroup | 跨模組角色群組 |
| PermissionAssignment | 使用者 / 角色 與 資源 / 權限 的關聯 |

#### 2.3 預期 Use Cases
| 用例 | 描述 |
|------|------|
| GrantPermission | 指派權限給角色或使用者 |
| RevokePermission | 撤銷指定權限 |
| CheckAccess | 驗證使用者能否操作資源 |
| GetEffectivePermissions | 彙整使用者有效權限集合 |

#### 2.4 預期 API
| Method | Path | 功能 |
|---------|------|------|
| GET | `/access-control/effective-permissions` | 查詢有效權限 |
| POST | `/access-control/assign` | 指派權限 |
| DELETE | `/access-control/revoke` | 移除權限 |

#### 2.5 設計方向
- 採 **Node-based 層級授權**
- 權限繼承邏輯：
  - 子節點繼承祖先節點權限
- 權限合併策略：
  - User → Roles → Node → Resource
- 未來整合 Subscription 模組（根據方案控制權限）

---

### 🧩 Module: Finance
> 狀態：設計中（參照 Finhub 會計系統結構）

- 支援應收應付、月帳統計、群組結算  
- 主要實體：`CashTransaction`, `Reconciliation`, `Receivable`, `Payable`, `GroupStatement`  
- 結算策略：
  - 結算時鎖定 BaseAmount（確保歷史一致性）
- 將整合 Node-based 權限以決定可見範圍（上線/下線）

---

### 🧩 Module: Subscription
> 狀態：設計中（計劃與 AccessControl 整合）

- 功能：控制方案權限、試用期、升級與降級
- 實體：`Plan`, `Subscription`, `FeatureLimit`
- 授權策略：方案可限制 AccessControl 層的權限集合

---

### 🧩 Module: Audit
> 狀態：規劃中  
- 功能：記錄系統行為、登入事件、操作歷程
- 預計提供：
  - API Request Log
  - Domain Event Log
  - Permission Check Audit

---

## 3. 系統橫切規格（Cross-cutting Specification）

### 3.1 Domain Events
- 格式：`{Aggregate}.{EventName}DomainEvent`
- 範例：`UserRegisteredDomainEvent`, `EmailVerificationCreatedDomainEvent`
- 流程：DomainEvent → Outbox → Hangfire Job → 外部投遞

### 3.2 Pipeline Behaviors
- `ValidationPipelineBehavior`
- `RequestLoggingPipelineBehavior`
- `TransactionalPipelineBehavior`

### 3.3 Result & Error Model
- 成功：`Result.Success(value)`
- 失敗：`Result.Failure(Error.Validation(...))`
- 錯誤分類：
  - Validation / Failure / Unauthorized / Conflict / NotFound

### 3.4 Infrastructure
- EF Core（`ApplicationDbContext`、snake_case 命名）
- Redis（快取 / 分散式鎖）
- Hangfire（Outbox、排程）
- Serilog + Seq（結構化日誌）

### 3.5 Testing
- 使用 `xUnit` + `FluentAssertions`
- Testcontainers（Postgres / Redis）
- FunctionalTests 驗證端點行為
- 測試命名慣例：`{Feature}Tests.cs`

### 3.6 CI/CD
- GitLab CI Pipeline：
  - Build → Test → Publish → Deploy
- Docker Image 命名：
  - `local/web-api:{date}-{hash}`

---

## 4. 規格維護原則（Maintenance Policy）

- 每個模組必須有 `SPEC.md`
- Codex 根據以下事件自動更新：
  - 新 CQRS Handler → 更新 Use Case 表
  - 新 Endpoint → 更新 API 表
  - 新 Permission → 更新權限表
- 全域索引維護於 `/docs/SYSTEM_SPEC.md`
- 未開發模組（如 AccessControl）保持佔位符，不生成具體文件直到開發開始。

---

> 📘 Maintained by Codex  
> 更新策略：PR 觸發檢查 → 對應模組 SPEC → 同步 SYSTEM_SPEC 索引
