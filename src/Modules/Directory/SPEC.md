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

## Invitations

### Goal
以 **Invitation 為單一真相** 管理「邀請加入 Workspace」流程。**Membership 一律在 `Accepted` 之後才建立**（外部與內部一致），可見度靠 Invitation 的投影與通知達成。

### Terms
- **External flow**：受邀者透過 Email 連結接受。
- **Internal flow**：受邀者為系統使用者，透過站內 Notice 接受。
- **Outbox**：所有跨聚合/跨模組副作用（Email、Notice、Membership 建立）皆以 Outbox 事件處理。

### State Machine
`Pending → Accepted | Declined | Expired | Revoked`
- 只有 `Pending` 可轉移；其他為終態。
- `Expired`: 由排程批次將逾期 `Pending` → `Expired`。
- `Revoked`: 管理者撤銷。

### Invariants
- 同一 `(workspace_id, invitee_user_id)` 或 `(workspace_id, email)` **僅允許 1 張 Pending**。
- `Accepted` **冪等**：重複接受不會重複建 Membership。
- **Membership 只由 `InvitationAccepted` 事件處理器建立**。

### Commands (CQRS)
- `InviteMemberCommand(workspace_id, email, position_ids)`  
  - 內部：`email → invitee_user_id`（必須已註冊）。  
  - 外部：只存 `email` + `token_hash`（展示用 token 僅出現在連結中）。  
  - 去重檢查：  
    - 內部：`ExistsPendingForInviteeAsync(workspace_id, invitee_user_id)`  
    - 外部：`ExistsPendingForEmailAsync(workspace_id, email)`  
  - 成功 → 建立 `Invitation(Pending)` → 發 Outbox：
    - 內部：`MemberInvitedInternal`
    - 外部：`MemberInvitedExternal`

- `AcceptInternalInvitationCommand(invitation_id)`（需登入且為 invitee）  
  - 驗證：`invitation.Pending && user == invitee_user_id` → `Invitation.Accept(now)` → Outbox `InvitationAccepted`.

- `AcceptExternalInvitationCommand(invitation_id, token)`（匿名可呼叫）  
  - 驗證：`token` 與 `token_hash` **常數時間比對**、`invitation.Pending`、未逾期 → `Accept(now)` → Outbox `InvitationAccepted`.

- 其他（必要時再補）：`DeclineInvitationCommand`、`RevokeInvitationCommand`、`ResendInvitationCommand`.

### Domain Events (Outbox payload 最小化)
- `MemberInvitedInternal(invitation_id, tenant_id, workspace_id, invitee_user_id, position_ids, expires_at_utc)`
- `MemberInvitedExternal(invitation_id, tenant_id, workspace_id, email, position_ids, expires_at_utc, locale?)`
- `InvitationAccepted(invitation_id, tenant_id, workspace_id, invitee_user_id or email, position_ids)`
- `InvitationDeclined/Expired/Revoked(...)`

> 事件只帶必要識別資訊；模板/連結/i18n 由通知層決定。

### Event Handlers (Outbox Consumers)
- **EmailSender**（訂閱 `MemberInvitedExternal`）  
  - 由 **Notifications/Communications 模組或 Infrastructure** 負責。  
  - 依 tenant/locale 選模板，`ILinkFactory.BuildInvitationAcceptLink(invitation_id, token)` 產連結。  
  - 以 `invitation_id` 作 idempotency key。

- **NoticeCreator**（訂閱 `MemberInvitedInternal`）  
  - 建立站內 Notice，指向 `invitation_id`（不需 membership_id）。

- **MembershipCreator**（訂閱 `InvitationAccepted`）  
  - **唯一**建立 Membership 的地方。  
  - `IWorkspaceRepository.CreateMembershipForInviteeAsync(workspace, user_id or email→user_id, position_ids)`  
  - 冪等：以 `invitation_id` 檢查是否已處理。

- （可選）**ProjectionUpdaters**（訂閱全部 Invitation 事件）  
  - 維護查詢模型（見下）。

### Read Models (Projections)
- `WorkspaceInvitationList`（管理端/工作區頁面）  
  - 欄位：`invitation_id, workspace_id, tenant_id, email/invitee_user_id, position_names, status, created_at, expires_at`
- `MyInvitations`（被邀請者個人頁）  
  - 欄位：`invitation_id, workspace_id/name, inviter, status, created_at, expires_at, positions`

> UI 的「已邀請名單」與可見度全部依賴這些投影，**不需**先建 Membership。

### Repositories (關鍵介面)
- `IInvitationRepository`
  - `ExistsPendingForInviteeAsync(workspace_id, invitee_user_id)`
  - `ExistsPendingForEmailAsync(workspace_id, email)`
  - `GetByIdAsync(invitation_id)`, `AddAsync(invitation)`, `ListExpiredPendingIdsAsync(now)`
- `IWorkspaceRepository`
  - `GetByIdAsync(workspace_id)`
  - `CreateMembershipForInviteeAsync(workspace, user_id, position_ids)`（冪等；接受後才呼叫）

### Permissions
- 邀請人需要 `directory.workspaces.invite`。  
- 檢視邀請清單需要 `directory.workspaces.invite.view`（或管理者）。  
- 被邀請者查看自己的邀請只需登入；**未接受前不賦予任何 Workspace 權限**。

### Security
- 外部 token：**只存雜湊**（含 salt）；常數時間比對；接受/拒絕後立即失效。  
- 外部接受 API：避免洩露「是否存在/是否有效」，統一回通用結果頁。  
- 節流：同 workspace+email 在短時間內限制重複邀請（Notifications 層可實作）。

### Expiration Job
- 週期性：掃描 `ListExpiredPendingIdsAsync(now)` → `Invitation.Expire(now)` → 投影/通知更新。

### Idempotency
- 指令層：`Accept` 允許重複呼叫（返回 Accepted 結果）。  
- 事件層：Email/Notice/MembershipCreator 以 `invitation_id` 作唯一鍵保證不重做。

### Responsibility Split
- **Directory**：維護 Invitation 聚合、狀態遷移、發 Outbox 事件、維護投影。  
- **Notifications/Communications**：處理由 `MemberInvitedExternal` 觸發的寄信（模板、品牌、i18n、重寄、供應商抽象）。  
- **Workspace/Membership（仍在 Directory）**：只在 `InvitationAccepted` 的 Handler 建立 Membership。

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
