# Sales Master Data (Customer + Mature Bird Standard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give `Customer` the fields the sales module needs (identity documents, vehicle plate, credit limit toggle, TOP, savings/installment per kg, registration branch), make "operating branches" a real many-to-many that filters which customers can be ordered for, and add a `MatureBirdStandard` master that will supply Minimum Weight on the future order form.

**Architecture:** Additive Prisma changes only — nine nullable columns on `customers`, a `customer_branches` link table shaped exactly like the existing `supplier_categories`, and a two-table master (`mature_bird_standards` + `mature_bird_standard_ranges`). Backend work is one extended service (`CustomerService`) plus one new service that copies the `MatureBirdTypeService` pattern. Frontend is one rewritten dialog, one new master page, and an optional `branchId` prop on `CustomerCombobox` so existing callers are unaffected.

**Tech Stack:** NestJS 11 + Prisma 7 + class-validator (backend, `breeding-app/`), Next.js 16 + shadcn/ui + next-intl (frontend, `breeding-dashboard/`), PostgreSQL.

**Spec:** `docs/superpowers/specs/2026-09-26-sales-master-data-design.md`

## Global Constraints

- All Prisma queries filter by `tenantId` (CLAUDE.md).
- Soft delete via `deletedAt`; never hard-delete domain entities. Link rows (`customer_branches`) and child rows (`mature_bird_standard_ranges`) are replaced wholesale — they carry no `deletedAt`.
- API responses are wrapped `{ data, statusCode, timestamp }` by `TransformInterceptor` — `curl` output below shows the `data` part only.
- `ValidationPipe` runs with `whitelist: true, forbidNonWhitelisted: true, transform: true` (`breeding-app/src/main.ts:17`) — any request property not declared on a DTO returns 400.
- Prisma client is generated to `breeding-app/generated/prisma/client.js`. Run `npx prisma generate` after every schema change.
- i18n keys go in **both** `breeding-dashboard/messages/en.json` and `messages/id.json`.
- Frontend API types live in `breeding-dashboard/src/types/api.ts`.
- `creditLimitEnabled` defaults to `false`: existing customers stay unrestricted, matching today's behaviour where `creditLimit` is never enforced.
- `installmentPerKg` is **stored only**. No deduction logic anywhere — stakeholder question B4 is unanswered (spec Non-goal).
- Build check before every commit: `cd breeding-app && npm run build`, or `cd breeding-dashboard && npx tsc --noEmit && npm run build`.
- Each sub-repo is its own git repo. `cd` into `breeding-app/` or `breeding-dashboard/` before `git` commands. Commit style: `feat(master-data):`, `fix(master-data):`.
- Commit messages end with the trailer `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Dev backend on `http://localhost:3002/api`; seeded login `admin@demo.farm` / `password123`; login response carries the token at `data.accessToken`.
- The shell has an `rtk` wrapper that can mangle `curl` output. Run verification commands as `rtk proxy curl ...` and write responses to a file before parsing.

## Review Focus

Input classes the spec implies but that no feature step naturally exercises. Each has a test pinned to the task that owns the code.

1. **A branch that is soft-deleted after a customer was linked to it** — the customer's `operatingBranches` must stop returning it, otherwise the edit dialog resends a dead id and every save returns 400. This exact bug shipped for suppliers and was fixed in `1adf940`. *(Task 2, Step 6)*
2. **The same branch id sent twice in `operatingBranchIds`** — must be de-duplicated before `createMany`, or Postgres raises a composite-primary-key violation and the request 500s. *(Task 2, Step 6)*
3. **A range whose `valueTo` is below its `valueFrom`, or a standard with no ranges at all** — both must be rejected, otherwise the master holds data that can never match a weight. *(Task 3, Step 5)*
4. **Branch ids belonging to another tenant** (both `registrationBranchId` and `operatingBranchIds`) — must be rejected, or one tenant links its customers to another tenant's branches. *(Task 2, Step 6)*
5. **Negative `topDays` / `savingsPerKg` / `installmentPerKg`** — must be rejected at the DTO, or nonsense values flow into the sales module later. *(Task 2, Step 6)*

---

## File Structure

### Backend (`breeding-app/`)

| Action | Path | Responsibility |
|---|---|---|
| Modify | `prisma/schema.prisma` | `Customer` columns + relations, `CustomerBranch`, `MatureBirdStandard`, `MatureBirdStandardRange`, `Branch` back-relations |
| Create (generated) | `prisma/migrations/<ts>_add_sales_master_data/` | one additive migration |
| Modify | `prisma/seed.ts` | customer fields + operating branches, one sample standard |
| Modify | `src/modules/master-data/dto/create-customer.dto.ts` | nine new fields + `operatingBranchIds` |
| Create | `src/modules/master-data/dto/query-customer.dto.ts` | `QueryCustomerDto extends PaginationDto` with `branchId?` |
| Modify | `src/modules/master-data/services/customer.service.ts` | branch validation, replace-all links, `branchId` filter, includes |
| Modify | `src/modules/master-data/controllers/customer.controller.ts` | use `QueryCustomerDto` |
| Create | `src/modules/master-data/dto/create-mature-bird-standard.dto.ts` | name + nested ranges |
| Create | `src/modules/master-data/dto/update-mature-bird-standard.dto.ts` | `PartialType` |
| Create | `src/modules/master-data/services/mature-bird-standard.service.ts` | CRUD + range replace-all |
| Create | `src/modules/master-data/controllers/mature-bird-standard.controller.ts` | `/api/mature-bird-standards` |
| Modify | `src/modules/master-data/master-data.module.ts` | register the new service/controller |

### Frontend (`breeding-dashboard/`)

| Action | Path | Responsibility |
|---|---|---|
| Modify | `src/types/api.ts` | `Customer` new fields, `CustomerBranchLink`, `MatureBirdStandard`, `MatureBirdStandardRange` |
| Modify | `messages/en.json`, `messages/id.json` | new keys for both namespaces |
| Modify | `src/components/forms/customer-combobox.tsx` | optional `branchId` prop |
| Modify | `src/app/(dashboard)/customers/page.tsx` | grouped dialog, new columns |
| Create | `src/app/(dashboard)/mature-bird-standards/page.tsx` | master page with range editor |
| Modify | `src/lib/constants.ts` | sidebar entry |
| Modify | `src/components/layout/breadcrumbs.tsx` | breadcrumb label |

---

## Task 1: Prisma schema, migration and seed

**Files:**
- Modify: `breeding-app/prisma/schema.prisma` — `Customer` (~line 605), `Branch` (~line 252), plus three new models
- Modify: `breeding-app/prisma/seed.ts` — customers block (~line 212)
- Create (generated): `breeding-app/prisma/migrations/<timestamp>_add_sales_master_data/migration.sql`

**Interfaces:**
- Produces: Prisma delegates `customerBranch`, `matureBirdStandard`, `matureBirdStandardRange`; `Customer` fields `registrationBranchId`, `idCardNumber`, `taxNumber`, `vehiclePlate`, `creditLimitEnabled`, `topDays`, `savingsPerKg`, `installmentPerKg`, `lastPaymentDate`; relations `customer.operatingBranches`, `customer.registrationBranch`.

- [ ] **Step 1: Extend the `Customer` model**

In `prisma/schema.prisma`, replace the `Customer` model with:

```prisma
model Customer {
  id                   String    @id @default(uuid())
  tenantId             String    @map("tenant_id")
  code                 String
  name                 String
  contactPerson        String?   @map("contact_person")
  phone                String?
  email                String?
  address              String?
  city                 String?
  creditLimit          Decimal?  @map("credit_limit") @db.Decimal(18, 2)
  performance          String?
  collateral           String?
  registrationBranchId String?   @map("registration_branch_id")
  idCardNumber         String?   @map("id_card_number")
  taxNumber            String?   @map("tax_number")
  vehiclePlate         String?   @map("vehicle_plate")
  creditLimitEnabled   Boolean   @default(false) @map("credit_limit_enabled")
  topDays              Int?      @map("top_days")
  savingsPerKg         Decimal?  @map("savings_per_kg") @db.Decimal(18, 2)
  installmentPerKg     Decimal?  @map("installment_per_kg") @db.Decimal(18, 2)
  lastPaymentDate      DateTime? @map("last_payment_date") @db.Date
  createdAt            DateTime  @default(now()) @map("created_at")
  updatedAt            DateTime  @updatedAt @map("updated_at")
  deletedAt            DateTime? @map("deleted_at")

  registrationBranch Branch?          @relation("CustomerRegistrationBranch", fields: [registrationBranchId], references: [id])
  operatingBranches  CustomerBranch[]

  salesOrders   SalesOrder[]
  deliveries    Delivery[]
  salesInvoices SalesInvoice[]
  salesPayments SalesPayment[]

  @@unique([tenantId, code])
  @@index([registrationBranchId])
  @@map("customers")
}
```

Before replacing, run `sed -n '605,640p' prisma/schema.prisma` and confirm the existing field and relation list matches what is preserved above. If the model has anything extra, keep it — report as a concern rather than dropping it.

- [ ] **Step 2: Add the three new models**

Directly after the `Customer` model, add:

```prisma
model CustomerBranch {
  customerId String @map("customer_id")
  branchId   String @map("branch_id")

  customer Customer @relation(fields: [customerId], references: [id])
  branch   Branch   @relation("BranchOperatingCustomers", fields: [branchId], references: [id])

  @@id([customerId, branchId])
  @@index([branchId])
  @@map("customer_branches")
}

model MatureBirdStandard {
  id        String    @id @default(uuid())
  tenantId  String    @map("tenant_id")
  name      String
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")
  deletedAt DateTime? @map("deleted_at")

  ranges MatureBirdStandardRange[]

  @@unique([tenantId, name])
  @@map("mature_bird_standards")
}

model MatureBirdStandardRange {
  id         String  @id @default(uuid())
  standardId String  @map("standard_id")
  valueFrom  Decimal @map("value_from") @db.Decimal(18, 4)
  valueTo    Decimal @map("value_to") @db.Decimal(18, 4)

  standard MatureBirdStandard @relation(fields: [standardId], references: [id], onDelete: Cascade)

  @@index([standardId])
  @@map("mature_bird_standard_ranges")
}
```

- [ ] **Step 3: Add `Branch` back-relations**

In `model Branch`, inside the block that already lists `customers`-adjacent relations (near `salesOrders  SalesOrder[]`), add these two lines:

```prisma
  registeredCustomers Customer[]       @relation("CustomerRegistrationBranch")
  operatingCustomers  CustomerBranch[] @relation("BranchOperatingCustomers")
```

- [ ] **Step 4: Seed the new fields**

In `prisma/seed.ts`, replace the customers block (currently two `prisma.customer.create` calls plus a `console.log`) with:

```ts
  const custPasar = await prisma.customer.create({
    data: {
      tenantId: tenant.id, code: 'CUS-001', name: 'Pasar Induk Kramat Jati',
      address: 'Jakarta Timur', city: 'Jakarta',
      registrationBranchId: branchMain.id,
      idCardNumber: '3171010101800001', taxNumber: '01.234.567.8-901.000',
      vehiclePlate: 'B 1234 ABC',
      creditLimitEnabled: true, creditLimit: 50000000, topDays: 3,
      savingsPerKg: 100,
      operatingBranches: { create: [{ branchId: branchMain.id }] },
    },
  });
  const custRestoran = await prisma.customer.create({
    data: {
      tenantId: tenant.id, code: 'CUS-002', name: 'Restoran Ayam Goreng Suharti',
      address: 'Jl. Ir. H. Juanda, Bogor', city: 'Bogor',
      registrationBranchId: branchMain.id,
      vehiclePlate: 'F 5678 DEF',
      topDays: 7,
      operatingBranches: { create: [{ branchId: branchMain.id }] },
    },
  });
  console.log(`[Customer] Created 2 customers`);

  const stdBroiler = await prisma.matureBirdStandard.create({
    data: {
      tenantId: tenant.id, name: 'Standar Broiler',
      ranges: {
        create: [
          { valueFrom: 0.8, valueTo: 1.2 },
          { valueFrom: 1.2, valueTo: 1.8 },
          { valueFrom: 1.8, valueTo: 2.5 },
        ],
      },
    },
  });
  console.log(`[MatureBirdStandard] Created 1 standard with 3 ranges`);
  void stdBroiler;
```

`branchMain` is the branch constant created earlier in the seed. Confirm its variable name with `grep -n "branch.*await prisma.branch.create" prisma/seed.ts` and use whatever it is.

- [ ] **Step 5: Generate the migration and client**

```bash
cd breeding-app && npx prisma migrate dev --name add_sales_master_data && npx prisma generate
```

Expected: nine `ADD COLUMN` statements on `customers`, one index, one FK to `branches`, then `CREATE TABLE` for `customer_branches`, `mature_bird_standards`, `mature_bird_standard_ranges` with their indexes and FKs. No `UPDATE` and no data-loss warning.

If the shell's `rtk` wrapper makes `npx` fail with "No such file or directory", run it as `rtk proxy npx ...`.

If Prisma asks an interactive question or reports drift, stop and report BLOCKED with the exact output.

- [ ] **Step 6: Build**

```bash
cd breeding-app && npm run build
```

Expected: success. Nothing references the new fields yet.

- [ ] **Step 7: Commit**

```bash
cd breeding-app && git add prisma/schema.prisma prisma/migrations prisma/seed.ts && git commit -m "feat(master-data): add customer sales fields, operating branches and mature bird standard

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 2: Customer DTO, service and controller

**Files:**
- Modify: `breeding-app/src/modules/master-data/dto/create-customer.dto.ts`
- Create: `breeding-app/src/modules/master-data/dto/query-customer.dto.ts`
- Modify: `breeding-app/src/modules/master-data/services/customer.service.ts`
- Modify: `breeding-app/src/modules/master-data/controllers/customer.controller.ts`

**Interfaces:**
- Consumes: Prisma delegates from Task 1.
- Produces:
  - `POST /customers` and `PATCH /customers/:id` accept `registrationBranchId?`, `idCardNumber?`, `taxNumber?`, `vehiclePlate?`, `creditLimitEnabled?`, `topDays?`, `savingsPerKg?`, `installmentPerKg?`, `operatingBranchIds?: string[]` on top of the existing fields.
  - `PATCH` with `operatingBranchIds: []` clears the links; omitting the key leaves them untouched.
  - `GET /customers?branchId=<uuid>` returns only customers operating in that branch.
  - Customer responses include `registrationBranch: { id, code, name } | null` and `operatingBranches: { customerId, branchId, branch: { id, code, name } }[]`, excluding soft-deleted branches.

- [ ] **Step 1: Extend `CreateCustomerDto`**

Replace `src/modules/master-data/dto/create-customer.dto.ts` with:

```ts
import {
  IsArray, IsBoolean, IsEmail, IsInt, IsNotEmpty, IsNumber, IsOptional,
  IsString, IsUUID, MaxLength, Min,
} from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { Decimal } from '@prisma/client/runtime/client';

export class CreateCustomerDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  code?: string;

  @ApiProperty()
  @IsNotEmpty()
  @IsString()
  name: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  contactPerson?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  phone?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsEmail()
  email?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  address?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  city?: string;

  @ApiPropertyOptional({ type: Number })
  @IsOptional()
  @Type(() => Number)
  creditLimit?: Decimal;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  performance?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  collateral?: string;

  @ApiPropertyOptional({ format: 'uuid', description: 'Branch the customer was registered at' })
  @IsOptional()
  @IsUUID()
  registrationBranchId?: string;

  @ApiPropertyOptional({ description: 'National ID (KTP) number' })
  @IsOptional()
  @IsString()
  @MaxLength(50)
  idCardNumber?: string;

  @ApiPropertyOptional({ description: 'Tax number (NPWP)' })
  @IsOptional()
  @IsString()
  @MaxLength(50)
  taxNumber?: string;

  @ApiPropertyOptional({ description: 'Plate number of the vehicle the customer collects with' })
  @IsOptional()
  @IsString()
  @MaxLength(50)
  vehiclePlate?: string;

  @ApiPropertyOptional({ description: 'Enforce the credit limit for this customer' })
  @IsOptional()
  @IsBoolean()
  creditLimitEnabled?: boolean;

  @ApiPropertyOptional({ description: 'Payment terms in days' })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  topDays?: number;

  @ApiPropertyOptional({ type: Number, description: 'Savings withheld per kg' })
  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  savingsPerKg?: number;

  @ApiPropertyOptional({ type: Number, description: 'Debt installment per kg (stored only, not yet applied)' })
  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  installmentPerKg?: number;

  @ApiPropertyOptional({ type: [String], format: 'uuid', description: 'Branches this customer may be ordered for' })
  @IsOptional()
  @IsArray()
  @IsUUID('4', { each: true })
  operatingBranchIds?: string[];
}
```

`UpdateCustomerDto` is `PartialType(CreateCustomerDto)` — confirm with `cat src/modules/master-data/dto/update-customer.dto.ts`; if so it picks the new fields up with no change.

- [ ] **Step 2: Create `QueryCustomerDto`**

Create `src/modules/master-data/dto/query-customer.dto.ts`:

```ts
import { IsOptional, IsUUID } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

export class QueryCustomerDto extends PaginationDto {
  @ApiPropertyOptional({ format: 'uuid', description: 'Only customers operating in this branch' })
  @IsOptional()
  @IsUUID()
  branchId?: string;
}
```

- [ ] **Step 3: Rewrite `CustomerService`**

Replace `src/modules/master-data/services/customer.service.ts` with:

```ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service.js';
import { CreateCustomerDto } from '../dto/create-customer.dto.js';
import { UpdateCustomerDto } from '../dto/update-customer.dto.js';
import { QueryCustomerDto } from '../dto/query-customer.dto.js';
import { PaginatedResult } from '../../../common/dto/pagination.dto.js';
import { ReferenceNumberGenerator } from '../../../common/utils/reference-number.generator.js';
import { Prisma } from '../../../../generated/prisma/client.js';

@Injectable()
export class CustomerService {
  /**
   * Soft-deleted branches are filtered out: the edit dialog seeds its
   * checkboxes from this payload, and a dead id sent back would fail
   * validation and make the customer uneditable.
   */
  private readonly include = {
    registrationBranch: { select: { id: true, code: true, name: true } },
    operatingBranches: {
      where: { branch: { deletedAt: null } },
      include: { branch: { select: { id: true, code: true, name: true } } },
    },
  } as const;

  constructor(
    private readonly prisma: PrismaService,
    private readonly refGen: ReferenceNumberGenerator,
  ) {}

  async create(dto: CreateCustomerDto, tenantId: string) {
    const { operatingBranchIds, registrationBranchId, ...data } = dto;
    return this.prisma.$transaction(async (tx) => {
      const branchIds = await this.validateBranchIds(tx, tenantId, operatingBranchIds);
      if (registrationBranchId) {
        await this.validateBranchIds(tx, tenantId, [registrationBranchId]);
      }
      const code = data.code?.trim() || (await this.refGen.generate(tx, tenantId, 'CUS'));
      return tx.customer.create({
        data: {
          ...data,
          code,
          tenantId,
          registrationBranchId,
          operatingBranches: { create: branchIds.map((branchId) => ({ branchId })) },
        },
        include: this.include,
      });
    });
  }

  async findAll(tenantId: string, query: QueryCustomerDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(query.search && {
        name: { contains: query.search, mode: 'insensitive' as const },
      }),
      ...(query.branchId && {
        operatingBranches: { some: { branchId: query.branchId } },
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.customer.findMany({
        where,
        include: this.include,
        skip: query.skip,
        take: query.limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.customer.count({ where }),
    ]);
    return new PaginatedResult(data, total, query.page, query.limit);
  }

  async findOne(id: string, tenantId: string) {
    const record = await this.prisma.customer.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!record) throw new NotFoundException('Customer not found');
    return record;
  }

  async update(id: string, dto: UpdateCustomerDto, tenantId: string) {
    await this.findOne(id, tenantId);
    const { operatingBranchIds, ...data } = dto;
    return this.prisma.$transaction(async (tx) => {
      if (data.registrationBranchId) {
        await this.validateBranchIds(tx, tenantId, [data.registrationBranchId]);
      }
      if (operatingBranchIds !== undefined) {
        const branchIds = await this.validateBranchIds(tx, tenantId, operatingBranchIds);
        await tx.customerBranch.deleteMany({ where: { customerId: id } });
        if (branchIds.length > 0) {
          await tx.customerBranch.createMany({
            data: branchIds.map((branchId) => ({ customerId: id, branchId })),
          });
        }
      }
      return tx.customer.update({ where: { id }, data, include: this.include });
    });
  }

  async remove(id: string, tenantId: string) {
    await this.findOne(id, tenantId);
    return this.prisma.customer.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }

  /** Returns the de-duplicated ids after checking every one belongs to the tenant. */
  private async validateBranchIds(
    tx: Prisma.TransactionClient,
    tenantId: string,
    branchIds: string[] | undefined,
  ): Promise<string[]> {
    const ids = Array.from(new Set(branchIds ?? []));
    if (ids.length === 0) return [];
    const found = await tx.branch.count({
      where: { id: { in: ids }, tenantId, deletedAt: null },
    });
    if (found !== ids.length) {
      throw new BadRequestException('One or more branch IDs are invalid');
    }
    return ids;
  }
}
```

The `Set` in `validateBranchIds` is what stops a duplicated branch id from reaching `createMany` and raising a composite-primary-key violation.

- [ ] **Step 4: Point the controller at `QueryCustomerDto`**

In `src/modules/master-data/controllers/customer.controller.ts`, swap the `PaginationDto` import for `QueryCustomerDto` and change `findAll`:

```ts
import { QueryCustomerDto } from '../dto/query-customer.dto.js';
// remove: import { PaginationDto } from '../../../common/dto/pagination.dto.js';

  @ApiOperation({ summary: 'List all customers' })
  @Get()
  findAll(@CurrentTenant() tenantId: string, @Query() query: QueryCustomerDto) {
    return this.customerService.findAll(tenantId, query);
  }
```

- [ ] **Step 5: Build**

```bash
cd breeding-app && npm run build
```

Expected: success. If `Prisma.TransactionClient` does not resolve, check the import path depth — from `src/modules/master-data/services/` it is `'../../../../generated/prisma/client.js'`.

- [ ] **Step 6: Verify with curl**

Start the API (`npm run start:dev`) if it is not already listening on 3002, then run this script. It covers the happy path plus the five Review Focus cases.

```bash
cd breeding-app && cat > /tmp/verify-customer.py <<'PYEOF'
import json, subprocess
API = "http://localhost:3002/api"

def curl(args):
    out = subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout
    return json.loads(out)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

branches = curl([API + "/branches?limit=10"] + H)["data"]["data"]
BR = branches[0]["id"]
print("branch:", branches[0]["name"])

def post(path, body):
    return curl(["-X", "POST", API + path] + H + ["-d", json.dumps(body)])

def patch(path, body):
    return curl(["-X", "PATCH", API + path] + H + ["-d", json.dumps(body)])

# 1. backward compatible: no new fields
r = post("/customers", {"name": "Cust Plain"})
print("1 plain create ->", r["data"]["name"], "| expected: Cust Plain")
plain_id = r["data"]["id"]

# 2. full create with duplicated branch id (Review Focus 2)
r = post("/customers", {"name": "Cust Full", "registrationBranchId": BR,
                        "idCardNumber": "3171", "taxNumber": "01.2", "vehiclePlate": "B 1 XY",
                        "creditLimitEnabled": True, "creditLimit": 1000000, "topDays": 5,
                        "savingsPerKg": 150, "installmentPerKg": 25,
                        "operatingBranchIds": [BR, BR]})
full = r["data"]
print("2 duplicate branch id ->", len(full["operatingBranches"]), "| expected: 1")
full_id = full["id"]

# 3. cross-tenant / unknown branch id (Review Focus 4)
r = post("/customers", {"name": "Cust Bad", "operatingBranchIds": ["00000000-0000-4000-8000-000000000000"]})
print("3 unknown branch ->", r.get("statusCode"), "| expected: 400")
r = post("/customers", {"name": "Cust Bad2", "registrationBranchId": "00000000-0000-4000-8000-000000000000"})
print("4 unknown registration branch ->", r.get("statusCode"), "| expected: 400")

# 4. negative numbers (Review Focus 5)
for field, value in [("topDays", -1), ("savingsPerKg", -5), ("installmentPerKg", -5)]:
    r = post("/customers", {"name": "Cust Neg", field: value})
    print(f"5 negative {field} ->", r.get("statusCode"), "| expected: 400")

# 5. branchId filter
r = curl([API + "/customers?branchId=" + BR] + H)["data"]
names = [c["name"] for c in r["data"]]
print("6 filter by branch ->", "Cust Full" in names, "and", "Cust Plain" not in names, "| expected: True and True")

# 6. replace-all semantics
r = patch("/customers/" + full_id, {"operatingBranchIds": []})
print("7 clear links ->", len(r["data"]["operatingBranches"]), "| expected: 0")
r = patch("/customers/" + full_id, {"phone": "0812"})
print("8 untouched when key omitted ->", len(r["data"]["operatingBranches"]), "| expected: 0")
r = patch("/customers/" + full_id, {"operatingBranchIds": [BR]})
print("9 restore link ->", len(r["data"]["operatingBranches"]), "| expected: 1")

# 7. soft-deleted branch must disappear from the payload (Review Focus 1)
r = post("/branches", {"code": "BR-TMP", "name": "Temp Branch"})
tmp = r["data"]["id"]
patch("/customers/" + full_id, {"operatingBranchIds": [BR, tmp]})
curl(["-X", "DELETE", API + "/branches/" + tmp] + H)
after = curl([API + "/customers/" + full_id] + H)["data"]
ids = [b["branchId"] for b in after["operatingBranches"]]
print("10 soft-deleted branch hidden ->", tmp not in ids, "| expected: True")
r = patch("/customers/" + full_id, {"operatingBranchIds": ids})
print("11 still editable after branch delete ->", r.get("statusCode", 200), "| expected: 200")

# cleanup
for cid in [plain_id, full_id]:
    curl(["-X", "DELETE", API + "/customers/" + cid] + H)
PYEOF
python3 /tmp/verify-customer.py
```

Expected: every line matches its stated expectation. If `POST /branches` needs fields beyond `code` and `name`, check `src/modules/branch/dto/create-branch.dto.ts` and add them.

- [ ] **Step 7: Commit**

```bash
cd breeding-app && git add src/modules/master-data && git commit -m "feat(master-data): customer sales fields, operating branch links and branch filter

Operating branches are replace-all on update, de-duplicated before insert,
and soft-deleted branches are excluded from the response so the edit
dialog cannot resend a dead id.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 3: Mature bird standard module

**Files:**
- Create: `breeding-app/src/modules/master-data/dto/create-mature-bird-standard.dto.ts`
- Create: `breeding-app/src/modules/master-data/dto/update-mature-bird-standard.dto.ts`
- Create: `breeding-app/src/modules/master-data/services/mature-bird-standard.service.ts`
- Create: `breeding-app/src/modules/master-data/controllers/mature-bird-standard.controller.ts`
- Modify: `breeding-app/src/modules/master-data/master-data.module.ts`

**Interfaces:**
- Consumes: Prisma delegates from Task 1.
- Produces: `/api/mature-bird-standards` with `POST`, `GET` (paginated, `search` on name), `GET /:id`, `PATCH /:id`, `DELETE /:id`. Body shape `{ name: string, ranges: { valueFrom: number, valueTo: number }[] }`. Responses include `ranges` ordered by `valueFrom` ascending.

- [ ] **Step 1: Create the DTOs**

`dto/create-mature-bird-standard.dto.ts`:

```ts
import { IsArray, IsNotEmpty, IsNumber, IsString, Min, ValidateNested, ArrayMinSize } from 'class-validator';
import { Type } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';

export class MatureBirdStandardRangeDto {
  @ApiProperty({ example: 1.2 })
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  valueFrom: number;

  @ApiProperty({ example: 1.8 })
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  valueTo: number;
}

export class CreateMatureBirdStandardDto {
  @ApiProperty()
  @IsNotEmpty()
  @IsString()
  name: string;

  @ApiProperty({ type: [MatureBirdStandardRangeDto] })
  @IsArray()
  @ArrayMinSize(1)
  @ValidateNested({ each: true })
  @Type(() => MatureBirdStandardRangeDto)
  ranges: MatureBirdStandardRangeDto[];
}
```

`dto/update-mature-bird-standard.dto.ts`:

```ts
import { PartialType } from '@nestjs/swagger';
import { CreateMatureBirdStandardDto } from './create-mature-bird-standard.dto.js';

export class UpdateMatureBirdStandardDto extends PartialType(CreateMatureBirdStandardDto) {}
```

- [ ] **Step 2: Create the service**

`services/mature-bird-standard.service.ts`:

```ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service.js';
import { CreateMatureBirdStandardDto, MatureBirdStandardRangeDto } from '../dto/create-mature-bird-standard.dto.js';
import { UpdateMatureBirdStandardDto } from '../dto/update-mature-bird-standard.dto.js';
import { PaginationDto, PaginatedResult } from '../../../common/dto/pagination.dto.js';

@Injectable()
export class MatureBirdStandardService {
  private readonly include = {
    ranges: { orderBy: { valueFrom: 'asc' } },
  } as const;

  constructor(private readonly prisma: PrismaService) {}

  async create(dto: CreateMatureBirdStandardDto, tenantId: string) {
    this.assertRangesValid(dto.ranges);
    return this.prisma.matureBirdStandard.create({
      data: {
        tenantId,
        name: dto.name,
        ranges: { create: dto.ranges.map((r) => ({ valueFrom: r.valueFrom, valueTo: r.valueTo })) },
      },
      include: this.include,
    });
  }

  async findAll(tenantId: string, pagination: PaginationDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(pagination.search && {
        name: { contains: pagination.search, mode: 'insensitive' as const },
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.matureBirdStandard.findMany({
        where,
        include: this.include,
        skip: pagination.skip,
        take: pagination.limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.matureBirdStandard.count({ where }),
    ]);
    return new PaginatedResult(data, total, pagination.page, pagination.limit);
  }

  async findOne(id: string, tenantId: string) {
    const record = await this.prisma.matureBirdStandard.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!record) throw new NotFoundException('Mature bird standard not found');
    return record;
  }

  async update(id: string, dto: UpdateMatureBirdStandardDto, tenantId: string) {
    await this.findOne(id, tenantId);
    if (dto.ranges !== undefined) this.assertRangesValid(dto.ranges);
    return this.prisma.$transaction(async (tx) => {
      if (dto.ranges !== undefined) {
        await tx.matureBirdStandardRange.deleteMany({ where: { standardId: id } });
        await tx.matureBirdStandardRange.createMany({
          data: dto.ranges.map((r) => ({ standardId: id, valueFrom: r.valueFrom, valueTo: r.valueTo })),
        });
      }
      return tx.matureBirdStandard.update({
        where: { id },
        data: { name: dto.name },
        include: this.include,
      });
    });
  }

  async remove(id: string, tenantId: string) {
    await this.findOne(id, tenantId);
    return this.prisma.matureBirdStandard.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }

  /** A range that ends below where it starts can never match a weight. */
  private assertRangesValid(ranges: MatureBirdStandardRangeDto[]) {
    if (ranges.length === 0) {
      throw new BadRequestException('At least one range is required');
    }
    for (const r of ranges) {
      if (r.valueTo < r.valueFrom) {
        throw new BadRequestException(`Range ${r.valueFrom}–${r.valueTo} ends below where it starts`);
      }
    }
  }
}
```

- [ ] **Step 3: Create the controller**

`controllers/mature-bird-standard.controller.ts`:

```ts
import { Controller, Get, Post, Patch, Delete, Body, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { MatureBirdStandardService } from '../services/mature-bird-standard.service.js';
import { CreateMatureBirdStandardDto } from '../dto/create-mature-bird-standard.dto.js';
import { UpdateMatureBirdStandardDto } from '../dto/update-mature-bird-standard.dto.js';
import { JwtAuthGuard } from '../../../common/guards/jwt-auth.guard.js';
import { CurrentTenant } from '../../../common/decorators/current-tenant.decorator.js';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

@ApiTags('Mature Bird Standard')
@ApiBearerAuth()
@Controller('mature-bird-standards')
@UseGuards(JwtAuthGuard)
export class MatureBirdStandardController {
  constructor(private readonly service: MatureBirdStandardService) {}

  @ApiOperation({ summary: 'Create a mature bird standard' })
  @Post()
  create(@Body() dto: CreateMatureBirdStandardDto, @CurrentTenant() tenantId: string) {
    return this.service.create(dto, tenantId);
  }

  @ApiOperation({ summary: 'List all mature bird standards' })
  @Get()
  findAll(@CurrentTenant() tenantId: string, @Query() pagination: PaginationDto) {
    return this.service.findAll(tenantId, pagination);
  }

  @ApiOperation({ summary: 'Get mature bird standard by ID' })
  @Get(':id')
  findOne(@Param('id') id: string, @CurrentTenant() tenantId: string) {
    return this.service.findOne(id, tenantId);
  }

  @ApiOperation({ summary: 'Update a mature bird standard' })
  @Patch(':id')
  update(@Param('id') id: string, @Body() dto: UpdateMatureBirdStandardDto, @CurrentTenant() tenantId: string) {
    return this.service.update(id, dto, tenantId);
  }

  @ApiOperation({ summary: 'Delete a mature bird standard' })
  @Delete(':id')
  remove(@Param('id') id: string, @CurrentTenant() tenantId: string) {
    return this.service.remove(id, tenantId);
  }
}
```

- [ ] **Step 4: Register in `MasterDataModule`**

In `src/modules/master-data/master-data.module.ts` add the two imports next to the `MatureBirdType` ones, then add `MatureBirdStandardController` to `controllers`, and `MatureBirdStandardService` to both `providers` and `exports`:

```ts
import { MatureBirdStandardService } from './services/mature-bird-standard.service.js';
import { MatureBirdStandardController } from './controllers/mature-bird-standard.controller.js';
```

- [ ] **Step 5: Build and verify with curl**

```bash
cd breeding-app && npm run build
```

Expected: success. Then, with the API listening on 3002:

```bash
cd breeding-app && cat > /tmp/verify-standard.py <<'PYEOF'
import json, subprocess
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

def post(body):
    return curl(["-X", "POST", API + "/mature-bird-standards"] + H + ["-d", json.dumps(body)])

# happy path, ranges come back sorted
r = post({"name": "Std Test", "ranges": [{"valueFrom": 1.8, "valueTo": 2.5}, {"valueFrom": 0.8, "valueTo": 1.2}]})
sid = r["data"]["id"]
print("1 created ->", [float(x["valueFrom"]) for x in r["data"]["ranges"]], "| expected: [0.8, 1.8]")

# Review Focus 3: empty ranges and inverted range
print("2 empty ranges ->", post({"name": "Std Empty", "ranges": []}).get("statusCode"), "| expected: 400")
print("3 inverted range ->", post({"name": "Std Inv", "ranges": [{"valueFrom": 2.0, "valueTo": 1.0}]}).get("statusCode"), "| expected: 400")

# replace-all on update
r = curl(["-X", "PATCH", API + "/mature-bird-standards/" + sid] + H +
         ["-d", json.dumps({"ranges": [{"valueFrom": 3.0, "valueTo": 4.0}]})])
print("4 replace ranges ->", len(r["data"]["ranges"]), "| expected: 1")
r = curl(["-X", "PATCH", API + "/mature-bird-standards/" + sid] + H + ["-d", json.dumps({"name": "Std Renamed"})])
print("5 rename keeps ranges ->", r["data"]["name"], len(r["data"]["ranges"]), "| expected: Std Renamed 1")
print("6 inverted on update ->", curl(["-X", "PATCH", API + "/mature-bird-standards/" + sid] + H +
      ["-d", json.dumps({"ranges": [{"valueFrom": 5.0, "valueTo": 1.0}]})]).get("statusCode"), "| expected: 400")

curl(["-X", "DELETE", API + "/mature-bird-standards/" + sid] + H)
print("7 soft-deleted ->", curl([API + "/mature-bird-standards/" + sid] + H).get("statusCode"), "| expected: 404")
PYEOF
python3 /tmp/verify-standard.py
```

Expected: every line matches its stated expectation.

- [ ] **Step 6: Commit**

```bash
cd breeding-app && git add src/modules/master-data && git commit -m "feat(master-data): add mature bird standard master with weight ranges

Ranges are replaced wholesale on update and rejected when empty or when
valueTo falls below valueFrom.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 4: Frontend types, i18n and `CustomerCombobox` prop

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts` (`Customer` ~line 171)
- Modify: `breeding-dashboard/messages/en.json`, `messages/id.json`
- Modify: `breeding-dashboard/src/components/forms/customer-combobox.tsx`

**Interfaces:**
- Produces:
  - `Customer` gains `code`, `city?`, `registrationBranchId?`, `registrationBranch?`, `idCardNumber?`, `taxNumber?`, `vehiclePlate?`, `creditLimitEnabled`, `topDays?`, `savingsPerKg?`, `installmentPerKg?`, `lastPaymentDate?`, `operatingBranches?: CustomerBranchLink[]`.
  - New types `CustomerBranchLink`, `MatureBirdStandard`, `MatureBirdStandardRange`.
  - `<CustomerCombobox value onChange disabled? branchId? />`.
  - i18n keys listed in Step 2.

- [ ] **Step 1: Types**

In `src/types/api.ts`, replace the `Customer` interface and add the new ones directly after it:

```ts
export interface CustomerBranchLink {
  customerId: string;
  branchId: string;
  branch: Pick<Branch, "id" | "code" | "name">;
}

export interface Customer {
  id: string;
  code: string;
  name: string;
  contactPerson?: string;
  phone?: string;
  email?: string;
  address?: string;
  city?: string;
  creditLimit?: string;
  creditLimitEnabled: boolean;
  performance?: string;
  collateral?: string;
  registrationBranchId?: string;
  registrationBranch?: Pick<Branch, "id" | "code" | "name">;
  idCardNumber?: string;
  taxNumber?: string;
  vehiclePlate?: string;
  topDays?: number;
  savingsPerKg?: string;
  installmentPerKg?: string;
  lastPaymentDate?: string;
  operatingBranches?: CustomerBranchLink[];
  createdAt: string;
  updatedAt: string;
}

export interface MatureBirdStandardRange {
  id: string;
  standardId: string;
  valueFrom: string;
  valueTo: string;
}

export interface MatureBirdStandard {
  id: string;
  name: string;
  ranges: MatureBirdStandardRange[];
  createdAt: string;
  updatedAt: string;
}
```

Confirm a `Branch` interface exists above this point with `id`, `code`, `name` — `grep -n "export interface Branch" src/types/api.ts`. If it lacks `code`, use `Pick<Branch, "id" | "name">` in both places instead and note it in the report.

- [ ] **Step 2: i18n**

In `messages/en.json`, inside `"customers"` add:

```json
    "code": "Customer code",
    "city": "City",
    "registrationBranch": "Registration area",
    "idCardNumber": "ID card no. (KTP)",
    "taxNumber": "Tax no. (NPWP)",
    "vehiclePlate": "Vehicle plate no.",
    "creditLimitEnabled": "Enforce credit limit",
    "topDays": "Payment terms (days)",
    "savingsPerKg": "Savings per kg",
    "installmentPerKg": "Installment per kg",
    "installmentHint": "Stored only — the deduction flow is not built yet.",
    "operatingBranches": "Operating areas",
    "noBranches": "No areas yet — create them under Branches first.",
    "groupIdentity": "Identity",
    "groupDocuments": "Documents",
    "groupContact": "Contact",
    "groupTerms": "Terms & conditions",
    "groupOperations": "Operations"
```

In `messages/id.json`, inside `"customers"`:

```json
    "code": "Kode customer",
    "city": "Kota",
    "registrationBranch": "Area pendaftaran",
    "idCardNumber": "No. KTP",
    "taxNumber": "NPWP",
    "vehiclePlate": "No. plat mobil",
    "creditLimitEnabled": "Aktifkan pembatasan plafon",
    "topDays": "TOP pembayaran (hari)",
    "savingsPerKg": "Tabungan per kg",
    "installmentPerKg": "Cicilan per kg",
    "installmentHint": "Baru disimpan — mekanisme pemotongan belum dibangun.",
    "operatingBranches": "Area operasional",
    "noBranches": "Belum ada area — buat dulu di menu Cabang.",
    "groupIdentity": "Identitas",
    "groupDocuments": "Dokumen",
    "groupContact": "Kontak",
    "groupTerms": "Syarat & ketentuan",
    "groupOperations": "Operasional"
```

Add a new top-level namespace to both files for the new master page. `en.json`:

```json
  "matureBirdStandards": {
    "title": "Mature Bird Standards",
    "description": "Weight ranges used as the minimum-weight reference on sales orders.",
    "entity": "standard",
    "name": "Standard name",
    "ranges": "Weight ranges",
    "valueFrom": "From",
    "valueTo": "To",
    "addRange": "Add range",
    "rangeRequired": "Add at least one range",
    "rangeInvalid": "\"To\" must not be lower than \"From\"",
    "searchPlaceholder": "Search standards..."
  },
```

`id.json`:

```json
  "matureBirdStandards": {
    "title": "Standarisasi Ayam Besar",
    "description": "Rentang bobot yang dipakai sebagai acuan bobot minimum di pesanan penjualan.",
    "entity": "standarisasi",
    "name": "Nama standarisasi",
    "ranges": "Rentang bobot",
    "valueFrom": "Dari",
    "valueTo": "Sampai",
    "addRange": "Tambah rentang",
    "rangeRequired": "Tambahkan minimal satu rentang",
    "rangeInvalid": "\"Sampai\" tidak boleh lebih kecil dari \"Dari\"",
    "searchPlaceholder": "Cari standarisasi..."
  },
```

Verify both files still parse:

```bash
cd breeding-dashboard && python3 -c "import json;[json.load(open(f)) for f in ['messages/en.json','messages/id.json']];print('json ok')"
```

- [ ] **Step 3: Add `branchId` to `CustomerCombobox`**

In `src/components/forms/customer-combobox.tsx`, extend the props and the fetch. The existing `useEffect` calls `fetchPaginated<Customer>("/customers", { limit: 50, search })`; change the interface, the signature and that call:

```tsx
interface CustomerComboboxProps {
  value: string;
  onChange: (id: string) => void;
  disabled?: boolean;
  /** Only list customers whose operating areas include this branch. */
  branchId?: string;
}

export function CustomerCombobox({ value, onChange, disabled, branchId }: CustomerComboboxProps) {
```

```tsx
    fetchPaginated<Customer>("/customers", {
      limit: 50,
      search,
      ...(branchId && { extra: { branchId } }),
    })
```

Add `branchId` to that `useEffect`'s dependency array alongside `search`.

- [ ] **Step 4: Type-check and build**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: clean. The customers page does not yet use the new fields, so nothing else breaks.

- [ ] **Step 5: Commit**

```bash
cd breeding-dashboard && git add src/types/api.ts messages src/components/forms/customer-combobox.tsx && git commit -m "feat(master-data): customer sales types, i18n keys and branch filter on CustomerCombobox

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 5: Customers page — grouped dialog and new columns

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/customers/page.tsx`

**Interfaces:**
- Consumes: types and i18n from Task 4; `POST/PATCH /customers` from Task 2; `GET /branches`.

- [ ] **Step 1: Imports and state**

Merge `useEffect` into the existing react import, and add:

```tsx
import { fetchPaginated } from "@/lib/api";
import { Checkbox } from "@/components/ui/checkbox";
import { Badge } from "@/components/ui/badge";
import { BranchCombobox } from "@/components/forms/branch-combobox";
import { Branch, Customer } from "@/types/api";
```

Replace the block of `form*` states with:

```tsx
  const [formCode, setFormCode] = useState("");
  const [formName, setFormName] = useState("");
  const [formRegistrationBranchId, setFormRegistrationBranchId] = useState("");
  const [formAddress, setFormAddress] = useState("");
  const [formCity, setFormCity] = useState("");
  const [formIdCardNumber, setFormIdCardNumber] = useState("");
  const [formTaxNumber, setFormTaxNumber] = useState("");
  const [formVehiclePlate, setFormVehiclePlate] = useState("");
  const [formContactPerson, setFormContactPerson] = useState("");
  const [formPhone, setFormPhone] = useState("");
  const [formEmail, setFormEmail] = useState("");
  const [formCreditLimitEnabled, setFormCreditLimitEnabled] = useState(false);
  const [formCreditLimit, setFormCreditLimit] = useState("");
  const [formTopDays, setFormTopDays] = useState("");
  const [formSavingsPerKg, setFormSavingsPerKg] = useState("");
  const [formInstallmentPerKg, setFormInstallmentPerKg] = useState("");
  const [formOperatingBranchIds, setFormOperatingBranchIds] = useState<string[]>([]);
  const [branches, setBranches] = useState<Branch[]>([]);
```

Load the branch list when the dialog opens, and add the toggle helper:

```tsx
  useEffect(() => {
    if (!dialogOpen) return;
    fetchPaginated<Branch>("/branches", { limit: 100 })
      .then((res) => setBranches(res.data))
      .catch(() => setBranches([]));
  }, [dialogOpen]);

  function toggleBranch(id: string, checked: boolean) {
    setFormOperatingBranchIds((prev) =>
      checked ? Array.from(new Set([...prev, id])) : prev.filter((b) => b !== id)
    );
  }
```

- [ ] **Step 2: Columns**

Replace the columns array with:

```tsx
  const columns: Column<Customer>[] = [
    {
      header: t('code'),
      cell: (row) => <span className="font-mono text-xs">{row.code}</span>,
      className: "w-[120px]",
    },
    {
      header: tc('name'),
      accessorKey: "name",
    },
    {
      header: t('registrationBranch'),
      cell: (row) => row.registrationBranch?.name || "-",
    },
    {
      header: t('operatingBranches'),
      cell: (row) =>
        row.operatingBranches && row.operatingBranches.length > 0 ? (
          <div className="flex flex-wrap gap-1">
            {row.operatingBranches.map((b) => (
              <Badge key={b.branchId} variant="secondary">{b.branch.name}</Badge>
            ))}
          </div>
        ) : (
          "-"
        ),
    },
    {
      header: t('creditLimit'),
      cell: (row) =>
        row.creditLimitEnabled && row.creditLimit
          ? Number(row.creditLimit).toLocaleString()
          : "-",
    },
    {
      header: tc('created'),
      cell: (row) => formatDate(row.createdAt),
      className: "w-[150px]",
    },
    {
      header: tc('actions'),
      cell: (row) => (
        <div className="flex items-center gap-1">
          <Button variant="ghost" size="sm" onClick={(e) => { e.stopPropagation(); handleEdit(row); }}>
            <Pencil className="h-4 w-4" />
          </Button>
          <Button variant="ghost" size="sm" onClick={(e) => { e.stopPropagation(); handleDeleteClick(row); }}>
            <Trash2 className="h-4 w-4 text-destructive" />
          </Button>
        </div>
      ),
      className: "w-[100px]",
    },
  ];
```

- [ ] **Step 3: `handleCreate`, `handleEdit`, `handleSubmit`**

```tsx
  function handleCreate() {
    setEditingCustomer(null);
    setFormCode(""); setFormName(""); setFormRegistrationBranchId("");
    setFormAddress(""); setFormCity("");
    setFormIdCardNumber(""); setFormTaxNumber(""); setFormVehiclePlate("");
    setFormContactPerson(""); setFormPhone(""); setFormEmail("");
    setFormCreditLimitEnabled(false); setFormCreditLimit("");
    setFormTopDays(""); setFormSavingsPerKg(""); setFormInstallmentPerKg("");
    setFormOperatingBranchIds([]);
    setDialogOpen(true);
  }

  function handleEdit(customer: Customer) {
    setEditingCustomer(customer);
    setFormCode(customer.code ?? "");
    setFormName(customer.name);
    setFormRegistrationBranchId(customer.registrationBranchId ?? "");
    setFormAddress(customer.address ?? "");
    setFormCity(customer.city ?? "");
    setFormIdCardNumber(customer.idCardNumber ?? "");
    setFormTaxNumber(customer.taxNumber ?? "");
    setFormVehiclePlate(customer.vehiclePlate ?? "");
    setFormContactPerson(customer.contactPerson ?? "");
    setFormPhone(customer.phone ?? "");
    setFormEmail(customer.email ?? "");
    setFormCreditLimitEnabled(customer.creditLimitEnabled ?? false);
    setFormCreditLimit(customer.creditLimit ?? "");
    setFormTopDays(customer.topDays != null ? String(customer.topDays) : "");
    setFormSavingsPerKg(customer.savingsPerKg ?? "");
    setFormInstallmentPerKg(customer.installmentPerKg ?? "");
    setFormOperatingBranchIds((customer.operatingBranches ?? []).map((b) => b.branchId));
    setDialogOpen(true);
  }

  async function handleSubmit() {
    if (!formName.trim()) {
      toast.error(tc('required', { field: tc('name') }));
      return;
    }

    setIsSubmitting(true);
    try {
      const body = {
        name: formName.trim(),
        ...(formCode.trim() && { code: formCode.trim() }),
        ...(formRegistrationBranchId && { registrationBranchId: formRegistrationBranchId }),
        ...(formAddress.trim() && { address: formAddress.trim() }),
        ...(formCity.trim() && { city: formCity.trim() }),
        ...(formIdCardNumber.trim() && { idCardNumber: formIdCardNumber.trim() }),
        ...(formTaxNumber.trim() && { taxNumber: formTaxNumber.trim() }),
        ...(formVehiclePlate.trim() && { vehiclePlate: formVehiclePlate.trim() }),
        ...(formContactPerson.trim() && { contactPerson: formContactPerson.trim() }),
        ...(formPhone.trim() && { phone: formPhone.trim() }),
        ...(formEmail.trim() && { email: formEmail.trim() }),
        creditLimitEnabled: formCreditLimitEnabled,
        ...(formCreditLimit.trim() && { creditLimit: formCreditLimit.trim() }),
        ...(formTopDays.trim() && { topDays: Number(formTopDays) }),
        ...(formSavingsPerKg.trim() && { savingsPerKg: Number(formSavingsPerKg) }),
        ...(formInstallmentPerKg.trim() && { installmentPerKg: Number(formInstallmentPerKg) }),
        operatingBranchIds: formOperatingBranchIds,
      };

      if (editingCustomer) {
        await fetchApi(`/customers/${editingCustomer.id}`, { method: "PATCH", body: JSON.stringify(body) });
        toast.success(tc('entityUpdated', { entity: t('entity') }));
      } else {
        await fetchApi("/customers", { method: "POST", body: JSON.stringify(body) });
        toast.success(tc('entityCreated', { entity: t('entity') }));
      }

      setDialogOpen(false);
      refetch();
    } catch (error) {
      toast.error(
        error instanceof Error
          ? error.message
          : editingCustomer
            ? tc('entityUpdateFailed', { entity: t('entity') })
            : tc('entityCreateFailed', { entity: t('entity') })
      );
    } finally {
      setIsSubmitting(false);
    }
  }
```

- [ ] **Step 4: Dialog body**

Replace the dialog's field container with the grouped layout. Widen the dialog so two columns fit: change `<DialogContent>` to `<DialogContent className="max-w-2xl max-h-[85vh] overflow-y-auto">`.

```tsx
          <div className="space-y-5">
            <section className="space-y-3">
              <h4 className="text-sm font-semibold text-muted-foreground">{t('groupIdentity')}</h4>
              <div className="grid gap-3 sm:grid-cols-2">
                <div className="space-y-2">
                  <Label htmlFor="customer-code">{t('code')}</Label>
                  <Input id="customer-code" placeholder={tc('enterFieldOptional', { field: t('code') })}
                    value={formCode} onChange={(e) => setFormCode(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-name">{tc('name')}</Label>
                  <Input id="customer-name" placeholder={tc('enterField', { field: tc('name') })}
                    value={formName} onChange={(e) => setFormName(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label>{t('registrationBranch')}</Label>
                  <BranchCombobox value={formRegistrationBranchId} onChange={setFormRegistrationBranchId} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-city">{t('city')}</Label>
                  <Input id="customer-city" placeholder={tc('enterFieldOptional', { field: t('city') })}
                    value={formCity} onChange={(e) => setFormCity(e.target.value)} />
                </div>
                <div className="space-y-2 sm:col-span-2">
                  <Label htmlFor="customer-address">{tc('address')}</Label>
                  <Input id="customer-address" placeholder={tc('enterFieldOptional', { field: tc('address') })}
                    value={formAddress} onChange={(e) => setFormAddress(e.target.value)} />
                </div>
              </div>
            </section>

            <section className="space-y-3">
              <h4 className="text-sm font-semibold text-muted-foreground">{t('groupDocuments')}</h4>
              <div className="grid gap-3 sm:grid-cols-3">
                <div className="space-y-2">
                  <Label htmlFor="customer-ktp">{t('idCardNumber')}</Label>
                  <Input id="customer-ktp" value={formIdCardNumber} onChange={(e) => setFormIdCardNumber(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-npwp">{t('taxNumber')}</Label>
                  <Input id="customer-npwp" value={formTaxNumber} onChange={(e) => setFormTaxNumber(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-plate">{t('vehiclePlate')}</Label>
                  <Input id="customer-plate" value={formVehiclePlate} onChange={(e) => setFormVehiclePlate(e.target.value)} />
                </div>
              </div>
            </section>

            <section className="space-y-3">
              <h4 className="text-sm font-semibold text-muted-foreground">{t('groupContact')}</h4>
              <div className="grid gap-3 sm:grid-cols-3">
                <div className="space-y-2">
                  <Label htmlFor="customer-contact">{t('contactPerson')}</Label>
                  <Input id="customer-contact" value={formContactPerson} onChange={(e) => setFormContactPerson(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-phone">{tc('phone')}</Label>
                  <Input id="customer-phone" value={formPhone} onChange={(e) => setFormPhone(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-email">{tc('email')}</Label>
                  <Input id="customer-email" type="email" value={formEmail} onChange={(e) => setFormEmail(e.target.value)} />
                </div>
              </div>
            </section>

            <section className="space-y-3">
              <h4 className="text-sm font-semibold text-muted-foreground">{t('groupTerms')}</h4>
              <div className="flex items-center gap-2">
                <Checkbox id="customer-plafon" checked={formCreditLimitEnabled}
                  onCheckedChange={(v) => setFormCreditLimitEnabled(v === true)} />
                <Label htmlFor="customer-plafon" className="font-normal">{t('creditLimitEnabled')}</Label>
              </div>
              <div className="grid gap-3 sm:grid-cols-2">
                <div className="space-y-2">
                  <Label htmlFor="customer-credit-limit">{t('creditLimit')}</Label>
                  <Input id="customer-credit-limit" type="number" min={0} disabled={!formCreditLimitEnabled}
                    value={formCreditLimit} onChange={(e) => setFormCreditLimit(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-top">{t('topDays')}</Label>
                  <Input id="customer-top" type="number" min={0}
                    value={formTopDays} onChange={(e) => setFormTopDays(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-savings">{t('savingsPerKg')}</Label>
                  <Input id="customer-savings" type="number" min={0}
                    value={formSavingsPerKg} onChange={(e) => setFormSavingsPerKg(e.target.value)} />
                </div>
                <div className="space-y-2">
                  <Label htmlFor="customer-installment">{t('installmentPerKg')}</Label>
                  <Input id="customer-installment" type="number" min={0}
                    value={formInstallmentPerKg} onChange={(e) => setFormInstallmentPerKg(e.target.value)} />
                  <p className="text-xs text-muted-foreground">{t('installmentHint')}</p>
                </div>
              </div>
            </section>

            <section className="space-y-3">
              <h4 className="text-sm font-semibold text-muted-foreground">{t('groupOperations')}</h4>
              <Label>{t('operatingBranches')}</Label>
              {branches.length === 0 ? (
                <p className="text-sm text-muted-foreground">{t('noBranches')}</p>
              ) : (
                <div className="grid grid-cols-2 gap-2 rounded-md border p-3">
                  {branches.map((b) => (
                    <div key={b.id} className="flex items-center gap-2">
                      <Checkbox id={`customer-branch-${b.id}`}
                        checked={formOperatingBranchIds.includes(b.id)}
                        onCheckedChange={(v) => toggleBranch(b.id, v === true)} />
                      <Label htmlFor={`customer-branch-${b.id}`} className="font-normal">{b.name}</Label>
                    </div>
                  ))}
                </div>
              )}
            </section>
          </div>
```

- [ ] **Step 5: Type-check, build, browser smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: clean. Then with backend and `npm run dev` running, open `/customers`:

1. Create `CUS-TEST` / "Test Customer", pick a registration area, tick two operating areas, leave the credit-limit checkbox off — confirm the Limit Plafon input is disabled.
2. Save. The row shows the code, registration area and two badges.
3. Edit it, tick the credit-limit checkbox, enter `1000000`, save, reopen — every value comes back including the two ticked areas.
4. Untick both areas, save, reopen — no badges, no error.

- [ ] **Step 6: Commit**

```bash
cd breeding-dashboard && git add "src/app/(dashboard)/customers/page.tsx" && git commit -m "feat(master-data): grouped customer dialog with sales fields and operating areas

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 6: Mature bird standards page

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/mature-bird-standards/page.tsx`
- Modify: `breeding-dashboard/src/lib/constants.ts` (Master Data nav group, ~line 71)
- Modify: `breeding-dashboard/src/components/layout/breadcrumbs.tsx` (~line 20)

**Interfaces:**
- Consumes: `MatureBirdStandard`, `MatureBirdStandardRange` and the `matureBirdStandards` i18n namespace from Task 4; `/api/mature-bird-standards` from Task 3.

- [ ] **Step 1: Create the page**

Create `src/app/(dashboard)/mature-bird-standards/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useQueryState, parseAsInteger } from "nuqs";
import { toast } from "sonner";
import { Plus, Pencil, Trash2, X } from "lucide-react";
import { useTranslations } from "next-intl";

import { DataTable, Column } from "@/components/shared/data-table";
import { PageHeader } from "@/components/shared/page-header";
import { ConfirmDialog } from "@/components/shared/confirm-dialog";
import { usePaginated } from "@/hooks/use-api";
import { fetchApi } from "@/lib/api";
import { formatDate } from "@/lib/utils";
import { MatureBirdStandard } from "@/types/api";

import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Badge } from "@/components/ui/badge";
import {
  Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle,
} from "@/components/ui/dialog";

interface RangeDraft {
  valueFrom: string;
  valueTo: string;
}

export default function MatureBirdStandardsPage() {
  const t = useTranslations('matureBirdStandards');
  const tc = useTranslations('common');

  const [page, setPage] = useQueryState("page", parseAsInteger.withDefault(1));
  const [search, setSearch] = useQueryState("search", { defaultValue: "" });

  const { data: standards, meta, isLoading, refetch } = usePaginated<MatureBirdStandard>(
    "/mature-bird-standards",
    { page, limit: 10, search }
  );

  const [dialogOpen, setDialogOpen] = useState(false);
  const [editing, setEditing] = useState<MatureBirdStandard | null>(null);
  const [formName, setFormName] = useState("");
  const [formRanges, setFormRanges] = useState<RangeDraft[]>([{ valueFrom: "", valueTo: "" }]);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const [deleteDialogOpen, setDeleteDialogOpen] = useState(false);
  const [deleting, setDeleting] = useState<MatureBirdStandard | null>(null);

  const columns: Column<MatureBirdStandard>[] = [
    { header: t('name'), accessorKey: "name" },
    {
      header: t('ranges'),
      cell: (row) => (
        <div className="flex flex-wrap gap-1">
          {row.ranges.map((r) => (
            <Badge key={r.id} variant="secondary">
              {Number(r.valueFrom)} – {Number(r.valueTo)}
            </Badge>
          ))}
        </div>
      ),
    },
    { header: tc('created'), cell: (row) => formatDate(row.createdAt), className: "w-[150px]" },
    {
      header: tc('actions'),
      cell: (row) => (
        <div className="flex items-center gap-1">
          <Button variant="ghost" size="sm" onClick={(e) => { e.stopPropagation(); handleEdit(row); }}>
            <Pencil className="h-4 w-4" />
          </Button>
          <Button variant="ghost" size="sm" onClick={(e) => { e.stopPropagation(); setDeleting(row); setDeleteDialogOpen(true); }}>
            <Trash2 className="h-4 w-4 text-destructive" />
          </Button>
        </div>
      ),
      className: "w-[100px]",
    },
  ];

  function handleCreate() {
    setEditing(null);
    setFormName("");
    setFormRanges([{ valueFrom: "", valueTo: "" }]);
    setDialogOpen(true);
  }

  function handleEdit(row: MatureBirdStandard) {
    setEditing(row);
    setFormName(row.name);
    setFormRanges(
      row.ranges.length > 0
        ? row.ranges.map((r) => ({ valueFrom: String(Number(r.valueFrom)), valueTo: String(Number(r.valueTo)) }))
        : [{ valueFrom: "", valueTo: "" }]
    );
    setDialogOpen(true);
  }

  function updateRange(index: number, key: keyof RangeDraft, value: string) {
    setFormRanges((prev) => prev.map((r, i) => (i === index ? { ...r, [key]: value } : r)));
  }

  async function handleSubmit() {
    if (!formName.trim()) {
      toast.error(tc('required', { field: t('name') }));
      return;
    }
    const filled = formRanges.filter((r) => r.valueFrom.trim() && r.valueTo.trim());
    if (filled.length === 0) {
      toast.error(t('rangeRequired'));
      return;
    }
    if (filled.some((r) => Number(r.valueTo) < Number(r.valueFrom))) {
      toast.error(t('rangeInvalid'));
      return;
    }

    setIsSubmitting(true);
    try {
      const body = {
        name: formName.trim(),
        ranges: filled.map((r) => ({ valueFrom: Number(r.valueFrom), valueTo: Number(r.valueTo) })),
      };
      if (editing) {
        await fetchApi(`/mature-bird-standards/${editing.id}`, { method: "PATCH", body: JSON.stringify(body) });
        toast.success(tc('entityUpdated', { entity: t('entity') }));
      } else {
        await fetchApi("/mature-bird-standards", { method: "POST", body: JSON.stringify(body) });
        toast.success(tc('entityCreated', { entity: t('entity') }));
      }
      setDialogOpen(false);
      refetch();
    } catch (error) {
      toast.error(
        error instanceof Error
          ? error.message
          : editing
            ? tc('entityUpdateFailed', { entity: t('entity') })
            : tc('entityCreateFailed', { entity: t('entity') })
      );
    } finally {
      setIsSubmitting(false);
    }
  }

  async function handleDelete() {
    if (!deleting) return;
    try {
      await fetchApi(`/mature-bird-standards/${deleting.id}`, { method: "DELETE" });
      toast.success(tc('entityDeleted', { entity: t('entity') }));
      setDeleteDialogOpen(false);
      setDeleting(null);
      refetch();
    } catch {
      toast.error(tc('entityDeleteFailed', { entity: t('entity') }));
    }
  }

  return (
    <div className="space-y-6">
      <PageHeader
        title={t('title')}
        description={t('description')}
        actions={
          <Button onClick={handleCreate}>
            <Plus className="mr-2 h-4 w-4" />
            {tc('newEntity', { entity: t('entity') })}
          </Button>
        }
      />

      <DataTable
        columns={columns}
        data={standards}
        isLoading={isLoading}
        search={search}
        onSearchChange={(value) => { setSearch(value); setPage(1); }}
        searchPlaceholder={t('searchPlaceholder')}
        page={page}
        totalPages={meta?.totalPages || 1}
        onPageChange={setPage}
        total={meta?.total}
        emptyTitle={tc('noResults', { entity: t('title') })}
        emptyDescription={tc('getStartedAlt', { entity: t('entity') })}
        emptyAction={
          <Button onClick={handleCreate}>
            <Plus className="mr-2 h-4 w-4" />
            {tc('newEntity', { entity: t('entity') })}
          </Button>
        }
      />

      <Dialog open={dialogOpen} onOpenChange={setDialogOpen}>
        <DialogContent>
          <DialogHeader>
            <DialogTitle>
              {editing ? tc('editEntity', { entity: t('entity') }) : tc('newEntity', { entity: t('entity') })}
            </DialogTitle>
            <DialogDescription>
              {editing ? tc('updateDetails', { entity: t('entity') }) : tc('fillDetailsAlt', { entity: t('entity') })}
            </DialogDescription>
          </DialogHeader>

          <div className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="standard-name">{t('name')}</Label>
              <Input id="standard-name" value={formName} onChange={(e) => setFormName(e.target.value)} />
            </div>

            <div className="space-y-2">
              <Label>{t('ranges')}</Label>
              <div className="space-y-2">
                {formRanges.map((r, i) => (
                  <div key={i} className="flex items-end gap-2">
                    <div className="flex-1 space-y-1">
                      <Label className="text-xs" htmlFor={`range-from-${i}`}>{t('valueFrom')}</Label>
                      <Input id={`range-from-${i}`} type="number" min={0} step="0.01"
                        value={r.valueFrom} onChange={(e) => updateRange(i, "valueFrom", e.target.value)} />
                    </div>
                    <div className="flex-1 space-y-1">
                      <Label className="text-xs" htmlFor={`range-to-${i}`}>{t('valueTo')}</Label>
                      <Input id={`range-to-${i}`} type="number" min={0} step="0.01"
                        value={r.valueTo} onChange={(e) => updateRange(i, "valueTo", e.target.value)} />
                    </div>
                    <Button variant="ghost" size="sm" disabled={formRanges.length === 1}
                      onClick={() => setFormRanges((prev) => prev.filter((_, idx) => idx !== i))}>
                      <X className="h-4 w-4" />
                    </Button>
                  </div>
                ))}
              </div>
              <Button variant="outline" size="sm"
                onClick={() => setFormRanges((prev) => [...prev, { valueFrom: "", valueTo: "" }])}>
                <Plus className="mr-2 h-4 w-4" />
                {t('addRange')}
              </Button>
            </div>
          </div>

          <DialogFooter>
            <Button variant="outline" onClick={() => setDialogOpen(false)} disabled={isSubmitting}>
              {tc('cancel')}
            </Button>
            <Button onClick={handleSubmit} disabled={isSubmitting}>
              {isSubmitting ? tc('saving') : editing ? tc('updateEntity', { entity: t('entity') }) : tc('createEntity', { entity: t('entity') })}
            </Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>

      <ConfirmDialog
        open={deleteDialogOpen}
        onOpenChange={setDeleteDialogOpen}
        title={tc('deleteEntity', { entity: t('entity') })}
        description={tc('confirmDelete', { name: deleting?.name ?? '' })}
        onConfirm={handleDelete}
        variant="destructive"
        confirmLabel={tc('delete')}
      />
    </div>
  );
}
```

- [ ] **Step 2: Sidebar and breadcrumb**

In `src/lib/constants.ts`, in the Master Data group right after the `Mature Bird Types` entry, add:

```ts
      { title: "Mature Bird Standards", url: "/mature-bird-standards", icon: Target },
```

`Target` is already imported for the neighbouring entry — confirm with `grep -n "Target" src/lib/constants.ts`.

In `src/components/layout/breadcrumbs.tsx`, after the `"mature-bird-types"` line add:

```ts
  "mature-bird-standards": "Mature Bird Standards",
```

- [ ] **Step 3: Type-check, build, browser smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: clean. Then in the browser, open `/mature-bird-standards`:

1. The seeded "Standar Broiler" row shows three badges.
2. Create a new standard with two ranges — saves, badges appear sorted ascending.
3. Edit it, remove one range with the X button, save, reopen — one range remains.
4. Enter a range whose "To" is below "From" and save — a toast appears and the dialog stays open.
5. Delete the standard you created — it disappears from the list.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard && git add "src/app/(dashboard)/mature-bird-standards" src/lib/constants.ts src/components/layout/breadcrumbs.tsx && git commit -m "feat(master-data): add mature bird standards page with range editor

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Task 7: Update the finding doc

**Files:**
- Modify: `docs/superpowers/specs/2026-09-13-sales-module-finding.md`
- Modify: `docs/superpowers/specs/2026-09-26-sales-master-data-design.md`

- [ ] **Step 1: Mark the spec implemented**

In `2026-09-26-sales-master-data-design.md`, change `**Status:** Design` to:

```markdown
**Status:** Implemented (2026-09-26) — lihat plan `docs/superpowers/plans/2026-09-26-sales-master-data.md`
```

- [ ] **Step 2: Update the stage table**

In `2026-09-13-sales-module-finding.md`, in the table under "Opsi solusi", change the **S-A** and **S-B2** rows' Status column from `Siap` to `Selesai (2026-09-26)`.

- [ ] **Step 3: Commit**

```bash
cd /Users/alva.e202511001/Desktop/project/breeding && git add docs/superpowers/specs && git commit -m "docs(sales): mark master data spec implemented

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Self-review

**Spec coverage**

| Spec section | Task |
|---|---|
| §1 `Customer` new columns | 1 |
| §1 `CustomerBranch` | 1 |
| §1 `MatureBirdStandard` + ranges | 1 |
| §1 migration (additive, no backfill) | 1 |
| §2 Customer DTO fields | 2 |
| §2 create/update branch validation + replace-all | 2 |
| §2 `branchId` filter | 2 |
| §2 includes with soft-delete guard | 2 |
| §2 `MatureBirdStandardService` + module wiring | 3 |
| §2 seed | 1 |
| §3 customer dialog grouped in 5 sections | 5 |
| §3 customer table columns | 5 |
| §3 `CustomerCombobox` `branchId` prop | 4 |
| §3 `/mature-bird-standards` page + nav | 6 |
| §3 types & i18n | 4 |
| §4 backend curl checks | 2 (Step 6), 3 (Step 5) |
| §4 frontend browser checks | 5 (Step 5), 6 (Step 3) |
| §5 implementation order | task order 1→6 |
| Non-goal: no installment logic | enforced — `installmentPerKg` is only written and displayed, with a hint in the UI |

**Type consistency:** `operatingBranchIds` (request) vs `operatingBranches` (response) used consistently in Tasks 2, 4, 5. `CustomerBranchLink.branchId` used in Task 5's badge key and in `handleEdit`. `MatureBirdStandardRangeDto` named the same in Tasks 3's DTO file and service import. `valueFrom`/`valueTo` spelled identically in schema, DTO, service, types and page.

**Review Focus coverage:** items 1, 2, 4 and 5 are exercised by Task 2 Step 6; item 3 by Task 3 Step 5.
