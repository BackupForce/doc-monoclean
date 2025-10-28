# Finance Module Specification

> 狀態：設計中

## 1. 模組目的（Purpose）
提供企業財務運作所需的應收、應付、月帳與群組結算能力，並確保數據一致性與可追蹤性。

## 2. Domain Model（初稿）
| 實體 | 類型 | 說明 |
|------|------|------|
| CashTransaction | Aggregate Root | 記錄單筆現金流（收入／支出）。 |
| Receivable | Aggregate Root | 客戶應收帳款，支援分期與折讓。 |
| Payable | Aggregate Root | 供應商應付帳款，支援付款排程。 |
| Reconciliation | Aggregate Root | 月帳結算，鎖定 BaseAmount 與匯率。 |
| GroupStatement | Aggregate Root | 跨部門／公司群組結算，整合多筆 Reconciliation。 |
| JournalEntry | Entity | 分錄資料，協助會計整合。 |
| SettlementRule | Value Object | 定義結算策略（時間區間、節點、匯率）。 |

## 3. Use Cases（規劃）
| Use Case | 描述 |
|----------|------|
| RecordCashTransactionCommand | 建立現金流紀錄並同步到總帳。 |
| CreateReceivableCommand | 建立應收帳款並綁定客戶節點。 |
| CreatePayableCommand | 建立應付帳款並綁定供應商節點。 |
| CloseMonthCommand | 建立 Reconciliation，鎖定結算區間。 |
| GenerateGroupStatementCommand | 聚合多個公司或部門的結算結果。 |
| GetLedgerQuery | 查詢指定節點的帳務明細。 |

## 4. API（預期）
- `POST /finance/cash-transactions`
- `POST /finance/receivables`
- `POST /finance/payables`
- `POST /finance/monthly-close`
- `POST /finance/group-statements`
- `GET /finance/ledgers`

## 5. 權限需求（預期）
- `finance.transactions.create`
- `finance.receivables.manage`
- `finance.payables.manage`
- `finance.reconciliation.close`
- `finance.group-statements.generate`
- `finance.ledgers.read`

## 6. Domain Events（預期）
- `CashTransactionRecordedDomainEvent`
- `ReceivableCreatedDomainEvent`
- `PayableCreatedDomainEvent`
- `MonthClosedDomainEvent`
- `GroupStatementGeneratedDomainEvent`

## 7. 外部整合
| 系統 | 說明 |
|------|------|
| Directory 模組 | 決定節點可見範圍，與 Node-based 權限整合。 |
| AccessControl 模組 | 控制財務資料的授權與審批。 |
| Audit 模組 | 紀錄財務操作的審計軌跡。 |

## 8. 測試策略（預期）
- 整合測試：Testcontainers 啟動 PostgreSQL，驗證結算鎖定與交易一致性。
- 演算法測試：針對 GroupStatement 聚合與匯率換算的單元測試。
- 事件流程測試：驗證 Domain Event → Outbox → Hangfire 工作流程。

## 9. TODO
- 設計匯率與多幣別支援方案。
- 與外部會計系統（如 ERP）資料交換。
- 建立財務報表模板與導出格式。
