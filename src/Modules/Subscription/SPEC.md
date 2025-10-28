# Subscription Module Specification

> 狀態：設計中

## 1. 模組目的（Purpose）
管理方案（Plans）、訂閱（Subscriptions）與功能限制（Feature Limits），並與 AccessControl 模組整合以限制授權範圍。

## 2. Domain Model（草稿）
| 實體 | 類型 | 說明 |
|------|------|------|
| Plan | Aggregate Root | 定義方案與對應功能權限集合。 |
| Subscription | Aggregate Root | 客戶或公司對特定方案的訂閱狀態，含週期與到期資訊。 |
| FeatureLimit | Entity | 方案限制設定（數量、頻率、布林值）。 |
| UsageRecord | Entity | 記錄方案使用量以供限制判斷。 |
| TrialPolicy | Value Object | 試用規則（天數、轉換條件）。 |

## 3. Use Cases（規劃）
| Use Case | 描述 |
|----------|------|
| CreatePlanCommand | 建立方案與權限集合。 |
| UpdatePlanCommand | 調整方案資訊與價格。 |
| ActivateSubscriptionCommand | 建立或啟用訂閱。 |
| CancelSubscriptionCommand | 取消訂閱或排程終止。 |
| RecordUsageCommand | 紀錄功能使用量並檢查限制。 |
| GetSubscriptionStatusQuery | 取得公司目前方案與剩餘額度。 |

## 4. API（預期）
- `POST /subscription/plans`
- `PATCH /subscription/plans/{id}`
- `POST /subscription/subscriptions`
- `POST /subscription/subscriptions/{id}/cancel`
- `POST /subscription/usage`
- `GET /subscription/status`

## 5. 權限需求（預期）
- `subscription.plans.manage`
- `subscription.subscriptions.manage`
- `subscription.usage.record`
- `subscription.status.read`

## 6. Domain Events（預期）
- `PlanCreatedDomainEvent`
- `SubscriptionActivatedDomainEvent`
- `SubscriptionCancelledDomainEvent`
- `UsageLimitReachedDomainEvent`

## 7. 外部整合
| 模組 | 說明 |
|------|------|
| AccessControl | 根據方案啟用或關閉權限集合。 |
| Billing（未定義） | 付款與續約處理。 |
| Directory | 限制可建立的部門／成員數量。 |

## 8. 測試策略（預期）
- 單元測試：檢查 FeatureLimit 與 UsageRecord 的限制邏輯。
- 整合測試：模擬方案升級／降級流程與 AccessControl 同步。
- 時序測試：確保到期排程（Hangfire）能準時停用授權。

## 9. TODO
- 設計與付款系統的介接流程。
- 決定升級／降級對權限的漸進式影響策略。
- 規劃多區貨幣與稅率處理。
