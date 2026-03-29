# Breeding ERP System

Multi-tenant Farm ERP for poultry/livestock breeding operations.

## Fresh Setup (Clone from Scratch)

### Prerequisites

- **Node.js** >= 20.x
- **npm** >= 10.x
- **PostgreSQL** >= 15 (running locally or accessible)
- **Git**

### Step 1: Clone the orchestration repo

```bash
git clone https://github.com/daffapraramadhana/breeding-agentic.git breeding
cd breeding
```

### Step 2: Clone the sub-repos

Each app has its own Git repository. Clone them inside the `breeding/` directory:

```bash
# Backend API
git clone https://github.com/daffapraramadhana/breeding-app.git breeding-app

# Frontend Dashboard
git clone https://github.com/daffapraramadhana/breeding-web.git breeding-dashboard
```

### Step 3: Configure environment variables

**Backend** — create `breeding-app/.env`:

```env
PORT=3002
DATABASE_URL="postgresql://<USER>:<PASSWORD>@localhost:5432/breeding_app?schema=public"
JWT_SECRET="change-me-in-production"
```

**Frontend** — create `breeding-dashboard/.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:3002/api
```

### Step 4: Install dependencies & set up database

```bash
# Backend
cd breeding-app
npm install
npx prisma generate
npx prisma migrate dev
npx prisma db seed
cd ..

# Frontend
cd breeding-dashboard
npm install
cd ..
```

### Step 5: Run

```bash
# Terminal 1 — Backend (port 3002)
cd breeding-app && npm run start:dev

# Terminal 2 — Frontend (port 3000)
cd breeding-dashboard && npm run dev
```

## Git Remotes

This project uses three separate Git repositories:

| Repo | Purpose | Remote URL |
|------|---------|------------|
| `breeding/` (root) | Orchestration, docs, specs, CLAUDE.md | `https://github.com/daffapraramadhana/breeding-agentic.git` |
| `breeding-app/` | NestJS backend API | `https://github.com/daffapraramadhana/breeding-app.git` |
| `breeding-dashboard/` | Next.js frontend | `https://github.com/daffapraramadhana/breeding-web.git` |

Each sub-repo also has an internal MDD remote (`mdd`) pointing to `git.mdd.co.id`. The `origin` remote on each sub-repo points to GitHub.

**AI agents / users**: always `cd` into the correct directory before running `git` commands — each folder is its own repo with its own history.

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
