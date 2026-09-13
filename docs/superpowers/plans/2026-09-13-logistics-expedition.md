# Logistics Expedition (Supplier Classification · Goods Transfer Dispatch · Goods Receipt Carrier) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let suppliers be classified by product category (with an "expedition" category marker), record carrier + delivery-note number on goods transfers (with a `PREPARING → dispatch → IN_TRANSIT` lifecycle and per-line BPB reference) and on goods receipts (header only).

**Architecture:** Three phases, each independently shippable. Phase A adds `ProductCategory.isExpedition` and a `SupplierCategory` join table, exposes `GET /suppliers?expedition=true`, and a reusable `SupplierService.assertExpeditionSupplier()`. Phase B adds `PREPARING` status, carrier/delivery-note/dispatch columns on `GoodsTransfer`, a `POST /goods-transfers/:id/dispatch` endpoint guarded to MANAGER+, and the BPB reference on transfer lines. Phase C adds carrier/delivery-note columns on `GoodsReceipt`. No stock movement is introduced anywhere.

**Tech Stack:** NestJS 11 + Prisma 7 + class-validator (backend, `breeding-app/`), Next.js 16 + shadcn/ui + next-intl (frontend, `breeding-dashboard/`), PostgreSQL.

**Spec:** `docs/superpowers/specs/2026-09-13-logistics-expedition-design.md`

**Testing note:** The backend has no service-level test suite (only `app.controller.spec.ts`). Following established practice in this repo (see `docs/superpowers/plans/2026-04-19-goods-consumption-receipt-reference.md`), verification is `npm run build` + `curl` against the dev server with seeded data, and `npx tsc --noEmit && npm run build` + browser smoke for the frontend. Every backend task ends with a build; every phase ends with curl verification.

## Global Constraints

- All Prisma queries filter by `tenantId` (CLAUDE.md).
- Soft delete via `deletedAt`; never hard-delete.
- API responses are wrapped `{ data, statusCode, timestamp }` by `TransformInterceptor` — `curl` output below shows the `data` part only.
- `ValidationPipe` runs with `whitelist: true, forbidNonWhitelisted: true, transform: true` (`breeding-app/src/main.ts:17`) — any request property not declared on a DTO returns 400.
- Prisma client is generated to `breeding-app/generated/prisma/client.js`; import enums from `'../../../generated/prisma/client.js'` (adjust depth per file). Run `npx prisma generate` after every schema change.
- i18n keys go in **both** `breeding-dashboard/messages/en.json` and `messages/id.json`.
- Frontend API types live in `breeding-dashboard/src/types/api.ts`.
- Roles: `SUPER_ADMIN`, `TENANT_ADMIN`, `MANAGER`, `STAFF`. Dispatch requires MANAGER or above.
- `deliveryNoteNumber`: manual input, optional, not unique, max 100 chars (spec "Asumsi terbuka").
- **No stock movement** for goods transfer in this plan (spec Non-goal).
- Build check backend before every commit: `cd breeding-app && npm run build`.
- Each sub-repo is its own git repo. `cd` into `breeding-app/` or `breeding-dashboard/` before `git` commands. Commit style: `feat(master-data):`, `feat(transfer):`, `feat(procurement):`, `fix(...)`.
- Dev backend on `http://localhost:3002/api`; seeded logins `admin@demo.farm` (TENANT_ADMIN), `manager@demo.farm` (MANAGER), `staff@demo.farm` (STAFF), password `password123`.

---

## File Structure

### Backend (`breeding-app/`)

| Action | Path | Responsibility |
|---|---|---|
| Modify | `prisma/schema.prisma` | `ProductCategory.isExpedition`, `SupplierCategory` model, `TransferStatus.PREPARING`, new columns on `GoodsTransfer`, `GoodsTransferLine`, `GoodsReceipt`, back-relations on `Supplier`, `User`, `GoodsReceiptLine` |
| Create (generated) | `prisma/migrations/<ts>_add_supplier_categories/`, `<ts>_add_goods_transfer_expedition/`, `<ts>_add_goods_receipt_carrier/` | one migration per phase |
| Modify | `prisma/seed.ts` | expedition category, 2 expedition suppliers, classify existing suppliers |
| Modify | `src/modules/inventory/dto/create-product-category.dto.ts` | `isExpedition?` |
| Modify | `src/modules/inventory/services/product-category.service.ts` | persist `isExpedition` on create |
| Modify | `src/modules/master-data/dto/create-supplier.dto.ts` | `categoryIds?` |
| Create | `src/modules/master-data/dto/query-supplier.dto.ts` | `QuerySupplierDto extends PaginationDto` with `expedition?` |
| Modify | `src/modules/master-data/services/supplier.service.ts` | category writes, `expedition` filter, includes, `assertExpeditionSupplier()` |
| Modify | `src/modules/master-data/controllers/supplier.controller.ts` | use `QuerySupplierDto` |
| Modify | `src/common/constants/status-transitions.constant.ts` | `PREPARING` row |
| Modify | `src/modules/transfer/dto/create-goods-transfer.dto.ts` | `carrierSupplierId?`, `deliveryNoteNumber?`, line `goodsReceiptLineId?` |
| Create | `src/modules/transfer/dto/dispatch-goods-transfer.dto.ts` | `DispatchGoodsTransferDto` |
| Create | `src/modules/transfer/dto/query-goods-transfer.dto.ts` | `QueryGoodsTransferDto extends PaginationDto` with `status?` |
| Modify | `src/modules/transfer/goods-transfer.service.ts` | create/update validation, `dispatch()`, status guard, includes, status filter |
| Modify | `src/modules/transfer/goods-transfer.controller.ts` | `POST :id/dispatch` (MANAGER+), `QueryGoodsTransferDto` |
| Modify | `src/modules/transfer/transfer.module.ts` | import `MasterDataModule` |
| Modify | `src/modules/procurement/dto/create-goods-receipt.dto.ts` | `carrierSupplierId?`, `deliveryNoteNumber?` |
| Modify | `src/modules/procurement/goods-receipt.service.ts` | validate + persist + include carrier |
| Modify | `src/modules/procurement/procurement.module.ts` | import `MasterDataModule` |

### Frontend (`breeding-dashboard/`)

| Action | Path | Responsibility |
|---|---|---|
| Modify | `src/types/api.ts` | `ProductCategory.isExpedition`, `Supplier.code/categories`, `TransferStatus` + `PREPARING`, `GoodsTransfer`/`GoodsTransferLine`/`GoodsReceipt` new fields |
| Modify | `src/lib/constants.ts` | `TRANSFER_STATUS_TRANSITIONS.PREPARING` |
| Modify | `messages/en.json`, `messages/id.json` | new keys (listed per task) |
| Modify | `src/app/(dashboard)/product-categories/page.tsx` | `isExpedition` checkbox + column |
| Modify | `src/app/(dashboard)/suppliers/page.tsx` | `code` field (bug fix), category checkboxes, category badges; remove fields backend doesn't accept |
| Modify | `src/components/forms/supplier-combobox.tsx` | `expeditionOnly` prop |
| Modify | `src/app/(dashboard)/goods-transfers/new/page.tsx` | carrier + delivery note inputs, BPB refs on lines |
| Create | `src/app/(dashboard)/goods-transfers/[id]/dispatch-dialog.tsx` | dispatch modal |
| Modify | `src/app/(dashboard)/goods-transfers/[id]/page.tsx` | expedition card, dispatch button, BPB column |
| Modify | `src/app/(dashboard)/goods-transfers/page.tsx` | status filter |
| Modify | `src/app/(dashboard)/goods-receipts/new/page.tsx` | carrier + delivery note inputs |
| Modify | `src/app/(dashboard)/goods-receipts/[id]/page.tsx` | show carrier + delivery note |

---

# Phase A — Master data: supplier classification

### Task 1: Prisma schema + migration for `SupplierCategory` and `ProductCategory.isExpedition`

**Files:**
- Modify: `breeding-app/prisma/schema.prisma:442-458` (ProductCategory), `:566-584` (Supplier)
- Create (generated): `breeding-app/prisma/migrations/<timestamp>_add_supplier_categories/migration.sql`

**Interfaces:**
- Produces: Prisma models `productCategory.isExpedition: boolean`, `supplierCategory { supplierId, categoryId }`, relations `supplier.categories`, `productCategory.supplierCategories`.

- [ ] **Step 1: Edit `ProductCategory` model**

In `breeding-app/prisma/schema.prisma`, replace the `ProductCategory` model body so it reads:

```prisma
model ProductCategory {
  id               String    @id @default(uuid())
  tenantId         String    @map("tenant_id")
  name             String
  purchasePurpose  String?   @map("purchase_purpose")
  overheadCategory String?   @map("overhead_category")
  priceType        String?   @map("price_type")
  isExpedition     Boolean   @default(false) @map("is_expedition")
  createdAt        DateTime  @default(now()) @map("created_at")
  updatedAt        DateTime  @updatedAt @map("updated_at")
  deletedAt        DateTime? @map("deleted_at")

  products           Product[]
  purchaseOrders     PurchaseOrder[]
  supplierCategories SupplierCategory[]

  @@unique([tenantId, name])
  @@map("product_categories")
}
```

- [ ] **Step 2: Edit `Supplier` model and add `SupplierCategory`**

Replace the `Supplier` model and add the join model directly after it:

```prisma
model Supplier {
  id        String    @id @default(uuid())
  tenantId  String    @map("tenant_id")
  code      String
  name      String
  address   String?
  status    String?
  createdAt DateTime  @default(now()) @map("created_at")
  updatedAt DateTime  @updatedAt @map("updated_at")
  deletedAt DateTime? @map("deleted_at")

  products       Product[]
  purchaseOrders PurchaseOrder[]
  goodsReceipts  GoodsReceipt[]
  goodsReturns   GoodsReturn[]
  categories     SupplierCategory[]

  @@unique([tenantId, code])
  @@map("suppliers")
}

model SupplierCategory {
  supplierId String @map("supplier_id")
  categoryId String @map("category_id")

  supplier Supplier        @relation(fields: [supplierId], references: [id])
  category ProductCategory @relation(fields: [categoryId], references: [id])

  @@id([supplierId, categoryId])
  @@index([categoryId])
  @@map("supplier_categories")
}
```

- [ ] **Step 3: Generate migration and client**

```bash
cd breeding-app && npx prisma migrate dev --name add_supplier_categories && npx prisma generate
```

Expected: a new folder under `prisma/migrations/` containing `ALTER TABLE "product_categories" ADD COLUMN "is_expedition" BOOLEAN NOT NULL DEFAULT false;` and `CREATE TABLE "supplier_categories" (...)`. No data loss warning.

- [ ] **Step 4: Build**

```bash
cd breeding-app && npm run build
```

Expected: success (no code references the new fields yet).

- [ ] **Step 5: Commit**

```bash
cd breeding-app && git add prisma/schema.prisma prisma/migrations && git commit -m "feat(master-data): add supplier category classification and expedition marker

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: `ProductCategory` DTO + service accept `isExpedition`

**Files:**
- Modify: `breeding-app/src/modules/inventory/dto/create-product-category.dto.ts`
- Modify: `breeding-app/src/modules/inventory/services/product-category.service.ts:51-61`

**Interfaces:**
- Produces: `POST/PATCH /product-categories` accept `isExpedition?: boolean`; responses include `isExpedition`.

- [ ] **Step 1: Add `isExpedition` to the create DTO**

Replace the file content with:

```ts
import { IsString, IsOptional, IsBoolean } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateProductCategoryDto {
  @ApiProperty()
  @IsString()
  name: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  purchasePurpose?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  overheadCategory?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  priceType?: string;

  @ApiPropertyOptional({ description: 'Marks this category as expedition / freight service' })
  @IsOptional()
  @IsBoolean()
  isExpedition?: boolean;
}
```

`UpdateProductCategoryDto` is `PartialType(CreateProductCategoryDto)` and picks the field up automatically; the service's `update` spreads `dto`, so no change there.

- [ ] **Step 2: Persist on create**

In `product-category.service.ts`, inside `create()`, add the field to `data`:

```ts
  async create(tenantId: string, dto: CreateProductCategoryDto) {
    return this.prisma.productCategory.create({
      data: {
        tenantId,
        name: dto.name,
        purchasePurpose: dto.purchasePurpose,
        overheadCategory: dto.overheadCategory,
        priceType: dto.priceType,
        isExpedition: dto.isExpedition ?? false,
      },
    });
  }
```

- [ ] **Step 3: Build**

```bash
cd breeding-app && npm run build
```

Expected: success.

- [ ] **Step 4: Commit**

```bash
cd breeding-app && git add src/modules/inventory && git commit -m "feat(inventory): accept isExpedition on product category

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: Supplier DTOs, service (categories, expedition filter, assert helper), controller

**Files:**
- Modify: `breeding-app/src/modules/master-data/dto/create-supplier.dto.ts`
- Create: `breeding-app/src/modules/master-data/dto/query-supplier.dto.ts`
- Modify: `breeding-app/src/modules/master-data/services/supplier.service.ts`
- Modify: `breeding-app/src/modules/master-data/controllers/supplier.controller.ts:22-27`

**Interfaces:**
- Produces:
  - `POST /suppliers` body `{ code, name, address?, status?, categoryIds?: string[] }`
  - `PATCH /suppliers/:id` same shape, all optional; `categoryIds: []` clears classification; omitted `categoryIds` leaves it untouched
  - `GET /suppliers?expedition=true` — only suppliers with at least one `isExpedition` category
  - Supplier responses include `categories: { supplierId, categoryId, category: { id, name, isExpedition } }[]`
  - `SupplierService.assertExpeditionSupplier(tenantId: string, supplierId: string): Promise<void>` — throws `NotFoundException` if supplier not in tenant / soft-deleted, `BadRequestException('Supplier is not classified as expedition')` otherwise if not classified. Used by Tasks 9 and 14.

- [ ] **Step 1: Add `categoryIds` to `CreateSupplierDto`**

Replace the file with:

```ts
import { IsNotEmpty, IsOptional, IsString, IsArray, IsUUID } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateSupplierDto {
  @ApiProperty()
  @IsNotEmpty()
  @IsString()
  code: string;

  @ApiProperty()
  @IsNotEmpty()
  @IsString()
  name: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  address?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  status?: string;

  @ApiPropertyOptional({ type: [String], format: 'uuid', description: 'Product category IDs this supplier is classified under' })
  @IsOptional()
  @IsArray()
  @IsUUID('4', { each: true })
  categoryIds?: string[];
}
```

- [ ] **Step 2: Create `QuerySupplierDto`**

Create `breeding-app/src/modules/master-data/dto/query-supplier.dto.ts`:

```ts
import { IsOptional, IsBoolean } from 'class-validator';
import { Transform } from 'class-transformer';
import { ApiPropertyOptional } from '@nestjs/swagger';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

export class QuerySupplierDto extends PaginationDto {
  @ApiPropertyOptional({ description: 'Only suppliers classified under an expedition category' })
  @IsOptional()
  @Transform(({ value }) => value === 'true' || value === true)
  @IsBoolean()
  expedition?: boolean;
}
```

- [ ] **Step 3: Rewrite `SupplierService`**

Replace `breeding-app/src/modules/master-data/services/supplier.service.ts` with:

```ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service.js';
import { CreateSupplierDto } from '../dto/create-supplier.dto.js';
import { UpdateSupplierDto } from '../dto/update-supplier.dto.js';
import { QuerySupplierDto } from '../dto/query-supplier.dto.js';
import { PaginatedResult } from '../../../common/dto/pagination.dto.js';

@Injectable()
export class SupplierService {
  private readonly include = {
    categories: {
      include: {
        category: { select: { id: true, name: true, isExpedition: true } },
      },
    },
  } as const;

  constructor(private readonly prisma: PrismaService) {}

  async create(dto: CreateSupplierDto, tenantId: string) {
    const { categoryIds, ...data } = dto;
    return this.prisma.$transaction(async (tx) => {
      const ids = await this.validateCategoryIds(tx, tenantId, categoryIds);
      return tx.supplier.create({
        data: {
          ...data,
          tenantId,
          categories: { create: ids.map((categoryId) => ({ categoryId })) },
        },
        include: this.include,
      });
    });
  }

  async findAll(tenantId: string, query: QuerySupplierDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(query.search && {
        name: { contains: query.search, mode: 'insensitive' as const },
      }),
      ...(query.expedition && {
        categories: {
          some: { category: { isExpedition: true, deletedAt: null } },
        },
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.supplier.findMany({
        where,
        include: this.include,
        skip: query.skip,
        take: query.limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.supplier.count({ where }),
    ]);
    return new PaginatedResult(data, total, query.page, query.limit);
  }

  async findOne(id: string, tenantId: string) {
    const record = await this.prisma.supplier.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!record) throw new NotFoundException('Supplier not found');
    return record;
  }

  async update(id: string, dto: UpdateSupplierDto, tenantId: string) {
    await this.findOne(id, tenantId);
    const { categoryIds, ...data } = dto;
    return this.prisma.$transaction(async (tx) => {
      if (categoryIds !== undefined) {
        const ids = await this.validateCategoryIds(tx, tenantId, categoryIds);
        await tx.supplierCategory.deleteMany({ where: { supplierId: id } });
        if (ids.length > 0) {
          await tx.supplierCategory.createMany({
            data: ids.map((categoryId) => ({ supplierId: id, categoryId })),
          });
        }
      }
      return tx.supplier.update({ where: { id }, data, include: this.include });
    });
  }

  async remove(id: string, tenantId: string) {
    await this.findOne(id, tenantId);
    return this.prisma.supplier.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }

  /**
   * Throws unless the supplier exists in the tenant and is classified under
   * at least one active category marked isExpedition.
   * Used by goods-transfer dispatch and goods-receipt create.
   */
  async assertExpeditionSupplier(tenantId: string, supplierId: string): Promise<void> {
    const supplier = await this.prisma.supplier.findFirst({
      where: { id: supplierId, tenantId, deletedAt: null },
      select: {
        id: true,
        categories: {
          where: { category: { isExpedition: true, deletedAt: null } },
          select: { categoryId: true },
        },
      },
    });
    if (!supplier) throw new NotFoundException('Supplier not found');
    if (supplier.categories.length === 0) {
      throw new BadRequestException('Supplier is not classified as expedition');
    }
  }

  /** Returns the de-duplicated ids after checking every one belongs to the tenant. */
  private async validateCategoryIds(
    tx: Parameters<Parameters<PrismaService['$transaction']>[0]>[0],
    tenantId: string,
    categoryIds: string[] | undefined,
  ): Promise<string[]> {
    const ids = Array.from(new Set(categoryIds ?? []));
    if (ids.length === 0) return [];
    const found = await tx.productCategory.count({
      where: { id: { in: ids }, tenantId, deletedAt: null },
    });
    if (found !== ids.length) {
      throw new BadRequestException('One or more category IDs are invalid');
    }
    return ids;
  }
}
```

If the `tx` parameter type above fails to compile under this Prisma version, replace it with `Prisma.TransactionClient` imported from `'../../../../generated/prisma/client.js'`.

- [ ] **Step 4: Controller uses `QuerySupplierDto`**

In `supplier.controller.ts`, replace the `PaginationDto` import and `findAll`:

```ts
import { QuerySupplierDto } from '../dto/query-supplier.dto.js';
// remove: import { PaginationDto } from '../../../common/dto/pagination.dto.js';

  @ApiOperation({ summary: 'List all suppliers' })
  @Get()
  findAll(@CurrentTenant() tenantId: string, @Query() query: QuerySupplierDto) {
    return this.supplierService.findAll(tenantId, query);
  }
```

- [ ] **Step 5: Build**

```bash
cd breeding-app && npm run build
```

Expected: success. If `tx.supplierCategory` is not found, re-run `npx prisma generate`.

- [ ] **Step 6: Commit**

```bash
cd breeding-app && git add src/modules/master-data && git commit -m "feat(master-data): supplier category classification and expedition filter

- categoryIds on create/update (replace-all on update)
- GET /suppliers?expedition=true
- SupplierService.assertExpeditionSupplier for other modules

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: Seed expedition category and suppliers

**Files:**
- Modify: `breeding-app/prisma/seed.ts:101-113` (categories), `:181-187` (suppliers)

**Interfaces:**
- Produces: seed constants `catExpedition`, `supplierExpeditionCo`, `supplierExpeditionPerson` for later seed sections (Task 8 uses `supplierExpeditionCo`).

- [ ] **Step 1: Add the expedition category**

After the `catEquipment` creation (line ~112), add:

```ts
  const catExpedition = await prisma.productCategory.create({
    data: { tenantId: tenant.id, name: 'Jasa Ekspedisi', purchasePurpose: 'Logistics', isExpedition: true },
  });
  console.log(`[ProductCategory] Created 5 categories`);
```

and delete the previous `console.log(\`[ProductCategory] Created 4 categories\`);` line.

- [ ] **Step 2: Classify existing suppliers and add two expedition suppliers**

Replace the supplier block (lines ~181-187) with:

```ts
  const supplierJapfa = await prisma.supplier.create({
    data: {
      tenantId: tenant.id, code: 'SUP-001', name: 'PT Japfa Comfeed', address: 'Jl. Industri No. 10, Sidoarjo',
      categories: { create: [{ categoryId: catFeed.id }] },
    },
  });
  const supplierCargill = await prisma.supplier.create({
    data: {
      tenantId: tenant.id, code: 'SUP-002', name: 'PT Cargill Indonesia', address: 'Jl. Raya Bogor KM 28',
      categories: { create: [{ categoryId: catFeed.id }, { categoryId: catMedicine.id }] },
    },
  });
  const supplierExpeditionCo = await prisma.supplier.create({
    data: {
      tenantId: tenant.id, code: 'SUP-003', name: 'CV Restu Ibu Ekspedisi', address: 'Jl. Raya Mojokerto No. 5',
      categories: { create: [{ categoryId: catExpedition.id }] },
    },
  });
  const supplierExpeditionPerson = await prisma.supplier.create({
    data: {
      tenantId: tenant.id, code: 'SUP-004', name: 'Agus Purwanto (Truk Sewa)', address: 'Desa Keranding',
      categories: { create: [{ categoryId: catExpedition.id }] },
    },
  });
  console.log(`[Supplier] Created 4 suppliers (2 expedition)`);
```

`supplierExpeditionPerson` is intentionally unused for now; Task 8 uses `supplierExpeditionCo`. If ESLint complains about the unused const, prefix it `_supplierExpeditionPerson`.

- [ ] **Step 3: Reset and re-seed**

```bash
cd breeding-app && npx prisma migrate reset --force
```

Expected: migrations apply, seed prints `[ProductCategory] Created 5 categories` and `[Supplier] Created 4 suppliers (2 expedition)`.

- [ ] **Step 4: Curl verification**

Start the API (`npm run start:dev`) in another terminal, then:

```bash
TOKEN=$(curl -s -X POST http://localhost:3002/api/auth/login -H 'Content-Type: application/json' -d '{"email":"admin@demo.farm","password":"password123"}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["data"]["accessToken"])')
curl -s "http://localhost:3002/api/suppliers?expedition=true" -H "Authorization: Bearer $TOKEN" | python3 -c 'import sys,json;d=json.load(sys.stdin)["data"];print([s["name"] for s in d["data"]])'
curl -s "http://localhost:3002/api/suppliers" -H "Authorization: Bearer $TOKEN" | python3 -c 'import sys,json;d=json.load(sys.stdin)["data"];print(d["meta"]["total"])'
```

Expected: first prints `['Agus Purwanto (Truk Sewa)', 'CV Restu Ibu Ekspedisi']`, second prints `4`. If the login response key is not `accessToken`, check `src/modules/auth/auth.service.ts` and adjust.

Also verify replace-all update and cross-tenant rejection:

```bash
SUP=$(curl -s "http://localhost:3002/api/suppliers?expedition=true" -H "Authorization: Bearer $TOKEN" | python3 -c 'import sys,json;print(json.load(sys.stdin)["data"]["data"][0]["id"])')
curl -s -X PATCH "http://localhost:3002/api/suppliers/$SUP" -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{"categoryIds":[]}' | python3 -c 'import sys,json;print(len(json.load(sys.stdin)["data"]["categories"]))'
curl -s -X PATCH "http://localhost:3002/api/suppliers/$SUP" -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d '{"categoryIds":["00000000-0000-4000-8000-000000000000"]}' | python3 -c 'import sys,json;print(json.load(sys.stdin)["statusCode"])'
```

Expected: `0`, then `400`. Re-run `npx prisma migrate reset --force` afterwards to restore the seed.

- [ ] **Step 5: Commit**

```bash
cd breeding-app && git add prisma/seed.ts && git commit -m "chore(seed): add expedition category and expedition suppliers

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Frontend types, i18n, product-category `isExpedition` checkbox

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts:120-128` (ProductCategory), `:153-162` (Supplier)
- Modify: `breeding-dashboard/messages/en.json`, `messages/id.json` (`productCategories`, `suppliers` namespaces)
- Modify: `breeding-dashboard/src/app/(dashboard)/product-categories/page.tsx`

**Interfaces:**
- Produces: `ProductCategory.isExpedition: boolean`; `Supplier.code: string`, `Supplier.categories?: SupplierCategoryLink[]`; i18n keys `productCategories.isExpedition`, `productCategories.expeditionBadge`, `suppliers.code`, `suppliers.categories`, `suppliers.selectExpedition`, `suppliers.noExpeditionFound`, `suppliers.noCategories`.

- [ ] **Step 1: Types**

In `src/types/api.ts`, replace `ProductCategory` and `Supplier`:

```ts
export interface ProductCategory {
  id: string;
  name: string;
  purchasePurpose?: string;
  overheadCategory?: string;
  priceType?: string;
  isExpedition: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface SupplierCategoryLink {
  supplierId: string;
  categoryId: string;
  category: Pick<ProductCategory, "id" | "name" | "isExpedition">;
}

export interface Supplier {
  id: string;
  code: string;
  name: string;
  address?: string;
  status?: string;
  categories?: SupplierCategoryLink[];
  createdAt: string;
  updatedAt: string;
}
```

(`contactPerson`, `phone`, `email` are removed — the backend `Supplier` model has never had them; see Task 6.)

- [ ] **Step 2: i18n keys**

In `messages/en.json`, inside `"productCategories"` add:

```json
    "isExpedition": "Expedition / freight service category",
    "expeditionBadge": "Expedition"
```

inside `"suppliers"` add:

```json
    "code": "Supplier code",
    "categories": "Category classification",
    "noCategories": "No categories yet — create them under Product Categories.",
    "selectExpedition": "Select expedition...",
    "noExpeditionFound": "No expedition supplier found. Classify a supplier under an expedition category first."
```

In `messages/id.json`, `"productCategories"`:

```json
    "isExpedition": "Kategori jasa ekspedisi / angkutan",
    "expeditionBadge": "Ekspedisi"
```

`"suppliers"`:

```json
    "code": "Kode supplier",
    "categories": "Klasifikasi kategori",
    "noCategories": "Belum ada kategori — buat dulu di Kategori Produk.",
    "selectExpedition": "Pilih ekspedisi...",
    "noExpeditionFound": "Tidak ada supplier ekspedisi. Klasifikasikan supplier ke kategori ekspedisi dulu."
```

Keep JSON valid (commas between entries). Remove the now-unused `suppliers.contactPerson` key from both files only if nothing else references it: `grep -rn "contactPerson" src/` — the customers page uses `customers.contactPerson`, a different namespace, so `suppliers.contactPerson` can go.

- [ ] **Step 3: Product-category dialog checkbox + table badge**

In `src/app/(dashboard)/product-categories/page.tsx`:

Add imports:

```ts
import { Checkbox } from "@/components/ui/checkbox";
import { Badge } from "@/components/ui/badge";
```

Add state next to the other `form*` states (line ~50):

```ts
  const [formIsExpedition, setFormIsExpedition] = useState(false);
```

In `handleCreate()` reset it: `setFormIsExpedition(false);`. In `handleEdit(category)` set it: `setFormIsExpedition(category.isExpedition ?? false);`.

In `handleSubmit()`, add to the `body` object (alongside the `purchasePurpose` spread at line ~141):

```ts
        isExpedition: formIsExpedition,
```

Add a column before the actions column:

```tsx
    {
      header: t('expeditionBadge'),
      cell: (row) =>
        row.isExpedition ? <Badge variant="secondary">{t('expeditionBadge')}</Badge> : "-",
      className: "w-[120px]",
    },
```

In the dialog, after the `category-price-type` field block, add:

```tsx
            <div className="flex items-center gap-2">
              <Checkbox
                id="category-is-expedition"
                checked={formIsExpedition}
                onCheckedChange={(v) => setFormIsExpedition(v === true)}
              />
              <Label htmlFor="category-is-expedition">{t('isExpedition')}</Label>
            </div>
```

- [ ] **Step 4: Type-check and build**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: `tsc` will now report errors in `suppliers/page.tsx` (uses removed `contactPerson`/`phone`/`email`). That is expected and fixed in Task 6 — if the build blocks, proceed to Task 6 before committing, then commit both together.

- [ ] **Step 5: Commit** (if build passes; otherwise fold into Task 6's commit)

```bash
cd breeding-dashboard && git add src/types/api.ts messages "src/app/(dashboard)/product-categories/page.tsx" && git commit -m "feat(master-data): expedition marker on product categories

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: Supplier dialog — `code` field (bug fix), category checkboxes, category badges

**Pre-existing bug this task fixes:** the supplier dialog sends `contactPerson`, `phone`, `email` and omits `code`. The backend `CreateSupplierDto` requires `code` and `ValidationPipe` has `forbidNonWhitelisted: true`, so creating a supplier from the UI currently returns 400. This task aligns the dialog with the backend.

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/suppliers/page.tsx`

**Interfaces:**
- Consumes: `GET /product-categories?limit=100` (existing), `POST/PATCH /suppliers` with `categoryIds` (Task 3), i18n keys from Task 5.

- [ ] **Step 1: Replace form state**

Replace lines 48-54 (the `formName` … `formAddress` states) with:

```ts
  const [formCode, setFormCode] = useState("");
  const [formName, setFormName] = useState("");
  const [formAddress, setFormAddress] = useState("");
  const [formCategoryIds, setFormCategoryIds] = useState<string[]>([]);
  const [categories, setCategories] = useState<ProductCategory[]>([]);
```

Add imports:

```ts
import { useEffect } from "react"; // merge into the existing react import: import { useState, useEffect } from "react";
import { fetchPaginated } from "@/lib/api";
import { Checkbox } from "@/components/ui/checkbox";
import { Badge } from "@/components/ui/badge";
import { ProductCategory, Supplier } from "@/types/api";
```

Add after the state declarations — load categories once when the dialog opens:

```ts
  useEffect(() => {
    if (!dialogOpen) return;
    fetchPaginated<ProductCategory>("/product-categories", { limit: 100 })
      .then((res) => setCategories(res.data))
      .catch(() => setCategories([]));
  }, [dialogOpen]);

  function toggleCategory(id: string, checked: boolean) {
    setFormCategoryIds((prev) =>
      checked ? Array.from(new Set([...prev, id])) : prev.filter((c) => c !== id)
    );
  }
```

- [ ] **Step 2: Replace table columns**

Replace the `contactPerson`, `phone`, `email` columns with:

```tsx
    {
      header: t('code'),
      cell: (row) => <span className="font-mono text-xs">{row.code}</span>,
      className: "w-[120px]",
    },
    {
      header: t('categories'),
      cell: (row) =>
        row.categories && row.categories.length > 0 ? (
          <div className="flex flex-wrap gap-1">
            {row.categories.map((c) => (
              <Badge key={c.categoryId} variant={c.category.isExpedition ? "default" : "secondary"}>
                {c.category.name}
              </Badge>
            ))}
          </div>
        ) : (
          "-"
        ),
    },
```

Keep the `name`, `created`, and `actions` columns. Put `code` first, then `name`, then `categories`.

- [ ] **Step 3: Replace `handleCreate`, `handleEdit`, `handleSubmit`**

```ts
  function handleCreate() {
    setEditingSupplier(null);
    setFormCode("");
    setFormName("");
    setFormAddress("");
    setFormCategoryIds([]);
    setDialogOpen(true);
  }

  function handleEdit(supplier: Supplier) {
    setEditingSupplier(supplier);
    setFormCode(supplier.code);
    setFormName(supplier.name);
    setFormAddress(supplier.address || "");
    setFormCategoryIds((supplier.categories ?? []).map((c) => c.categoryId));
    setDialogOpen(true);
  }

  async function handleSubmit() {
    if (!formCode.trim()) {
      toast.error(tc('required', { field: t('code') }));
      return;
    }
    if (!formName.trim()) {
      toast.error(tc('required', { field: tc('name') }));
      return;
    }

    setIsSubmitting(true);
    try {
      const body = {
        code: formCode.trim(),
        name: formName.trim(),
        ...(formAddress.trim() && { address: formAddress.trim() }),
        categoryIds: formCategoryIds,
      };

      if (editingSupplier) {
        await fetchApi(`/suppliers/${editingSupplier.id}`, {
          method: "PATCH",
          body: JSON.stringify(body),
        });
        toast.success(tc('entityUpdated', { entity: t('entity') }));
      } else {
        await fetchApi("/suppliers", {
          method: "POST",
          body: JSON.stringify(body),
        });
        toast.success(tc('entityCreated', { entity: t('entity') }));
      }

      setDialogOpen(false);
      refetch();
    } catch (error) {
      toast.error(
        error instanceof Error
          ? error.message
          : editingSupplier
            ? tc('entityUpdateFailed', { entity: t('entity') })
            : tc('entityCreateFailed', { entity: t('entity') })
      );
    } finally {
      setIsSubmitting(false);
    }
  }
```

- [ ] **Step 4: Replace the dialog fields**

Replace the `<div className="space-y-4">…</div>` block inside `DialogContent` with:

```tsx
          <div className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="supplier-code">{t('code')}</Label>
              <Input
                id="supplier-code"
                placeholder={tc('enterField', { field: t('code') })}
                value={formCode}
                onChange={(e) => setFormCode(e.target.value)}
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="supplier-name">{tc('name')}</Label>
              <Input
                id="supplier-name"
                placeholder={tc('enterField', { field: tc('name') })}
                value={formName}
                onChange={(e) => setFormName(e.target.value)}
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="supplier-address">{tc('address')}</Label>
              <Input
                id="supplier-address"
                placeholder={tc('enterFieldOptional', { field: tc('address') })}
                value={formAddress}
                onChange={(e) => setFormAddress(e.target.value)}
              />
            </div>
            <div className="space-y-2">
              <Label>{t('categories')}</Label>
              {categories.length === 0 ? (
                <p className="text-sm text-muted-foreground">{t('noCategories')}</p>
              ) : (
                <div className="grid grid-cols-2 gap-2 rounded-md border p-3">
                  {categories.map((cat) => (
                    <div key={cat.id} className="flex items-center gap-2">
                      <Checkbox
                        id={`supplier-cat-${cat.id}`}
                        checked={formCategoryIds.includes(cat.id)}
                        onCheckedChange={(v) => toggleCategory(cat.id, v === true)}
                      />
                      <Label htmlFor={`supplier-cat-${cat.id}`} className="font-normal">
                        {cat.name}
                        {cat.isExpedition && (
                          <span className="ml-1 text-xs text-muted-foreground">
                            ({tc('expedition')})
                          </span>
                        )}
                      </Label>
                    </div>
                  ))}
                </div>
              )}
            </div>
          </div>
```

Add `"expedition": "Expedition"` to `common` in `en.json` and `"expedition": "Ekspedisi"` in `id.json`.

- [ ] **Step 5: Type-check, build, smoke test**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: clean. Then with backend + `npm run dev` running, open `/suppliers`, create a supplier with code `SUP-TEST`, tick `Jasa Ekspedisi`, save. Expected: row appears with a primary-colored `Jasa Ekspedisi` badge. Edit it, untick, save → badge gone.

- [ ] **Step 6: Commit**

```bash
cd breeding-dashboard && git add "src/app/(dashboard)/suppliers/page.tsx" messages src/types/api.ts "src/app/(dashboard)/product-categories/page.tsx" && git commit -m "feat(master-data): supplier category classification in dialog; fix supplier create payload

The dialog sent contactPerson/phone/email (not on the backend model) and
omitted the required code field, so creating a supplier from the UI
returned 400. Aligned with CreateSupplierDto and added category checkboxes.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 7: `SupplierCombobox` gets `expeditionOnly`

**Files:**
- Modify: `breeding-dashboard/src/components/forms/supplier-combobox.tsx`

**Interfaces:**
- Produces: `<SupplierCombobox value onChange disabled? expeditionOnly? />` — with `expeditionOnly`, fetches `/suppliers?expedition=true`, uses `suppliers.selectExpedition` / `suppliers.noExpeditionFound` texts. Used by Tasks 10, 11, 14.

- [ ] **Step 1: Replace the component**

```tsx
"use client";

import { useState, useEffect } from "react";
import { Check, ChevronsUpDown } from "lucide-react";
import { useTranslations } from "next-intl";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from "@/components/ui/command";
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover";
import { fetchPaginated } from "@/lib/api";
import { Supplier } from "@/types/api";

interface SupplierComboboxProps {
  value: string;
  onChange: (id: string) => void;
  disabled?: boolean;
  /** Only list suppliers classified under an expedition category. */
  expeditionOnly?: boolean;
}

export function SupplierCombobox({ value, onChange, disabled, expeditionOnly = false }: SupplierComboboxProps) {
  const t = useTranslations("suppliers");
  const [open, setOpen] = useState(false);
  const [suppliers, setSuppliers] = useState<Supplier[]>([]);
  const [search, setSearch] = useState("");
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    setIsLoading(true);
    fetchPaginated<Supplier>("/suppliers", {
      limit: 50,
      search,
      ...(expeditionOnly && { extra: { expedition: "true" } }),
    })
      .then((res) => setSuppliers(res.data))
      .catch(() => {})
      .finally(() => setIsLoading(false));
  }, [search, expeditionOnly]);

  const selected = suppliers.find((s) => s.id === value);
  const placeholder = expeditionOnly ? t("selectExpedition") : "Select supplier...";
  const emptyText = expeditionOnly ? t("noExpeditionFound") : "No suppliers found.";

  return (
    <Popover open={open} onOpenChange={setOpen}>
      <PopoverTrigger asChild>
        <Button
          variant="outline"
          role="combobox"
          aria-expanded={open}
          className="w-full justify-between font-normal"
          disabled={disabled}
        >
          {selected ? selected.name : placeholder}
          <ChevronsUpDown className="ml-2 h-4 w-4 shrink-0 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-[400px] p-0" align="start">
        <Command shouldFilter={false}>
          <CommandInput
            placeholder="Search suppliers..."
            value={search}
            onValueChange={setSearch}
          />
          <CommandList>
            {isLoading ? (
              <CommandEmpty>Loading...</CommandEmpty>
            ) : suppliers.length === 0 ? (
              <CommandEmpty>{emptyText}</CommandEmpty>
            ) : (
              <CommandGroup>
                {suppliers.map((supplier) => (
                  <CommandItem
                    key={supplier.id}
                    value={supplier.id}
                    onSelect={() => {
                      onChange(supplier.id);
                      setOpen(false);
                    }}
                  >
                    <Check
                      className={cn(
                        "mr-2 h-4 w-4",
                        value === supplier.id ? "opacity-100" : "opacity-0"
                      )}
                    />
                    <span>{supplier.name}</span>
                  </CommandItem>
                ))}
              </CommandGroup>
            )}
          </CommandList>
        </Command>
      </PopoverContent>
    </Popover>
  );
}
```

Known limitation kept from the original: when `value` is set but the selected supplier is not in the current 50-row page, the trigger shows the placeholder. The dispatch dialog (Task 11) passes the carrier name for display separately, so this doesn't affect the read path.

- [ ] **Step 2: Type-check and build**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Expected: clean. Existing callers pass no `expeditionOnly` and behave as before.

- [ ] **Step 3: Commit**

```bash
cd breeding-dashboard && git add src/components/forms/supplier-combobox.tsx && git commit -m "feat(forms): expeditionOnly filter on SupplierCombobox

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

# Phase B — Goods transfer: carrier, delivery note, PREPARING/dispatch, BPB reference

### Task 8: Prisma schema + migration for goods transfer

**Files:**
- Modify: `breeding-app/prisma/schema.prisma:78-84` (TransferStatus), `:229-` (User back-relation), `:566-` (Supplier back-relation), `:1188-1207` (GoodsReceiptLine back-relation), `:1212-1256` (GoodsTransfer, GoodsTransferLine)
- Modify: `breeding-app/prisma/seed.ts:525-538` (seed transfer)
- Create (generated): `prisma/migrations/<timestamp>_add_goods_transfer_expedition/`

**Interfaces:**
- Produces: `TransferStatus.PREPARING` (default); `goodsTransfer.carrierSupplierId | deliveryNoteNumber | dispatchedAt | dispatchedById`; relations `goodsTransfer.carrierSupplier`, `goodsTransfer.dispatchedBy`; `goodsTransferLine.goodsReceiptLineId` + `goodsTransferLine.goodsReceiptLine`.

- [ ] **Step 1: Enum**

```prisma
enum TransferStatus {
  PREPARING
  IN_TRANSIT
  RECEIVED
  PARTIAL
  DAMAGED_IN_TRANSIT
  CANCELLED
}
```

- [ ] **Step 2: `GoodsTransfer` and `GoodsTransferLine`**

Replace both models:

```prisma
model GoodsTransfer {
  id                   String         @id @default(uuid())
  tenantId             String         @map("tenant_id")
  branchId             String         @map("branch_id")
  fromWarehouseId      String         @map("from_warehouse_id")
  toWarehouseId        String         @map("to_warehouse_id")
  status               TransferStatus @default(PREPARING)
  transferNumber       String         @map("transfer_number")
  transferDate         DateTime       @map("transfer_date") @db.Date
  estimatedReceiptDate DateTime?      @map("estimated_receipt_date") @db.Date
  actualReceiptDate    DateTime?      @map("actual_receipt_date") @db.Date
  reason               String?        @db.Text
  notes                String?        @db.Text
  carrierSupplierId    String?        @map("carrier_supplier_id")
  deliveryNoteNumber   String?        @map("delivery_note_number")
  dispatchedAt         DateTime?      @map("dispatched_at")
  dispatchedById       String?        @map("dispatched_by_id")
  createdAt            DateTime       @default(now()) @map("created_at")
  updatedAt            DateTime       @updatedAt @map("updated_at")
  deletedAt            DateTime?      @map("deleted_at")

  branch          Branch    @relation(fields: [branchId], references: [id])
  fromWarehouse   Warehouse @relation("TransferFrom", fields: [fromWarehouseId], references: [id])
  toWarehouse     Warehouse @relation("TransferTo", fields: [toWarehouseId], references: [id])
  carrierSupplier Supplier? @relation("TransferCarrier", fields: [carrierSupplierId], references: [id])
  dispatchedBy    User?     @relation("TransferDispatchedBy", fields: [dispatchedById], references: [id])

  lines                  GoodsTransferLine[]
  logisticsShippingCosts LogisticsShippingCost[] @relation("TransferShippingCost")

  @@unique([tenantId, transferNumber])
  @@index([carrierSupplierId])
  @@map("goods_transfers")
}

model GoodsTransferLine {
  id                 String  @id @default(uuid())
  goodsTransferId    String  @map("goods_transfer_id")
  productId          String  @map("product_id")
  uomId              String  @map("uom_id")
  goodsReceiptLineId String? @map("goods_receipt_line_id")
  quantitySent       Decimal @map("quantity_sent") @db.Decimal(18, 4)
  quantityReceived   Decimal @default(0) @map("quantity_received") @db.Decimal(18, 4)
  quantityDamaged    Decimal @default(0) @map("quantity_damaged") @db.Decimal(18, 4)
  valueAmount        Decimal @default(0) @map("value_amount") @db.Decimal(18, 2)
  notes              String? @db.Text

  goodsTransfer    GoodsTransfer     @relation(fields: [goodsTransferId], references: [id])
  product          Product           @relation(fields: [productId], references: [id])
  uom              UnitOfMeasure     @relation(fields: [uomId], references: [id])
  goodsReceiptLine GoodsReceiptLine? @relation(fields: [goodsReceiptLineId], references: [id])

  @@index([goodsReceiptLineId])
  @@map("goods_transfer_lines")
}
```

Before replacing, `sed -n 1240,1256p prisma/schema.prisma` and confirm the existing line columns match the list above (`id`, `goodsTransferId`, `productId`, `uomId`, quantities, `valueAmount`, `notes`) so nothing is dropped.

- [ ] **Step 3: Back-relations**

In `model Supplier`, add after `categories SupplierCategory[]`:

```prisma
  goodsTransfersAsCarrier GoodsTransfer[] @relation("TransferCarrier")
```

In `model User`, add to the relations block (find the existing relation list in that model and append):

```prisma
  goodsTransfersDispatched GoodsTransfer[] @relation("TransferDispatchedBy")
```

In `model GoodsReceiptLine`, next to `consumptionLines GoodsConsumptionLine[]`, add:

```prisma
  transferLines    GoodsTransferLine[]
```

- [ ] **Step 4: Seed transfer stays `RECEIVED` but now carries expedition data**

In `prisma/seed.ts`, the goods transfer block (line ~525) becomes:

```ts
  await prisma.goodsTransfer.create({
    data: {
      tenantId: tenant.id, branchId: branchMain.id,
      fromWarehouseId: whMain.id, toWarehouseId: whFarm.id,
      transferNumber: 'GT-2026-03-001', transferDate: new Date('2026-03-05'),
      status: TransferStatus.RECEIVED, actualReceiptDate: new Date('2026-03-05'),
      carrierSupplierId: supplierExpeditionCo.id,
      deliveryNoteNumber: 'SJ/RI/2026/0031',
      dispatchedAt: new Date('2026-03-05T07:00:00Z'),
      dispatchedById: adminUser.id,
      lines: {
        create: [
          { productId: prdCornFeed.id, uomId: uomKg.id, quantitySent: 500, quantityReceived: 500 },
        ],
      },
    },
  });
```

`adminUser` — check the variable name used for the TENANT_ADMIN user at seed line ~78 (`const ... = await prisma.user.create({ data: { email: 'admin@demo.farm' ...`) and use that name.

- [ ] **Step 5: Migrate, generate, build**

```bash
cd breeding-app && npx prisma migrate dev --name add_goods_transfer_expedition && npx prisma generate && npm run build
```

Expected: migration contains `ALTER TYPE "TransferStatus" ADD VALUE 'PREPARING';`, `ALTER TABLE "goods_transfers" ALTER COLUMN "status" SET DEFAULT 'PREPARING'`, four new nullable columns, one on `goods_transfer_lines`, two indexes, three FKs. Build: **will fail** in `status-transitions.constant.ts` (`Record<TransferStatus, …>` missing `PREPARING`). That is Task 9 Step 1 — do it now before committing, then commit both files together in Task 9. If you prefer a green commit here, add the `PREPARING` row from Task 9 Step 1 in this task.

- [ ] **Step 6: Commit** (after Task 9 Step 1 makes the build green)

```bash
cd breeding-app && git add prisma src/common/constants/status-transitions.constant.ts && git commit -m "feat(transfer): PREPARING status, carrier/delivery-note/dispatch columns, BPB ref on lines

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 9: Goods transfer backend — DTOs, service, dispatch endpoint

**Files:**
- Modify: `breeding-app/src/common/constants/status-transitions.constant.ts:28-34`
- Modify: `breeding-app/src/modules/transfer/dto/create-goods-transfer.dto.ts`
- Create: `breeding-app/src/modules/transfer/dto/dispatch-goods-transfer.dto.ts`
- Create: `breeding-app/src/modules/transfer/dto/query-goods-transfer.dto.ts`
- Modify: `breeding-app/src/modules/transfer/goods-transfer.service.ts`
- Modify: `breeding-app/src/modules/transfer/goods-transfer.controller.ts`
- Modify: `breeding-app/src/modules/transfer/transfer.module.ts`

**Interfaces:**
- Consumes: `SupplierService.assertExpeditionSupplier(tenantId, supplierId)` (Task 3).
- Produces:
  - `POST /goods-transfers` → status `PREPARING`; body adds `carrierSupplierId?`, `deliveryNoteNumber?`, `lines[].goodsReceiptLineId?`
  - `PATCH /goods-transfers/:id` → 400 unless `PREPARING`
  - `POST /goods-transfers/:id/dispatch` body `{ carrierSupplierId?, deliveryNoteNumber? }` → MANAGER+; `PREPARING → IN_TRANSIT`
  - `PATCH /goods-transfers/:id/status` with `targetStatus: 'IN_TRANSIT'` → 400 `Use dispatch endpoint`
  - `GET /goods-transfers?status=PREPARING` filter
  - Responses include `carrierSupplier { id, name }`, `dispatchedBy { id, name }`, `lines[].goodsReceiptLine { id, goodsReceipt { id, receiptNumber } }`

- [ ] **Step 1: Transitions**

```ts
export const TRANSFER_STATUS_TRANSITIONS: Record<TransferStatus, TransferStatus[]> = {
  [TransferStatus.PREPARING]: [TransferStatus.IN_TRANSIT, TransferStatus.CANCELLED],
  [TransferStatus.IN_TRANSIT]: [TransferStatus.RECEIVED, TransferStatus.PARTIAL, TransferStatus.DAMAGED_IN_TRANSIT, TransferStatus.CANCELLED],
  [TransferStatus.RECEIVED]: [],
  [TransferStatus.PARTIAL]: [TransferStatus.RECEIVED],
  [TransferStatus.DAMAGED_IN_TRANSIT]: [],
  [TransferStatus.CANCELLED]: [],
};
```

`IN_TRANSIT` stays listed under `PREPARING` so `dispatch()` can reuse `validateTransition`; the generic endpoint blocks it explicitly (Step 5).

- [ ] **Step 2: Create DTO additions**

In `create-goods-transfer.dto.ts`, add `MaxLength` to the class-validator import, then:

In `CreateGoodsTransferLineDto`, after `uomId`:

```ts
  @ApiPropertyOptional({ format: 'uuid', description: 'Originating goods receipt line (BPB) in the source warehouse' })
  @IsOptional()
  @IsUUID()
  goodsReceiptLineId?: string;
```

In `CreateGoodsTransferDto`, after `notes`:

```ts
  @ApiPropertyOptional({ format: 'uuid', description: 'Expedition supplier carrying this transfer' })
  @IsOptional()
  @IsUUID()
  carrierSupplierId?: string;

  @ApiPropertyOptional({ description: 'Delivery note (surat jalan) number from the carrier', maxLength: 100 })
  @IsOptional()
  @IsString()
  @MaxLength(100)
  deliveryNoteNumber?: string;
```

`UpdateGoodsTransferDto` is `PartialType(CreateGoodsTransferDto)` — check `dto/update-goods-transfer.dto.ts`; if it is, nothing to add. If it lists fields explicitly, add the same two header fields there.

- [ ] **Step 3: Dispatch and query DTOs**

`dto/dispatch-goods-transfer.dto.ts`:

```ts
import { IsOptional, IsString, IsUUID, MaxLength } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class DispatchGoodsTransferDto {
  @ApiPropertyOptional({ format: 'uuid', description: 'Overrides the carrier stored on the transfer' })
  @IsOptional()
  @IsUUID()
  carrierSupplierId?: string;

  @ApiPropertyOptional({ description: 'Overrides the delivery note number stored on the transfer', maxLength: 100 })
  @IsOptional()
  @IsString()
  @MaxLength(100)
  deliveryNoteNumber?: string;
}
```

`dto/query-goods-transfer.dto.ts`:

```ts
import { IsOptional, IsEnum } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';
import { PaginationDto } from '../../common/dto/pagination.dto.js';
import { TransferStatus } from '../../../generated/prisma/client.js';

export class QueryGoodsTransferDto extends PaginationDto {
  @ApiPropertyOptional({ enum: TransferStatus })
  @IsOptional()
  @IsEnum(TransferStatus)
  status?: TransferStatus;
}
```

- [ ] **Step 4: Service**

Replace `goods-transfer.service.ts` with:

```ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';
import { ReferenceNumberGenerator } from '../../common/utils/reference-number.generator.js';
import { StatusTransitionService } from '../../common/services/status-transition.service.js';
import { SupplierService } from '../master-data/services/supplier.service.js';
import { CreateGoodsTransferDto, CreateGoodsTransferLineDto } from './dto/create-goods-transfer.dto.js';
import { UpdateGoodsTransferDto } from './dto/update-goods-transfer.dto.js';
import { DispatchGoodsTransferDto } from './dto/dispatch-goods-transfer.dto.js';
import { QueryGoodsTransferDto } from './dto/query-goods-transfer.dto.js';
import { TransferStatus, SystemRole } from '../../../generated/prisma/client.js';
import { Decimal } from '@prisma/client/runtime/client';
import { PaginatedResult } from '../../common/dto/pagination.dto.js';
import { TRANSFER_STATUS_TRANSITIONS } from '../../common/constants/status-transitions.constant.js';

@Injectable()
export class GoodsTransferService {
  private readonly include = {
    lines: {
      include: {
        product: true,
        uom: true,
        goodsReceiptLine: {
          include: { goodsReceipt: { select: { id: true, receiptNumber: true } } },
        },
      },
    },
    fromWarehouse: true,
    toWarehouse: true,
    carrierSupplier: { select: { id: true, name: true } },
    dispatchedBy: { select: { id: true, name: true } },
  } as const;

  constructor(
    private prisma: PrismaService,
    private refGenerator: ReferenceNumberGenerator,
    private statusTransition: StatusTransitionService,
    private supplierService: SupplierService,
  ) {}

  async create(tenantId: string, dto: CreateGoodsTransferDto) {
    if (dto.fromWarehouseId === dto.toWarehouseId) {
      throw new BadRequestException('Source and destination warehouses must be different');
    }
    if (dto.carrierSupplierId) {
      await this.supplierService.assertExpeditionSupplier(tenantId, dto.carrierSupplierId);
    }

    return this.prisma.$transaction(async (tx) => {
      const [fromWarehouse, toWarehouse] = await Promise.all([
        tx.warehouse.findFirst({ where: { id: dto.fromWarehouseId, tenantId, deletedAt: null } }),
        tx.warehouse.findFirst({ where: { id: dto.toWarehouseId, tenantId, deletedAt: null } }),
      ]);
      if (!fromWarehouse) throw new NotFoundException('Source warehouse not found');
      if (!toWarehouse) throw new NotFoundException('Destination warehouse not found');

      const branchId = dto.branchId ?? fromWarehouse.branchId ?? toWarehouse.branchId;
      if (!branchId) {
        throw new BadRequestException('Unable to determine branch for this transfer. Please provide branchId.');
      }

      await this.assertReceiptLinesInWarehouse(tx, tenantId, dto.fromWarehouseId, dto.lines);

      const transferNumber = await this.refGenerator.generate(tx, tenantId, 'GT');

      const lines = dto.lines.map((line) => ({
        productId: line.productId,
        uomId: line.uomId,
        goodsReceiptLineId: line.goodsReceiptLineId,
        quantitySent: new Decimal(line.quantitySent),
        quantityReceived: line.quantityReceived ? new Decimal(line.quantityReceived) : new Decimal(0),
        quantityDamaged: line.quantityDamaged ? new Decimal(line.quantityDamaged) : new Decimal(0),
        valueAmount: line.valueAmount ? new Decimal(line.valueAmount) : new Decimal(0),
        notes: line.notes,
      }));

      return tx.goodsTransfer.create({
        data: {
          tenantId,
          branchId,
          transferNumber,
          fromWarehouseId: dto.fromWarehouseId,
          toWarehouseId: dto.toWarehouseId,
          transferDate: new Date(dto.transferDate),
          estimatedReceiptDate: dto.estimatedReceiptDate ? new Date(dto.estimatedReceiptDate) : null,
          reason: dto.reason,
          notes: dto.notes,
          carrierSupplierId: dto.carrierSupplierId,
          deliveryNoteNumber: dto.deliveryNoteNumber,
          lines: { create: lines },
        },
        include: this.include,
      });
    });
  }

  async findAll(tenantId: string, query: QueryGoodsTransferDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(query.status && { status: query.status }),
      ...(query.search && {
        transferNumber: { contains: query.search, mode: 'insensitive' as const },
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.goodsTransfer.findMany({
        where,
        include: this.include,
        orderBy: { createdAt: 'desc' },
        skip: query.skip,
        take: query.limit,
      }),
      this.prisma.goodsTransfer.count({ where }),
    ]);
    return new PaginatedResult(data, total, query.page, query.limit);
  }

  async findOne(tenantId: string, id: string) {
    const gt = await this.prisma.goodsTransfer.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!gt) throw new NotFoundException('Goods transfer not found');
    return gt;
  }

  async update(tenantId: string, id: string, dto: UpdateGoodsTransferDto) {
    const gt = await this.findOne(tenantId, id);
    if (gt.status !== TransferStatus.PREPARING) {
      throw new BadRequestException('Transfer can only be edited while preparing');
    }
    if (dto.carrierSupplierId) {
      await this.supplierService.assertExpeditionSupplier(tenantId, dto.carrierSupplierId);
    }
    const data: Record<string, any> = {};
    if (dto.branchId !== undefined) data.branchId = dto.branchId;
    if (dto.fromWarehouseId !== undefined) data.fromWarehouseId = dto.fromWarehouseId;
    if (dto.toWarehouseId !== undefined) data.toWarehouseId = dto.toWarehouseId;
    if (dto.transferDate !== undefined) data.transferDate = new Date(dto.transferDate);
    if (dto.estimatedReceiptDate !== undefined)
      data.estimatedReceiptDate = dto.estimatedReceiptDate ? new Date(dto.estimatedReceiptDate) : null;
    if (dto.reason !== undefined) data.reason = dto.reason;
    if (dto.notes !== undefined) data.notes = dto.notes;
    if (dto.carrierSupplierId !== undefined) data.carrierSupplierId = dto.carrierSupplierId || null;
    if (dto.deliveryNoteNumber !== undefined) data.deliveryNoteNumber = dto.deliveryNoteNumber || null;
    return this.prisma.goodsTransfer.update({ where: { id }, data, include: this.include });
  }

  async remove(tenantId: string, id: string) {
    await this.findOne(tenantId, id);
    return this.prisma.goodsTransfer.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }

  /**
   * PREPARING -> IN_TRANSIT. The only way into IN_TRANSIT, so dispatchedAt /
   * dispatchedById are always populated. No stock movement here (see spec non-goal).
   */
  async dispatch(tenantId: string, id: string, dto: DispatchGoodsTransferDto, userId: string, userRole: SystemRole) {
    const gt = await this.findOne(tenantId, id);
    this.statusTransition.validateTransition(gt.status, TransferStatus.IN_TRANSIT, userRole, TRANSFER_STATUS_TRANSITIONS);
    if (gt.lines.length === 0) {
      throw new BadRequestException('Transfer has no lines');
    }
    const carrierSupplierId = dto.carrierSupplierId ?? gt.carrierSupplierId;
    if (!carrierSupplierId) {
      throw new BadRequestException('Expedition is required to dispatch');
    }
    await this.supplierService.assertExpeditionSupplier(tenantId, carrierSupplierId);

    return this.prisma.goodsTransfer.update({
      where: { id },
      data: {
        status: TransferStatus.IN_TRANSIT,
        carrierSupplierId,
        deliveryNoteNumber: dto.deliveryNoteNumber ?? gt.deliveryNoteNumber,
        dispatchedAt: new Date(),
        dispatchedById: userId,
      },
      include: this.include,
    });
  }

  async transitionStatus(tenantId: string, id: string, targetStatus: TransferStatus, userRole: SystemRole) {
    if (targetStatus === TransferStatus.IN_TRANSIT) {
      throw new BadRequestException('Use dispatch endpoint');
    }
    const gt = await this.findOne(tenantId, id);
    this.statusTransition.validateTransition(gt.status, targetStatus, userRole, TRANSFER_STATUS_TRANSITIONS);

    const data: Record<string, any> = { status: targetStatus };
    if (targetStatus === TransferStatus.RECEIVED) {
      data.actualReceiptDate = new Date();
    }
    return this.prisma.goodsTransfer.update({ where: { id }, data, include: this.include });
  }

  /** Every referenced BPB line must belong to the tenant and to the source warehouse. */
  private async assertReceiptLinesInWarehouse(
    tx: Parameters<Parameters<PrismaService['$transaction']>[0]>[0],
    tenantId: string,
    fromWarehouseId: string,
    lines: CreateGoodsTransferLineDto[],
  ) {
    const ids = Array.from(new Set(lines.map((l) => l.goodsReceiptLineId).filter((v): v is string => !!v)));
    if (ids.length === 0) return;
    const found = await tx.goodsReceiptLine.count({
      where: {
        id: { in: ids },
        goodsReceipt: { tenantId, deletedAt: null, warehouseId: fromWarehouseId },
      },
    });
    if (found !== ids.length) {
      throw new BadRequestException('One or more goods receipt lines do not belong to the source warehouse');
    }
  }
}
```

Note: the generic `APPROVAL_MATRIX.IN_TRANSIT` currently allows STAFF. `validateTransition` in `dispatch()` therefore passes for STAFF at the service level; the **controller** guard (Step 5) is what enforces MANAGER+. Don't change the matrix — `IN_TRANSIT` is also used by `InternalTradeStatus`.

- [ ] **Step 5: Controller**

Replace `goods-transfer.controller.ts` with:

```ts
import {
  Controller, Get, Post, Patch, Delete, Body, Param, Query, UseGuards,
} from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { GoodsTransferService } from './goods-transfer.service.js';
import { CreateGoodsTransferDto } from './dto/create-goods-transfer.dto.js';
import { UpdateGoodsTransferDto } from './dto/update-goods-transfer.dto.js';
import { TransitionTransferStatusDto } from './dto/transition-status.dto.js';
import { DispatchGoodsTransferDto } from './dto/dispatch-goods-transfer.dto.js';
import { QueryGoodsTransferDto } from './dto/query-goods-transfer.dto.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { RolesGuard } from '../../common/guards/roles.guard.js';
import { Roles } from '../../common/decorators/roles.decorator.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { CurrentUser } from '../../common/decorators/current-user.decorator.js';
import type { JwtPayload } from '../../common/interfaces/request-with-user.interface.js';
import { TransferStatus, SystemRole } from '../../../generated/prisma/client.js';

@ApiTags('Goods Transfers')
@ApiBearerAuth()
@Controller('goods-transfers')
@UseGuards(JwtAuthGuard, RolesGuard)
export class GoodsTransferController {
  constructor(private readonly service: GoodsTransferService) {}

  @ApiOperation({ summary: 'Create a new goods transfer (status PREPARING)' })
  @Post()
  create(@Body() dto: CreateGoodsTransferDto, @CurrentTenant() tenantId: string) {
    return this.service.create(tenantId, dto);
  }

  @ApiOperation({ summary: 'List all goods transfers' })
  @Get()
  findAll(@CurrentTenant() tenantId: string, @Query() query: QueryGoodsTransferDto) {
    return this.service.findAll(tenantId, query);
  }

  @ApiOperation({ summary: 'Get a goods transfer by ID' })
  @Get(':id')
  findOne(@Param('id') id: string, @CurrentTenant() tenantId: string) {
    return this.service.findOne(tenantId, id);
  }

  @ApiOperation({ summary: 'Update a goods transfer (PREPARING only)' })
  @Patch(':id')
  update(
    @Param('id') id: string,
    @Body() dto: UpdateGoodsTransferDto,
    @CurrentTenant() tenantId: string,
  ) {
    return this.service.update(tenantId, id, dto);
  }

  @ApiOperation({ summary: 'Delete a goods transfer (soft delete)' })
  @Delete(':id')
  remove(@Param('id') id: string, @CurrentTenant() tenantId: string) {
    return this.service.remove(tenantId, id);
  }

  @ApiOperation({ summary: 'Dispatch: PREPARING -> IN_TRANSIT with carrier and delivery note' })
  @Post(':id/dispatch')
  @Roles(SystemRole.SUPER_ADMIN, SystemRole.TENANT_ADMIN, SystemRole.MANAGER)
  dispatch(
    @Param('id') id: string,
    @Body() dto: DispatchGoodsTransferDto,
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.service.dispatch(tenantId, id, dto, user.sub, user.role as SystemRole);
  }

  @ApiOperation({ summary: 'Transition goods transfer status (IN_TRANSIT not allowed here — use dispatch)' })
  @Patch(':id/status')
  transitionStatus(
    @Param('id') id: string,
    @Body() dto: TransitionTransferStatusDto,
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
  ) {
    return this.service.transitionStatus(
      tenantId,
      id,
      dto.targetStatus as TransferStatus,
      user.role as SystemRole,
    );
  }
}
```

`RolesGuard` returns `true` when no `@Roles` metadata is present, so adding it at class level only affects `dispatch`.

- [ ] **Step 6: Module imports `MasterDataModule`**

```ts
import { Module } from '@nestjs/common';
import { GoodsTransferService } from './goods-transfer.service.js';
import { GoodsTransferController } from './goods-transfer.controller.js';
import { ReferenceNumberGenerator } from '../../common/utils/reference-number.generator.js';
import { StatusTransitionService } from '../../common/services/status-transition.service.js';
import { MasterDataModule } from '../master-data/master-data.module.js';

@Module({
  imports: [MasterDataModule],
  controllers: [GoodsTransferController],
  providers: [
    GoodsTransferService,
    ReferenceNumberGenerator,
    StatusTransitionService,
  ],
  exports: [GoodsTransferService],
})
export class TransferModule {}
```

Verify the module file name: `ls src/modules/master-data/*.module.ts`.

- [ ] **Step 7: Build**

```bash
cd breeding-app && npm run build
```

Expected: success.

- [ ] **Step 8: Curl verification**

Reset DB (`npx prisma migrate reset --force`), start dev server, then:

```bash
API=http://localhost:3002/api
login() { curl -s -X POST $API/auth/login -H 'Content-Type: application/json' -d "{\"email\":\"$1\",\"password\":\"password123\"}" | python3 -c 'import sys,json;print(json.load(sys.stdin)["data"]["accessToken"])'; }
ADMIN=$(login admin@demo.farm); STAFF=$(login staff@demo.farm)
j() { python3 -c "import sys,json;d=json.load(sys.stdin);print($1)"; }

# ids
EXP=$(curl -s "$API/suppliers?expedition=true" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["id"]')
FEED=$(curl -s "$API/suppliers?search=Japfa" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["id"]')
WH=$(curl -s "$API/warehouses?limit=50" -H "Authorization: Bearer $ADMIN" | j '[w["id"] for w in d["data"]["data"]][:2]')
echo $WH   # copy the two ids into FROM / TO below
FROM=<first-id>; TO=<second-id>
PROD=$(curl -s "$API/products?limit=1" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["id"]')
UOM=$(curl -s "$API/products?limit=1" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["uomId"]')

# 1. create -> PREPARING
GT=$(curl -s -X POST $API/goods-transfers -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d "{\"fromWarehouseId\":\"$FROM\",\"toWarehouseId\":\"$TO\",\"transferDate\":\"2026-09-13\",\"lines\":[{\"productId\":\"$PROD\",\"uomId\":\"$UOM\",\"quantitySent\":\"10\"}]}" | j 'd["data"]["id"]+" "+d["data"]["status"]')
echo $GT          # expected: <id> PREPARING
GTID=${GT%% *}

# 2. non-expedition carrier rejected on create
curl -s -X POST $API/goods-transfers -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d "{\"fromWarehouseId\":\"$FROM\",\"toWarehouseId\":\"$TO\",\"transferDate\":\"2026-09-13\",\"carrierSupplierId\":\"$FEED\",\"lines\":[{\"productId\":\"$PROD\",\"uomId\":\"$UOM\",\"quantitySent\":\"1\"}]}" | j 'd["statusCode"]'   # 400

# 3. generic status -> IN_TRANSIT blocked
curl -s -X PATCH $API/goods-transfers/$GTID/status -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' -d '{"targetStatus":"IN_TRANSIT"}' | j 'd["statusCode"]'   # 400

# 4. dispatch without carrier -> 400
curl -s -X POST $API/goods-transfers/$GTID/dispatch -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' -d '{}' | j 'd["statusCode"]'   # 400

# 5. STAFF dispatch -> 403
curl -s -X POST $API/goods-transfers/$GTID/dispatch -H "Authorization: Bearer $STAFF" -H 'Content-Type: application/json' -d "{\"carrierSupplierId\":\"$EXP\"}" | j 'd["statusCode"]'   # 403

# 6. dispatch happy path
curl -s -X POST $API/goods-transfers/$GTID/dispatch -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' -d "{\"carrierSupplierId\":\"$EXP\",\"deliveryNoteNumber\":\"SJ-001\"}" | j 'd["data"]["status"]+" "+d["data"]["carrierSupplier"]["name"]+" "+d["data"]["dispatchedBy"]["name"]+" "+str(d["data"]["dispatchedAt"] is not None)'
# expected: IN_TRANSIT CV Restu Ibu Ekspedisi Admin True   (or Agus Purwanto... depending on EXP)

# 7. PATCH after dispatch -> 400 ; dispatch again -> 400
curl -s -X PATCH $API/goods-transfers/$GTID -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' -d '{"notes":"x"}' | j 'd["statusCode"]'   # 400
curl -s -X POST $API/goods-transfers/$GTID/dispatch -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' -d '{}' | j 'd["statusCode"]'   # 400

# 8. status filter
curl -s "$API/goods-transfers?status=IN_TRANSIT" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["meta"]["total"]'   # >= 1
curl -s "$API/goods-transfers?status=BOGUS" -H "Authorization: Bearer $ADMIN" | j 'd["statusCode"]'   # 400

# 9. no inventory movement was created by any of the above
curl -s "$API/inventory-movements?limit=100" -H "Authorization: Bearer $ADMIN" | j '[m for m in d["data"]["data"] if m.get("movementSource")=="TRANSFER"]'   # []
```

If the `/inventory-movements` route name differs, check `src/modules/inventory/controllers/` for the movement controller path. If `/products` doesn't expose `uomId`, take `UOM` from `/unit-of-measures?limit=1`.

- [ ] **Step 9: Commit**

```bash
cd breeding-app && git add src/modules/transfer src/common/constants/status-transitions.constant.ts && git commit -m "feat(transfer): dispatch endpoint, carrier validation, PREPARING guardrails, BPB line ref

- POST /goods-transfers/:id/dispatch (MANAGER+) is the only path to IN_TRANSIT
- PATCH only while PREPARING; carrier must be an expedition supplier
- lines[].goodsReceiptLineId validated against the source warehouse
- GET /goods-transfers?status= filter

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 10: Frontend — types, constants, i18n, transfer **create** page

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts:42` (TransferStatus), `:454-484` (GoodsTransferLine, GoodsTransfer)
- Modify: `breeding-dashboard/src/lib/constants.ts:214-217`
- Modify: `breeding-dashboard/messages/en.json`, `messages/id.json` (`goodsTransfers`, `goodsTransferNew`)
- Modify: `breeding-dashboard/src/app/(dashboard)/goods-transfers/new/page.tsx`

**Interfaces:**
- Consumes: `SupplierCombobox expeditionOnly` (Task 7), `LineItemsField` props `showGoodsReceiptRef showLineNotes warehouseId` (existing), `POST /goods-transfers` shape (Task 9).
- Produces: types `GoodsTransfer.carrierSupplierId? carrierSupplier? deliveryNoteNumber? dispatchedAt? dispatchedBy?`, `GoodsTransferLine.goodsReceiptLineId? goodsReceiptLine?`; i18n keys listed in Step 3.

- [ ] **Step 1: Types**

```ts
export type TransferStatus = "PREPARING" | "IN_TRANSIT" | "RECEIVED" | "PARTIAL" | "DAMAGED_IN_TRANSIT" | "CANCELLED";
```

```ts
export interface GoodsTransferLine {
  id: string;
  productId: string;
  product?: Product;
  uomId: string;
  uom?: UnitOfMeasure;
  quantitySent: string;
  quantityReceived?: string;
  quantityDamaged?: string;
  valueAmount?: string;
  notes?: string;
  goodsReceiptLineId?: string;
  goodsReceiptLine?: { id: string; goodsReceipt?: { id: string; receiptNumber: string } };
}

export interface GoodsTransfer {
  id: string;
  transferNumber: string;
  branchId: string;
  fromWarehouseId: string;
  toWarehouseId: string;
  fromWarehouse?: Warehouse;
  toWarehouse?: Warehouse;
  transferDate?: string;
  estimatedReceiptDate?: string;
  actualReceiptDate?: string;
  reason?: string;
  status: TransferStatus;
  notes?: string;
  carrierSupplierId?: string;
  carrierSupplier?: Pick<Supplier, "id" | "name">;
  deliveryNoteNumber?: string;
  dispatchedAt?: string;
  dispatchedById?: string;
  dispatchedBy?: Pick<User, "id" | "name">;
  lines: GoodsTransferLine[];
  createdAt: string;
  updatedAt: string;
}
```

- [ ] **Step 2: Constants**

```ts
export const TRANSFER_STATUS_TRANSITIONS: Record<string, string[]> = {
  PREPARING: ["CANCELLED"],
  IN_TRANSIT: ["RECEIVED", "PARTIAL", "DAMAGED_IN_TRANSIT", "CANCELLED"],
  PARTIAL: ["RECEIVED"],
};
```

`IN_TRANSIT` is deliberately not offered from `PREPARING` here — the Dispatch button (Task 11) handles it.

- [ ] **Step 3: i18n**

`en.json` → `"goodsTransferNew"` add:

```json
    "carrier": "Expedition",
    "deliveryNoteNumber": "Delivery note no.",
    "deliveryNotePlaceholder": "Number on the carrier's surat jalan (optional)",
    "expeditionSection": "Expedition"
```

`en.json` → `"goodsTransfers"` add:

```json
    "expedition": "Expedition",
    "carrier": "Expedition",
    "deliveryNoteNumber": "Delivery note no.",
    "dispatchedAt": "Dispatched at",
    "dispatchedBy": "Dispatched by",
    "notDispatched": "Not dispatched yet",
    "dispatch": "Dispatch",
    "dispatchTitle": "Dispatch transfer",
    "dispatchDescription": "Confirm the expedition and delivery note. Status becomes In Transit.",
    "dispatching": "Dispatching...",
    "dispatchSuccess": "Transfer dispatched",
    "carrierRequired": "Select an expedition before dispatching",
    "sourceReceipt": "Source BPB",
    "statusFilterAll": "All statuses"
```

`id.json` → `"goodsTransferNew"`:

```json
    "carrier": "Ekspedisi",
    "deliveryNoteNumber": "No. surat jalan",
    "deliveryNotePlaceholder": "Nomor di surat jalan ekspedisi (opsional)",
    "expeditionSection": "Ekspedisi"
```

`id.json` → `"goodsTransfers"`:

```json
    "expedition": "Ekspedisi",
    "carrier": "Ekspedisi",
    "deliveryNoteNumber": "No. surat jalan",
    "dispatchedAt": "Diberangkatkan pada",
    "dispatchedBy": "Diberangkatkan oleh",
    "notDispatched": "Belum diberangkatkan",
    "dispatch": "Berangkatkan",
    "dispatchTitle": "Berangkatkan pindah barang",
    "dispatchDescription": "Pastikan ekspedisi dan surat jalan. Status menjadi Dalam Perjalanan.",
    "dispatching": "Memproses...",
    "dispatchSuccess": "Pindah barang diberangkatkan",
    "carrierRequired": "Pilih ekspedisi dulu sebelum diberangkatkan",
    "sourceReceipt": "BPB asal",
    "statusFilterAll": "Semua status"
```

- [ ] **Step 4: Create page**

In `goods-transfers/new/page.tsx`:

Add import: `import { SupplierCombobox } from "@/components/forms/supplier-combobox";`

Extend form state:

```ts
  const [form, setForm] = useState({
    fromWarehouseId: "",
    toWarehouseId: "",
    transferDate: "",
    notes: "",
    carrierSupplierId: "",
    deliveryNoteNumber: "",
  });
```

When the source warehouse changes, clear every line's BPB ref (it belongs to the old warehouse). Replace the `fromWarehouse` combobox `onChange`:

```tsx
                <WarehouseGroupedCombobox
                  value={form.fromWarehouseId}
                  onChange={(id) => {
                    setForm((prev) => ({ ...prev, fromWarehouseId: id }));
                    setLines((prev) => prev.map((l) => ({ ...l, goodsReceiptLineId: undefined })));
                  }}
                  groupLabels={groupLabels}
                />
```

Add a second card between "transfer details" and "line items":

```tsx
        <Card>
          <CardHeader>
            <CardTitle>{t("expeditionSection")}</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid gap-6 md:grid-cols-2">
              <div className="space-y-2">
                <Label>{t("carrier")}</Label>
                <SupplierCombobox
                  value={form.carrierSupplierId}
                  onChange={(id) => setForm((prev) => ({ ...prev, carrierSupplierId: id }))}
                  expeditionOnly
                />
              </div>
              <div className="space-y-2">
                <Label htmlFor="deliveryNoteNumber">{t("deliveryNoteNumber")}</Label>
                <Input
                  id="deliveryNoteNumber"
                  maxLength={100}
                  value={form.deliveryNoteNumber}
                  onChange={(e) => setForm({ ...form, deliveryNoteNumber: e.target.value })}
                  placeholder={t("deliveryNotePlaceholder")}
                />
              </div>
            </div>
          </CardContent>
        </Card>
```

Line items — enable BPB ref + notes:

```tsx
            <LineItemsField
              lines={lines}
              onChange={setLines}
              showPrice={false}
              showGoodsReceiptRef
              showLineNotes
              warehouseId={form.fromWarehouseId}
            />
```

Submit body:

```ts
      const body = {
        fromWarehouseId: form.fromWarehouseId,
        toWarehouseId: form.toWarehouseId,
        transferDate: form.transferDate,
        notes: form.notes || undefined,
        carrierSupplierId: form.carrierSupplierId || undefined,
        deliveryNoteNumber: form.deliveryNoteNumber.trim() || undefined,
        lines: validLines.map((l) => ({
          productId: l.productId,
          quantitySent: String(l.quantity),
          uomId: l.uomId,
          goodsReceiptLineId: l.goodsReceiptLineId || undefined,
          notes: l.notes || undefined,
        })),
      };
```

- [ ] **Step 5: Type-check, build, smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Open `/goods-transfers/new`: pick source warehouse `Main Warehouse` (the seeded receipt landed there), product `Corn Feed` → the BPB combobox on the line lists `GR-…`. Pick expedition `CV Restu Ibu Ekspedisi`, save. Expected: redirect to detail with status badge `PREPARING`.

- [ ] **Step 6: Commit**

```bash
cd breeding-dashboard && git add src/types/api.ts src/lib/constants.ts messages "src/app/(dashboard)/goods-transfers/new/page.tsx" && git commit -m "feat(transfer): carrier, delivery note and BPB ref on transfer create form

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 11: Frontend — transfer **detail** page: expedition card, dispatch dialog, BPB column

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/goods-transfers/[id]/dispatch-dialog.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/goods-transfers/[id]/page.tsx`

**Interfaces:**
- Consumes: `POST /goods-transfers/:id/dispatch` (Task 9), `useAuth().user.role`, i18n keys (Task 10), `SupplierCombobox expeditionOnly` (Task 7).
- Produces: `<DispatchDialog open onOpenChange transfer onSuccess />`.

- [ ] **Step 1: Dispatch dialog component**

Create `dispatch-dialog.tsx`:

```tsx
"use client";

import { useEffect, useState } from "react";
import { useTranslations } from "next-intl";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { SupplierCombobox } from "@/components/forms/supplier-combobox";
import { fetchApi } from "@/lib/api";
import { GoodsTransfer } from "@/types/api";

interface DispatchDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  transfer: GoodsTransfer;
  onSuccess: () => void;
}

export function DispatchDialog({ open, onOpenChange, transfer, onSuccess }: DispatchDialogProps) {
  const t = useTranslations("goodsTransfers");
  const tc = useTranslations("common");
  const [carrierSupplierId, setCarrierSupplierId] = useState(transfer.carrierSupplierId ?? "");
  const [deliveryNoteNumber, setDeliveryNoteNumber] = useState(transfer.deliveryNoteNumber ?? "");
  const [isSubmitting, setIsSubmitting] = useState(false);

  // Re-prefill every time the dialog opens (transfer may have been edited meanwhile).
  useEffect(() => {
    if (open) {
      setCarrierSupplierId(transfer.carrierSupplierId ?? "");
      setDeliveryNoteNumber(transfer.deliveryNoteNumber ?? "");
    }
  }, [open, transfer.carrierSupplierId, transfer.deliveryNoteNumber]);

  async function handleDispatch() {
    if (!carrierSupplierId) {
      toast.error(t("carrierRequired"));
      return;
    }
    setIsSubmitting(true);
    try {
      await fetchApi(`/goods-transfers/${transfer.id}/dispatch`, {
        method: "POST",
        body: JSON.stringify({
          carrierSupplierId,
          deliveryNoteNumber: deliveryNoteNumber.trim() || undefined,
        }),
      });
      toast.success(t("dispatchSuccess"));
      onOpenChange(false);
      onSuccess();
    } catch (err) {
      toast.error(err instanceof Error ? err.message : tc("entityUpdateFailed", { entity: t("entity") }));
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>{t("dispatchTitle")}</DialogTitle>
          <DialogDescription>{t("dispatchDescription")}</DialogDescription>
        </DialogHeader>
        <div className="space-y-4">
          <div className="space-y-2">
            <Label>{t("carrier")} *</Label>
            <SupplierCombobox value={carrierSupplierId} onChange={setCarrierSupplierId} expeditionOnly />
            {transfer.carrierSupplier && carrierSupplierId === transfer.carrierSupplierId && (
              <p className="text-xs text-muted-foreground">{transfer.carrierSupplier.name}</p>
            )}
          </div>
          <div className="space-y-2">
            <Label htmlFor="dispatch-delivery-note">{t("deliveryNoteNumber")}</Label>
            <Input
              id="dispatch-delivery-note"
              maxLength={100}
              value={deliveryNoteNumber}
              onChange={(e) => setDeliveryNoteNumber(e.target.value)}
            />
          </div>
        </div>
        <DialogFooter>
          <Button variant="outline" onClick={() => onOpenChange(false)} disabled={isSubmitting}>
            {tc("cancel")}
          </Button>
          <Button onClick={handleDispatch} disabled={isSubmitting}>
            {isSubmitting ? t("dispatching") : t("dispatch")}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 2: Detail page changes**

In `goods-transfers/[id]/page.tsx`:

Imports to add:

```ts
import { useAuth } from "@/hooks/use-auth";
import { Truck } from "lucide-react";
import { DispatchDialog } from "./dispatch-dialog";
```

State + role:

```ts
  const { user } = useAuth();
  const canDispatch = ["SUPER_ADMIN", "TENANT_ADMIN", "MANAGER"].includes(user?.role ?? "");
  const [showDispatch, setShowDispatch] = useState(false);
```

Place these **above** the early `if (isLoading) return …` lines (hooks must run unconditionally).

In the `PageHeader` actions, before `<StatusAction …>`:

```tsx
            {gt.status === "PREPARING" && canDispatch && (
              <Button onClick={() => setShowDispatch(true)}>
                <Truck className="mr-2 h-4 w-4" />
                {t('dispatch')}
              </Button>
            )}
```

Add an **Expedition card** after the two-column grid and before the line-items card:

```tsx
      <Card>
        <CardHeader>
          <CardTitle>{t('expedition')}</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          <div className="flex justify-between">
            <span className="text-muted-foreground">{t('carrier')}</span>
            <span className="font-medium">{gt.carrierSupplier?.name || "—"}</span>
          </div>
          <Separator />
          <div className="flex justify-between">
            <span className="text-muted-foreground">{t('deliveryNoteNumber')}</span>
            <span className="font-mono">{gt.deliveryNoteNumber || "—"}</span>
          </div>
          <Separator />
          {gt.dispatchedAt ? (
            <>
              <div className="flex justify-between">
                <span className="text-muted-foreground">{t('dispatchedAt')}</span>
                <span>{formatDate(gt.dispatchedAt)}</span>
              </div>
              <Separator />
              <div className="flex justify-between">
                <span className="text-muted-foreground">{t('dispatchedBy')}</span>
                <span>{gt.dispatchedBy?.name || "—"}</span>
              </div>
            </>
          ) : (
            <p className="text-sm text-muted-foreground">{t('notDispatched')}</p>
          )}
        </CardContent>
      </Card>
```

Line table — add a BPB column after the product column:

```tsx
                  <TableHead>{t('sourceReceipt')}</TableHead>
```

```tsx
                    <TableCell>
                      {line.goodsReceiptLine?.goodsReceipt ? (
                        <Link
                          href={`/goods-receipts/${line.goodsReceiptLine.goodsReceipt.id}`}
                          className="text-primary underline"
                        >
                          {line.goodsReceiptLine.goodsReceipt.receiptNumber}
                        </Link>
                      ) : (
                        "—"
                      )}
                    </TableCell>
```

Mount the dialog before the closing `</div>` of the page:

```tsx
      <DispatchDialog
        open={showDispatch}
        onOpenChange={setShowDispatch}
        transfer={gt}
        onSuccess={refetch}
      />
```

- [ ] **Step 3: Type-check, build, smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

Log in as `manager@demo.farm`: open the `PREPARING` transfer from Task 10 → Dispatch button visible; click → dialog prefilled with the carrier; submit → toast, badge `IN_TRANSIT`, expedition card shows dispatched-at/by, Dispatch button gone, `StatusAction` now offers Received/Partial/…. Log in as `staff@demo.farm`: create a transfer → no Dispatch button.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard && git add "src/app/(dashboard)/goods-transfers/[id]" && git commit -m "feat(transfer): expedition card, dispatch dialog and BPB column on transfer detail

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 12: Frontend — transfer list status filter

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/goods-transfers/page.tsx`

**Interfaces:**
- Consumes: `GET /goods-transfers?status=` (Task 9), i18n `goodsTransfers.statusFilterAll` (Task 10).

- [ ] **Step 1: Add a status select bound to the URL**

Imports:

```ts
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
```

State (next to `search`):

```ts
  const [status, setStatus] = useQueryState("status", { defaultValue: "" });
```

Query:

```ts
  const { data, meta, isLoading } = usePaginated<GoodsTransfer>(
    "/goods-transfers",
    { page, limit: 20, search, extra: { status } }
  );
```

`fetchPaginated` skips empty `extra` values, so `""` means "all".

Render the select above the `DataTable` (inside the page's `space-y-6` wrapper, after `PageHeader`):

```tsx
      <div className="flex justify-end">
        <Select
          value={status || "ALL"}
          onValueChange={(v) => {
            setStatus(v === "ALL" ? "" : v);
            setPage(1);
          }}
        >
          <SelectTrigger className="w-[220px]">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="ALL">{t('statusFilterAll')}</SelectItem>
            {["PREPARING", "IN_TRANSIT", "PARTIAL", "RECEIVED", "DAMAGED_IN_TRANSIT", "CANCELLED"].map((s) => (
              <SelectItem key={s} value={s}>{s.replace(/_/g, " ")}</SelectItem>
            ))}
          </SelectContent>
        </Select>
      </div>
```

- [ ] **Step 2: Type-check, build, smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

`/goods-transfers?status=PREPARING` shows only preparing transfers; "All statuses" clears the param.

- [ ] **Step 3: Commit**

```bash
cd breeding-dashboard && git add "src/app/(dashboard)/goods-transfers/page.tsx" && git commit -m "feat(transfer): status filter on transfer list

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

# Phase C — Goods receipt: carrier + delivery note at header

### Task 13: Prisma schema + migration + backend for goods receipt carrier

**Files:**
- Modify: `breeding-app/prisma/schema.prisma:566-` (Supplier back-relation), `:1160-1186` (GoodsReceipt)
- Create (generated): `prisma/migrations/<timestamp>_add_goods_receipt_carrier/`
- Modify: `breeding-app/src/modules/procurement/dto/create-goods-receipt.dto.ts`
- Modify: `breeding-app/src/modules/procurement/goods-receipt.service.ts:21-73, 75-98`
- Modify: `breeding-app/src/modules/procurement/procurement.module.ts:12`

**Interfaces:**
- Consumes: `SupplierService.assertExpeditionSupplier` (Task 3).
- Produces: `POST /goods-receipts` body adds `carrierSupplierId?`, `deliveryNoteNumber?`; responses include `carrierSupplier { id, name }` and `deliveryNoteNumber`.

- [ ] **Step 1: Schema**

In `model GoodsReceipt`, add after `notes`:

```prisma
  carrierSupplierId  String?       @map("carrier_supplier_id")
  deliveryNoteNumber String?       @map("delivery_note_number")
```

add relation after `warehouse`:

```prisma
  carrierSupplier Supplier? @relation("ReceiptCarrier", fields: [carrierSupplierId], references: [id])
```

add index before `@@map`:

```prisma
  @@index([carrierSupplierId])
```

In `model Supplier`, add after `goodsTransfersAsCarrier`:

```prisma
  goodsReceiptsAsCarrier  GoodsReceipt[]  @relation("ReceiptCarrier")
```

Note `Supplier.goodsReceipts GoodsReceipt[]` (the existing, unnamed relation for `GoodsReceipt.supplier`) stays. Prisma requires the existing `GoodsReceipt.supplier` relation to be **named** now that there are two relations to `Supplier`: change it to

```prisma
  supplier      Supplier      @relation("ReceiptSupplier", fields: [supplierId], references: [id])
```

and `Supplier.goodsReceipts  GoodsReceipt[] @relation("ReceiptSupplier")`. Naming an existing relation changes no SQL.

- [ ] **Step 2: Migrate + generate**

```bash
cd breeding-app && npx prisma migrate dev --name add_goods_receipt_carrier && npx prisma generate
```

Expected: two nullable columns, one index, one FK on `goods_receipts`. No table rewrite.

- [ ] **Step 3: DTO**

In `create-goods-receipt.dto.ts`, add `MaxLength` to the class-validator import; in `CreateGoodsReceiptDto` after `notes`:

```ts
  @ApiPropertyOptional({ format: 'uuid', description: 'Expedition supplier that delivered the goods' })
  @IsOptional()
  @IsUUID()
  carrierSupplierId?: string;

  @ApiPropertyOptional({ description: 'Delivery note (surat jalan) number', maxLength: 100 })
  @IsOptional()
  @IsString()
  @MaxLength(100)
  deliveryNoteNumber?: string;
```

- [ ] **Step 4: Service**

Inject `SupplierService`:

```ts
import { SupplierService } from '../master-data/services/supplier.service.js';
// constructor:
    private supplierService: SupplierService,
```

At the top of `create()` (before `$transaction`):

```ts
    if (dto.carrierSupplierId) {
      await this.supplierService.assertExpeditionSupplier(tenantId, dto.carrierSupplierId);
    }
```

In the `tx.goodsReceipt.create` data, after `notes: dto.notes,`:

```ts
          carrierSupplierId: dto.carrierSupplierId,
          deliveryNoteNumber: dto.deliveryNoteNumber,
```

In both `findAll` and `findOne`, change the `include` to:

```ts
include: {
  lines: { include: { product: true, uom: true } },
  purchaseOrder: true,
  supplier: true,
  warehouse: true,
  carrierSupplier: { select: { id: true, name: true } },
},
```

- [ ] **Step 5: Module**

`procurement.module.ts`: `imports: [InventoryModule, MasterDataModule]` with `import { MasterDataModule } from '../master-data/master-data.module.js';`.

- [ ] **Step 6: Build + curl**

```bash
cd breeding-app && npm run build
```

Then (reuse the `login`/`j` helpers from Task 9):

```bash
PO=$(curl -s "$API/purchase-orders?limit=1" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["id"]')
WHM=$(curl -s "$API/warehouses?limit=1" -H "Authorization: Bearer $ADMIN" | j 'd["data"]["data"][0]["id"]')
# non-expedition carrier -> 400
curl -s -X POST $API/goods-receipts -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d "{\"purchaseOrderId\":\"$PO\",\"warehouseId\":\"$WHM\",\"receiptDate\":\"2026-09-13\",\"carrierSupplierId\":\"$FEED\",\"lines\":[{\"productId\":\"$PROD\",\"uomId\":\"$UOM\",\"quantitySent\":\"1\",\"quantityReceived\":\"1\",\"quantityDamaged\":\"0\"}]}" | j 'd["statusCode"]'   # 400
# expedition carrier -> 201 with carrierSupplier
curl -s -X POST $API/goods-receipts -H "Authorization: Bearer $ADMIN" -H 'Content-Type: application/json' \
  -d "{\"purchaseOrderId\":\"$PO\",\"warehouseId\":\"$WHM\",\"receiptDate\":\"2026-09-13\",\"carrierSupplierId\":\"$EXP\",\"deliveryNoteNumber\":\"SJ-GR-01\",\"lines\":[{\"productId\":\"$PROD\",\"uomId\":\"$UOM\",\"quantitySent\":\"1\",\"quantityReceived\":\"1\",\"quantityDamaged\":\"0\"}]}" | j 'd["data"]["deliveryNoteNumber"]'   # SJ-GR-01
```

`findOne` includes `carrierSupplier`; verify with `GET /goods-receipts/<id>`.

- [ ] **Step 7: Commit**

```bash
cd breeding-app && git add prisma src/modules/procurement && git commit -m "feat(procurement): carrier and delivery note on goods receipt header

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 14: Frontend — goods receipt create + detail

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts:308-323` (GoodsReceipt)
- Modify: `breeding-dashboard/messages/en.json`, `messages/id.json` (`goodsReceipts`)
- Modify: `breeding-dashboard/src/app/(dashboard)/goods-receipts/new/page.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/goods-receipts/[id]/page.tsx`

**Interfaces:**
- Consumes: `SupplierCombobox expeditionOnly` (Task 7), `POST /goods-receipts` (Task 13).

- [ ] **Step 1: Types**

```ts
export interface GoodsReceipt {
  id: string;
  receiptNumber: string;
  purchaseOrderId: string;
  purchaseOrder?: PurchaseOrder;
  branchId: string;
  supplierId?: string;
  supplier?: Supplier;
  warehouseId: string;
  warehouse?: Warehouse;
  notes?: string;
  carrierSupplierId?: string;
  carrierSupplier?: Pick<Supplier, "id" | "name">;
  deliveryNoteNumber?: string;
  status: ReceiptStatus;
  lines: GoodsReceiptLine[];
  createdAt: string;
  updatedAt: string;
}
```

- [ ] **Step 2: i18n** — `goodsReceipts` namespace

`en.json`:

```json
    "carrier": "Expedition",
    "deliveryNoteNumber": "Delivery note no.",
    "deliveryNotePlaceholder": "Number on the carrier's surat jalan (optional)"
```

`id.json`:

```json
    "carrier": "Ekspedisi",
    "deliveryNoteNumber": "No. surat jalan",
    "deliveryNotePlaceholder": "Nomor di surat jalan ekspedisi (opsional)"
```

- [ ] **Step 3: Create page**

Import `SupplierCombobox`. Extend state:

```ts
  const [form, setForm] = useState({
    purchaseOrderId: "",
    warehouseId: "",
    receiptDate: "",
    notes: "",
    carrierSupplierId: "",
    deliveryNoteNumber: "",
  });
```

After the `receiptDate` field and before the `notes` textarea, add:

```tsx
            <div className="space-y-2">
              <Label>{t('carrier')}</Label>
              <SupplierCombobox
                value={form.carrierSupplierId}
                onChange={(id) => setForm({ ...form, carrierSupplierId: id })}
                expeditionOnly
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="deliveryNoteNumber">{t('deliveryNoteNumber')}</Label>
              <Input
                id="deliveryNoteNumber"
                maxLength={100}
                value={form.deliveryNoteNumber}
                onChange={(e) => setForm({ ...form, deliveryNoteNumber: e.target.value })}
                placeholder={t('deliveryNotePlaceholder')}
              />
            </div>
```

Body additions:

```ts
        carrierSupplierId: form.carrierSupplierId || undefined,
        deliveryNoteNumber: form.deliveryNoteNumber.trim() || undefined,
```

- [ ] **Step 4: Detail page**

In the receipt-information card, after the `warehouse` row and before `created`:

```tsx
            <Separator />
            <div className="flex justify-between">
              <span className="text-muted-foreground">{t('carrier')}</span>
              <span className="font-medium">{gr.carrierSupplier?.name || "—"}</span>
            </div>
            <Separator />
            <div className="flex justify-between">
              <span className="text-muted-foreground">{t('deliveryNoteNumber')}</span>
              <span className="font-mono">{gr.deliveryNoteNumber || "—"}</span>
            </div>
```

- [ ] **Step 5: Type-check, build, smoke**

```bash
cd breeding-dashboard && npx tsc --noEmit && npm run build
```

`/goods-receipts/new`: expedition combobox lists only the two expedition suppliers; create with `SJ-GR-UI`; detail shows both values.

- [ ] **Step 6: Commit**

```bash
cd breeding-dashboard && git add src/types/api.ts messages "src/app/(dashboard)/goods-receipts" && git commit -m "feat(procurement): carrier and delivery note on goods receipt form and detail

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 15: Docs — mark spec implemented, note follow-ups

**Files:**
- Modify: `docs/superpowers/specs/2026-09-13-logistics-expedition-design.md:4`
- Modify: `docs/superpowers/specs/2026-09-05-logistics-expedition-finding.md` (Lampiran B)

- [ ] **Step 1: Spec status**

Change `**Status:** Design` → `**Status:** Implemented (YYYY-MM-DD) — lihat plan \`docs/superpowers/plans/2026-09-13-logistics-expedition.md\``.

- [ ] **Step 2: Finding doc revision row**

Add to Lampiran B (top of table):

```markdown
| YYYY-MM-DD | Spec 2026-09-13 diimplementasi: klasifikasi supplier, dispatch transfer, carrier di receipt. Stock movement transfer masih terbuka (spec terpisah). Bug lama diperbaiki: form supplier di dashboard tidak mengirim `code`. |
```

- [ ] **Step 3: Commit (root repo)**

```bash
cd /Users/alva.e202511001/Desktop/project/breeding && git add docs/superpowers && git commit -m "docs(logistics): mark expedition spec implemented

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Self-review

**Spec coverage**

| Spec section | Tasks |
|---|---|
| §1 schema (`isExpedition`, `SupplierCategory`) | 1 |
| §1 migration / seed | 1, 4 |
| §1 backend ProductCategory DTO | 2 |
| §1 backend Supplier DTOs, replace-all, `expedition` filter, includes, `assertExpeditionSupplier` | 3 |
| §1 frontend product-categories checkbox + badge | 5 |
| §1 frontend suppliers dialog + badges | 6 |
| §1 `SupplierCombobox expeditionOnly` | 7 |
| §1 types + i18n | 5, 6 |
| §2 schema (enum, columns, relations, index), migration, seed | 8 |
| §2 transitions + `IN_TRANSIT` guard | 9 |
| §2 create/PATCH/dispatch/status API, includes, status filter | 9 |
| §2 permissions (MANAGER+ dispatch) | 9 |
| §2 BPB validation against source warehouse | 9 |
| §2 frontend create page (carrier, surat jalan, BPB refs, reset on warehouse change) | 10 |
| §2 frontend detail (card, dispatch button/modal, read-only after dispatch, BPB column, receive only after dispatch) | 11 (`StatusAction` + constants in 10 handle receive visibility) |
| §2 frontend list filter + badge | 12 (badge color for `PREPARING` already exists in `STATUS_COLORS`) |
| §2 types, constants, i18n | 10 |
| §3 schema, migration, DTO, service, module | 13 |
| §3 frontend create + detail + types + i18n | 14 |
| §4 tests | curl in 4, 9, 13; browser smoke in 6, 10, 11, 12, 14 (per repo practice) |
| §5 docs | 15 |
| Non-goal: no stock movement | 9 curl step 9 checks no `TRANSFER` movements |

Not covered on purpose: "lines table read-only when status ≠ PREPARING" — the detail page's line table is already read-only (no edit-in-place exists); an edit page for transfers does not exist, so there is nothing to lock. Spec §2 "Edit button" on the expedition card is therefore omitted; carrier/surat jalan can be changed in the dispatch dialog, and `PATCH` remains available via API.

**Type consistency**: `assertExpeditionSupplier(tenantId, supplierId)` — same order in Tasks 3, 9, 13. `SupplierCategoryLink.categoryId` used in Task 6. `GoodsTransfer.dispatchedBy` shape `Pick<User,"id"|"name">` matches backend select. `TRANSFER_STATUS_TRANSITIONS` frontend `PREPARING: ["CANCELLED"]` vs backend `[IN_TRANSIT, CANCELLED]` — intentional (frontend excludes the dispatch path).
