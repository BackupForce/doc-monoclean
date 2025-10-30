# Directory Module Specification (consolidated)

## 1. Purpose
提供租戶／部門／職位與成員關係的階層式目錄；**不擁有權限判斷**，僅發布事件讓 AccessControl 建立/維護資源節點與授權。

## 2. Domain Model
### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 | 關聯 |
|------|------|------|------|
| Tenant | Aggregate Root | 組織根節點 | Tenant→Departments（一對多） |
| Department | Aggregate Root | 部門節點（支援階層） | Department→Children（一對多）、Department→Members（多對多） |
| Membership | Entity | 使用者在部門的關係 | Membership→Positions（多對多）；僅持有 `UserId` |
| Position | Entity | 權限模板（轉交 AC 展開） | Position→Permissions（多對多；為模板定義） |
| Invitation | Aggregate | 邀請加入流程（具狀態機） | Invitation→Membership（一對一） |
| DepartmentRelation | Value Object | ancestor/descendant/depth | 隸屬 Department |

### 2.2 Value Objects
- **DepartmentCode**（唯一，snake_case）
- **DepartmentName**（2–100）
- **InvitationToken**（具有效期/單次）
- **NodePath**（`/` 分隔）
- **DepartmentStatus**：`Active` / `Archived`
- **InvitationStatus**：`Pending` / `Accepted` / `Expired` / `Cancelled`

## 3. Use Cases (CQRS)
| Use Case | Request | Response | 描述 |
|----------|---------|----------|------|
| CreateTenantCommand | Name, OwnerId | Result<TenantId> | 建立租戶與 root 部門，發 `TenantCreated` |
| CreateDepartmentCommand | TenantId, ParentDepartmentId, Name, Code? | Result<DepartmentId> | 建立部門，發 `DepartmentCreated` |
| UpdateDepartmentCommand | DepartmentId, Name, Status | Result | 發 `DepartmentRenamed` 或狀態改變事件 |
| MoveDepartmentCommand | DepartmentId, NewParentId | Result | 重建 Relation，發 `DepartmentMoved` |
| InviteMemberCommand | DepartmentId, Email, PositionIds | Result<InvitationId> | 建立邀請，發 `MemberInvited` |
| AcceptInvitationCommand | Token, CurrentUserId | Result<MembershipId> | 成為成員，發 `MemberJoinedDepartment` |
| RemoveMemberCommand | MembershipId | Result | 移除成員，發 `MemberRemoved` |
| AssignPositionCommand | MembershipId, PositionId | Result | 指派職位模板，發 `PositionAssigned` |
| GetOrganizationTreeQuery | TenantId | Result<OrganizationTreeDto> | 查樹狀結構 |
| GetDepartmentMembersQuery | DepartmentId | Result<List<MemberDto>> | 查部門成員 |
| GetPendingInvitationsQuery | TenantId | Result<List<InvitationDto>> | 查邀請清單 |

## 4. API Endpoints
| Method | Path | Handler | 權限 |
|--------|------|---------|------|
| POST | `/directory/tenants` | CreateTenantCommand | `directory.tenants.manage` |
| POST | `/directory/departments` | CreateDepartmentCommand | `directory.departments.manage` |
| PATCH | `/directory/departments/{id}` | UpdateDepartmentCommand | `directory.departments.manage` |
| PATCH | `/directory/departments/{id}/move` | MoveDepartmentCommand | `directory.departments.manage` |
| POST | `/directory/invitations` | InviteMemberCommand | `directory.members.invite` |
| POST | `/directory/invitations/accept` | AcceptInvitationCommand | Public |
| DELETE | `/directory/members/{id}` | RemoveMemberCommand | `directory.members.manage` |
| POST | `/directory/members/{id}/positions` | AssignPositionCommand | `directory.members.manage` |
| GET | `/directory/tenants/{id}/tree` | GetOrganizationTreeQuery | `directory.departments.read` |
| GET | `/directory/departments/{id}/members` | GetDepartmentMembersQuery | `directory.members.read` |
| GET | `/directory/invitations` | GetPendingInvitationsQuery | `directory.invitations.read` |

## 5. Permissions（由 AC 註冊與判斷）
- `directory.tenants.manage`
- `directory.departments.manage`
- `directory.departments.read`
- `directory.members.invite`
- `directory.members.manage`
- `directory.members.read`
- `directory.positions.manage`
- `directory.invitations.read`

## 6. Domain Events（Outbox）
- `TenantCreated`
- `DepartmentCreated`
- `DepartmentRenamed`
- `DepartmentMoved`
- `MemberInvited`
- `MemberJoinedDepartment`
- `MemberRemoved`
- `PositionAssigned`
- `InvitationExpired`（排程或 lazy 驗證）

## 7. Integration（Emit ➜ AccessControl）
- 事件觸發 AC 建立/更新 ResourceNode 與賦權
- Position 僅為**模板**；實際賦權在 AC 展開

## 8. Testing Strategy
- 單元：DepartmentRelation 建構/搬移；Invitation 狀態機
- 整合：Testcontainers 驗證 Closure Table/MP；事件外送冪等
- API：Invite→Accept→AssignPosition 的全流程
