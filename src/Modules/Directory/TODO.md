# Directory 模組待辦事項

## 功能開發
- [ ] 依據 `CreateCompanyCommand` 建立公司與 root 部門流程。
- [ ] 依據 `CreateDepartmentCommand` 建立部門並支援父子節點關係。
- [ ] 依據 `UpdateDepartmentCommand` 更新部門名稱與狀態。
- [ ] 依據 `InviteMemberCommand` 建立邀請並觸發 Email 寄送。
- [ ] 依據 `AcceptInvitationCommand` 完成邀請接受與成員建立。
- [ ] 依據 `AssignPositionCommand` 指派職位並同步權限模板。
- [ ] 實作 `GetOrganizationTreeQuery` 以輸出完整組織樹狀結構。

## 權限與節點管理
- [ ] 建立部門節點的層級繼承與 `DepartmentRelation` 儲存邏輯。
- [ ] 設計 `DepartmentCode`、`DepartmentName`、`NodePath` 等 Value Object 的驗證規則。
- [ ] 建立 `directory.*` 權限代碼對應的授權檢查與 AccessControl 模組同步機制。

## 邀請與成員
- [ ] 實作 `Membership` 與 `Position` 的多對多關聯與權限套用。
- [ ] 建立邀請流程的 Token 驗證與到期處理。
- [ ] 與 Identity 模組整合以驗證受邀使用者。

## 事件與整合
- [ ] 發佈並處理 `CompanyCreatedDomainEvent` 與 `DepartmentCreatedDomainEvent`。
- [ ] 追蹤 `DepartmentStatusChangedDomainEvent` 以更新節點狀態。
- [ ] 實作 `MemberInvitedDomainEvent` 與 `MemberJoinedDepartmentDomainEvent` 的流程監控。
- [ ] 與 AccessControl 模組建立節點同步與權限指派整合。
- [ ] 與 SMTP 服務整合以寄送部門邀請信。

## 測試策略
- [ ] 撰寫組合測試以驗證部門階層新增／刪除時的節點繼承。
- [ ] 使用 Testcontainers 建立 PostgreSQL 整合測試以確認層級儲存結構。
- [ ] 撰寫 API 測試覆蓋邀請流程與權限檢查。

## 開放問題
- [ ] 評估是否支援跨公司共享部門的需求與設計調整。
- [ ] 研究大型公司情境下節點繼承查詢的效能優化方案。
- [ ] 與 Subscription 模組協調以限制可建立的部門數量。
