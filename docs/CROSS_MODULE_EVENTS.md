# Cross-Module Event Mapping

| Producer | Event | Consumer | Action |
|----------|------|----------|--------|
| Directory | TenantCreated | AccessControl | 建 Root ResourceNode |
| Directory | DepartmentCreated | AccessControl | 建 Node + Relation |
| Directory | DepartmentRenamed | AccessControl | 更新節點名稱 |
| Directory | DepartmentMoved | AccessControl | 重建 Relation |
| Directory | MemberInvited | — | 通知/稽核（可選） |
| Directory | MemberJoinedDepartment | AccessControl | 依 Position 模板展開並授權 |
| Directory | MemberRemoved | AccessControl | 撤權 |
| Directory | PositionAssigned | AccessControl | 重新計算並下發授權 |
| Directory | InvitationExpired | AccessControl | 清理暫存授權（可選） |
| Identity | UserRegistered | AccessControl | 初始化主體快取（可選） |
| Identity | UserLocked | AccessControl | 標記使用者授權為鎖定（可選） |

> 所有事件皆走 Outbox；消費端需冪等。
