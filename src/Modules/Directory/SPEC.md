# Directory Module Specification (consolidated)

## 1. Purpose
提供租戶／Workspace／職位與成員關係的協作目錄；**不擁有權限判斷**，僅發布事件讓 AccessControl 建立/維護資源節點與授權。本系統以 Workspace 為協作命名空間的根；如需階層，可使用 WorkspaceRelation（closure table 或 materialized path），否則 Workspace 亦可採用扁平模式。

## 2. Domain Model
### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 | 關聯 |
|------|------|------|------|
| Tenant | Aggregate Root | 組織根節點 | Tenant→Workspaces（一對多） |
| Workspace | Aggregate Root | 協作空間節點（支援階層） | Workspace→Children（一對多）、Workspace→Members（多對多） |
| Membership | Entity | 使用者在 Workspace 的關係 | Membership→Positions（多對多）；僅持有 `UserId` |
| Position | Entity | 權限模板（轉交 AC 展開） | Position→Permissions（多對多；為模板定義） |
| Invitation | Aggregate | 邀請加入流程（具狀態機） | Invitation→Membership（一對一） |
| WorkspaceRelation | Value Object | ancestor/descendant/depth | 隸屬 Workspace |

### 2.2 Value Objects
- **WorkspaceCode**（唯一，snake_case）
- **WorkspaceName**（2–100）
- **InvitationToken**（具有效期/單次）
- **NodePath**（`/` 分隔）
- **WorkspaceStatus**：`Active` / `Archived`
- **InvitationStatus**：`Pending` / `Accepted` / `Expired` / `Cancelled`

## 3. Use Cases (CQRS)
| Use Case | Request | Response | 描述 |
|----------|---------|----------|------|
| CreateTenantCommand | Name, OwnerId | Result<TenantId> | 建立租戶與 root Workspace，發 `TenantCreated` |
| CreateWorkspaceCommand | TenantId, ParentWorkspaceId?, Name, Code? | Result<WorkspaceId> | 建立 Workspace，發 `WorkspaceCreated` |
| UpdateWorkspaceCommand | WorkspaceId, Name, Status | Result | 發 `WorkspaceRenamed` 或狀態改變事件 |
| MoveWorkspaceCommand | WorkspaceId, NewParentId | Result | 重建 Relation，發 `WorkspaceMoved` |
| InviteMemberCommand | WorkspaceId, Email, PositionIds | Result<InvitationId> | 建立邀請，發 `MemberInvited` |
| AcceptInvitationCommand | Token, CurrentUserId | Result<MembershipId> | 成為成員，發 `MemberJoinedWorkspace` |
| RemoveMemberCommand | MembershipId | Result | 移除成員，發 `RemovedFromWorkspace` |
| AssignPositionCommand | MembershipId, PositionId | Result | 指派職位模板，發 `PositionAssigned` |
| GetWorkspaceTreeQuery | TenantId | Result<WorkspaceTreeDto> | 查 Workspace 樹狀結構 |
| GetWorkspaceMembersQuery | WorkspaceId | Result<List<MemberDto>> | 查 Workspace 成員 |
| GetPendingInvitationsQuery | TenantId | Result<List<InvitationDto>> | 查邀請清單 |

## 4. API Endpoints
| Method | Path | Handler | 權限 |
|--------|------|---------|------|
| POST | `/directory/tenants` | CreateTenantCommand | `directory.tenants.manage` |
| POST | `/directory/workspaces` | CreateWorkspaceCommand | `directory.workspaces.manage` |
| PATCH | `/directory/workspaces/{id}` | UpdateWorkspaceCommand | `directory.workspaces.manage` |
| PATCH | `/directory/workspaces/{id}/move` | MoveWorkspaceCommand | `directory.workspaces.manage` |
| POST | `/directory/invitations` | InviteMemberCommand | `directory.members.invite` |
| POST | `/directory/invitations/accept` | AcceptInvitationCommand | Public |
| DELETE | `/directory/members/{id}` | RemoveMemberCommand | `directory.members.manage` |
| POST | `/directory/members/{id}/positions` | AssignPositionCommand | `directory.members.manage` |
| GET | `/directory/tenants/{id}/tree` | GetWorkspaceTreeQuery | `directory.workspaces.read` |
| GET | `/directory/workspaces/{id}/members` | GetWorkspaceMembersQuery | `directory.members.read` |
| GET | `/directory/invitations` | GetPendingInvitationsQuery | `directory.invitations.read` |

## 5. Permissions（由 AC 註冊與判斷）
- `directory.tenants.manage`
- `directory.workspaces.manage`
- `directory.workspaces.read`
- `directory.members.invite`
- `directory.members.manage`
- `directory.members.read`
- `directory.positions.manage`
- `directory.invitations.read`

## 6. Domain Events（Outbox）
- `TenantCreated`
- `WorkspaceCreated`
- `WorkspaceRenamed`
- `WorkspaceMoved`
- `MemberInvited`
- `MemberJoinedWorkspace`
- `RemovedFromWorkspace`
- `PositionAssigned`
- `InvitationExpired`（排程或 lazy 驗證）

## 7. Integration（Emit ➜ AccessControl）
- 事件觸發 AC 建立/更新 ResourceNode 與賦權
- Position 僅為**模板**；實際賦權在 AC 展開

## 8. Testing Strategy
- 單元：WorkspaceRelation 建構/搬移；Invitation 狀態機
- 整合：Testcontainers 驗證 Closure Table/MP；事件外送冪等
- API：Invite→Accept→AssignPosition 的全流程
