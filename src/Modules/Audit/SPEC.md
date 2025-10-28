# Audit Module Specification

> 狀態：規劃中

## 1. 模組目的（Purpose）
提供全系統的操作審計能力，記錄 API 請求、Domain Event、權限檢查與登入活動，確保可追溯性。

## 2. Domain Model（預期）
| 實體 | 類型 | 說明 |
|------|------|------|
| AuditLog | Aggregate Root | 通用審計紀錄，包含 Actor、Action、Resource、Timestamp。 |
| ApiRequestLog | Entity | 紀錄 HTTP 請求與回應摘要。 |
| DomainEventLog | Entity | 紀錄 Domain Event Payload 與處理狀態。 |
| PermissionCheckLog | Entity | 紀錄授權檢查結果（Allow / Deny）。 |
| CorrelationId | Value Object | 串接跨模組操作的追蹤編號。 |

## 3. Use Cases（預期）
| Use Case | 描述 |
|----------|------|
| RecordApiRequestCommand | 保存 API 請求與回應資訊。 |
| RecordDomainEventCommand | 保存 Domain Event 詳細內容與結果。 |
| RecordPermissionCheckCommand | 紀錄授權檢查與決策。 |
| QueryAuditLogsQuery | 以時間／Actor／Resource 篩選審計資料。 |
| ExportAuditReportCommand | 匯出 CSV/JSON 報表。 |

## 4. Data Retention Policy（預期）
- 預設保留 180 天，透過 Hangfire 排程清理。
- 重要事件（安全相關）提升至 365 天。
- 支援以 Subscription 方案調整保留天數。

## 5. 外部整合
| 模組／系統 | 說明 |
|------------|------|
| Seq | 透過 Serilog Sink 傳送結構化日誌。 |
| Hangfire | 定期清除過期審計資料。 |
| AccessControl | 驗證使用者是否能讀取審計記錄。 |

## 6. TODO
- 設計查詢索引與分區策略以提升大量資料查詢效能。
- 定義敏感資料遮罩規則。
- 評估與 SIEM 平台整合的可能性。
