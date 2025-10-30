# AccessControl Module Specification (v0.1)

## 1. Purpose
作為**唯一的授權系統（Single Source of Truth）**，管理 `Resource(Node)`、`Permission`、`RoleGroup` 與 `PermissionAssignment`；提供有效權限查詢與檢查，並透過事件與其他模組整合。

## 2. Domain Model
### 2.1 Aggregates & Entities
| 實體 | 類型 | 說明 | 關聯 |
|------|------|------|------|
| ResourceNode | Aggregate Root | 對應外部系統的實體資源（如 Tenant/Department），可階層化 | ResourceNode→Children（一對多） |
| NodeRelation | Value Object | Closure Table/Materialized Path metadata：ancestor/descendant/depth | 隸屬 ResourceNode |
| Permission | Entity | 系統權限碼的登錄與描述（集中註冊） | — |
| RoleGroup | Aggregate Root | 權限集合的邏輯群組 | RoleGroup→Permissions（多對多） |
| PermissionAssignment | Entity | 對 `UserId` / `RoleGroup` / `ResourceNode` 的授權/拒絕紀錄 | — |

> 註：AccessControl 不擁有 `User`、`Tenant`、`Department` 的資料，只保存其對應的 `NodeId` 與外部自然鍵。

### 2.2 Value Objects
- **PermissionCode**：`<module>.<resource>.<action>`，如 `directory.departments.manage`
- **AssignmentScope**：`Global` / `NodeScoped`
- **Decision**：`Allow` / `Deny`（Deny 具最高優先）

## 3. Use Cases (CQRS)
| Use Case | Request | Response | 描述 |
|----------|---------|----------|------|
| RegisterPermissionsCommand | List<PermissionCode> | Result | 同步三模組的權限碼到本模組（種子/對齊） |
| GrantPermissionCommand | Subject(UserId or RoleGroupId), PermissionCode, NodeId?, Decision | Result | 指派/覆寫授權，支援節點範圍 |
| RevokePermissionCommand | Subject, PermissionCode, NodeId? | Result | 撤銷授權 |
| GetEffectivePermissionsQuery | UserId, NodeId | Result<List<PermissionCode>> | 計算使用者於節點的有效權限（含繼承/拒絕） |
| CheckAccessQuery | UserId, NodeId, PermissionCode | Result<bool> | 快速檢查單一權限 |

## 4. API Endpoints
| Method | Path | Handler | 權限 |
|--------|------|---------|------|
| POST | `/access-control/permissions:sync` | RegisterPermissionsCommand | `access.permissions.manage` |
| POST | `/access-control/assign` | GrantPermissionCommand | `access.assign` |
| DELETE | `/access-control/assign` | RevokePermissionCommand | `access.assign` |
| GET | `/access-control/effective-permissions` | GetEffectivePermissionsQuery | `access.permissions.read` |
| GET | `/access-control/check` | CheckAccessQuery | `access.permissions.read` |

## 5. Permission Registry
- `access.permissions.manage`
- `access.permissions.read`
- `access.assign`
- Identity 相關（由 Identity 模組提供權限定義、這裡註冊）：`identity.users.read`, `identity.users.manage`, ...
- Directory 相關（由 Directory 模組提供權限定義、這裡註冊）：`directory.tenants.manage`, `directory.departments.manage`, `directory.members.invite`, `directory.members.manage`, `directory.departments.read`, `directory.positions.manage`, `directory.invitations.read`

## 6. Integration (Events In)
AccessControl 訂閱以下事件以維護 ResourceNode 與授權：
- `TenantCreated`
- `DepartmentCreated`
- `DepartmentRenamed`
- `DepartmentMoved`
- `MemberJoinedDepartment`
- `MemberRemoved`
- `PositionAssigned`
- （可選）Identity：`UserRegistered`, `UserLocked`

> **冪等**：以外部自然鍵（TenantId/DepartmentId/UserId）+ 事件版本號避免重複處理  
> **最小寫入**：僅在狀態變更時更新資料；Effective 計算結果用 Redis 快取（key：`user:{UserId}:node:{NodeId}`）

## 7. Authorization Rules
- **繼承**：祖先節點 `Allow` → 子孫繼承；顯式 `Deny` 最高優先、可覆蓋 Allow。
- **合併順序**：User 覆寫 > RoleGroup > Node Allow → 最終判定。
- **Scope**：若無 `NodeId` 即為全域授權；節點授權只在分支內有效。

## 8. Testing Strategy
- 單元：合併順序、Deny 優先、節點繼承。
- 整合：事件冪等、RegisterPermissions 與 Seed 對齊、Redis 快取命中率。
- 端到端：Directory 邀請→加入→PositionAssigned→EffectivePermissions 應包含模板權限。

## 9. TODO
- Deny/Allow 與多層 RoleGroup 疊加的最佳化
- 大租戶快取失效策略
