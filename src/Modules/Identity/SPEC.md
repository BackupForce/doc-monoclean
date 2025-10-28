# Identity Module Specification

## 1. 模組目的（Purpose）
提供 MonoClean 系統的身分識別與授權核心能力，包括使用者管理、登入註冊流程、Email 驗證、OAuth 整合與角色／權限派發。

## 2. Domain Model

### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 | 關聯 |
|------|------|------|------|
| User | Aggregate Root | 系統使用者，包含登入憑證與個人資料。 | User→Roles（多對多）、User→RefreshTokens（一對多） |
| Role | Entity | 權限群組，可分派至使用者或節點。 | Role→Permissions（多對多） |
| Permission | Entity | 系統操作權限，以 Node 層級綁定資源。 | Permission→Nodes（多對多） |
| Node | Entity | 節點層級結構，用於權限繼承與授權邊界。 | NodeRelation→Node（一對多） |
| NodeRelation | Value Object | 表示節點 ancestor/descendant 關係與深度。 | 隸屬於 Node |
| RefreshToken | Aggregate | 管理重新簽發存取權杖的安全性。 | 關聯 User |
| EmailVerification | Aggregate | 管理 Email 驗證碼與有效期限。 | 關聯 User |

### 2.2 Value Objects
- **HashedPassword**：封裝雜湊與驗證邏輯。
- **DisplayName**：確保顯示名稱長度與格式。
- **Email**：標準化與驗證 Email 字串。
- **PermissionCode**：符合 `identity.*` 命名規範。
- **StronglyTypedId**：所有識別碼皆採強型別實作。

## 3. Use Cases (CQRS)
| Use Case | Request | Response | 描述 |
|----------|---------|----------|------|
| RegisterUserCommand | Email, Password, DisplayName | Result<UserId> | 建立使用者並發布 `UserRegisteredDomainEvent`。 |
| VerifyEmailCommand | Token | Result | 驗證 Email 並啟用帳號。 |
| LoginCommand | Email, Password | Result<TokenPair> | 驗證憑證並產生 Access/Refresh Token。 |
| RefreshTokenCommand | RefreshToken | Result<AccessToken> | 重新簽發 Access Token。 |
| RequestEmailVerificationCommand | Email | Result | 重新寄送驗證信。 |
| GetPermissionsQuery | UserId | Result<Permission[]> | 彙整有效權限集合。 |
| GetCurrentUserQuery | ClaimsPrincipal | Result<UserProfile> | 取得登入者資訊。 |

## 4. API Endpoints
| Method | Path | Handler | 權限 |
|--------|------|---------|------|
| POST | `/auth/register` | RegisterUserCommand | Public |
| POST | `/auth/login` | LoginCommand | Public |
| POST | `/auth/refresh` | RefreshTokenCommand | Public |
| POST | `/auth/verify-email` | VerifyEmailCommand | Public |
| POST | `/auth/request-email` | RequestEmailVerificationCommand | Public |
| GET | `/auth/me` | GetCurrentUserQuery | Authorized |
| GET | `/permissions` | GetPermissionsQuery | Authorized |

## 5. Permissions
- `identity.users.create`
- `identity.users.read`
- `identity.users.update`
- `identity.users.delete`
- `identity.roles.manage`
- `identity.permissions.read`

權限繼承：基於 Node 層級，祖先節點權限會套用至子節點。

## 6. Domain Events
- `UserRegisteredDomainEvent`
- `EmailVerificationCreatedDomainEvent`
- `PasswordChangedDomainEvent`
- `UserLockedDomainEvent`

所有 Domain Event 皆透過 Outbox Pattern 發佈，Hangfire Job 會進行外部投遞。

## 7. 外部整合
| 整合對象 | 用途 |
|------------|------|
| SMTP | 寄送註冊與重發驗證信 |
| Google OAuth | 第三方登入 |
| Seq | 追蹤登入事件與安全審計 |

## 8. 測試策略
- 單元測試：使用 `xUnit` + `FluentAssertions` 驗證 CQRS Handler。
- 整合測試：Testcontainers 啟動 PostgreSQL 與 Redis，驗證資料存取與快取。
- 功能測試：模擬 API 流程（Register → Verify → Login → Refresh）。

## 9. TODO & Open Questions
- Passwordless / WebAuthn 登入支援？
- Node 權限繼承對高維度組織的效能評估。
- RefreshToken 多設備策略（單一 vs 多重）。
