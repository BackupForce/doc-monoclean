# MonoClean Documentation

MonoClean 是一套以 **Modular Monolith** 為核心、採用 **Clean Architecture + DDD + CQRS** 的企業系統範例。本文件庫專注於維護其架構與模組規格，協助團隊快速掌握系統整體輪廓。

## 內容總覽
- [系統總覽規格（SYSTEM_SPEC）](docs/SYSTEM_SPEC.md)
- 模組規格：
  - [Identity](src/Modules/Identity/SPEC.md)
  - [Directory](src/Modules/Directory/SPEC.md)
  - [AccessControl](src/Modules/AccessControl/SPEC.md)
  - [Finance](src/Modules/Finance/SPEC.md)
  - [Subscription](src/Modules/Subscription/SPEC.md)
  - [Audit](src/Modules/Audit/SPEC.md)

## 維護原則
- 所有識別碼採用強型別 ID 與 `snake_case` 資料庫命名。
- 新增／調整 CQRS Handler、API 或權限時，需同步更新對應模組 `SPEC.md` 與 `docs/SYSTEM_SPEC.md`。
- 保持文件與實作同步，以利 Codex 自動維護與審查流程。

## 參考資源
- .NET 8、EF Core、Dapper、MediatR、PostgreSQL、Redis、Hangfire、Seq
- React、TypeScript、Ant Design Pro、Axios
- Docker Compose（Postgres、Redis、Seq、WebApi）

歡迎依據團隊需求擴充更多模組與文件，確保 MonoClean 在擴展時維持乾淨且一致的架構。
