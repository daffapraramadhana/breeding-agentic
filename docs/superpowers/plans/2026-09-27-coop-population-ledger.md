# Ledger Populasi Kandang — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Membangun populasi ayam berjalan per siklus kandang dengan ledger gerakan yang bisa diaudit, supaya form pesanan penjualan (S-B) punya angka sisa stok.

**Architecture:** Meniru pasangan `InventoryStock`/`InventoryMovement` yang sudah ada: satu tabel saldo (`CoopPopulation`) dan satu ledger append-only (`CoopPopulationMovement`), dengan grain `ProjectCoop`. Satu service memegang satu-satunya pintu tulis ke saldo; pembaruan saldo memakai `updateMany` bersyarat sehingga penjagaan saldo-tidak-negatif dan keamanan balapan datang dari kunci baris Postgres, bukan dari baca-lalu-tulis. Dua dokumen input (deplesi harian, pindah ayam) dan integrasi chick-in menulis lewat pintu itu.

**Tech Stack:** NestJS 11, Prisma 7 (client di `generated/prisma/`), PostgreSQL 15, Next.js 16, shadcn/ui, next-intl.

**Spec:** `docs/superpowers/specs/2026-09-27-coop-population-ledger-design.md`

## Global Constraints

- Semua query difilter `tenantId`; `ProjectCoop` tidak punya `tenantId`, jadi penyaringan lewat `project: { tenantId }`.
- Soft delete lewat `deletedAt`; jangan pernah hard-delete kecuali meniru perilaku yang sudah ada (lihat Task 7).
- Respons dibungkus `{ data, statusCode, timestamp }` oleh `TransformInterceptor` — skrip verifikasi membaca `r["data"]`.
- `ValidationPipe` memakai `whitelist: true, forbidNonWhitelisted: true, transform: true` — properti yang tidak dideklarasikan di DTO menghasilkan 400.
- Ekor adalah bilangan bulat: semua kolom jumlah bertipe `Int`, semua DTO memakai `@IsInt()`.
- Ledger append-only: tidak ada `update` atau `delete` pada `CoopPopulationMovement`.
- Pelanggaran `@@unique` dipetakan ke `ConflictException` berpesan manusiawi, tidak pernah dibiarkan bocor sebagai P2002.
- Kunci i18n ditambahkan ke `messages/en.json` **dan** `messages/id.json`, termasuk namespace `navigation` yang dibaca sidebar.
- Jalankan `cd breeding-app && npm run build` sebelum commit backend. **Peringatan:** `npm run build` menghapus `dist/` dan mematikan `npm run start:dev` yang sedang jalan — hidupkan lagi dev server sesudahnya.
- Repo tidak punya jest yang jalan (`src/app.controller.spec.ts` gagal parse sejak sebelum cabang mana pun). Verifikasi memakai skrip Python+curl yang dijalankan sampai **gagal dulu** sebelum implementasi.
- Perintah curl harus diawali `rtk proxy` karena pembungkus shell merusak keluaran curl polos.

## Review Focus

Lima kelas masukan yang spec implikasikan tetapi tidak dilatih langkah fitur mana pun. Tiap baris punya tes yang ditempelkan ke task pemiliknya.

1. **Dua gerakan `OUT` bersamaan pada kandang yang sama** — saldo tidak boleh jadi negatif dan tepat satu permintaan boleh lulus. Dijamin oleh `updateMany` bersyarat, diuji dengan dua permintaan paralel sungguhan di Task 3 Step 7.
2. **`projectCoopId` milik tenant lain** di setiap rute — harus 400, bukan membocorkan atau memodifikasi kandang tenant lain. Diuji dengan tenant kedua yang nyata di Task 2 Step 8b, dan dengan id tak dikenal di Task 5 dan 6.
3. **Entri deplesi diubah ke angka yang membuat saldo negatif** — ditolak, dan ledger tidak boleh menyisakan gerakan separuh jalan. Diuji di Task 5.
4. **Pindah ayam yang gagal di sisi tujuan** — sisi asal tidak boleh tersisa gerakan `OUT`. Diuji di Task 6.
5. **Jumlah nol, negatif, atau pecahan** di koreksi manual, deplesi, dan pindah — ditolak di DTO sebelum menyentuh ledger. Diuji di Task 3, 5, dan 6.

---

## Struktur berkas

**Backend (`breeding-app`)** — modul `project` memakai berkas datar, bukan subfolder `services/`; ikuti pola itu.

| Berkas | Tanggung jawab |
|---|---|
| `prisma/schema.prisma` | 4 model baru, 1 enum baru, relasi balik di `ProjectCoop` |
| `prisma/migrations/<ts>_add_coop_population_ledger/migration.sql` | migrasi aditif |
| `src/modules/project/coop-population.service.ts` | **satu-satunya** pintu tulis saldo: `applyMovement`, `allocate`, `release`, `adjust`, pembacaan |
| `src/modules/project/coop-population.controller.ts` | rute baca + koreksi manual |
| `src/modules/project/coop-depletion.service.ts` | entri deplesi harian |
| `src/modules/project/coop-depletion.controller.ts` | rute deplesi |
| `src/modules/project/coop-bird-transfer.service.ts` | pindah ayam antar kandang |
| `src/modules/project/coop-bird-transfer.controller.ts` | rute pindah ayam |
| `src/modules/project/dto/*.dto.ts` | DTO untuk ketiganya |
| `src/modules/project/project-chick-in.service.ts` | **diubah** — menerbitkan gerakan |
| `src/modules/project/project.module.ts` | **diubah** — pendaftaran |
| `prisma/seed.ts` | **diubah** — chick-in lewat jalur baru |

**Frontend (`breeding-dashboard`)**

| Berkas | Tanggung jawab |
|---|---|
| `src/types/api.ts` | tipe baru |
| `src/lib/constants.ts` | tiga entri nav di grup `Projects` |
| `messages/en.json`, `messages/id.json` | kunci i18n |
| `src/components/forms/project-coop-combobox.tsx` | pemilih siklus-kandang |
| `src/app/(dashboard)/coop-populations/page.tsx` | daftar saldo + panel riwayat + dialog koreksi |
| `src/app/(dashboard)/coop-depletions/page.tsx` | entri deplesi harian |
| `src/app/(dashboard)/coop-bird-transfers/page.tsx` | pindah ayam |

---

### Task 1: Schema dan migrasi

**Files:**
- Modify: `breeding-app/prisma/schema.prisma`
- Create: `breeding-app/prisma/migrations/<timestamp>_add_coop_population_ledger/migration.sql` (dibuat `prisma migrate dev`)

**Interfaces:**
- Consumes: enum `MovementType { IN, OUT }` yang sudah ada (schema.prisma baris 24), model `ProjectCoop` yang sudah ada.
- Produces: delegate Prisma `coopPopulation`, `coopPopulationMovement`, `coopDepletionEntry`, `coopBirdTransfer`; enum `BirdMovementSource`.

- [ ] **Step 1: Tambahkan enum dan empat model ke schema**

Letakkan enum tepat di bawah `enum MovementSource` yang sudah ada, dan model-modelnya tepat setelah `model ProjectChickIn`.

```prisma
enum BirdMovementSource {
  CHICK_IN
  MORTALITY
  CULLING
  SALES_REALIZATION
  TRANSFER_IN
  TRANSFER_OUT
  ADJUSTMENT
}

model CoopPopulation {
  id                String   @id @default(uuid())
  tenantId          String   @map("tenant_id")
  projectCoopId     String   @unique @map("project_coop_id")
  quantityOnHand    Int      @default(0) @map("quantity_on_hand")
  quantityAllocated Int      @default(0) @map("quantity_allocated")
  quantityAvailable Int      @default(0) @map("quantity_available")
  lastUpdatedAt     DateTime @default(now()) @map("last_updated_at")

  projectCoop ProjectCoop @relation(fields: [projectCoopId], references: [id])

  @@index([tenantId])
  @@map("coop_populations")
}

model CoopPopulationMovement {
  id             String             @id @default(uuid())
  tenantId       String             @map("tenant_id")
  projectCoopId  String             @map("project_coop_id")
  movementType   MovementType       @map("movement_type")
  movementSource BirdMovementSource @map("movement_source")
  sourceDocId    String?            @map("source_doc_id")
  sourceDocType  String?            @map("source_doc_type")
  quantityBefore Int                @map("quantity_before")
  quantity       Int
  quantityAfter  Int                @map("quantity_after")
  movementDate   DateTime           @map("movement_date") @db.Date
  notes          String?            @db.Text
  createdBy      String?            @map("created_by")
  createdAt      DateTime           @default(now()) @map("created_at")

  projectCoop ProjectCoop @relation(fields: [projectCoopId], references: [id])

  @@index([projectCoopId, movementDate])
  @@index([tenantId])
  @@map("coop_population_movements")
}

model CoopDepletionEntry {
  id             String    @id @default(uuid())
  tenantId       String    @map("tenant_id")
  projectCoopId  String    @map("project_coop_id")
  entryDate      DateTime  @map("entry_date") @db.Date
  mortalityCount Int       @default(0) @map("mortality_count")
  cullingCount   Int       @default(0) @map("culling_count")
  notes          String?   @db.Text
  createdBy      String?   @map("created_by")
  createdAt      DateTime  @default(now()) @map("created_at")
  updatedAt      DateTime  @updatedAt @map("updated_at")
  deletedAt      DateTime? @map("deleted_at")

  projectCoop ProjectCoop @relation(fields: [projectCoopId], references: [id])

  @@unique([projectCoopId, entryDate])
  @@index([tenantId])
  @@map("coop_depletion_entries")
}

model CoopBirdTransfer {
  id                       String    @id @default(uuid())
  tenantId                 String    @map("tenant_id")
  transferNumber           String    @map("transfer_number")
  sourceProjectCoopId      String    @map("source_project_coop_id")
  destinationProjectCoopId String    @map("destination_project_coop_id")
  quantity                 Int
  transferDate             DateTime  @map("transfer_date") @db.Date
  notes                    String?   @db.Text
  createdBy                String?   @map("created_by")
  createdAt                DateTime  @default(now()) @map("created_at")
  updatedAt                DateTime  @updatedAt @map("updated_at")
  deletedAt                DateTime? @map("deleted_at")

  sourceProjectCoop      ProjectCoop @relation("BirdTransferSource", fields: [sourceProjectCoopId], references: [id])
  destinationProjectCoop ProjectCoop @relation("BirdTransferDestination", fields: [destinationProjectCoopId], references: [id])

  @@unique([tenantId, transferNumber])
  @@index([sourceProjectCoopId])
  @@index([destinationProjectCoopId])
  @@map("coop_bird_transfers")
}
```

- [ ] **Step 2: Tambahkan relasi balik di `ProjectCoop`**

Di `model ProjectCoop`, tepat setelah baris `workers  ProjectWorker[]`:

```prisma
  population          CoopPopulation?
  populationMovements CoopPopulationMovement[]
  depletionEntries    CoopDepletionEntry[]
  birdTransfersOut    CoopBirdTransfer[]       @relation("BirdTransferSource")
  birdTransfersIn     CoopBirdTransfer[]       @relation("BirdTransferDestination")
```

- [ ] **Step 3: Buat migrasi**

Run: `cd breeding-app && npx prisma migrate dev --name add_coop_population_ledger`
Expected: migrasi dibuat dan diterapkan, `prisma generate` jalan otomatis.

- [ ] **Step 4: Periksa SQL-nya aditif**

Run: `cd breeding-app && grep -c "UPDATE\|DROP\|ALTER COLUMN" prisma/migrations/*_add_coop_population_ledger/migration.sql`
Expected: `0` — migrasi hanya boleh `CREATE TYPE`, `CREATE TABLE`, `CREATE INDEX`, `ALTER TABLE ... ADD CONSTRAINT`.

- [ ] **Step 5: Build**

Run: `cd breeding-app && npm run build`
Expected: keluar dengan status 0.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(project): add coop population ledger schema"
```

---

### Task 2: Service populasi — pintu tulis dan pembacaan

**Files:**
- Create: `breeding-app/src/modules/project/coop-population.service.ts`
- Create: `breeding-app/src/modules/project/coop-population.controller.ts`
- Create: `breeding-app/src/modules/project/dto/query-coop-population.dto.ts`
- Modify: `breeding-app/src/modules/project/project.module.ts`
- Test: `scratchpad/verify-population.py`

**Interfaces:**
- Consumes: delegate Prisma dari Task 1.
- Produces: `CoopPopulationService.applyMovement(tx, params)`, `.resolveProjectCoop(tx, projectCoopId, tenantId)`, `.findAll(tenantId, query)`, `.findOne(tenantId, projectCoopId)`, `.findMovements(tenantId, projectCoopId, pagination)`. Tipe `ApplyMovementParams` diekspor dari berkas service. Rute `GET /coop-populations`, `GET /coop-populations/:projectCoopId`, `GET /coop-populations/:projectCoopId/movements`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-population.py`:

```python
import json, subprocess, time
RUN = str(int(time.time()))[-6:]
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

fails = []
def check(label, actual, expected):
    ok = actual == expected
    if not ok: fails.append(label)
    print(f"{'PASS' if ok else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

rows = curl([API + "/coop-populations?limit=50"] + H)["data"]["data"]
check("1 list returns project coops", len(rows) > 0, True)
pc = rows[0]
check("2 row carries a population block", isinstance(pc.get("population"), dict), True)
check("3 coop name is included", isinstance(pc["coop"]["name"], str), True)

one = curl([API + "/coop-populations/" + pc["id"]] + H)["data"]
check("4 single read matches list", one["id"], pc["id"])

mv = curl([API + "/coop-populations/" + pc["id"] + "/movements?limit=5"] + H)["data"]
check("5 movements endpoint paginates", isinstance(mv["data"], list), True)

bogus = curl([API + "/coop-populations/00000000-0000-4000-8000-000000000000"] + H)
check("6 unknown project coop rejected", bogus.get("statusCode"), 400)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-population.py`
Expected: gagal dengan `KeyError: 'data'` pada baris `rows = ...` karena rute `/coop-populations` belum ada (Nest menjawab 404 tanpa kunci `data`).

- [ ] **Step 3: Tulis service**

Create `breeding-app/src/modules/project/coop-population.service.ts`:

```ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';
import { Prisma, MovementType, BirdMovementSource } from '../../../generated/prisma/client.js';
import { PaginationDto, PaginatedResult } from '../../common/dto/pagination.dto.js';
import { QueryCoopPopulationDto } from './dto/query-coop-population.dto.js';

export interface ApplyMovementParams {
  tenantId: string;
  projectCoopId: string;
  movementType: MovementType;
  movementSource: BirdMovementSource;
  quantity: number;
  movementDate: Date;
  sourceDocId?: string;
  sourceDocType?: string;
  notes?: string;
  createdBy?: string;
}

@Injectable()
export class CoopPopulationService {
  constructor(private readonly prisma: PrismaService) {}

  /**
   * Satu-satunya pintu tulis ke saldo populasi.
   *
   * Saldo diperbarui dengan updateMany bersyarat, bukan baca-lalu-tulis:
   * syarat `quantityOnHand >= quantity` membuat Postgres mengunci baris dan
   * menolak gerakan yang membuat saldo negatif dalam satu perintah. Dua
   * permintaan bersamaan karena itu berbaris, bukan saling menimpa.
   */
  async applyMovement(
    tx: Prisma.TransactionClient,
    params: ApplyMovementParams,
  ) {
    const { tenantId, projectCoopId, movementType, quantity } = params;

    if (!Number.isInteger(quantity) || quantity <= 0) {
      throw new BadRequestException('Movement quantity must be a positive whole number');
    }

    await tx.coopPopulation.upsert({
      where: { projectCoopId },
      update: {},
      create: { tenantId, projectCoopId },
    });

    const delta = movementType === 'IN' ? quantity : -quantity;
    const guard = movementType === 'OUT' ? { quantityOnHand: { gte: quantity } } : {};

    const updated = await tx.coopPopulation.updateMany({
      where: { projectCoopId, tenantId, ...guard },
      data: {
        quantityOnHand: { increment: delta },
        quantityAvailable: { increment: delta },
        lastUpdatedAt: new Date(),
      },
    });
    if (updated.count !== 1) {
      throw new BadRequestException('Populasi kandang tidak mencukupi');
    }

    const after = await tx.coopPopulation.findUniqueOrThrow({ where: { projectCoopId } });

    return tx.coopPopulationMovement.create({
      data: {
        tenantId,
        projectCoopId,
        movementType,
        movementSource: params.movementSource,
        sourceDocId: params.sourceDocId,
        sourceDocType: params.sourceDocType,
        quantityBefore: after.quantityOnHand - delta,
        quantity,
        quantityAfter: after.quantityOnHand,
        movementDate: params.movementDate,
        notes: params.notes,
        createdBy: params.createdBy,
      },
    });
  }

  /** `ProjectCoop` tidak punya tenantId sendiri — penyaringan lewat proyeknya. */
  async resolveProjectCoop(
    tx: Prisma.TransactionClient | PrismaService,
    projectCoopId: string,
    tenantId: string,
  ) {
    const pc = await tx.projectCoop.findFirst({
      where: { id: projectCoopId, project: { tenantId, deletedAt: null } },
      include: {
        coop: { select: { id: true, code: true, name: true, farmId: true, branchId: true } },
        project: { select: { id: true, tenantId: true, isActive: true } },
      },
    });
    if (!pc) {
      throw new BadRequestException('Project coop not found for this tenant');
    }
    return pc;
  }

  private readonly include = {
    coop: { select: { id: true, code: true, name: true } },
    project: { select: { id: true, startDate: true, isActive: true } },
    population: true,
  } as const;

  private withZeroPopulation<T extends { population: unknown }>(row: T) {
    if (row.population) return row;
    return {
      ...row,
      population: { quantityOnHand: 0, quantityAllocated: 0, quantityAvailable: 0 },
    };
  }

  async findAll(tenantId: string, query: QueryCoopPopulationDto) {
    const where = {
      project: {
        tenantId,
        deletedAt: null,
        ...(query.projectId && { id: query.projectId }),
        ...(query.includeInactive ? {} : { isActive: true }),
      },
      ...(query.coopId && { coopId: query.coopId }),
      coop: {
        deletedAt: null,
        ...(query.branchId && { branchId: query.branchId }),
        ...(query.farmId && { farmId: query.farmId }),
      },
    };

    const [data, total] = await Promise.all([
      this.prisma.projectCoop.findMany({
        where,
        include: this.include,
        skip: query.skip,
        take: query.limit,
        orderBy: { id: 'asc' },
      }),
      this.prisma.projectCoop.count({ where }),
    ]);

    return new PaginatedResult(
      data.map((row) => this.withZeroPopulation(row)),
      total,
      query.page,
      query.limit,
    );
  }

  async findOne(tenantId: string, projectCoopId: string) {
    await this.resolveProjectCoop(this.prisma, projectCoopId, tenantId);
    const row = await this.prisma.projectCoop.findFirstOrThrow({
      where: { id: projectCoopId },
      include: this.include,
    });
    return this.withZeroPopulation(row);
  }

  async findMovements(tenantId: string, projectCoopId: string, pagination: PaginationDto) {
    await this.resolveProjectCoop(this.prisma, projectCoopId, tenantId);
    const where = { projectCoopId, tenantId };
    const [data, total] = await Promise.all([
      this.prisma.coopPopulationMovement.findMany({
        where,
        skip: pagination.skip,
        take: pagination.limit,
        orderBy: [{ movementDate: 'desc' }, { createdAt: 'desc' }],
      }),
      this.prisma.coopPopulationMovement.count({ where }),
    ]);
    return new PaginatedResult(data, total, pagination.page, pagination.limit);
  }
}
```

- [ ] **Step 4: Tulis DTO query**

Create `breeding-app/src/modules/project/dto/query-coop-population.dto.ts`:

```ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsUUID, IsBoolean } from 'class-validator';
import { Transform } from 'class-transformer';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

export class QueryCoopPopulationDto extends PaginationDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  projectId?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  coopId?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  branchId?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  farmId?: string;

  @ApiPropertyOptional({ description: 'Sertakan siklus yang sudah tidak aktif' })
  @IsOptional()
  @Transform(({ value }) => value === true || value === 'true')
  @IsBoolean()
  includeInactive?: boolean;
}
```

- [ ] **Step 5: Tulis controller**

Create `breeding-app/src/modules/project/coop-population.controller.ts`:

```ts
import { Controller, Get, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { CoopPopulationService } from './coop-population.service.js';
import { QueryCoopPopulationDto } from './dto/query-coop-population.dto.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { PaginationDto } from '../../common/dto/pagination.dto.js';

@ApiTags('coop-populations')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('coop-populations')
export class CoopPopulationController {
  constructor(private readonly service: CoopPopulationService) {}

  @Get()
  @ApiOperation({ summary: 'Daftar populasi berjalan per siklus kandang' })
  findAll(@CurrentTenant() tenantId: string, @Query() query: QueryCoopPopulationDto) {
    return this.service.findAll(tenantId, query);
  }

  @Get(':projectCoopId')
  @ApiOperation({ summary: 'Populasi satu siklus kandang' })
  findOne(@CurrentTenant() tenantId: string, @Param('projectCoopId') projectCoopId: string) {
    return this.service.findOne(tenantId, projectCoopId);
  }

  @Get(':projectCoopId/movements')
  @ApiOperation({ summary: 'Riwayat gerakan populasi' })
  findMovements(
    @CurrentTenant() tenantId: string,
    @Param('projectCoopId') projectCoopId: string,
    @Query() pagination: PaginationDto,
  ) {
    return this.service.findMovements(tenantId, projectCoopId, pagination);
  }
}
```

- [ ] **Step 6: Daftarkan di module**

Di `breeding-app/src/modules/project/project.module.ts`, tambahkan impor `CoopPopulationService` dan `CoopPopulationController`, lalu masukkan controller ke array `controllers`, service ke `providers` **dan** `exports` (Task 7 dan S-C memakainya dari luar).

- [ ] **Step 7: Build dan jalankan skrip**

Run: `cd breeding-app && npm run build && npm run start:dev` (di latar), lalu `python3 scratchpad/verify-population.py`
Expected: `RESULT: all checks pass` — 6 dari 6.

- [ ] **Step 8: Uji balapan dan isolasi tenant**

Tambahkan ke `scratchpad/verify-population.py`, sebelum baris `print()` terakhir:

```python
# Review Focus 1: dua OUT bersamaan tidak boleh membuat saldo negatif.
# Dijalankan lewat koreksi manual di Task 3; di sini cukup dibuktikan bahwa
# OUT yang melebihi saldo ditolak dan tidak menyisakan baris ledger.
before = curl([API + "/coop-populations/" + pc["id"] + "/movements?limit=1"] + H)["data"]["meta"]["total"]
over = curl(["-X", "POST", API + "/coop-populations/" + pc["id"] + "/adjustments"] + H +
            ["-d", json.dumps({"quantity": 999999999, "direction": "OUT",
                               "reason": "over " + RUN, "movementDate": "2026-09-27"})])
check("7 OUT beyond balance rejected", over.get("statusCode"), 400)
after_total = curl([API + "/coop-populations/" + pc["id"] + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("8 rejected movement left no ledger row", after_total, before)
```

Langkah ini bergantung pada rute koreksi dari Task 3; jalankan lagi setelah Task 3 selesai.

- [ ] **Step 8b: Uji saldo cocok dengan ledger, dan isolasi tenant sungguhan**

Pemeriksaan 6 di Step 1 memakai UUID karangan, yang menempuh jalur kode yang sama tetapi tidak membuktikan kandang tenant **lain** tertolak. Ini membuktikannya dengan tenant kedua yang nyata.

Create `scratchpad/verify-tenant-isolation.mjs`:

```js
import { PrismaClient } from '../breeding-app/generated/prisma/client.js';

const prisma = new PrismaClient();
const fails = [];
const check = (label, actual, expected) => {
  const ok = JSON.stringify(actual) === JSON.stringify(expected);
  if (!ok) fails.push(label);
  console.log(`${ok ? 'PASS' : 'FAIL'} ${label}: got ${JSON.stringify(actual)}, expected ${JSON.stringify(expected)}`);
};

const API = 'http://localhost:3002/api';
const login = await fetch(`${API}/auth/login`, {
  method: 'POST', headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email: 'admin@demo.farm', password: 'password123' }),
}).then((r) => r.json());
const H = { 'Content-Type': 'application/json', Authorization: `Bearer ${login.data.accessToken}` };

// Saldo harus sama dengan quantityAfter gerakan terakhir — spec Verifikasi #1.
const pop = await prisma.coopPopulation.findFirstOrThrow();
const last = await prisma.coopPopulationMovement.findFirst({
  where: { projectCoopId: pop.projectCoopId },
  orderBy: { createdAt: 'desc' },
});
check('1 balance equals last movement quantityAfter', pop.quantityOnHand, last.quantityAfter);

// Tenant kedua yang nyata, bukan UUID karangan.
const RUN = String(Date.now()).slice(-6);
const other = await prisma.tenant.create({ data: { name: `Other ${RUN}`, slug: `other-${RUN}` } });
const branch = await prisma.branch.create({ data: { tenantId: other.id, code: `OB-${RUN}`, name: 'Other Branch' } });
const farm = await prisma.farm.create({ data: { tenantId: other.id, branchId: branch.id, code: `OF-${RUN}`, name: 'Other Farm' } });
const coop = await prisma.coop.create({ data: { tenantId: other.id, farmId: farm.id, branchId: branch.id, code: `OC-${RUN}`, name: 'Other Coop', capacity: 100 } });
const project = await prisma.project.create({ data: { tenantId: other.id, branchId: branch.id, farmId: farm.id, startDate: new Date() } });
const foreign = await prisma.projectCoop.create({ data: { projectId: project.id, coopId: coop.id } });

const read = await fetch(`${API}/coop-populations/${foreign.id}`, { headers: H }).then((r) => r.json());
check('2 foreign tenant coop not readable', read.statusCode, 400);

const mv = await fetch(`${API}/coop-populations/${foreign.id}/movements`, { headers: H }).then((r) => r.json());
check('3 foreign tenant movements not readable', mv.statusCode, 400);

const adj = await fetch(`${API}/coop-populations/${foreign.id}/adjustments`, {
  method: 'POST', headers: H,
  body: JSON.stringify({ quantity: 1, direction: 'IN', reason: 'x', movementDate: '2026-09-27' }),
}).then((r) => r.json());
check('4 foreign tenant not adjustable', adj.statusCode, 400);

const list = await fetch(`${API}/coop-populations?limit=100`, { headers: H }).then((r) => r.json());
check('5 foreign coop absent from list', list.data.data.some((r) => r.id === foreign.id), false);

await prisma.projectCoop.delete({ where: { id: foreign.id } });
await prisma.project.delete({ where: { id: project.id } });
await prisma.coop.delete({ where: { id: coop.id } });
await prisma.farm.delete({ where: { id: farm.id } });
await prisma.branch.delete({ where: { id: branch.id } });
await prisma.tenant.delete({ where: { id: other.id } });

console.log('\nRESULT:', fails.length ? `${fails.length} FAILING: ${fails}` : 'all checks pass');
await prisma.$disconnect();
```

Jalankan dulu sebelum menulis `findOne`/`findMovements` untuk melihat pemeriksaan 2-4 gagal, lalu lagi setelahnya.

Run: `node scratchpad/verify-tenant-isolation.mjs`
Expected: `RESULT: all checks pass` — 5 dari 5. Pemeriksaan 4 baru bisa lulus setelah Task 3; sampai saat itu jalankan skrip ini tanpa blok pemeriksaan 4.

Kalau `prisma.tenant.create` menolak karena kolom wajib yang belum tercantum, baca bentuk `model Tenant` di `breeding-app/prisma/schema.prisma` dan lengkapi — jangan melewati langkah ini.

- [ ] **Step 9: Commit**

```bash
cd breeding-app
git add src/modules/project
git commit -m "feat(project): coop population service with conditional balance guard"
```

---

### Task 3: Koreksi manual

**Files:**
- Create: `breeding-app/src/modules/project/dto/create-coop-adjustment.dto.ts`
- Modify: `breeding-app/src/modules/project/coop-population.service.ts`
- Modify: `breeding-app/src/modules/project/coop-population.controller.ts`
- Test: `scratchpad/verify-adjustment.py`

**Interfaces:**
- Consumes: `CoopPopulationService.applyMovement(tx, params)`, `.resolveProjectCoop(tx, id, tenantId)` dari Task 2.
- Produces: `CoopPopulationService.adjust(tenantId, projectCoopId, dto, userId)`; rute `POST /coop-populations/:projectCoopId/adjustments`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-adjustment.py`:

```python
import json, subprocess, time
RUN = str(int(time.time()))[-6:]
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

fails = []
def check(label, actual, expected):
    ok = actual == expected
    if not ok: fails.append(label)
    print(f"{'PASS' if ok else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

pc = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]
PID = pc["id"]
start = pc["population"]["quantityOnHand"]

def adjust(body):
    return curl(["-X", "POST", API + "/coop-populations/" + PID + "/adjustments"] + H + ["-d", json.dumps(body)])

r = adjust({"quantity": 100, "direction": "IN", "reason": "opname " + RUN, "movementDate": "2026-09-27"})
check("1 IN adjustment applied", r["data"]["quantityAfter"], start + 100)
check("2 ledger records the reason", r["data"]["notes"], "opname " + RUN)
check("3 movement source is ADJUSTMENT", r["data"]["movementSource"], "ADJUSTMENT")

r = adjust({"quantity": 40, "direction": "OUT", "reason": "opname2 " + RUN, "movementDate": "2026-09-27"})
check("4 OUT adjustment applied", r["data"]["quantityAfter"], start + 60)

check("5 zero quantity rejected", adjust({"quantity": 0, "direction": "IN", "reason": "z", "movementDate": "2026-09-27"}).get("statusCode"), 400)
check("6 negative quantity rejected", adjust({"quantity": -5, "direction": "IN", "reason": "z", "movementDate": "2026-09-27"}).get("statusCode"), 400)
check("7 fractional quantity rejected", adjust({"quantity": 1.5, "direction": "IN", "reason": "z", "movementDate": "2026-09-27"}).get("statusCode"), 400)
check("8 missing reason rejected", adjust({"quantity": 5, "direction": "IN", "movementDate": "2026-09-27"}).get("statusCode"), 400)
check("9 bad direction rejected", adjust({"quantity": 5, "direction": "SIDEWAYS", "reason": "z", "movementDate": "2026-09-27"}).get("statusCode"), 400)

# bersihkan: kembalikan saldo ke posisi semula
adjust({"quantity": 60, "direction": "OUT", "reason": "cleanup " + RUN, "movementDate": "2026-09-27"})
final = curl([API + "/coop-populations/" + PID] + H)["data"]["population"]["quantityOnHand"]
check("10 balance restored", final, start)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-adjustment.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 1 karena rute `adjustments` belum ada.

- [ ] **Step 3: Tulis DTO**

Create `breeding-app/src/modules/project/dto/create-coop-adjustment.dto.ts`:

```ts
import { ApiProperty } from '@nestjs/swagger';
import { IsInt, IsPositive, IsIn, IsString, IsNotEmpty, IsDateString } from 'class-validator';

export class CreateCoopAdjustmentDto {
  @ApiProperty({ description: 'Jumlah ekor, selalu positif' })
  @IsInt()
  @IsPositive()
  quantity!: number;

  @ApiProperty({ enum: ['IN', 'OUT'] })
  @IsIn(['IN', 'OUT'])
  direction!: 'IN' | 'OUT';

  @ApiProperty({ description: 'Alasan koreksi, wajib — tersimpan di catatan gerakan' })
  @IsString()
  @IsNotEmpty()
  reason!: string;

  @ApiProperty({ example: '2026-09-27' })
  @IsDateString()
  movementDate!: string;
}
```

- [ ] **Step 4: Tambahkan `adjust()` ke service**

Di `coop-population.service.ts`, tambahkan impor `CreateCoopAdjustmentDto` dan metode:

```ts
  async adjust(
    tenantId: string,
    projectCoopId: string,
    dto: CreateCoopAdjustmentDto,
    userId?: string,
  ) {
    return this.prisma.$transaction(async (tx) => {
      await this.resolveProjectCoop(tx, projectCoopId, tenantId);
      return this.applyMovement(tx, {
        tenantId,
        projectCoopId,
        movementType: dto.direction as MovementType,
        movementSource: 'ADJUSTMENT',
        quantity: dto.quantity,
        movementDate: new Date(dto.movementDate),
        notes: dto.reason,
        createdBy: userId,
      });
    });
  }
```

- [ ] **Step 5: Tambahkan rute**

Di `coop-population.controller.ts`, tambahkan impor `Post`, `Body`, `RolesGuard`, `Roles`, `SystemRole`, `CurrentUser`, `JwtPayload`, `CreateCoopAdjustmentDto`, dan metode:

```ts
  @Post(':projectCoopId/adjustments')
  @UseGuards(RolesGuard)
  @Roles(SystemRole.MANAGER)
  @ApiOperation({ summary: 'Koreksi manual populasi kandang' })
  adjust(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('projectCoopId') projectCoopId: string,
    @Body() dto: CreateCoopAdjustmentDto,
  ) {
    return this.service.adjust(tenantId, projectCoopId, dto, user.sub);
  }
```

Repo ini memasang penjaga di tingkat kelas (`breeding-app/src/modules/transfer/goods-transfer.controller.ts:21`), jadi ubah dekorator kelas `coop-population.controller.ts` menjadi `@UseGuards(JwtAuthGuard, RolesGuard)` dan **hapus** `@UseGuards(RolesGuard)` dari metode — sisakan `@Roles(SystemRole.MANAGER)` saja. `RolesGuard` mengembalikan true saat tidak ada metadata `@Roles`, jadi rute baca tetap terbuka untuk semua peran.

`JwtPayload` memakai `sub` sebagai id pengguna (`src/modules/auth/jwt.strategy.ts:19`), jadi `user.sub` benar.

- [ ] **Step 6: Build dan jalankan kedua skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi dev server, lalu `python3 scratchpad/verify-adjustment.py && python3 scratchpad/verify-population.py`
Expected: keduanya `RESULT: all checks pass` — koreksi 10/10, populasi 8/8 (pemeriksaan 7 dan 8 dari Task 2 Step 8 sekarang bisa jalan).

- [ ] **Step 7: Uji balapan (Review Focus 1)**

Dua gerakan `OUT` bersamaan yang totalnya melebihi saldo: tepat satu harus lulus. Kalau saldo ditulis dengan baca-lalu-tulis, keduanya lulus dan saldo jadi negatif.

Create `scratchpad/verify-race.py`:

```python
import json, subprocess, time
from concurrent.futures import ThreadPoolExecutor
API = "http://localhost:3002/api"
RUN = str(int(time.time()))[-6:]

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

pc = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]
PID = pc["id"]

def adjust(qty, direction, reason):
    return curl(["-X", "POST", API + "/coop-populations/" + PID + "/adjustments"] + H +
                ["-d", json.dumps({"quantity": qty, "direction": direction,
                                   "reason": reason, "movementDate": "2026-09-27"})])

# saldo tepat 100 di atas posisi awal
start = curl([API + "/coop-populations/" + PID] + H)["data"]["population"]["quantityOnHand"]
adjust(100, "IN", "race setup " + RUN)

with ThreadPoolExecutor(max_workers=2) as ex:
    a, b = [f.result() for f in [ex.submit(adjust, 100, "OUT", "race A " + RUN),
                                 ex.submit(adjust, 100, "OUT", "race B " + RUN)]]

ok = sum(1 for r in (a, b) if r.get("statusCode") != 400)
end = curl([API + "/coop-populations/" + PID] + H)["data"]["population"]["quantityOnHand"]

fails = []
def check(label, actual, expected):
    if actual != expected: fails.append(label)
    print(f"{'PASS' if actual == expected else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

check("1 exactly one concurrent OUT succeeded", ok, 1)
check("2 balance returned to its starting value", end, start)
check("3 balance never went negative", end >= 0, True)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

Run: `python3 scratchpad/verify-race.py`
Expected: `RESULT: all checks pass` — 3 dari 3. Kalau pemeriksaan 1 memberi `2`, `applyMovement` tidak memakai `updateMany` bersyarat; perbaiki di sana, jangan di skripnya.

- [ ] **Step 8: Commit**

```bash
cd breeding-app
git add src/modules/project
git commit -m "feat(project): manual population adjustment with mandatory reason"
```

---

### Task 4: Alokasi

**Files:**
- Modify: `breeding-app/src/modules/project/coop-population.service.ts`
- Test: `scratchpad/verify-allocation.py`

**Interfaces:**
- Consumes: `resolveProjectCoop` dari Task 2.
- Produces: `CoopPopulationService.allocate(tenantId, projectCoopId, quantity)` dan `.release(tenantId, projectCoopId, quantity)`, keduanya mengembalikan `CoopPopulation` terbaru. **Dipanggil oleh S-B, belum ada pemanggil di plan ini.**

**Catatan untuk S-C:** spec menyebut `SALES_REALIZATION` sebagai gerakan yang "disediakan metodenya, dipanggil di S-C". Metode itu adalah `applyMovement` yang sudah publik dan diekspor lewat `ProjectModule` sejak Task 2 — S-C memanggilnya dengan `movementSource: 'SALES_REALIZATION'` dan `sourceDocType: 'SalesRealization'`. Tidak ada metode khusus yang perlu dibuat di plan ini; membuatnya sekarang berarti menebak bentuk realisasi yang belum dirancang.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Karena belum ada rute HTTP untuk alokasi — dan memang tidak boleh ada, alokasi dipicu oleh pesanan — skrip ini memanggil service lewat skrip Node sekali jalan.

Create `scratchpad/verify-allocation.mjs`:

```js
import { PrismaClient } from '../breeding-app/generated/prisma/client.js';

const prisma = new PrismaClient();
const fails = [];
const check = (label, actual, expected) => {
  const ok = JSON.stringify(actual) === JSON.stringify(expected);
  if (!ok) fails.push(label);
  console.log(`${ok ? 'PASS' : 'FAIL'} ${label}: got ${JSON.stringify(actual)}, expected ${JSON.stringify(expected)}`);
};

const pc = await prisma.projectCoop.findFirstOrThrow({ include: { project: true } });
const tenantId = pc.project.tenantId;

const { CoopPopulationService } = await import('../breeding-app/dist/modules/project/coop-population.service.js');
const svc = new CoopPopulationService(prisma);

const start = await prisma.coopPopulation.upsert({
  where: { projectCoopId: pc.id },
  update: {},
  create: { tenantId, projectCoopId: pc.id, quantityOnHand: 100, quantityAvailable: 100 },
});
if (start.quantityAvailable < 30) {
  await prisma.coopPopulation.update({
    where: { projectCoopId: pc.id },
    data: { quantityOnHand: 100, quantityAllocated: 0, quantityAvailable: 100 },
  });
}

const a = await svc.allocate(tenantId, pc.id, 30);
check('1 allocate reduces available', a.quantityAvailable, 70);
check('2 allocate raises allocated', a.quantityAllocated, 30);
check('3 allocate leaves on-hand alone', a.quantityOnHand, 100);

const ledger = await prisma.coopPopulationMovement.count({
  where: { projectCoopId: pc.id, movementSource: 'ADJUSTMENT', notes: { contains: 'allocate' } },
});
check('4 allocation writes no ledger row', ledger, 0);

let overflowed = false;
try { await svc.allocate(tenantId, pc.id, 1000); } catch { overflowed = true; }
check('5 allocate beyond available rejected', overflowed, true);

const r = await svc.release(tenantId, pc.id, 30);
check('6 release restores available', r.quantityAvailable, 100);
check('7 release clears allocated', r.quantityAllocated, 0);

let overReleased = false;
try { await svc.release(tenantId, pc.id, 5); } catch { overReleased = true; }
check('8 release beyond allocated rejected', overReleased, true);

console.log('\nRESULT:', fails.length ? `${fails.length} FAILING: ${fails}` : 'all checks pass');
await prisma.$disconnect();
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `cd breeding-app && npm run build && node ../scratchpad/verify-allocation.mjs`
Expected: gagal dengan `TypeError: svc.allocate is not a function`.

- [ ] **Step 3: Tambahkan `allocate` dan `release`**

Di `coop-population.service.ts`:

```ts
  /**
   * Alokasi tidak menulis ledger: tidak ada ayam yang berpindah, hanya saldo
   * yang dipesan. Syarat `gte` membuat Postgres menolak alokasi berlebih
   * dalam satu perintah, jadi dua pesanan bersamaan tidak bisa memesan ekor
   * yang sama.
   */
  async allocate(tenantId: string, projectCoopId: string, quantity: number) {
    return this.moveAllocation(tenantId, projectCoopId, quantity, 'allocate');
  }

  async release(tenantId: string, projectCoopId: string, quantity: number) {
    return this.moveAllocation(tenantId, projectCoopId, quantity, 'release');
  }

  private async moveAllocation(
    tenantId: string,
    projectCoopId: string,
    quantity: number,
    mode: 'allocate' | 'release',
  ) {
    if (!Number.isInteger(quantity) || quantity <= 0) {
      throw new BadRequestException('Allocation quantity must be a positive whole number');
    }

    const guard =
      mode === 'allocate'
        ? { quantityAvailable: { gte: quantity } }
        : { quantityAllocated: { gte: quantity } };
    const sign = mode === 'allocate' ? 1 : -1;

    const updated = await this.prisma.coopPopulation.updateMany({
      where: { projectCoopId, tenantId, ...guard },
      data: {
        quantityAllocated: { increment: sign * quantity },
        quantityAvailable: { increment: -sign * quantity },
        lastUpdatedAt: new Date(),
      },
    });
    if (updated.count !== 1) {
      throw new BadRequestException(
        mode === 'allocate'
          ? 'Sisa populasi tidak mencukupi'
          : 'Alokasi yang dilepas melebihi yang tercatat',
      );
    }

    return this.prisma.coopPopulation.findUniqueOrThrow({ where: { projectCoopId } });
  }
```

- [ ] **Step 4: Jalankan dan pastikan lulus**

Run: `cd breeding-app && npm run build && node ../scratchpad/verify-allocation.mjs`
Expected: `RESULT: all checks pass` — 8 dari 8.

- [ ] **Step 5: Commit**

```bash
cd breeding-app
git add src/modules/project/coop-population.service.ts
git commit -m "feat(project): population allocation mechanism for sales orders"
```

---

### Task 5: Entri deplesi harian

**Files:**
- Create: `breeding-app/src/modules/project/coop-depletion.service.ts`
- Create: `breeding-app/src/modules/project/coop-depletion.controller.ts`
- Create: `breeding-app/src/modules/project/dto/create-coop-depletion.dto.ts`
- Create: `breeding-app/src/modules/project/dto/update-coop-depletion.dto.ts`
- Create: `breeding-app/src/modules/project/dto/query-coop-depletion.dto.ts`
- Modify: `breeding-app/src/modules/project/project.module.ts`
- Test: `scratchpad/verify-depletion.py`

**Interfaces:**
- Consumes: `CoopPopulationService.applyMovement(tx, params)` dan `.resolveProjectCoop(tx, id, tenantId)`.
- Produces: rute `GET/POST /coop-depletions`, `PATCH/DELETE /coop-depletions/:id`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-depletion.py`:

```python
import json, subprocess, time
RUN = str(int(time.time()))[-6:]
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

fails = []
def check(label, actual, expected):
    ok = actual == expected
    if not ok: fails.append(label)
    print(f"{'PASS' if ok else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

def onhand(pid):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]["quantityOnHand"]

pc = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]
PID = pc["id"]

# pastikan ada stok untuk dikurangi
curl(["-X", "POST", API + "/coop-populations/" + PID + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 500, "direction": "IN", "reason": "seed " + RUN, "movementDate": "2026-09-27"})])
start = onhand(PID)

DATE = "2026-09-%02d" % (int(RUN[-2:]) % 27 + 1)

r = curl(["-X", "POST", API + "/coop-depletions"] + H +
         ["-d", json.dumps({"projectCoopId": PID, "entryDate": DATE, "mortalityCount": 10, "cullingCount": 5})])
eid = r["data"]["id"]
check("1 entry created", r["data"]["mortalityCount"], 10)
check("2 balance dropped by both counts", onhand(PID), start - 15)

mv = curl([API + "/coop-populations/" + PID + "/movements?limit=5"] + H)["data"]["data"]
sources = sorted(m["movementSource"] for m in mv[:2])
check("3 two movements written", sources, ["CULLING", "MORTALITY"])

# Review Focus: perubahan menulis gerakan SELISIH, bukan gerakan penuh kedua
r = curl(["-X", "PATCH", API + "/coop-depletions/" + eid] + H +
         ["-d", json.dumps({"mortalityCount": 7})])
check("4 balance corrected upward by 3", onhand(PID), start - 12)
top = curl([API + "/coop-populations/" + PID + "/movements?limit=1"] + H)["data"]["data"][0]
check("5 correction is an IN movement", top["movementType"], "IN")
check("6 correction quantity is the delta", top["quantity"], 3)

check("7 duplicate entry for same coop+date rejected",
      curl(["-X", "POST", API + "/coop-depletions"] + H +
           ["-d", json.dumps({"projectCoopId": PID, "entryDate": DATE, "mortalityCount": 1})]).get("statusCode"), 409)

check("8 negative count rejected",
      curl(["-X", "POST", API + "/coop-depletions"] + H +
           ["-d", json.dumps({"projectCoopId": PID, "entryDate": "2026-08-01", "mortalityCount": -1})]).get("statusCode"), 400)
check("9 fractional count rejected",
      curl(["-X", "POST", API + "/coop-depletions"] + H +
           ["-d", json.dumps({"projectCoopId": PID, "entryDate": "2026-08-02", "mortalityCount": 1.5})]).get("statusCode"), 400)
check("10 unknown project coop rejected",
      curl(["-X", "POST", API + "/coop-depletions"] + H +
           ["-d", json.dumps({"projectCoopId": "00000000-0000-4000-8000-000000000000",
                              "entryDate": "2026-08-03", "mortalityCount": 1})]).get("statusCode"), 400)

# Review Focus 3: perubahan yang membuat saldo negatif ditolak dan tidak menyisakan gerakan
before_count = curl([API + "/coop-populations/" + PID + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("11 update beyond balance rejected",
      curl(["-X", "PATCH", API + "/coop-depletions/" + eid] + H +
           ["-d", json.dumps({"mortalityCount": 999999999})]).get("statusCode"), 400)
check("12 rejected update left no ledger row",
      curl([API + "/coop-populations/" + PID + "/movements?limit=1"] + H)["data"]["meta"]["total"], before_count)

# hapus mengembalikan seluruhnya
curl(["-X", "DELETE", API + "/coop-depletions/" + eid] + H)
check("13 delete reverses the whole entry", onhand(PID), start)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-depletion.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 1 karena rute `/coop-depletions` belum ada.

- [ ] **Step 3: Tulis DTO**

Create `breeding-app/src/modules/project/dto/create-coop-depletion.dto.ts`:

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsUUID, IsDateString, IsInt, Min, IsOptional, IsString } from 'class-validator';

export class CreateCoopDepletionDto {
  @ApiProperty()
  @IsUUID()
  projectCoopId!: string;

  @ApiProperty({ example: '2026-09-27' })
  @IsDateString()
  entryDate!: string;

  @ApiPropertyOptional({ default: 0 })
  @IsOptional()
  @IsInt()
  @Min(0)
  mortalityCount?: number;

  @ApiPropertyOptional({ default: 0 })
  @IsOptional()
  @IsInt()
  @Min(0)
  cullingCount?: number;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  notes?: string;
}
```

Create `breeding-app/src/modules/project/dto/update-coop-depletion.dto.ts`:

```ts
import { PartialType, OmitType } from '@nestjs/swagger';
import { CreateCoopDepletionDto } from './create-coop-depletion.dto.js';

/** Kandang dan tanggal tidak bisa dipindah — hapus lalu buat lagi. */
export class UpdateCoopDepletionDto extends PartialType(
  OmitType(CreateCoopDepletionDto, ['projectCoopId', 'entryDate'] as const),
) {}
```

Create `breeding-app/src/modules/project/dto/query-coop-depletion.dto.ts`:

```ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsUUID, IsDateString } from 'class-validator';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

export class QueryCoopDepletionDto extends PaginationDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  projectCoopId?: string;

  @ApiPropertyOptional({ example: '2026-09-01' })
  @IsOptional()
  @IsDateString()
  dateFrom?: string;

  @ApiPropertyOptional({ example: '2026-09-30' })
  @IsOptional()
  @IsDateString()
  dateTo?: string;
}
```

- [ ] **Step 4: Tulis service**

Create `breeding-app/src/modules/project/coop-depletion.service.ts`:

```ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';
import { Prisma, BirdMovementSource } from '../../../generated/prisma/client.js';
import { PaginatedResult } from '../../common/dto/pagination.dto.js';
import { CoopPopulationService } from './coop-population.service.js';
import { CreateCoopDepletionDto } from './dto/create-coop-depletion.dto.js';
import { UpdateCoopDepletionDto } from './dto/update-coop-depletion.dto.js';
import { QueryCoopDepletionDto } from './dto/query-coop-depletion.dto.js';

const SOURCE_DOC_TYPE = 'CoopDepletionEntry';

@Injectable()
export class CoopDepletionService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly population: CoopPopulationService,
  ) {}

  private readonly include = {
    projectCoop: {
      select: { id: true, coopName: true, coop: { select: { id: true, code: true, name: true } } },
    },
  } as const;

  /**
   * Kunci unik (kandang, tanggal) bertahan melewati soft delete, jadi entri
   * yang dihapus masih memegang tanggalnya. Dipetakan ke 409 supaya pengguna
   * tidak melihat string Prisma mentah.
   */
  private rethrowDuplicateEntry(error: unknown): never {
    if (
      typeof error === 'object' && error !== null &&
      (error as { code?: string }).code === 'P2002'
    ) {
      throw new ConflictException('Entri deplesi untuk kandang dan tanggal ini sudah ada');
    }
    throw error;
  }

  /**
   * Menulis gerakan selisih per jenis. Nilai 10 → 7 menghasilkan gerakan IN
   * sebanyak 3, bukan gerakan OUT kedua — ledger tetap append-only dan saldo
   * tetap benar.
   */
  private async applyDelta(
    tx: Prisma.TransactionClient,
    tenantId: string,
    projectCoopId: string,
    entryId: string,
    movementDate: Date,
    source: BirdMovementSource,
    before: number,
    after: number,
    userId?: string,
  ) {
    const diff = after - before;
    if (diff === 0) return;
    await this.population.applyMovement(tx, {
      tenantId,
      projectCoopId,
      movementType: diff > 0 ? 'OUT' : 'IN',
      movementSource: source,
      quantity: Math.abs(diff),
      movementDate,
      sourceDocId: entryId,
      sourceDocType: SOURCE_DOC_TYPE,
      createdBy: userId,
    });
  }

  async create(tenantId: string, dto: CreateCoopDepletionDto, userId?: string) {
    return this.prisma
      .$transaction(async (tx) => {
        await this.population.resolveProjectCoop(tx, dto.projectCoopId, tenantId);
        const entryDate = new Date(dto.entryDate);
        const mortality = dto.mortalityCount ?? 0;
        const culling = dto.cullingCount ?? 0;

        const entry = await tx.coopDepletionEntry.create({
          data: {
            tenantId,
            projectCoopId: dto.projectCoopId,
            entryDate,
            mortalityCount: mortality,
            cullingCount: culling,
            notes: dto.notes,
            createdBy: userId,
          },
        });

        await this.applyDelta(tx, tenantId, dto.projectCoopId, entry.id, entryDate, 'MORTALITY', 0, mortality, userId);
        await this.applyDelta(tx, tenantId, dto.projectCoopId, entry.id, entryDate, 'CULLING', 0, culling, userId);

        return tx.coopDepletionEntry.findUniqueOrThrow({
          where: { id: entry.id },
          include: this.include,
        });
      })
      .catch((error) => this.rethrowDuplicateEntry(error));
  }

  async findAll(tenantId: string, query: QueryCoopDepletionDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(query.projectCoopId && { projectCoopId: query.projectCoopId }),
      ...((query.dateFrom || query.dateTo) && {
        entryDate: {
          ...(query.dateFrom && { gte: new Date(query.dateFrom) }),
          ...(query.dateTo && { lte: new Date(query.dateTo) }),
        },
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.coopDepletionEntry.findMany({
        where,
        include: this.include,
        skip: query.skip,
        take: query.limit,
        orderBy: [{ entryDate: 'desc' }, { createdAt: 'desc' }],
      }),
      this.prisma.coopDepletionEntry.count({ where }),
    ]);
    return new PaginatedResult(data, total, query.page, query.limit);
  }

  async findOne(tenantId: string, id: string) {
    const entry = await this.prisma.coopDepletionEntry.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!entry) throw new NotFoundException('Entri deplesi tidak ditemukan');
    return entry;
  }

  async update(tenantId: string, id: string, dto: UpdateCoopDepletionDto, userId?: string) {
    const current = await this.findOne(tenantId, id);

    return this.prisma.$transaction(async (tx) => {
      const mortality = dto.mortalityCount ?? current.mortalityCount;
      const culling = dto.cullingCount ?? current.cullingCount;

      await this.applyDelta(tx, tenantId, current.projectCoopId, id, current.entryDate,
        'MORTALITY', current.mortalityCount, mortality, userId);
      await this.applyDelta(tx, tenantId, current.projectCoopId, id, current.entryDate,
        'CULLING', current.cullingCount, culling, userId);

      await tx.coopDepletionEntry.update({
        where: { id },
        data: { mortalityCount: mortality, cullingCount: culling, notes: dto.notes },
      });

      return tx.coopDepletionEntry.findUniqueOrThrow({ where: { id }, include: this.include });
    });
  }

  async remove(tenantId: string, id: string, userId?: string) {
    const current = await this.findOne(tenantId, id);

    return this.prisma.$transaction(async (tx) => {
      await this.applyDelta(tx, tenantId, current.projectCoopId, id, current.entryDate,
        'MORTALITY', current.mortalityCount, 0, userId);
      await this.applyDelta(tx, tenantId, current.projectCoopId, id, current.entryDate,
        'CULLING', current.cullingCount, 0, userId);

      return tx.coopDepletionEntry.update({
        where: { id },
        data: { deletedAt: new Date() },
      });
    });
  }
}
```

- [ ] **Step 5: Tulis controller**

Create `breeding-app/src/modules/project/coop-depletion.controller.ts`:

```ts
import { Controller, Get, Post, Patch, Delete, Body, Param, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { CoopDepletionService } from './coop-depletion.service.js';
import { CreateCoopDepletionDto } from './dto/create-coop-depletion.dto.js';
import { UpdateCoopDepletionDto } from './dto/update-coop-depletion.dto.js';
import { QueryCoopDepletionDto } from './dto/query-coop-depletion.dto.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { RolesGuard } from '../../common/guards/roles.guard.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { CurrentUser } from '../../common/decorators/current-user.decorator.js';
import { JwtPayload } from '../../common/interfaces/jwt-payload.interface.js';

@ApiTags('coop-depletions')
@ApiBearerAuth()
@Controller('coop-depletions')
@UseGuards(JwtAuthGuard, RolesGuard)
export class CoopDepletionController {
  constructor(private readonly service: CoopDepletionService) {}

  @Get()
  @ApiOperation({ summary: 'Daftar entri deplesi harian' })
  findAll(@CurrentTenant() tenantId: string, @Query() query: QueryCoopDepletionDto) {
    return this.service.findAll(tenantId, query);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Satu entri deplesi' })
  findOne(@CurrentTenant() tenantId: string, @Param('id') id: string) {
    return this.service.findOne(tenantId, id);
  }

  @Post()
  @ApiOperation({ summary: 'Catat deplesi harian satu kandang' })
  create(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Body() dto: CreateCoopDepletionDto,
  ) {
    return this.service.create(tenantId, dto, user.sub);
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Perbaiki entri deplesi' })
  update(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
    @Body() dto: UpdateCoopDepletionDto,
  ) {
    return this.service.update(tenantId, id, dto, user.sub);
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Hapus entri deplesi dan kembalikan populasinya' })
  remove(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
  ) {
    return this.service.remove(tenantId, id, user.sub);
  }
}
```

Tidak ada `@Roles` di sini — STAFF boleh mengisi deplesi, dan `RolesGuard` mengembalikan true saat metadata `@Roles` tidak ada.

Periksa jalur impor `JwtPayload` dengan `grep -rn "JwtPayload" breeding-app/src/modules/transfer/goods-transfer.controller.ts` dan samakan kalau berbeda.

- [ ] **Step 6: Daftarkan di module**

Tambahkan `CoopDepletionService` ke `providers` dan `CoopDepletionController` ke `controllers` di `project.module.ts`.

- [ ] **Step 7: Build dan jalankan**

Run: `cd breeding-app && npm run build`, hidupkan lagi dev server, lalu `python3 scratchpad/verify-depletion.py`
Expected: `RESULT: all checks pass` — 13 dari 13.

- [ ] **Step 8: Commit**

```bash
cd breeding-app
git add src/modules/project
git commit -m "feat(project): daily depletion entries with delta corrections"
```

---

### Task 6: Pindah ayam antar kandang

**Files:**
- Create: `breeding-app/src/modules/project/coop-bird-transfer.service.ts`
- Create: `breeding-app/src/modules/project/coop-bird-transfer.controller.ts`
- Create: `breeding-app/src/modules/project/dto/create-coop-bird-transfer.dto.ts`
- Create: `breeding-app/src/modules/project/dto/query-coop-bird-transfer.dto.ts`
- Modify: `breeding-app/src/modules/project/project.module.ts`
- Test: `scratchpad/verify-bird-transfer.py`

**Interfaces:**
- Consumes: `CoopPopulationService.applyMovement`, `.resolveProjectCoop`; `ReferenceNumberGenerator.generate(tx, tenantId, prefix)`.
- Produces: rute `GET/POST /coop-bird-transfers`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-bird-transfer.py`:

```python
import json, subprocess, time
RUN = str(int(time.time()))[-6:]
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

fails = []
def check(label, actual, expected):
    ok = actual == expected
    if not ok: fails.append(label)
    print(f"{'PASS' if ok else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

def onhand(pid):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]["quantityOnHand"]

rows = curl([API + "/coop-populations?limit=10"] + H)["data"]["data"]
assert len(rows) >= 2, "butuh minimal dua siklus kandang di seed"
SRC, DST = rows[0]["id"], rows[1]["id"]

for pid in (SRC, DST):
    curl(["-X", "POST", API + "/coop-populations/" + pid + "/adjustments"] + H +
         ["-d", json.dumps({"quantity": 200, "direction": "IN", "reason": "seed " + RUN, "movementDate": "2026-09-27"})])
src0, dst0 = onhand(SRC), onhand(DST)

def transfer(body):
    return curl(["-X", "POST", API + "/coop-bird-transfers"] + H + ["-d", json.dumps(body)])

r = transfer({"sourceProjectCoopId": SRC, "destinationProjectCoopId": DST,
              "quantity": 50, "transferDate": "2026-09-27"})
check("1 transfer numbered", r["data"]["transferNumber"].startswith("CBT-"), True)
check("2 source reduced", onhand(SRC), src0 - 50)
check("3 destination raised", onhand(DST), dst0 + 50)

src_mv = curl([API + "/coop-populations/" + SRC + "/movements?limit=1"] + H)["data"]["data"][0]
dst_mv = curl([API + "/coop-populations/" + DST + "/movements?limit=1"] + H)["data"]["data"][0]
check("4 source movement source", src_mv["movementSource"], "TRANSFER_OUT")
check("5 destination movement source", dst_mv["movementSource"], "TRANSFER_IN")

check("6 same coop rejected",
      transfer({"sourceProjectCoopId": SRC, "destinationProjectCoopId": SRC,
                "quantity": 5, "transferDate": "2026-09-27"}).get("statusCode"), 400)
check("7 quantity beyond source balance rejected",
      transfer({"sourceProjectCoopId": SRC, "destinationProjectCoopId": DST,
                "quantity": 999999999, "transferDate": "2026-09-27"}).get("statusCode"), 400)
check("8 zero quantity rejected",
      transfer({"sourceProjectCoopId": SRC, "destinationProjectCoopId": DST,
                "quantity": 0, "transferDate": "2026-09-27"}).get("statusCode"), 400)
check("9 fractional quantity rejected",
      transfer({"sourceProjectCoopId": SRC, "destinationProjectCoopId": DST,
                "quantity": 2.5, "transferDate": "2026-09-27"}).get("statusCode"), 400)

# Review Focus 4: tujuan tidak sah tidak boleh menyisakan gerakan di sisi asal
before = curl([API + "/coop-populations/" + SRC + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("10 unknown destination rejected",
      transfer({"sourceProjectCoopId": SRC,
                "destinationProjectCoopId": "00000000-0000-4000-8000-000000000000",
                "quantity": 5, "transferDate": "2026-09-27"}).get("statusCode"), 400)
check("11 failed transfer left source untouched",
      curl([API + "/coop-populations/" + SRC + "/movements?limit=1"] + H)["data"]["meta"]["total"], before)
check("12 source balance unchanged after failure", onhand(SRC), src0 - 50)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-bird-transfer.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 1 karena rute `/coop-bird-transfers` belum ada.

- [ ] **Step 3: Tulis DTO**

Create `breeding-app/src/modules/project/dto/create-coop-bird-transfer.dto.ts`:

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsUUID, IsInt, IsPositive, IsDateString, IsOptional, IsString } from 'class-validator';

export class CreateCoopBirdTransferDto {
  @ApiProperty()
  @IsUUID()
  sourceProjectCoopId!: string;

  @ApiProperty()
  @IsUUID()
  destinationProjectCoopId!: string;

  @ApiProperty({ description: 'Jumlah ekor yang dipindah' })
  @IsInt()
  @IsPositive()
  quantity!: number;

  @ApiProperty({ example: '2026-09-27' })
  @IsDateString()
  transferDate!: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  notes?: string;
}
```

Create `breeding-app/src/modules/project/dto/query-coop-bird-transfer.dto.ts`:

```ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsUUID } from 'class-validator';
import { PaginationDto } from '../../../common/dto/pagination.dto.js';

export class QueryCoopBirdTransferDto extends PaginationDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsUUID()
  projectCoopId?: string;
}
```

- [ ] **Step 4: Tulis service**

Create `breeding-app/src/modules/project/coop-bird-transfer.service.ts`:

```ts
import { Injectable, BadRequestException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';
import { ReferenceNumberGenerator } from '../../common/utils/reference-number.generator.js';
import { PaginatedResult } from '../../common/dto/pagination.dto.js';
import { CoopPopulationService } from './coop-population.service.js';
import { CreateCoopBirdTransferDto } from './dto/create-coop-bird-transfer.dto.js';
import { QueryCoopBirdTransferDto } from './dto/query-coop-bird-transfer.dto.js';

const SOURCE_DOC_TYPE = 'CoopBirdTransfer';

@Injectable()
export class CoopBirdTransferService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly population: CoopPopulationService,
    private readonly refGenerator: ReferenceNumberGenerator,
  ) {}

  private readonly include = {
    sourceProjectCoop: { select: { id: true, coopName: true, coop: { select: { code: true, name: true } } } },
    destinationProjectCoop: { select: { id: true, coopName: true, coop: { select: { code: true, name: true } } } },
  } as const;

  private rethrowDuplicateNumber(error: unknown): never {
    if (
      typeof error === 'object' && error !== null &&
      (error as { code?: string }).code === 'P2002'
    ) {
      throw new ConflictException('Nomor pindah ayam sudah dipakai');
    }
    throw error;
  }

  /**
   * Kedua sisi ditulis dalam satu transaksi: kalau sisi tujuan gagal — kandang
   * tenant lain, saldo tidak cukup — gerakan di sisi asal ikut dibatalkan,
   * sehingga tidak ada ayam yang hilang di tengah jalan.
   */
  async create(tenantId: string, dto: CreateCoopBirdTransferDto, userId?: string) {
    if (dto.sourceProjectCoopId === dto.destinationProjectCoopId) {
      throw new BadRequestException('Kandang asal dan tujuan tidak boleh sama');
    }

    return this.prisma
      .$transaction(async (tx) => {
        await this.population.resolveProjectCoop(tx, dto.sourceProjectCoopId, tenantId);
        await this.population.resolveProjectCoop(tx, dto.destinationProjectCoopId, tenantId);

        const transferNumber = await this.refGenerator.generate(tx, tenantId, 'CBT');
        const transferDate = new Date(dto.transferDate);

        const transfer = await tx.coopBirdTransfer.create({
          data: {
            tenantId,
            transferNumber,
            sourceProjectCoopId: dto.sourceProjectCoopId,
            destinationProjectCoopId: dto.destinationProjectCoopId,
            quantity: dto.quantity,
            transferDate,
            notes: dto.notes,
            createdBy: userId,
          },
        });

        await this.population.applyMovement(tx, {
          tenantId,
          projectCoopId: dto.sourceProjectCoopId,
          movementType: 'OUT',
          movementSource: 'TRANSFER_OUT',
          quantity: dto.quantity,
          movementDate: transferDate,
          sourceDocId: transfer.id,
          sourceDocType: SOURCE_DOC_TYPE,
          createdBy: userId,
        });

        await this.population.applyMovement(tx, {
          tenantId,
          projectCoopId: dto.destinationProjectCoopId,
          movementType: 'IN',
          movementSource: 'TRANSFER_IN',
          quantity: dto.quantity,
          movementDate: transferDate,
          sourceDocId: transfer.id,
          sourceDocType: SOURCE_DOC_TYPE,
          createdBy: userId,
        });

        return tx.coopBirdTransfer.findUniqueOrThrow({
          where: { id: transfer.id },
          include: this.include,
        });
      })
      .catch((error) => this.rethrowDuplicateNumber(error));
  }

  async findAll(tenantId: string, query: QueryCoopBirdTransferDto) {
    const where = {
      tenantId,
      deletedAt: null,
      ...(query.projectCoopId && {
        OR: [
          { sourceProjectCoopId: query.projectCoopId },
          { destinationProjectCoopId: query.projectCoopId },
        ],
      }),
    };
    const [data, total] = await Promise.all([
      this.prisma.coopBirdTransfer.findMany({
        where,
        include: this.include,
        skip: query.skip,
        take: query.limit,
        orderBy: [{ transferDate: 'desc' }, { createdAt: 'desc' }],
      }),
      this.prisma.coopBirdTransfer.count({ where }),
    ]);
    return new PaginatedResult(data, total, query.page, query.limit);
  }
}
```

- [ ] **Step 5: Tulis controller**

Create `breeding-app/src/modules/project/coop-bird-transfer.controller.ts`:

```ts
import { Controller, Get, Post, Body, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { CoopBirdTransferService } from './coop-bird-transfer.service.js';
import { CreateCoopBirdTransferDto } from './dto/create-coop-bird-transfer.dto.js';
import { QueryCoopBirdTransferDto } from './dto/query-coop-bird-transfer.dto.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { RolesGuard } from '../../common/guards/roles.guard.js';
import { Roles } from '../../common/decorators/roles.decorator.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { CurrentUser } from '../../common/decorators/current-user.decorator.js';
import { JwtPayload } from '../../common/interfaces/jwt-payload.interface.js';
import { SystemRole } from '../../../generated/prisma/client.js';

@ApiTags('coop-bird-transfers')
@ApiBearerAuth()
@Controller('coop-bird-transfers')
@UseGuards(JwtAuthGuard, RolesGuard)
export class CoopBirdTransferController {
  constructor(private readonly service: CoopBirdTransferService) {}

  @Get()
  @ApiOperation({ summary: 'Daftar pindah ayam antar kandang' })
  findAll(@CurrentTenant() tenantId: string, @Query() query: QueryCoopBirdTransferDto) {
    return this.service.findAll(tenantId, query);
  }

  @Post()
  @Roles(SystemRole.MANAGER)
  @ApiOperation({ summary: 'Pindahkan ayam antar kandang' })
  create(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Body() dto: CreateCoopBirdTransferDto,
  ) {
    return this.service.create(tenantId, dto, user.sub);
  }
}
```

- [ ] **Step 6: Daftarkan di module**

Tambahkan `CoopBirdTransferService` dan `ReferenceNumberGenerator` ke `providers`, `CoopBirdTransferController` ke `controllers`.

- [ ] **Step 7: Build dan jalankan**

Run: `cd breeding-app && npm run build`, hidupkan lagi dev server, lalu `python3 scratchpad/verify-bird-transfer.py`
Expected: `RESULT: all checks pass` — 12 dari 12.

- [ ] **Step 8: Commit**

```bash
cd breeding-app
git add src/modules/project
git commit -m "feat(project): bird transfer between coops in one transaction"
```

---

### Task 7: Integrasi chick-in

**Files:**
- Modify: `breeding-app/src/modules/project/project-chick-in.service.ts`
- Test: `scratchpad/verify-chickin.py`

**Interfaces:**
- Consumes: `CoopPopulationService.applyMovement`, `.resolveProjectCoop`.
- Produces: tidak ada rute baru — perilaku rute `projects/:projectId/coops/:projectCoopId/chick-ins` yang sudah ada bertambah.

**Catatan penting:** `ProjectChickInService.remove` saat ini melakukan **hard delete** dan `ProjectChickIn` tidak punya `deletedAt`. Pertahankan perilaku itu; yang ditambahkan hanya gerakan pembalik di transaksi yang sama. Mengubahnya jadi soft delete di luar lingkup plan ini.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-chickin.py`:

```python
import json, subprocess, time
RUN = str(int(time.time()))[-6:]
API = "http://localhost:3002/api"

def curl(args):
    return json.loads(subprocess.run(["rtk", "proxy", "curl", "-s"] + args, capture_output=True, text=True).stdout)

tok = curl(["-X", "POST", API + "/auth/login", "-H", "Content-Type: application/json",
            "-d", '{"email":"admin@demo.farm","password":"password123"}'])["data"]["accessToken"]
H = ["-H", "Authorization: Bearer " + tok, "-H", "Content-Type: application/json"]

fails = []
def check(label, actual, expected):
    ok = actual == expected
    if not ok: fails.append(label)
    print(f"{'PASS' if ok else 'FAIL'} {label}: got {actual!r}, expected {expected!r}")

def onhand(pid):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]["quantityOnHand"]

pc = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]
PID, PROJ = pc["id"], pc["project"]["id"]
BASE = API + "/projects/" + PROJ + "/coops/" + PID + "/chick-ins"
start = onhand(PID)

r = curl(["-X", "POST", BASE] + H + ["-d", json.dumps({"population": 800})])
cid = r["data"]["id"]
check("1 chick-in raises balance", onhand(PID), start + 800)

top = curl([API + "/coop-populations/" + PID + "/movements?limit=1"] + H)["data"]["data"][0]
check("2 movement source is CHICK_IN", top["movementSource"], "CHICK_IN")
check("3 movement links back to the chick-in", top["sourceDocId"], cid)

curl(["-X", "PATCH", BASE + "/" + cid] + H + ["-d", json.dumps({"population": 750})])
check("4 lowering the chick-in writes the difference", onhand(PID), start + 750)
top = curl([API + "/coop-populations/" + PID + "/movements?limit=1"] + H)["data"]["data"][0]
check("5 correction is an OUT movement of 50", [top["movementType"], top["quantity"]], ["OUT", 50])

curl(["-X", "DELETE", BASE + "/" + cid] + H)
check("6 deleting the chick-in reverses it fully", onhand(PID), start)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-chickin.py`
Expected: FAIL pada pemeriksaan 1 — saldo tidak berubah karena chick-in belum menulis gerakan.

- [ ] **Step 3: Ubah service**

Di `project-chick-in.service.ts`: suntikkan `CoopPopulationService`, bungkus `create`, `update`, dan `remove` dalam `$transaction`, dan terbitkan gerakan. Tenant diambil dari proyek lewat `resolveProjectCoop`.

```ts
  constructor(
    private readonly prisma: PrismaService,
    private readonly population: CoopPopulationService,
  ) {}

  async create(projectCoopId: string, dto: CreateProjectChickInDto) {
    return this.prisma.$transaction(async (tx) => {
      const pc = await tx.projectCoop.findFirst({
        where: { id: projectCoopId },
        include: { project: { select: { tenantId: true } } },
      });
      if (!pc) throw new NotFoundException('Project coop not found');

      const chickIn = await tx.projectChickIn.create({
        data: { ...dto, ...this.convertDates(dto), projectCoopId },
      });

      await this.population.applyMovement(tx, {
        tenantId: pc.project.tenantId,
        projectCoopId,
        movementType: 'IN',
        movementSource: 'CHICK_IN',
        quantity: chickIn.population,
        movementDate: chickIn.rearingStartDate ?? new Date(),
        sourceDocId: chickIn.id,
        sourceDocType: 'ProjectChickIn',
      });

      return chickIn;
    });
  }
```

```ts
  async update(id: string, projectCoopId: string, dto: UpdateProjectChickInDto) {
    const current = await this.findOne(id, projectCoopId);

    return this.prisma.$transaction(async (tx) => {
      const pc = await tx.projectCoop.findFirstOrThrow({
        where: { id: projectCoopId },
        include: { project: { select: { tenantId: true } } },
      });

      // Gerakan selisih, bukan gerakan penuh kedua: ledger tetap append-only
      // dan saldo tidak dihitung dua kali.
      if (dto.population !== undefined && dto.population !== current.population) {
        const diff = dto.population - current.population;
        await this.population.applyMovement(tx, {
          tenantId: pc.project.tenantId,
          projectCoopId,
          movementType: diff > 0 ? 'IN' : 'OUT',
          movementSource: 'CHICK_IN',
          quantity: Math.abs(diff),
          movementDate: current.rearingStartDate ?? new Date(),
          sourceDocId: id,
          sourceDocType: 'ProjectChickIn',
        });
      }

      return tx.projectChickIn.update({
        where: { id },
        data: { ...dto, ...this.convertDates(dto) },
      });
    });
  }

  async remove(id: string, projectCoopId: string) {
    const current = await this.findOne(id, projectCoopId);

    return this.prisma.$transaction(async (tx) => {
      const pc = await tx.projectCoop.findFirstOrThrow({
        where: { id: projectCoopId },
        include: { project: { select: { tenantId: true } } },
      });

      if (current.population > 0) {
        await this.population.applyMovement(tx, {
          tenantId: pc.project.tenantId,
          projectCoopId,
          movementType: 'OUT',
          movementSource: 'CHICK_IN',
          quantity: current.population,
          movementDate: current.rearingStartDate ?? new Date(),
          sourceDocId: id,
          sourceDocType: 'ProjectChickIn',
        });
      }

      // Hard delete dipertahankan: `ProjectChickIn` tidak punya `deletedAt`,
      // dan mengubahnya jadi soft delete di luar lingkup plan ini.
      return tx.projectChickIn.delete({ where: { id } });
    });
  }
```

- [ ] **Step 4: Daftarkan ketergantungan**

`CoopPopulationService` sudah ada di `providers` `ProjectModule` sejak Task 2, jadi injeksi langsung jalan. Pastikan tidak ada lingkaran impor: `CoopPopulationService` tidak boleh mengimpor `ProjectChickInService`.

- [ ] **Step 5: Build dan jalankan**

Run: `cd breeding-app && npm run build`, hidupkan lagi dev server, lalu `python3 scratchpad/verify-chickin.py`
Expected: `RESULT: all checks pass` — 6 dari 6.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add src/modules/project/project-chick-in.service.ts
git commit -m "feat(project): chick-in emits population movements"
```

---

### Task 8: Seed

**Files:**
- Modify: `breeding-app/prisma/seed.ts:449-461`

**Interfaces:**
- Consumes: delegate `coopPopulation` dan `coopPopulationMovement` dari Task 1.
- Produces: data pengembangan dengan saldo populasi yang terisi.

- [ ] **Step 1: Tulis saldo dan gerakan di seed**

Seed memakai `PrismaClient` langsung, bukan service Nest, jadi gerakan ditulis apa adanya. Ganti dua blok `prisma.projectChickIn.create` di `prisma/seed.ts:449-461` dengan pembantu berikut, diletakkan tepat di atasnya:

```ts
  async function seedChickIn(projectCoopId: string, population: number, dates: Record<string, Date>) {
    const chickIn = await prisma.projectChickIn.create({
      data: { projectCoopId, population, ...dates },
    });
    await prisma.coopPopulation.create({
      data: {
        tenantId: tenant.id,
        projectCoopId,
        quantityOnHand: population,
        quantityAvailable: population,
      },
    });
    await prisma.coopPopulationMovement.create({
      data: {
        tenantId: tenant.id,
        projectCoopId,
        movementType: 'IN',
        movementSource: 'CHICK_IN',
        sourceDocId: chickIn.id,
        sourceDocType: 'ProjectChickIn',
        quantityBefore: 0,
        quantity: population,
        quantityAfter: population,
        movementDate: dates.rearingStartDate ?? new Date(),
      },
    });
    return chickIn;
  }

  await seedChickIn(projCoopA.id, 5000, {
    rearingStartDate: new Date('2026-02-01'), rearingEndDate: new Date('2026-03-07'),
    harvestStartDate: new Date('2026-03-08'), harvestEndDate: new Date('2026-03-10'),
  });
  await seedChickIn(projCoopB.id, 3000, {
    rearingStartDate: new Date('2026-02-03'), rearingEndDate: new Date('2026-03-09'),
  });
```

Ganti `tenant.id` dengan nama variabel tenant yang dipakai seed di sekitar baris itu — periksa dengan `grep -n "tenantId:" prisma/seed.ts | head -3`.

- [ ] **Step 2: Jalankan seed dari nol**

Run: `cd breeding-app && npx prisma migrate reset --force`
Expected: migrasi diterapkan dan seed jalan tanpa galat.

**Ini menghapus basis data pengembangan lokal.** Minta izin manusia sebelum menjalankannya kalau belum diberikan untuk plan ini.

- [ ] **Step 3: Periksa saldonya terisi**

Run: `cd breeding-app && npx prisma db execute --stdin <<< "SELECT quantity_on_hand, quantity_available FROM coop_populations ORDER BY quantity_on_hand DESC;"`
Expected: dua baris — 5000 dan 3000, `quantity_available` sama dengan `quantity_on_hand`.

- [ ] **Step 4: Commit**

```bash
cd breeding-app
git add prisma/seed.ts
git commit -m "chore(project): seed coop population from chick-ins"
```

---

### Task 9: Tipe, i18n, dan navigasi frontend

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts`
- Modify: `breeding-dashboard/src/lib/constants.ts:76-81`
- Modify: `breeding-dashboard/messages/en.json`
- Modify: `breeding-dashboard/messages/id.json`
- Create: `breeding-dashboard/src/components/forms/project-coop-combobox.tsx`

**Interfaces:**
- Consumes: bentuk respons dari Task 2, 5, 6.
- Produces: tipe `CoopPopulationRow`, `CoopPopulationMovement`, `CoopDepletionEntry`, `CoopBirdTransfer`; komponen `<ProjectCoopCombobox value onChange disabled />`; namespace i18n `coopPopulations`, `coopDepletions`, `coopBirdTransfers`.

- [ ] **Step 1: Tambahkan tipe**

Di `breeding-dashboard/src/types/api.ts`:

```ts
export interface CoopPopulationBalance {
  quantityOnHand: number;
  quantityAllocated: number;
  quantityAvailable: number;
}

export interface CoopPopulationRow {
  id: string;
  coopName: string | null;
  coop: { id: string; code: string; name: string };
  project: { id: string; startDate: string; isActive: boolean };
  population: CoopPopulationBalance;
}

export interface CoopPopulationMovement {
  id: string;
  movementType: "IN" | "OUT";
  movementSource:
    | "CHICK_IN" | "MORTALITY" | "CULLING"
    | "SALES_REALIZATION" | "TRANSFER_IN" | "TRANSFER_OUT" | "ADJUSTMENT";
  sourceDocId: string | null;
  sourceDocType: string | null;
  quantityBefore: number;
  quantity: number;
  quantityAfter: number;
  movementDate: string;
  notes: string | null;
  createdAt: string;
}

export interface CoopDepletionEntry {
  id: string;
  projectCoopId: string;
  entryDate: string;
  mortalityCount: number;
  cullingCount: number;
  notes: string | null;
  createdAt: string;
  projectCoop: { id: string; coopName: string | null; coop: { id: string; code: string; name: string } };
}

export interface CoopBirdTransfer {
  id: string;
  transferNumber: string;
  quantity: number;
  transferDate: string;
  notes: string | null;
  createdAt: string;
  sourceProjectCoop: { id: string; coopName: string | null; coop: { code: string; name: string } };
  destinationProjectCoop: { id: string; coopName: string | null; coop: { code: string; name: string } };
}
```

- [ ] **Step 2: Tambahkan entri navigasi**

Di `breeding-dashboard/src/lib/constants.ts`, grup `Projects` (baris 76-81), setelah entri `Projects`:

```ts
      { title: "Coop Population", url: "/coop-populations", icon: Layers },
      { title: "Daily Depletion", url: "/coop-depletions", icon: ClipboardList },
      { title: "Bird Transfers", url: "/coop-bird-transfers", icon: Layers },
```

`Layers` dan `ClipboardList` sudah diimpor di berkas itu.

- [ ] **Step 3: Tambahkan kunci i18n ke kedua berkas**

Sidebar menerjemahkan judul navigasi lewat namespace `navigation`, bukan lewat namespace halaman — tanpa ketiga kunci ini judul menu muncul sebagai kunci mentah. Tambahkan ke `navigation` di **`en.json` dan `id.json`**:

```json
    "Coop Population": "Coop Population",
    "Daily Depletion": "Daily Depletion",
    "Bird Transfers": "Bird Transfers"
```

Di `id.json` isinya `"Populasi Kandang"`, `"Deplesi Harian"`, `"Pindah Ayam"`.

Lalu tiga namespace halaman di kedua berkas. `en.json`:

```json
  "coopPopulations": {
    "title": "Coop Population",
    "description": "Running bird population per rearing cycle",
    "entity": "Coop Population",
    "coop": "Coop",
    "cycleStart": "Cycle start",
    "onHand": "On hand",
    "allocated": "Allocated",
    "available": "Available",
    "movements": "Movement history",
    "movementDate": "Date",
    "movementSource": "Source",
    "quantityAfter": "Balance after",
    "adjust": "Correct",
    "adjustTitle": "Correct population",
    "direction": "Direction",
    "directionIn": "Add",
    "directionOut": "Subtract",
    "quantity": "Birds",
    "reason": "Reason",
    "reasonRequired": "A reason is required for a manual correction",
    "searchPlaceholder": "Search coop...",
    "noMovements": "No movements recorded yet"
  },
  "coopDepletions": {
    "title": "Daily Depletion",
    "description": "Mortality and culling per coop per day",
    "entity": "Depletion Entry",
    "coop": "Coop",
    "entryDate": "Date",
    "mortality": "Mortality",
    "culling": "Culling",
    "notes": "Notes",
    "searchPlaceholder": "Search coop...",
    "countRequired": "Record at least one bird under mortality or culling"
  },
  "coopBirdTransfers": {
    "title": "Bird Transfers",
    "description": "Birds moved between coops",
    "entity": "Bird Transfer",
    "transferNumber": "Number",
    "source": "From",
    "destination": "To",
    "quantity": "Birds",
    "transferDate": "Date",
    "notes": "Notes",
    "sourceAvailable": "Available at source",
    "sameCoop": "Source and destination must be different coops",
    "searchPlaceholder": "Search transfer..."
  }
```

`id.json` memakai kunci yang sama persis dengan nilai bahasa Indonesia — misalnya `"onHand": "Masuk"`, `"available": "Tersedia"`, `"mortality": "Mortalitas"`, `"culling": "Afkir"`, `"reasonRequired": "Koreksi manual wajib menyertakan alasan"`.

- [ ] **Step 4: Tulis combobox**

Create `breeding-dashboard/src/components/forms/project-coop-combobox.tsx`, meniru `src/components/forms/project-combobox.tsx` persis strukturnya, tetapi memuat dari `/coop-populations` dan menampilkan `${row.coop.code} — ${row.coop.name}` sebagai label, dengan sisa populasi di sebelah kanan:

```tsx
  useEffect(() => {
    setIsLoading(true);
    fetchPaginated<CoopPopulationRow>("/coop-populations", { limit: 50, search })
      .then((res) => setRows(res.data))
      .catch(() => {})
      .finally(() => setIsLoading(false));
  }, [search]);
```

Props: `{ value: string; onChange: (projectCoopId: string) => void; disabled?: boolean }`.

- [ ] **Step 5: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 6: Periksa kedua berkas pesan masih JSON yang sah**

Run: `cd breeding-dashboard && node -e "['en','id'].forEach(l=>{const m=require('./messages/'+l+'.json');console.log(l,Object.keys(m.coopPopulations).length,Object.keys(m.coopDepletions).length,Object.keys(m.coopBirdTransfers).length,['Coop Population','Daily Depletion','Bird Transfers'].every(k=>k in m.navigation))})"`
Expected: `en 22 10 12 true` dan `id 22 10 12 true` — jumlah kunci sama di kedua bahasa dan ketiga judul navigasi ada.

- [ ] **Step 7: Commit**

```bash
cd breeding-dashboard
git add src/types/api.ts src/lib/constants.ts messages src/components/forms/project-coop-combobox.tsx
git commit -m "feat(project): coop population types, i18n keys and project coop combobox"
```

---

### Task 10: Halaman Populasi Kandang

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/coop-populations/page.tsx`

**Interfaces:**
- Consumes: `CoopPopulationRow`, `CoopPopulationMovement` dari Task 9; rute Task 2 dan Task 3.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tulis halaman**

Meniru struktur `src/app/(dashboard)/mature-bird-standards/page.tsx`: `usePaginated`, `DataTable`, `PageHeader`, dialog shadcn. Kolom: Coop (`row.coop.code — row.coop.name`), Cycle start, On hand, Allocated, Available, Actions.

Baris yang diklik membuka `Dialog` berisi riwayat gerakan, dimuat lewat `fetchPaginated<CoopPopulationMovement>` dari `/coop-populations/${id}/movements`. Tombol **Correct** membuka dialog koreksi.

Dialog koreksi mengirim:

```ts
await fetchApi(`/coop-populations/${row.id}/adjustments`, {
  method: "POST",
  body: JSON.stringify({
    quantity: Number(quantity),
    direction,
    reason: reason.trim(),
    movementDate,
  }),
});
```

Validasi klien sebelum mengirim — tanpa ini backend menolak dengan 400 dan pengguna tidak tahu bidang mana yang salah:

```ts
if (!reason.trim()) {
  toast.error(t('reasonRequired'));
  return;
}
const qty = Number(quantity);
if (!Number.isInteger(qty) || qty <= 0) {
  toast.error(tc('required', { field: t('quantity') }));
  return;
}
```

Tangkap galat dengan memunculkan pesan asli dari API, seperti halaman customers:

```ts
} catch (error) {
  toast.error(error instanceof Error ? error.message : tc('entityUpdateFailed', { entity: t('entity') }));
}
```

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 3: Smoke browser**

Buka `http://localhost:3005/coop-populations`. Buktikan, dan catat hasilnya:
1. Tabel memuat dua kandang dari seed dengan On hand 5000 dan 3000.
2. Klik baris membuka riwayat; gerakan `CHICK_IN` terlihat.
3. Koreksi +100 dengan alasan tersimpan, saldo jadi 5100, dan gerakan `ADJUSTMENT` muncul paling atas.
4. Koreksi tanpa alasan memunculkan toast dan dialog tetap terbuka.
5. Koreksi −100 mengembalikan saldo ke 5000. Bersihkan sesudahnya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/coop-populations"
git commit -m "feat(project): coop population page with movement history and correction"
```

---

### Task 11: Halaman Deplesi Harian

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/coop-depletions/page.tsx`

**Interfaces:**
- Consumes: `CoopDepletionEntry`, `<ProjectCoopCombobox />` dari Task 9; rute Task 5.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tulis halaman**

Struktur sama seperti Task 10. Kolom: Date, Coop, Mortality, Culling, Notes, Actions (edit, hapus). Dialog berisi `<ProjectCoopCombobox />`, input tanggal, dua input angka, dan catatan.

Saat mengedit, kandang dan tanggal dikunci (`disabled`) karena backend menolak perubahan keduanya — DTO `UpdateCoopDepletionDto` menghilangkan kedua bidang itu, jadi mengirimnya menghasilkan 400 dari `forbidNonWhitelisted`.

Validasi klien:

```ts
const mortality = Number(formMortality || 0);
const culling = Number(formCulling || 0);
if (mortality + culling <= 0) {
  toast.error(t('countRequired'));
  return;
}
```

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 3: Smoke browser**

Buka `http://localhost:3005/coop-depletions`. Buktikan, dan catat hasilnya:
1. Entri baru dengan mortalitas 10 dan afkir 5 tersimpan.
2. Halaman populasi menunjukkan saldo kandang itu turun 15.
3. Entri yang sama diubah jadi mortalitas 7 — saldo naik 3, bukan turun lagi.
4. Entri kedua untuk kandang dan tanggal yang sama memunculkan toast 409 berbahasa manusia, bukan string Prisma.
5. Menghapus entri mengembalikan saldo ke posisi semula. Bersihkan sesudahnya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/coop-depletions"
git commit -m "feat(project): daily depletion page"
```

---

### Task 12: Halaman Pindah Ayam

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/coop-bird-transfers/page.tsx`

**Interfaces:**
- Consumes: `CoopBirdTransfer`, `<ProjectCoopCombobox />` dari Task 9; rute Task 6.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tulis halaman**

Kolom: Number, Date, From, To, Birds, Notes. Tidak ada aksi edit atau hapus — dokumen ini tidak bisa diubah setelah dibuat; pembetulan lewat koreksi manual di halaman populasi. Sebutkan itu di `DialogDescription` supaya pengguna tahu sebelum menyimpan.

Dialog: dua `<ProjectCoopCombobox />`, jumlah, tanggal, catatan. Begitu kandang asal dipilih, tampilkan sisa populasinya:

```ts
useEffect(() => {
  if (!formSource) { setSourceAvailable(null); return; }
  fetchApi<CoopPopulationRow>(`/coop-populations/${formSource}`)
    .then((res) => setSourceAvailable(res.population.quantityAvailable))
    .catch(() => setSourceAvailable(null));
}, [formSource]);
```

Validasi klien: kandang asal dan tujuan tidak boleh sama (`toast.error(t('sameCoop'))`), jumlah harus bilangan bulat positif.

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 3: Smoke browser**

Buka `http://localhost:3005/coop-bird-transfers`. Buktikan, dan catat hasilnya:
1. Pindah 50 ekor dari kandang A ke B tersimpan dengan nomor berawalan `CBT-`.
2. Halaman populasi menunjukkan A turun 50 dan B naik 50.
3. Memilih kandang yang sama di kedua sisi memunculkan toast dan dialog tetap terbuka.
4. Jumlah melebihi sisa kandang asal ditolak dengan pesan dari API. Bersihkan lewat koreksi manual sesudahnya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/coop-bird-transfers"
git commit -m "feat(project): bird transfer page"
```

---

### Task 13: Perbarui dokumen tahapan

**Files:**
- Modify: `docs/superpowers/specs/2026-09-27-coop-population-ledger-design.md`
- Modify: `docs/superpowers/specs/2026-09-13-sales-module-finding.md`

**Interfaces:**
- Consumes: hasil Task 1-12.
- Produces: tidak ada.

- [ ] **Step 1: Tandai spec sudah diimplementasi**

Ubah baris `**Status:**` di `2026-09-27-coop-population-ledger-design.md` menjadi:

```markdown
**Status:** Implemented (2026-09-27) — lihat plan `docs/superpowers/plans/2026-09-27-coop-population-ledger.md`
```

- [ ] **Step 2: Catat prasyarat S-B sudah terpenuhi**

Di tabel tahapan `2026-09-13-sales-module-finding.md`, pada baris **S-B**, tambahkan di kolom Status:

```
Siap — prasyarat sisa stok kandang terpenuhi oleh ledger populasi (2026-09-27)
```

- [ ] **Step 3: Periksa**

Run: `cd /Users/alva.e202511001/Desktop/project/breeding && grep -n "Implemented (2026-09-27)" docs/superpowers/specs/2026-09-27-coop-population-ledger-design.md && grep -n "ledger populasi" docs/superpowers/specs/2026-09-13-sales-module-finding.md`
Expected: masing-masing satu baris cocok.

- [ ] **Step 4: Commit**

```bash
cd /Users/alva.e202511001/Desktop/project/breeding
git add docs/superpowers/specs
git commit -m "docs(project): mark coop population ledger implemented"
```
