# Identity Module Specification (consolidated)

## 1. Purpose
唯一擁有 `User` 與帳號生命週期（註冊、登入、OAuth、Email 驗證、Token）。**不擁有 Node/Permission**；權限查詢交由 AccessControl。

## 2. Domain Model
### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 |
|------|------|------|
| User | Aggregate Root | Email/Password/OAuth/Lockout/Claims 等 |
| RefreshToken | Aggregate Root | 長期存取更新憑證 |

> 不再定義 Node/Permission；如需角色概念，轉為 AC 的 `RoleGroup` 管理。

## 3. Use Cases (CQRS)
| Use Case | 描述 |
|----------|------|
| Register/Login/OAuthCallback | 標準身份驗證流程 |
| VerifyEmail / RequestEmailVerification | Email 驗證 |
| IssueTokens / RefreshTokens | Token 發行與刷新 |
| GetCurrentUser | 查目前使用者 |
| **GetMyPermissions** | 代理呼叫 AC 的 `GetEffectivePermissions(userId, nodeId)` |

## 4. API Endpoints
- `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/google/*` …
- `/me`：目前使用者資訊
- `/me/permissions?nodeId=...`：**呼叫 AC** 回傳有效權限集合

## 5. Integration
### Out (Events)
- `UserRegistered`
- `UserLocked`（可選）

### In (Queries)
- 權限查詢改為呼叫 AccessControl：不在本模組自行判斷

## 6. Testing Strategy
- 單元：Token/Email 驗證/Lockout
- 整合：OAuth Callback 與 Token 發行
- 功能：`/me/permissions` 透過 AC 正確回傳權限
