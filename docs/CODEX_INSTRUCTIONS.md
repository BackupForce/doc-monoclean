# ⚙️ Codex Instructions（文件維護規範）

> This file defines all rules Codex must follow when maintaining documentation  
> for the MonoClean project.  
> 本文件定義 Codex 在維護 MonoClean 專案文件時必須遵循的所有規則。

---

## 🧩 1. Branch Policy（分支策略）

- All documentation work **must be done on the `doc-maintenance` branch**.  
  所有文件維護與修改，**必須在 `doc-maintenance` 分支上進行**。
- ❌ Do not commit directly to `main`.  
  不得直接在 `main` 分支提交任何變更。
- After completing a task, open a Pull Request from  
  `doc-maintenance → main` for review and merge.  
  完成任務後，請從 `doc-maintenance` 開 PR 至 `main`，待審查合併。

---

## 📁 2. File Structure Rules（檔案結構規則）

- The **single source of truth (SSOT)** for module specifications is:
/src/Modules/{Module}/SPEC.md
- The **global overview file** is:
/docs/SYSTEM_SPEC.md
- ❌ Never create folders like `/docs/Module.*`  
絕對不要建立 `/docs/Module.*` 這類資料夾（例如 `/docs/Module.Identity/`）。
- All updates must follow the SSOT principle —  
module specs belong under `/src/Modules/`, not `/docs/`.

---

## 🧾 3. Commit Message Convention（提交訊息規範）

Use clear, semantic commit messages written in **Traditional Chinese** (English optional for clarity).

使用明確、語意化的 Commit 訊息（以繁體中文撰寫，可附英文輔助）：

| 類型 | 範例 | 說明 |
|------|------|------|
| `docs:` | `docs: 更新 SYSTEM_SPEC 全域概述` | 修改全域文件 |
| `docs({module}):` | `docs(identity): 修正使用者註冊流程規格` | 修改特定模組規格 |
| `chore:` | `chore: 移除多餘測試檔案 hello_world.txt` | 清理或維護性變更 |

> 💡 English examples (for reference):  
> `docs: update SYSTEM_SPEC`  
> `docs(identity): revise SPEC`  
> `chore: remove redundant file`

---

## ⚙️ 4. Execution Workflow（執行流程）

1. **Always read this file first** before performing any modification.  
 每次執行任務前，請先閱讀並遵守本文件。
2. Follow the latest rules described here.  
 嚴格依照本文件的最新規則執行。
3. When executing a new task, refer to both:  
 - This document (as the general policy)  
 - The new task instruction provided in context  
 每次任務請同時遵守：  
 - 本文件（一般規範）  
 - 當前任務指令（即時目標）

---

## 🈶 5. Language Preference（語言偏好）

- All Codex responses, explanations, commit messages, and PR titles  
**must be written in Traditional Chinese (繁體中文)**.  
所有 Codex 回覆、說明、Commit 訊息與 PR 標題，**皆須使用繁體中文**。
- English may be used only for code, file paths, and branch names.  
英文僅可用於程式碼、檔案路徑與分支名稱。
- Example / 範例：
✅ docs: 更新 AccessControl 模組規格
❌ docs: update AccessControl spec

---

## 🚫 6. Prohibited Actions（禁止事項）

- ❌ Do not create new folders under `/docs` for modules.  
禁止在 `/docs` 下建立模組資料夾。
- ❌ Do not rename existing SPEC.md files.  
禁止更動既有 `SPEC.md` 檔案名稱。
- ❌ Do not overwrite unrelated files outside `/docs` and `/src/Modules`.  
禁止修改與文件無關的程式碼或設定檔。

---

## 🧠 7. Example Task Prompt（範例任務指令）
Read and follow /docs/CODEX_INSTRUCTIONS.md.
Then perform the following task:
Delete the folder /docs/Module.Identity/
Delete the file /hello_world.txt
Update AccessControl status in /docs/SYSTEM_SPEC.md
Commit each change separately with semantic messages

---

## ✅ 8. Expected Result（預期結構）

After cleanup, the project should maintain the following structure:
docs/
SYSTEM_SPEC.md
CODEX_INSTRUCTIONS.md
src/
Modules/
Identity/SPEC.md
Directory/SPEC.md
AccessControl/SPEC.md
Finance/SPEC.md
Subscription/SPEC.md
Audit/SPEC.md
All module specs live under `/src/Modules/`;  
`/docs` only contains global-level documentation.

---

_Last updated: 2025-10-28_

