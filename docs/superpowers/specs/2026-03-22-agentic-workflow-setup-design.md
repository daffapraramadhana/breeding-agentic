# Agentic Development Workflow Setup

**Date:** 2026-03-22
**Status:** Approved
**Approach:** Monorepo Wrapper + Per-Repo Context (Approach B)

## Problem

Three friction points when pair-programming with Claude on this project:

1. **Context loss** — every new session requires re-explaining project structure, conventions, and patterns
2. **Permission spam** — routine commands (build, test, git, prisma) trigger approval prompts
3. **Cross-repo coordination** — features span both repos (breeding-app backend + breeding-dashboard frontend) with no unified context

## Solution

Create a unified Claude Code workspace at the parent `breeding/` directory with layered context files and consolidated permissions.

## Components

### 1. Parent Git Repository (`breeding/`)

Initialize a lightweight git repo at the parent directory. Purpose: give Claude a unified workspace context.

- `.gitignore` ignores sub-repos (`breeding-app/`, `breeding-dashboard/`), `.env*` files, and OS artifacts — sub-repos keep their own git history
- Parent repo only tracks its own config files (`CLAUDE.md`, `.claude/`, `docs/`)

### 2. Root `CLAUDE.md` (`breeding/CLAUDE.md`)

System-wide context loaded at the start of every Claude session. Covers:

- **System overview:** Multi-tenant Farm ERP for poultry/livestock breeding. Two repos, PostgreSQL via Prisma 7, 74 models
- **Cross-repo conventions:**
  - Feature work: backend API first, then frontend integration
  - API contract: responses wrapped in `{ data, statusCode, timestamp }`
  - Auth: JWT Bearer tokens, 4 roles (SUPER_ADMIN, TENANT_ADMIN, MANAGER, STAFF)
  - All domain entities are tenant-scoped (tenantId)
  - Soft deletes via `deletedAt`
- **Development workflow:**
  - Backend: `cd breeding-app && npm run start:dev` (port 3002 via `.env`, default fallback is 3000)
  - Frontend: `cd breeding-dashboard && npm run dev` (port 3000)
  - Database: `npx prisma generate` after schema changes, `npx prisma migrate dev` for migrations
  - Commit style: conventional commits (`feat:`, `fix:`, `refactor:`, `docs:`)
- **Key rules:**
  - Always include tenantId filtering in queries
  - Use existing DTO patterns (Create*, Update*, Query* with class-validator)
  - Use shared combobox components on frontend for entity selection
  - i18n: add keys to both `en.json` and `id.json`
  - API types live in `breeding-dashboard/src/types/api.ts`

### 3. Backend `CLAUDE.md` (`breeding-app/CLAUDE.md`)

Backend-specific conventions:

- Module structure: `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.dto/` folder per module. Larger modules (logistics, marketing, master-data, standards, inventory) use `controllers/` and `services/` subfolders
- Prisma client: generated to `generated/prisma/` — imports use relative paths from there
- Guards: `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(...)` on protected endpoints
- Custom decorators: `@CurrentUser()`, `@CurrentTenant()` for JWT payload extraction
- Reference numbers: `ReferenceNumberGenerator` for document codes (PO, SO, GR, etc.)
- Status transitions: `StatusTransitionService` with `APPROVAL_MATRIX` constant
- Events: `@nestjs/event-emitter` for side effects (inventory movements after goods receipt)
- Swagger: decorate all endpoints with `@ApiTags`, `@ApiOperation`
- Build check: `npm run build` before committing
- Seed data: `npx prisma db seed` to reset test data

### 4. Frontend `CLAUDE.md` (`breeding-dashboard/CLAUDE.md`)

Frontend-specific conventions:

- App Router with `(auth)` and `(dashboard)` route groups
- All dashboard pages are `"use client"` — use `useApi<T>()` or `usePaginated<T>()` for data fetching
- Forms: React Hook Form + Zod schemas, follow existing form patterns
- UI primitives: shadcn/ui in `components/ui/`, shared components in `components/shared/`
- Entity selection: use existing combobox components from `components/forms/`
- Data tables: `DataTable` component from `components/shared/data-table`
- Styling: Tailwind only, `cn()` utility, oklch CSS variables for theming
- Status badges: `StatusBadge` component with workflow status colors
- i18n: `useTranslations()`, update both locale files
- No test framework — skip test file creation

### 5. Permissions (`breeding/.claude/settings.json`)

Consolidated allow-list:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npx prisma *)",
      "Bash(npx nest *)",
      "Bash(npx next *)",
      "Bash(npx tsc *)",
      "Bash(git *)",
      "Bash(ls *)",
      "Bash(cat *)",
      "Bash(curl *)",
      "Bash(node *)",
      "Bash(xargs *)"
    ]
  }
}
```

Intentionally excludes: `rm`, `npm install` (to stay aware of new deps), destructive operations.

### 6. Cleanup

Delete permission entries from the following files (the parent `settings.json` consolidates everything). Preserve the `.claude/` directory structure and `worktrees/` folder in breeding-app:

- `breeding-app/.claude/settings.json` — delete file (only had `xargs` permission, now in parent)
- `breeding-app/.claude/settings.local.json` — delete file (ad-hoc permissions now in parent)
- `breeding-dashboard/.claude/settings.local.json` — delete file (ad-hoc permissions now in parent)

Note: `settings.local.json` files may regenerate as users approve new permissions during sessions. This is normal Claude Code behavior.

## Files to Create/Modify

| File | Action |
|------|--------|
| `breeding/.gitignore` | Create |
| `breeding/CLAUDE.md` | Create |
| `breeding/.claude/settings.json` | Create |
| `breeding-app/CLAUDE.md` | Create |
| `breeding-dashboard/CLAUDE.md` | Create |
| `breeding-app/.claude/settings.json` | Delete |
| `breeding-app/.claude/settings.local.json` | Delete |
| `breeding-dashboard/.claude/settings.local.json` | Delete |

## Out of Scope

- Hooks (auto-linting, auto-build) — can add later if needed
- Worktree templates — existing worktree setup in breeding-app is sufficient
- Cross-repo scripts — pair-programming style means Claude coordinates naturally
- Git submodules — unnecessary complexity, nested repos work fine
