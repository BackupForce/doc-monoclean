# Directory Module Specification

## 1. 模組目的（Purpose）
提供租戶／組織的階層式目錄管理，涵蓋租戶、部門、職位與成員關係，並支援節點式授權模型。

## 2. Domain Model

### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 | 關聯 |
|------|------|------|------|
| Tenant | Aggregate Root | 組織根節點，維護租戶層級的設定。 | Tenant→Departments（一對多） |
| Department | Aggregate Root | 部門節點，支援上下層結構與節點權限。 | Department→Children（一對多）、Department→Members（多對多） |
| Membership | Entity | 使用者在指定部門的成員關係與角色。 | Membership→Positions（多對多） |
| Position | Entity | 定義職位與權限模板，可套用於成員。 | Position→Permissions（多對多） |
| Invitation | Aggregate | 邀請新成員加入租戶或部門的流程。 | Invitation→Membership（一對一） |
| DepartmentRelation | Value Object | 儲存 ancestor / descendant / depth。 | 隸屬於 Department |

### 2.2 Value Objects
- **DepartmentCode**：唯一識別部門的代碼（`snake_case`）。
- **DepartmentName**：部門名稱長度 2-100 字元。
- **InvitationToken**：具有效期的邀請代碼。
- **NodePath**：階層路徑（以 `/` 分隔）。

## 3. Use Cases (CQRS)
| Use Case | Request | Response | 描述 |
|----------|---------|----------|------|
| CreateTenantCommand | Name, OwnerId | Result<TenantId> | 建立租戶並建立 root 部門。 |
| CreateDepartmentCommand | TenantId, ParentDepartmentId, Name | Result<DepartmentId> | 在指定節點下建立部門。 |
| UpdateDepartmentCommand | DepartmentId, Name, Status | Result | 修改部門屬性。 |
| InviteMemberCommand | DepartmentId, Email, RoleIds | Result<InvitationId> | 建立邀請並寄送 Email。 |
| AcceptInvitationCommand | Token | Result<MembershipId> | 受邀者接受並成為成員。 |
| AssignPositionCommand | MembershipId, PositionId | Result | 指派職位與權限模板。 |
| GetOrganizationTreeQuery | TenantId | Result<OrganizationTreeDto> | 取得樹狀結構。 |

## 4. API Endpoints（預期）
| Method | Path | Handler | 權限 |
|--------|------|---------|------|
| POST | `/directory/tenants` | CreateTenantCommand | `directory.tenants.manage` |
| POST | `/directory/departments` | CreateDepartmentCommand | `directory.departments.manage` |
| PATCH | `/directory/departments/{id}` | UpdateDepartmentCommand | `directory.departments.manage` |
| POST | `/directory/invitations` | InviteMemberCommand | `directory.members.invite` |
| POST | `/directory/invitations/accept` | AcceptInvitationCommand | Public |
| POST | `/directory/members/{id}/positions` | AssignPositionCommand | `directory.members.manage` |
| GET | `/directory/tenants/{id}/tree` | GetOrganizationTreeQuery | `directory.departments.read` |

## 5. Permissions
- `directory.tenants.manage`
- `directory.departments.manage`
- `directory.departments.read`
- `directory.members.invite`
- `directory.members.manage`
- `directory.positions.manage`

## 6. Domain Events
- `TenantCreatedDomainEvent`
- `DepartmentCreatedDomainEvent`
- `DepartmentStatusChangedDomainEvent`
- `MemberInvitedDomainEvent`
- `MemberJoinedDepartmentDomainEvent`

## 7. 外部整合
| 整合對象 | 用途 |
|------------|------|
| Identity 模組 | 驗證使用者存在與基礎資料。
| AccessControl 模組 | 同步 Node 與權限指派。
| SMTP | 寄送部門邀請信。

## 8. 測試策略
- 組合測試：驗證部門階層新增／刪除時的 Node 繼承。
- 整合測試：Testcontainers 驗證 PostgreSQL 中的層級結構儲存。
- API 測試：保證邀請流程與權限檢查。

## 9. TODO & Open Questions
- 是否支援跨租戶共享部門？
- 當租戶規模極大時，Node 繼承查詢的效能調校。
- 與 Subscription 模組整合以限制可建立的部門數。
