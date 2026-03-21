# Breeding ERP System

Multi-tenant Farm ERP for poultry/livestock breeding operations.

## Architecture

- **breeding-app/** — NestJS 11 backend API (port 3002 via `.env`, fallback 3000)
- **breeding-dashboard/** — Next.js 16 frontend (port 3000)
- **Database** — PostgreSQL via Prisma 7 (74 models, 29 enums)

## Cross-Repo Conventions

- Feature work: build backend API first, then frontend integration
- API contract: all responses wrapped in `{ data, statusCode, timestamp }` via TransformInterceptor
- Auth: JWT Bearer tokens, 4 roles — `SUPER_ADMIN`, `TENANT_ADMIN`, `MANAGER`, `STAFF`
- All domain entities are tenant-scoped (`tenantId` field)
- Soft deletes via `deletedAt` field (never hard-delete)
- API types on frontend live in `breeding-dashboard/src/types/api.ts` — keep in sync with backend DTOs

## Development Workflow

```bash
# Backend
cd breeding-app && npm run start:dev

# Frontend
cd breeding-dashboard && npm run dev

# Database
cd breeding-app && npx prisma generate       # after schema changes
cd breeding-app && npx prisma migrate dev     # create/apply migrations
cd breeding-app && npx prisma db seed         # reset seed data
```

## Commit Style

Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`

Scope with module name when applicable: `feat(logistics):`, `fix(inventory):`

## Key Rules

- Always filter by `tenantId` in database queries
- Use existing DTO patterns: `Create*Dto`, `Update*Dto`, `Query*Dto` with class-validator
- Use shared combobox components on frontend for entity selection (see `breeding-dashboard/src/components/forms/`)
- i18n: add keys to both `breeding-dashboard/messages/en.json` and `id.json`
- Build check backend before committing: `cd breeding-app && npm run build`
