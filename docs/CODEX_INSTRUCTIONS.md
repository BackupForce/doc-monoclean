# Codex Instructions

## 1. Branch Policy
- All documentation work must be done on the `doc-maintenance` branch.
- Do not commit directly to `main`.
- After completing a task, open a Pull Request from `doc-maintenance` → `main`.

## 2. File Structure Rules
- The single source of truth (SSOT) for module specs:
  - `/src/Modules/{Module}/SPEC.md`
- Global overview file:
  - `/docs/SYSTEM_SPEC.md`
- Never create `/docs/Module.*` folders.

## 3. Commit Message Convention
Use clear, semantic commits:
- `docs: update SYSTEM_SPEC`
- `docs({module}): revise SPEC`
- `chore: remove redundant file`

## 4. Execution Workflow
1. Always read this file first before performing any modification.
2. Follow the latest rules described here.
3. When executing a new task, refer to both:
   - This document (as the general policy)
   - The new task instruction provided in context.

## 5. Example Task Prompt
