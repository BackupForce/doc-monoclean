# AccessControl Module Specification

> 狀態：規劃中（Placeholder）

## 1. 模組目的（Purpose）
統一管理 MonoClean 內所有資源的授權邏輯，提供 Policy Provider、權限合併與有效權限查詢能力。

## 2. Domain Model（預期）
| 實體 | 類型 | 說明 |
|------|------|------|
| AccessPolicy | Aggregate Root | 定義資源與可允許的操作（CRUD、Scope）。 |
| Resource | Entity | 描述可授權的資源（Node、Endpoint、Domain Object）。 |
| RoleGroup | Entity | 跨模組角色群組，對應 Identity 模組中的 Role。 |
| PermissionAssignment | Aggregate | 儲存指派關係（User/Role → Resource/Permission）。 |
| AccessRule | Value Object | 表示繼承、優先順序與條件。 |

## 3. Use Cases（預期）
| Use Case | 描述 |
|----------|------|
| GrantPermissionCommand | 指派權限給指定使用者／角色。 |
| RevokePermissionCommand | 撤銷既有權限。 |
| CheckAccessQuery | 驗證使用者是否可以操作特定資源。 |
| GetEffectivePermissionsQuery | 計算使用者在節點下的有效權限集合。 |
| SyncModulePermissionsCommand | 從模組定義中更新權限列表。 |

## 4. API（預期）
| Method | Path | 功能 |
|--------|------|------|
| GET | `/access-control/effective-permissions` | 查詢有效權限。 |
| POST | `/access-control/assign` | 指派權限。 |
| DELETE | `/access-control/revoke` | 移除權限。 |

## 5. 權限策略（預期）
- 採 **Node-based 層級繼承**，子節點繼承祖先的 allow 權限。
- 支援 **顯式拒絕（deny override）** 以阻止繼承權限。
- 權限合併順序：User Overrides → RoleGroup → Node → Resource Default。
- 與 Subscription 模組整合，提供方案層級的權限上限。

## 6. 外部整合（預期）
| 模組 | 整合內容 |
|------|----------|
| Identity | 取得使用者與角色資料。 |
| Directory | 取得節點階層資訊。 |
| Subscription | 根據方案限制授權。 |

## 7. TODO
- 定義 AccessPolicy 與 PermissionAssignment 的資料庫結構。
- 設計快取策略（Redis）以降低授權查詢成本。
- 擬定審計記錄與撤銷機制。
