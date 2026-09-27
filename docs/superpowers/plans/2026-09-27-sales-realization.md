# Realisasi DO (S-C) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mencatat penyerahan ayam ke customer — satu dokumen realisasi per DO dengan satu baris per pengambilan — yang memotong populasi kandang dan melepaskan janji yang sudah ditunaikan.

**Architecture:** Header `SalesRealization` 1:1 ke `SalesOrder`, banyak `SalesRealizationLine`. Setiap baris menulis gerakan `OUT`/`SALES_REALIZATION` lewat `CoopPopulationService.applyMovement` lalu melepas alokasi sebanyak `min(ekor baris, sisa tahanan DO di kandang itu)`, keduanya dalam satu transaksi, gerakan dulu supaya pemeriksaan fisik yang menentukan. Jumlah yang benar-benar dilepas disimpan di barisnya, sehingga penghapusan menahan kembali angka yang sama persis alih-alih menghitung ulang.

**Tech Stack:** NestJS 11, Prisma 7 (client di `generated/prisma/`), PostgreSQL 15, Next.js 16, shadcn/ui, next-intl.

**Spec:** `docs/superpowers/specs/2026-09-27-sales-realization-design.md`

## Global Constraints

- Satu realisasi per DO: `@@unique` pada `salesOrderId` (R1).
- `dtpsNumber` wajib, dan unik per tenant lewat `@@unique([tenantId, dtpsNumber])` di **basis data** — bukan pemeriksaan di service (R2).
- `avgWeightKg` selalu dihitung backend dari `totalWeightKg ÷ birdCount`; nilai dari klien tidak dideklarasikan di DTO sehingga `forbidNonWhitelisted` menolaknya 400 (R4).
- Realisasi boleh melebihi pesanan dan tidak diblok; tidak ada kanal peringatan baru di API (R5).
- `installmentPerKg` dan `adminFeePerKg` disimpan tapi tidak diposting ke mana pun (R6).
- Realisasi tidak pernah mengubah status DO — ia memeriksa status, tidak menggerakkannya.
- Realisasi hanya boleh dari kandang yang ada di baris pesanan DO tersebut.
- Status DO yang boleh direalisasi: `APPROVED`, `REALIZATION_APPROVAL`, `REALIZING`, `REALIZING_DO_LIMIT`.
- Semua query difilter `tenantId`; header pakai `deletedAt`, baris tidak punya.
- Respons dibungkus `{ data, statusCode, timestamp }` oleh `TransformInterceptor` — skrip verifikasi membaca `r["data"]`.
- `ValidationPipe` memakai `whitelist: true, forbidNonWhitelisted: true, transform: true`.
- `RolesGuard` mencocokkan peran **persis**, tanpa hierarki: tulis `@Roles(SystemRole.SUPER_ADMIN, SystemRole.TENANT_ADMIN, SystemRole.MANAGER)`.
- `JwtPayload` diimpor dari `common/interfaces/request-with-user.interface.js` dengan `import type`; id pengguna di `user.sub`.
- Pelanggaran `@@unique` dipetakan ke `ConflictException` berpesan manusiawi, tidak pernah bocor sebagai P2002.
- Kunci i18n ke `messages/en.json` **dan** `id.json`.
- Jalankan `cd breeding-app && npm run build` sebelum commit backend. **Peringatan:** build menghapus `dist/` dan mematikan `npm run start:dev`; hentikan lalu hidupkan ulang preview `api` sesudahnya — sekadar `reused` tidak cukup.
- `npx prisma migrate dev` **tidak** meregenerasi client di setup ini; jalankan `npx prisma generate` sendiri.
- Tidak ada jest yang jalan. Verifikasi memakai skrip Python+curl yang dijalankan sampai **gagal dulu**, diawali `rtk proxy`.
- Skrip verifikasi harus membersihkan pesanan dan realisasi yang dibuatnya, dan memberi tanggal gerakan di ujung tahun (`2026-12-xx`) supaya barisnya muncul di halaman pertama riwayat yang diurutkan `movementDate desc`.

## Review Focus

Lima kelas masukan yang spec implikasikan tetapi langkah fitur tidak melatihnya.

1. **Dua penambahan baris bersamaan pada realisasi yang sama** — stok tidak boleh terpotong dua kali untuk satu baris, dan alokasi tidak boleh dilepas dua kali. Diuji di Task 3.
2. **Kandang yang tidak ada di baris pesanan** — ditolak 400, dan tidak menyisakan gerakan. Diuji di Task 3.
3. **Realisasi melebihi pesanan** — diterima, tapi alokasi yang dilepas berhenti di nol; `quantityAllocated` tidak boleh jadi negatif. Diuji di Task 3.
4. **Mengubah baris ke jumlah yang tidak muat** — keadaan lama harus utuh kembali, bukan hilang separuh. Diuji di Task 4.
5. **DTPS kembar lintas realisasi di tenant yang sama** — 409 dari kunci unik basis data, bukan dari pemeriksaan service yang bisa dilewati balapan. Diuji di Task 3.

---

## Struktur berkas

**Backend (`breeding-app`)** — modul `sales` memakai berkas datar.

| Berkas | Tanggung jawab |
|---|---|
| `prisma/schema.prisma` | enum `WeighingLocation`, 2 model, 2 relasi balik |
| `prisma/migrations/<ts>_add_sales_realization/migration.sql` | migrasi aditif |
| `src/modules/sales/sales-realization.service.ts` | header + baris, pintu tunggal ke populasi |
| `src/modules/sales/sales-realization.controller.ts` | 7 rute |
| `src/modules/sales/dto/create-sales-realization.dto.ts` | header |
| `src/modules/sales/dto/update-sales-realization.dto.ts` | header, PartialType |
| `src/modules/sales/dto/create-realization-line.dto.ts` | baris |
| `src/modules/sales/dto/update-realization-line.dto.ts` | baris, PartialType |
| `src/modules/sales/sales.module.ts` | pendaftaran |

**Frontend (`breeding-dashboard`)**

| Berkas | Tanggung jawab |
|---|---|
| `src/types/api.ts` | tipe realisasi |
| `messages/en.json`, `messages/id.json` | namespace `salesRealization` |
| `src/app/(dashboard)/sales-orders/[id]/realization/page.tsx` | halaman realisasi |
| `src/app/(dashboard)/sales-orders/[id]/page.tsx` | tombol Realisasi + ringkasan |

---

### Task 1: Schema dan migrasi

**Files:**
- Modify: `breeding-app/prisma/schema.prisma`
- Create: `breeding-app/prisma/migrations/<timestamp>_add_sales_realization/migration.sql`

**Interfaces:**
- Consumes: model `SalesOrder`, `ProjectCoop` yang sudah ada.
- Produces: enum `WeighingLocation`; delegate `salesRealization`, `salesRealizationLine`; relasi `SalesOrder.realization`, `ProjectCoop.salesRealizationLines`.

- [ ] **Step 1: Tambahkan enum**

Tepat di bawah `enum RecipientType` yang sudah ada:

```prisma
enum WeighingLocation {
  ORIGIN_COOP
  DESTINATION_CUSTOMER
}
```

- [ ] **Step 2: Tambahkan kedua model**

Letakkan setelah `model SalesOrderLine`:

```prisma
model SalesRealization {
  id               String           @id @default(uuid())
  tenantId         String           @map("tenant_id")
  salesOrderId     String           @unique @map("sales_order_id")
  realizationDate  DateTime         @map("realization_date") @db.Date
  weighingLocation WeighingLocation @map("weighing_location")
  vehiclePlate     String?          @map("vehicle_plate")
  notes            String?          @db.Text
  createdBy        String?          @map("created_by")
  createdAt        DateTime         @default(now()) @map("created_at")
  updatedAt        DateTime         @updatedAt @map("updated_at")
  deletedAt        DateTime?        @map("deleted_at")

  salesOrder SalesOrder             @relation(fields: [salesOrderId], references: [id])
  lines      SalesRealizationLine[]

  @@index([tenantId])
  @@map("sales_realizations")
}

model SalesRealizationLine {
  id                  String   @id @default(uuid())
  tenantId            String   @map("tenant_id")
  realizationId       String   @map("realization_id")
  sourceProjectCoopId String   @map("source_project_coop_id")
  pickupDate          DateTime @map("pickup_date") @db.Date
  dtpsNumber          String   @map("dtps_number")
  birdCount           Int      @map("bird_count")
  totalWeightKg       Decimal  @map("total_weight_kg") @db.Decimal(18, 4)
  avgWeightKg         Decimal  @map("avg_weight_kg") @db.Decimal(18, 4)
  isCulled            Boolean  @default(false) @map("is_culled")
  unitPrice           Decimal? @map("unit_price") @db.Decimal(18, 4)
  discountPerKg       Decimal? @map("discount_per_kg") @db.Decimal(18, 2)
  adminFeePerKg       Decimal? @map("admin_fee_per_kg") @db.Decimal(18, 2)
  savingsPerKg        Decimal? @map("savings_per_kg") @db.Decimal(18, 2)
  installmentPerKg    Decimal? @map("installment_per_kg") @db.Decimal(18, 2)
  totalPrice          Decimal? @map("total_price") @db.Decimal(18, 2)
  allocationReleased  Int      @default(0) @map("allocation_released")
  lineNotes           String?  @map("line_notes") @db.Text
  createdAt           DateTime @default(now()) @map("created_at")
  updatedAt           DateTime @updatedAt @map("updated_at")

  realization       SalesRealization @relation(fields: [realizationId], references: [id], onDelete: Cascade)
  sourceProjectCoop ProjectCoop      @relation(fields: [sourceProjectCoopId], references: [id])

  @@unique([tenantId, dtpsNumber])
  @@index([realizationId])
  @@index([sourceProjectCoopId])
  @@map("sales_realization_lines")
}
```

- [ ] **Step 3: Tambahkan relasi balik**

Di `model SalesOrder`, setelah `salesInvoices SalesInvoice[]`:

```prisma
  realization SalesRealization?
```

Di `model ProjectCoop`, setelah `salesOrderLines SalesOrderLine[]`:

```prisma
  salesRealizationLines SalesRealizationLine[]
```

- [ ] **Step 4: Buat migrasi dan regenerasi client**

Run: `cd breeding-app && npx prisma migrate dev --name add_sales_realization && npx prisma generate`
Expected: migrasi dibuat dan diterapkan, lalu `Generated Prisma Client`.

- [ ] **Step 5: Periksa migrasi aditif**

Run: `cd breeding-app && grep -cE '^\s*(UPDATE|DELETE|DROP|TRUNCATE|ALTER TABLE [^ ]+ (DROP|ALTER) COLUMN)' prisma/migrations/*_add_sales_realization/migration.sql`
Expected: `0`. Jangan pakai pola longgar `grep 'UPDATE'` — itu ikut mencocokkan `ON UPDATE CASCADE`.

- [ ] **Step 6: Build**

Run: `cd breeding-app && npm run build`
Expected: keluar dengan status 0.

- [ ] **Step 7: Commit**

```bash
cd breeding-app
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(sales): add sales realization schema"
```

---

### Task 2: Header realisasi

**Files:**
- Create: `breeding-app/src/modules/sales/sales-realization.service.ts`
- Create: `breeding-app/src/modules/sales/sales-realization.controller.ts`
- Create: `breeding-app/src/modules/sales/dto/create-sales-realization.dto.ts`
- Create: `breeding-app/src/modules/sales/dto/update-sales-realization.dto.ts`
- Modify: `breeding-app/src/modules/sales/sales.module.ts`
- Test: `scratchpad/verify-realization-header.py`

**Interfaces:**
- Consumes: delegate Task 1; `CoopPopulationService` sudah tersedia lewat `ProjectModule` yang diimpor `SalesModule` sejak S-B.
- Produces: `SalesRealizationService.findByOrder(tenantId, salesOrderId)`, `.create(tenantId, salesOrderId, dto, userId)`, `.update(tenantId, id, dto)`, `.remove(tenantId, id)`, helper privat `resolveOrderForRealization(tx, tenantId, salesOrderId)` dan konstanta `REALIZABLE_STATUSES`. Rute `GET/POST /sales-orders/:id/realization`, `PATCH/DELETE /sales-realizations/:id`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-realization-header.py`:

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

BR = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
PC = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]["id"]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "RH Cust " + RUN, "code": "RHC-" + RUN,
                               "operatingBranchIds": [BR["id"]], "vehiclePlate": "B 1 " + RUN})])["data"]
curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 5000, "direction": "IN", "reason": "rh " + RUN, "movementDate": "2026-12-01"})])

made = []
def make_order(qty=100):
    r = curl(["-X", "POST", API + "/sales-orders"] + H +
             ["-d", json.dumps({"branchId": BR["id"], "customerId": cust["id"], "recipientType": "CUSTOMER",
                                "orderDate": "2026-09-27", "validUntil": "2026-12-31",
                                "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                           "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}]})])["data"]
    made.append(r["id"])
    return r

def approve(oid):
    return curl(["-X", "PATCH", API + "/sales-orders/" + oid + "/status"] + H +
                ["-d", json.dumps({"targetStatus": "APPROVED"})])

def realize(oid, **over):
    body = {"realizationDate": "2026-12-05", "weighingLocation": "ORIGIN_COOP"}
    body.update(over)
    return curl(["-X", "POST", API + "/sales-orders/" + oid + "/realization"] + H + ["-d", json.dumps(body)])

pending = make_order()
check("1 a pending order cannot be realized yet", realize(pending["id"]).get("statusCode"), 409)

approve(pending["id"])
r = realize(pending["id"])
check("2 an approved order can be realized", r["data"]["weighingLocation"], "ORIGIN_COOP")
check("3 the plate is copied from the customer", r["data"]["vehiclePlate"], cust["vehiclePlate"])
check("4 it starts with no lines", len(r["data"]["lines"]), 0)
rid = r["data"]["id"]

check("5 a second realization for the same order is refused", realize(pending["id"]).get("statusCode"), 409)

got = curl([API + "/sales-orders/" + pending["id"] + "/realization"] + H)["data"]
check("6 it can be read back from the order", got["id"], rid)

upd = curl(["-X", "PATCH", API + "/sales-realizations/" + rid] + H +
           ["-d", json.dumps({"weighingLocation": "DESTINATION_CUSTOMER", "notes": "catatan " + RUN})])
check("7 the header can be edited", upd["data"]["weighingLocation"], "DESTINATION_CUSTOMER")
check("8 and keeps the note", upd["data"]["notes"], "catatan " + RUN)

check("9 an unknown weighing location is refused",
      realize(make_order()["id"], weighingLocation="SOMEWHERE").get("statusCode"), 400)

rejected = make_order()
approve(rejected["id"])
curl(["-X", "PATCH", API + "/sales-orders/" + rejected["id"] + "/status"] + H +
     ["-d", json.dumps({"targetStatus": "REJECTED"})])
check("10 a rejected order cannot be realized", realize(rejected["id"]).get("statusCode"), 409)

check("11 an order from another tenant is refused",
      realize("00000000-0000-4000-8000-000000000000").get("statusCode"), 400)

curl(["-X", "DELETE", API + "/sales-realizations/" + rid] + H)
check("12 a deleted realization is gone from the order",
      curl([API + "/sales-orders/" + pending["id"] + "/realization"] + H).get("statusCode"), 404)

for oid in made:
    curl(["-X", "DELETE", API + "/sales-orders/" + oid] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-realization-header.py`
Expected: FAIL pada pemeriksaan 1 — rute belum ada, sehingga `statusCode` yang kembali 404, bukan 409.

- [ ] **Step 3: Tulis DTO header**

Create `breeding-app/src/modules/sales/dto/create-sales-realization.dto.ts`:

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsDateString, IsEnum, IsOptional, IsString } from 'class-validator';
import { WeighingLocation } from '../../../../generated/prisma/client.js';

export class CreateSalesRealizationDto {
  @ApiProperty({ example: '2026-12-05' })
  @IsDateString()
  realizationDate: string;

  @ApiProperty({ enum: WeighingLocation, description: 'Timbang Kandang atau Timbang Kirim (A4)' })
  @IsEnum(WeighingLocation)
  weighingLocation: WeighingLocation;

  @ApiPropertyOptional({ description: 'Default disalin dari plat di master customer' })
  @IsOptional()
  @IsString()
  vehiclePlate?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  notes?: string;
}
```

Create `breeding-app/src/modules/sales/dto/update-sales-realization.dto.ts`:

```ts
import { PartialType } from '@nestjs/swagger';
import { CreateSalesRealizationDto } from './create-sales-realization.dto.js';

export class UpdateSalesRealizationDto extends PartialType(CreateSalesRealizationDto) {}
```

- [ ] **Step 4: Tulis service**

Create `breeding-app/src/modules/sales/sales-realization.service.ts`:

```ts
import { Injectable, NotFoundException, BadRequestException, ConflictException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';
import { Prisma, SalesStatus } from '../../../generated/prisma/client.js';
import { CoopPopulationService } from '../project/coop-population.service.js';
import { CreateSalesRealizationDto } from './dto/create-sales-realization.dto.js';
import { UpdateSalesRealizationDto } from './dto/update-sales-realization.dto.js';

@Injectable()
export class SalesRealizationService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly population: CoopPopulationService,
  ) {}

  /** Realisasi hanya masuk akal setelah pesanan disetujui dan belum berakhir. */
  private static readonly REALIZABLE_STATUSES: SalesStatus[] = [
    SalesStatus.APPROVED,
    SalesStatus.REALIZATION_APPROVAL,
    SalesStatus.REALIZING,
    SalesStatus.REALIZING_DO_LIMIT,
  ];

  private readonly include = {
    lines: {
      include: {
        sourceProjectCoop: {
          select: { id: true, coop: { select: { code: true, name: true } } },
        },
      },
      orderBy: { pickupDate: 'asc' as const },
    },
  } as const;

  private async resolveOrderForRealization(
    tx: Prisma.TransactionClient,
    tenantId: string,
    salesOrderId: string,
  ) {
    const order = await tx.salesOrder.findFirst({
      where: { id: salesOrderId, tenantId, deletedAt: null },
      include: { lines: true, customer: { select: { vehiclePlate: true } } },
    });
    if (!order) throw new BadRequestException('Pesanan tidak ditemukan untuk tenant ini');
    if (!SalesRealizationService.REALIZABLE_STATUSES.includes(order.status)) {
      throw new ConflictException('Pesanan ini belum siap direalisasi');
    }
    return order;
  }

  private rethrowDuplicate(error: unknown): never {
    if (typeof error === 'object' && error !== null && (error as { code?: string }).code === 'P2002') {
      const target = String((error as { meta?: { target?: unknown } }).meta?.target ?? '');
      if (target.includes('dtps')) throw new ConflictException('Nomor DTPS ini sudah dipakai');
      throw new ConflictException('Pesanan ini sudah punya realisasi');
    }
    throw error;
  }

  async create(tenantId: string, salesOrderId: string, dto: CreateSalesRealizationDto, userId?: string) {
    return this.prisma
      .$transaction(async (tx) => {
        const order = await this.resolveOrderForRealization(tx, tenantId, salesOrderId);
        const created = await tx.salesRealization.create({
          data: {
            tenantId,
            salesOrderId,
            realizationDate: new Date(dto.realizationDate),
            weighingLocation: dto.weighingLocation,
            // B6: plat diambil dari master saat realisasi dan disimpan sebagai
            // salinan — plat yang berubah nanti tidak mengubah dokumen ini.
            vehiclePlate: dto.vehiclePlate ?? order.customer?.vehiclePlate ?? null,
            notes: dto.notes,
            createdBy: userId,
          },
          include: this.include,
        });
        return created;
      })
      .catch((error) => this.rethrowDuplicate(error));
  }

  async findByOrder(tenantId: string, salesOrderId: string) {
    const found = await this.prisma.salesRealization.findFirst({
      where: { salesOrderId, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!found) throw new NotFoundException('Pesanan ini belum punya realisasi');
    return found;
  }

  async findOne(tenantId: string, id: string) {
    const found = await this.prisma.salesRealization.findFirst({
      where: { id, tenantId, deletedAt: null },
      include: this.include,
    });
    if (!found) throw new NotFoundException('Realisasi tidak ditemukan');
    return found;
  }

  async update(tenantId: string, id: string, dto: UpdateSalesRealizationDto) {
    await this.findOne(tenantId, id);
    return this.prisma.salesRealization.update({
      where: { id },
      data: {
        ...(dto.realizationDate !== undefined && { realizationDate: new Date(dto.realizationDate) }),
        ...(dto.weighingLocation !== undefined && { weighingLocation: dto.weighingLocation }),
        ...(dto.vehiclePlate !== undefined && { vehiclePlate: dto.vehiclePlate }),
        ...(dto.notes !== undefined && { notes: dto.notes }),
      },
      include: this.include,
    });
  }
}
```

`remove` ditulis di Task 5 — ia harus membalik setiap barisnya, dan barisnya belum ada sampai Task 3.

- [ ] **Step 5: Tulis controller**

Create `breeding-app/src/modules/sales/sales-realization.controller.ts`:

```ts
import { Controller, Get, Post, Patch, Body, Param, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { SalesRealizationService } from './sales-realization.service.js';
import { CreateSalesRealizationDto } from './dto/create-sales-realization.dto.js';
import { UpdateSalesRealizationDto } from './dto/update-sales-realization.dto.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { RolesGuard } from '../../common/guards/roles.guard.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { CurrentUser } from '../../common/decorators/current-user.decorator.js';
import type { JwtPayload } from '../../common/interfaces/request-with-user.interface.js';

@ApiTags('Sales Realizations')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard, RolesGuard)
@Controller()
export class SalesRealizationController {
  constructor(private readonly service: SalesRealizationService) {}

  @Get('sales-orders/:salesOrderId/realization')
  @ApiOperation({ summary: 'Realisasi milik sebuah pesanan' })
  findByOrder(@CurrentTenant() tenantId: string, @Param('salesOrderId') salesOrderId: string) {
    return this.service.findByOrder(tenantId, salesOrderId);
  }

  @Post('sales-orders/:salesOrderId/realization')
  @ApiOperation({ summary: 'Buka realisasi untuk sebuah pesanan' })
  create(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('salesOrderId') salesOrderId: string,
    @Body() dto: CreateSalesRealizationDto,
  ) {
    return this.service.create(tenantId, salesOrderId, dto, user.sub);
  }

  @Patch('sales-realizations/:id')
  @ApiOperation({ summary: 'Ubah header realisasi' })
  update(
    @CurrentTenant() tenantId: string,
    @Param('id') id: string,
    @Body() dto: UpdateSalesRealizationDto,
  ) {
    return this.service.update(tenantId, id, dto);
  }
}
```

Rute `DELETE /sales-realizations/:id` ditambahkan di Task 5.

- [ ] **Step 6: Daftarkan di module**

Di `sales.module.ts`, tambahkan `SalesRealizationController` ke `controllers` dan `SalesRealizationService` ke `providers` serta `exports`.

- [ ] **Step 7: Build, hidupkan ulang preview, jalankan skrip**

Run: `cd breeding-app && npm run build`, lalu **stop** dan **start** preview `api`, lalu `python3 scratchpad/verify-realization-header.py`
Expected: pemeriksaan 1-11 lulus; pemeriksaan 12 masih gagal karena `DELETE` belum ada — itu benar dan ditutup di Task 5.

- [ ] **Step 8: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): realization header, one per sales order"
```

---

### Task 3: Menambah baris realisasi

**Files:**
- Create: `breeding-app/src/modules/sales/dto/create-realization-line.dto.ts`
- Modify: `breeding-app/src/modules/sales/sales-realization.service.ts`
- Modify: `breeding-app/src/modules/sales/sales-realization.controller.ts`
- Test: `scratchpad/verify-realization-line.py`

**Interfaces:**
- Consumes: `resolveOrderForRealization`, `REALIZABLE_STATUSES`, `include`, `rethrowDuplicate` dari Task 2; `CoopPopulationService.applyMovement(tx, params)`, `.release(tx, tenantId, projectCoopId, quantity)`, `.resolveProjectCoop(tx, projectCoopId, tenantId)`.
- Produces: `SalesRealizationService.addLine(tenantId, realizationId, dto, userId)`, helper privat `heldByOrderOnCoop(tx, order, realizationId, projectCoopId, excludeLineId?)` dan `applyLineEffects(tx, tenantId, order, realizationId, input)`. Rute `POST /sales-realizations/:id/lines`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-realization-line.py`:

```python
import json, subprocess, time
from concurrent.futures import ThreadPoolExecutor
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

BR = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
rows = curl([API + "/coop-populations?limit=10"] + H)["data"]["data"]
PC = rows[0]["id"]
OTHER_PC = rows[1]["id"]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "RL Cust " + RUN, "code": "RLC-" + RUN,
                               "operatingBranchIds": [BR["id"]]})])["data"]

def pop(pid=PC):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]

curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 5000, "direction": "IN", "reason": "rl " + RUN, "movementDate": "2026-12-01"})])

made = []
def order_with_realization(qty=100):
    o = curl(["-X", "POST", API + "/sales-orders"] + H +
             ["-d", json.dumps({"branchId": BR["id"], "customerId": cust["id"], "recipientType": "CUSTOMER",
                                "orderDate": "2026-09-27", "validUntil": "2026-12-31",
                                "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                           "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}]})])["data"]
    made.append(o["id"])
    curl(["-X", "PATCH", API + "/sales-orders/" + o["id"] + "/status"] + H +
         ["-d", json.dumps({"targetStatus": "APPROVED"})])
    r = curl(["-X", "POST", API + "/sales-orders/" + o["id"] + "/realization"] + H +
             ["-d", json.dumps({"realizationDate": "2026-12-05", "weighingLocation": "ORIGIN_COOP"})])["data"]
    return o, r

def add_line(rid, **over):
    body = {"sourceProjectCoopId": PC, "pickupDate": "2026-12-06",
            "dtpsNumber": "DTPS-" + RUN + "-" + str(len(made)) + "-" + str(int(time.time() * 1000))[-5:],
            "birdCount": 40, "totalWeightKg": 88.0, "unitPrice": 25000}
    body.update(over)
    return curl(["-X", "POST", API + "/sales-realizations/" + rid + "/lines"] + H + ["-d", json.dumps(body)])

o, r = order_with_realization(100)
onhand0, alloc0 = pop()["quantityOnHand"], pop()["quantityAllocated"]

line = add_line(r["id"])
check("1 the line is stored", line["data"]["birdCount"], 40)
check("2 avg weight is derived, not taken from the client", float(line["data"]["avgWeightKg"]), 2.2)
check("3 stock drops by the bird count", pop()["quantityOnHand"], onhand0 - 40)
check("4 the order's hold is released by the same amount", pop()["quantityAllocated"], alloc0 - 40)
check("5 the release is recorded on the line", line["data"]["allocationReleased"], 40)

check("6 a client-sent avgWeightKg is refused", add_line(r["id"], avgWeightKg=9).get("statusCode"), 400)

# Review Focus 2: kandang yang tidak dijanjikan pesanan
before_mv = curl([API + "/coop-populations/" + OTHER_PC + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("7 a coop the order never promised is refused",
      add_line(r["id"], sourceProjectCoopId=OTHER_PC).get("statusCode"), 400)
check("8 and it left no movement behind",
      curl([API + "/coop-populations/" + OTHER_PC + "/movements?limit=1"] + H)["data"]["meta"]["total"], before_mv)

# Review Focus 5: DTPS kembar lintas realisasi
o2, r2 = order_with_realization(50)
dup = "DTPS-DUP-" + RUN
add_line(r2["id"], dtpsNumber=dup, birdCount=10, totalWeightKg=22.0)
check("9 a DTPS already used in this tenant is refused",
      add_line(r["id"], dtpsNumber=dup, birdCount=10, totalWeightKg=22.0).get("statusCode"), 409)

# Review Focus 3: realisasi melebihi pesanan
o3, r3 = order_with_realization(20)
a0 = pop()["quantityAllocated"]
over = add_line(r3["id"], birdCount=60, totalWeightKg=132.0)
check("10 realizing more than ordered is accepted", over["data"]["birdCount"], 60)
check("11 the release stops at what the order held", over["data"]["allocationReleased"], 20)
check("12 and the coop's hold never goes negative", pop()["quantityAllocated"] >= 0, True)
check("13 the hold dropped by exactly the order's share", pop()["quantityAllocated"], a0 - 20)

before_over = curl([API + "/coop-populations/" + PC + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("14 a bird count beyond the coop's stock is refused",
      add_line(r3["id"], birdCount=999999, totalWeightKg=10.0).get("statusCode"), 400)
check("14b and the refused line left no movement",
      curl([API + "/coop-populations/" + PC + "/movements?limit=1"] + H)["data"]["meta"]["total"], before_over)
check("14c a coop from another tenant is refused",
      add_line(r3["id"], sourceProjectCoopId="00000000-0000-4000-8000-000000000000").get("statusCode"), 400)

# realisasi memeriksa status DO, tidak menggerakkannya
status_before = curl([API + "/sales-orders/" + o3["id"]] + H)["data"]["status"]
add_line(r3["id"], birdCount=1, totalWeightKg=2.2)
check("14d recording a pickup does not move the order status",
      curl([API + "/sales-orders/" + o3["id"]] + H)["data"]["status"], status_before)
check("15 zero birds are refused", add_line(r3["id"], birdCount=0, totalWeightKg=1.0).get("statusCode"), 400)
check("16 a missing DTPS is refused",
      curl(["-X", "POST", API + "/sales-realizations/" + r3["id"] + "/lines"] + H +
           ["-d", json.dumps({"sourceProjectCoopId": PC, "pickupDate": "2026-12-06",
                              "birdCount": 5, "totalWeightKg": 11.0})]).get("statusCode"), 400)

# Review Focus 1: dua penambahan bersamaan
o4, r4 = order_with_realization(200)
onhand_before = pop()["quantityOnHand"]
with ThreadPoolExecutor(max_workers=2) as ex:
    f1 = ex.submit(add_line, r4["id"], birdCount=30, totalWeightKg=66.0)
    f2 = ex.submit(add_line, r4["id"], birdCount=30, totalWeightKg=66.0)
    a, b = f1.result(), f2.result()
won = sum(1 for x in (a, b) if x.get("data"))
check("17 concurrent adds each deduct once", pop()["quantityOnHand"], onhand_before - 30 * won)

for oid in made:
    curl(["-X", "DELETE", API + "/sales-orders/" + oid] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-realization-line.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 1 — rute `lines` belum ada.

- [ ] **Step 3: Tulis DTO baris**

Create `breeding-app/src/modules/sales/dto/create-realization-line.dto.ts`:

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsUUID, IsDateString, IsInt, IsPositive, IsNumber, IsOptional, IsBoolean, IsString, IsNotEmpty, Min } from 'class-validator';
import { Type } from 'class-transformer';

export class CreateRealizationLineDto {
  @ApiProperty({ format: 'uuid', description: 'Kandang asal — wajib ada di baris pesanan' })
  @IsUUID()
  sourceProjectCoopId: string;

  @ApiProperty({ example: '2026-12-06', description: 'Tanggal truk mengambil' })
  @IsDateString()
  pickupDate: string;

  @ApiProperty({ description: 'Nomor slip timbang, unik per tenant (A3)' })
  @IsString()
  @IsNotEmpty()
  dtpsNumber: string;

  @ApiProperty({ example: 40 })
  @IsInt()
  @IsPositive()
  birdCount: number;

  @ApiProperty({ example: 88.0, description: 'Tonase hasil timbang' })
  @IsNumber({ maxDecimalPlaces: 4 })
  @IsPositive()
  @Type(() => Number)
  totalWeightKg: number;

  @ApiPropertyOptional({ default: false })
  @IsOptional()
  @IsBoolean()
  isCulled?: boolean;

  @ApiPropertyOptional({ description: 'Harga jual per kg' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 4 })
  @Min(0)
  @Type(() => Number)
  unitPrice?: number;

  @ApiPropertyOptional({ description: 'Diskon per kg' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Type(() => Number)
  discountPerKg?: number;

  @ApiPropertyOptional({ description: 'Biaya admin per kg — disimpan, belum diposting (Q-SO-8)' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Type(() => Number)
  adminFeePerKg?: number;

  @ApiPropertyOptional({ description: 'Tabungan per kg' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Type(() => Number)
  savingsPerKg?: number;

  @ApiPropertyOptional({ description: 'Cicilan per kg — disimpan, mekanismenya belum dibangun (Q-CU-4)' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Type(() => Number)
  installmentPerKg?: number;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  lineNotes?: string;
}
```

`avgWeightKg`, `totalPrice` dan `allocationReleased` sengaja tidak dideklarasikan: ketiganya diturunkan backend, dan `forbidNonWhitelisted` menolak 400 kalau klien mengirimnya — itulah yang membuktikan pemeriksaan 6.

- [ ] **Step 4: Tambahkan logika baris ke service**

Di `sales-realization.service.ts`:

```ts
  /**
   * Sisa ekor yang masih ditahan pesanan ini di sebuah kandang: yang dijanjikan
   * baris pesanan, dikurangi yang sudah direalisasi di kandang yang sama.
   *
   * `excludeLineId` dipakai saat mengubah baris — baris yang sedang diganti
   * tidak boleh ikut dihitung sebagai "sudah direalisasi".
   */
  private async heldByOrderOnCoop(
    tx: Prisma.TransactionClient,
    order: { lines: { sourceProjectCoopId: string | null; birdCount: number | null }[] },
    realizationId: string,
    projectCoopId: string,
    excludeLineId?: string,
  ) {
    const promised = order.lines
      .filter((l) => l.sourceProjectCoopId === projectCoopId)
      .reduce((sum, l) => sum + (l.birdCount ?? 0), 0);

    const realized = await tx.salesRealizationLine.aggregate({
      where: {
        realizationId,
        sourceProjectCoopId: projectCoopId,
        ...(excludeLineId && { id: { not: excludeLineId } }),
      },
      _sum: { birdCount: true },
    });

    return Math.max(0, promised - (realized._sum.birdCount ?? 0));
  }

  async addLine(tenantId: string, realizationId: string, dto: CreateRealizationLineDto, userId?: string) {
    return this.prisma
      .$transaction(async (tx) => {
        const realization = await tx.salesRealization.findFirst({
          where: { id: realizationId, tenantId, deletedAt: null },
        });
        if (!realization) throw new NotFoundException('Realisasi tidak ditemukan');

        const order = await this.resolveOrderForRealization(tx, tenantId, realization.salesOrderId);
        await this.population.resolveProjectCoop(tx, dto.sourceProjectCoopId, tenantId);

        // Tanpa ini, seseorang bisa memotong stok kandang yang tidak pernah
        // dijanjikan pesanan ini.
        const promised = order.lines.some((l) => l.sourceProjectCoopId === dto.sourceProjectCoopId);
        if (!promised) {
          throw new BadRequestException('Kandang ini tidak ada di baris pesanan');
        }

        const stillHeld = await this.heldByOrderOnCoop(
          tx, order, realizationId, dto.sourceProjectCoopId,
        );
        const toRelease = Math.min(dto.birdCount, stillHeld);

        // Gerakan dulu: ini pemeriksaan fisik yang menentukan, dan kalau
        // ayamnya tidak ada seluruh transaksi batal sebelum alokasi tersentuh.
        await this.population.applyMovement(tx, {
          tenantId,
          projectCoopId: dto.sourceProjectCoopId,
          movementType: 'OUT',
          movementSource: 'SALES_REALIZATION',
          quantity: dto.birdCount,
          movementDate: new Date(dto.pickupDate),
          sourceDocId: realizationId,
          sourceDocType: 'SalesRealization',
          createdBy: userId,
        });

        if (toRelease > 0) {
          await this.population.release(tx, tenantId, dto.sourceProjectCoopId, toRelease);
        }

        await tx.salesRealizationLine.create({
          data: { tenantId, realizationId, ...this.mapLine(dto), allocationReleased: toRelease },
        });

        return tx.salesRealization.findUniqueOrThrow({
          where: { id: realizationId },
          include: this.include,
        });
      })
      .catch((error) => this.rethrowDuplicate(error));
  }

  /**
   * R4: arahnya terbalik dari pesanan. Di DO, AVG x Qty = Tonase. Di realisasi
   * truk ditimbang dan ayam dihitung, jadi AVG = Tonase / Qty.
   */
  private mapLine(dto: CreateRealizationLineDto) {
    const totalWeightKg = new Decimal(dto.totalWeightKg);
    const avgWeightKg = totalWeightKg.div(new Decimal(dto.birdCount));
    return {
      sourceProjectCoopId: dto.sourceProjectCoopId,
      pickupDate: new Date(dto.pickupDate),
      dtpsNumber: dto.dtpsNumber,
      birdCount: dto.birdCount,
      totalWeightKg,
      avgWeightKg,
      isCulled: dto.isCulled ?? false,
      unitPrice: dto.unitPrice != null ? new Decimal(dto.unitPrice) : null,
      discountPerKg: dto.discountPerKg != null ? new Decimal(dto.discountPerKg) : null,
      adminFeePerKg: dto.adminFeePerKg != null ? new Decimal(dto.adminFeePerKg) : null,
      savingsPerKg: dto.savingsPerKg != null ? new Decimal(dto.savingsPerKg) : null,
      installmentPerKg: dto.installmentPerKg != null ? new Decimal(dto.installmentPerKg) : null,
      totalPrice: dto.unitPrice != null ? totalWeightKg.mul(new Decimal(dto.unitPrice)) : null,
      lineNotes: dto.lineNotes ?? null,
    };
  }
```

Tambahkan impor:

```ts
import { Decimal } from '@prisma/client/runtime/client';
import { CreateRealizationLineDto } from './dto/create-realization-line.dto.js';
```

- [ ] **Step 5: Tambahkan rute**

Di `sales-realization.controller.ts`:

```ts
  @Post('sales-realizations/:id/lines')
  @ApiOperation({ summary: 'Catat satu pengambilan' })
  addLine(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
    @Body() dto: CreateRealizationLineDto,
  ) {
    return this.service.addLine(tenantId, id, dto, user.sub);
  }
```

Rutenya mengembalikan header beserta seluruh barisnya, sehingga klien tidak perlu memanggil ulang. Pemeriksaan 1, 2 dan 5 di skrip membaca `data.lines[0]` — **ubah skripnya** kalau kamu memilih mengembalikan barisnya saja; jangan ubah kodenya untuk mencocokkan skrip.

- [ ] **Step 6: Build, hidupkan ulang preview, jalankan skrip**

Run: `cd breeding-app && npm run build`, stop lalu start preview `api`, lalu `python3 scratchpad/verify-realization-line.py`
Expected: `RESULT: all checks pass` — 17 dari 17.

- [ ] **Step 7: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): realization lines deduct stock and settle the hold"
```

---

### Task 4: Mengubah dan menghapus baris

**Files:**
- Create: `breeding-app/src/modules/sales/dto/update-realization-line.dto.ts`
- Modify: `breeding-app/src/modules/sales/sales-realization.service.ts`
- Modify: `breeding-app/src/modules/sales/sales-realization.controller.ts`
- Test: `scratchpad/verify-realization-edit.py`

**Interfaces:**
- Consumes: `addLine`, `heldByOrderOnCoop`, `mapLine` dari Task 3.
- Produces: `SalesRealizationService.updateLine(tenantId, lineId, dto, userId)`, `.removeLine(tenantId, lineId, userId)`, helper privat `reverseLine(tx, tenantId, line, userId)`. Rute `PATCH/DELETE /sales-realization-lines/:id`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-realization-edit.py`:

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

BR = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
PC = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]["id"]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "RE Cust " + RUN, "code": "REC-" + RUN,
                               "operatingBranchIds": [BR["id"]]})])["data"]
curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 5000, "direction": "IN", "reason": "re " + RUN, "movementDate": "2026-12-01"})])

def pop():
    return curl([API + "/coop-populations/" + PC] + H)["data"]["population"]

made = []
o = curl(["-X", "POST", API + "/sales-orders"] + H +
         ["-d", json.dumps({"branchId": BR["id"], "customerId": cust["id"], "recipientType": "CUSTOMER",
                            "orderDate": "2026-09-27", "validUntil": "2026-12-31",
                            "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                       "birdCount": 100, "avgWeightKg": 2.0, "unitPrice": 25000}]})])["data"]
made.append(o["id"])
curl(["-X", "PATCH", API + "/sales-orders/" + o["id"] + "/status"] + H +
     ["-d", json.dumps({"targetStatus": "APPROVED"})])
r = curl(["-X", "POST", API + "/sales-orders/" + o["id"] + "/realization"] + H +
         ["-d", json.dumps({"realizationDate": "2026-12-05", "weighingLocation": "ORIGIN_COOP"})])["data"]

onhand0, alloc0 = pop()["quantityOnHand"], pop()["quantityAllocated"]
added = curl(["-X", "POST", API + "/sales-realizations/" + r["id"] + "/lines"] + H +
             ["-d", json.dumps({"sourceProjectCoopId": PC, "pickupDate": "2026-12-06",
                                "dtpsNumber": "DTPS-E-" + RUN, "birdCount": 40,
                                "totalWeightKg": 88.0, "unitPrice": 25000})])["data"]
LID = added["lines"][0]["id"]
check("1 stock dropped by 40", pop()["quantityOnHand"], onhand0 - 40)
check("2 the hold dropped by 40", pop()["quantityAllocated"], alloc0 - 40)

up = curl(["-X", "PATCH", API + "/sales-realization-lines/" + LID] + H +
          ["-d", json.dumps({"birdCount": 25, "totalWeightKg": 55.0})])
check("3 lowering the line returns the difference", pop()["quantityOnHand"], onhand0 - 25)
check("4 and re-holds the difference", pop()["quantityAllocated"], alloc0 - 25)
check("5 avg weight is recomputed", float(up["data"]["lines"][0]["avgWeightKg"]), 2.2)

curl(["-X", "PATCH", API + "/sales-realization-lines/" + LID] + H +
     ["-d", json.dumps({"birdCount": 60, "totalWeightKg": 132.0})])
check("6 raising the line takes more", pop()["quantityOnHand"], onhand0 - 60)

# Review Focus 4: perubahan yang tidak muat harus meninggalkan keadaan lama utuh
before_on, before_alloc = pop()["quantityOnHand"], pop()["quantityAllocated"]
before_mv = curl([API + "/coop-populations/" + PC + "/movements?limit=1"] + H)["data"]["meta"]["total"]
check("7 an impossible edit is refused",
      curl(["-X", "PATCH", API + "/sales-realization-lines/" + LID] + H +
           ["-d", json.dumps({"birdCount": 999999, "totalWeightKg": 10.0})]).get("statusCode"), 400)
check("8 stock survived the refused edit", pop()["quantityOnHand"], before_on)
check("9 the hold survived it too", pop()["quantityAllocated"], before_alloc)
check("10 and no movement was left behind",
      curl([API + "/coop-populations/" + PC + "/movements?limit=1"] + H)["data"]["meta"]["total"], before_mv)
check("11 the line still reads 60",
      curl([API + "/sales-orders/" + o["id"] + "/realization"] + H)["data"]["lines"][0]["birdCount"], 60)

curl(["-X", "DELETE", API + "/sales-realization-lines/" + LID] + H)
check("12 deleting restores the stock", pop()["quantityOnHand"], onhand0)
check("13 and re-holds exactly what it released", pop()["quantityAllocated"], alloc0)
check("14 the realization has no lines left",
      len(curl([API + "/sales-orders/" + o["id"] + "/realization"] + H)["data"]["lines"]), 0)

for oid in made:
    curl(["-X", "DELETE", API + "/sales-orders/" + oid] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-realization-edit.py`
Expected: FAIL pada pemeriksaan 3 — rute `PATCH /sales-realization-lines/:id` belum ada, jadi saldo tidak bergerak.

- [ ] **Step 3: Tulis DTO**

Create `breeding-app/src/modules/sales/dto/update-realization-line.dto.ts`:

```ts
import { PartialType, OmitType } from '@nestjs/swagger';
import { CreateRealizationLineDto } from './create-realization-line.dto.js';

/**
 * Kandang asal tidak bisa dipindah: memindahkannya berarti memulangkan ekor ke
 * satu kandang dan mengambil dari kandang lain dalam satu langkah yang tidak
 * terbaca di ledger. Hapus barisnya lalu buat yang baru.
 */
export class UpdateRealizationLineDto extends PartialType(
  OmitType(CreateRealizationLineDto, ['sourceProjectCoopId'] as const),
) {}
```

- [ ] **Step 4: Tambahkan balik-lalu-terapkan-ulang ke service**

```ts
  /** Membalik efek satu baris: ayamnya kembali, janjinya ditahan lagi. */
  private async reverseLine(
    tx: Prisma.TransactionClient,
    tenantId: string,
    line: {
      id: string; realizationId: string; sourceProjectCoopId: string;
      birdCount: number; allocationReleased: number; pickupDate: Date;
    },
    userId?: string,
  ) {
    // IN dulu: gerakan ini menaikkan stok DAN ketersediaan, sehingga penahanan
    // berikutnya selalu muat. Urutan sebaliknya bisa gagal menahan dan membuat
    // barisnya tidak bisa dihapus sama sekali.
    await this.population.applyMovement(tx, {
      tenantId,
      projectCoopId: line.sourceProjectCoopId,
      movementType: 'IN',
      movementSource: 'SALES_REALIZATION',
      quantity: line.birdCount,
      movementDate: line.pickupDate,
      sourceDocId: line.realizationId,
      sourceDocType: 'SalesRealization',
      notes: 'Pembalikan baris realisasi',
      createdBy: userId,
    });

    if (line.allocationReleased > 0) {
      await this.population.allocate(tx, tenantId, line.sourceProjectCoopId, line.allocationReleased);
    }
  }

  async removeLine(tenantId: string, lineId: string, userId?: string) {
    return this.prisma.$transaction(async (tx) => {
      const line = await tx.salesRealizationLine.findFirst({ where: { id: lineId, tenantId } });
      if (!line) throw new NotFoundException('Baris realisasi tidak ditemukan');

      const deleted = await tx.salesRealizationLine.deleteMany({ where: { id: lineId, tenantId } });
      if (deleted.count !== 1) throw new NotFoundException('Baris realisasi tidak ditemukan');

      await this.reverseLine(tx, tenantId, line, userId);

      return tx.salesRealization.findUniqueOrThrow({
        where: { id: line.realizationId },
        include: this.include,
      });
    });
  }

  /**
   * Dibalik lalu diterapkan ulang, bukan dihitung selisihnya. Selisih pada ekor,
   * tonase dan harga sekaligus lebih mudah salah, dan ledger populasi memang
   * dirancang menerima gerakan pembalik.
   */
  async updateLine(tenantId: string, lineId: string, dto: UpdateRealizationLineDto, userId?: string) {
    return this.prisma
      .$transaction(async (tx) => {
        const line = await tx.salesRealizationLine.findFirst({ where: { id: lineId, tenantId } });
        if (!line) throw new NotFoundException('Baris realisasi tidak ditemukan');

        const realization = await tx.salesRealization.findFirstOrThrow({
          where: { id: line.realizationId, tenantId, deletedAt: null },
        });
        const order = await this.resolveOrderForRealization(tx, tenantId, realization.salesOrderId);

        await this.reverseLine(tx, tenantId, line, userId);

        const merged = {
          sourceProjectCoopId: line.sourceProjectCoopId,
          pickupDate: (dto.pickupDate ?? line.pickupDate.toISOString().slice(0, 10)) as string,
          dtpsNumber: dto.dtpsNumber ?? line.dtpsNumber,
          birdCount: dto.birdCount ?? line.birdCount,
          totalWeightKg: dto.totalWeightKg ?? Number(line.totalWeightKg),
          isCulled: dto.isCulled ?? line.isCulled,
          unitPrice: dto.unitPrice ?? (line.unitPrice != null ? Number(line.unitPrice) : undefined),
          discountPerKg: dto.discountPerKg ?? (line.discountPerKg != null ? Number(line.discountPerKg) : undefined),
          adminFeePerKg: dto.adminFeePerKg ?? (line.adminFeePerKg != null ? Number(line.adminFeePerKg) : undefined),
          savingsPerKg: dto.savingsPerKg ?? (line.savingsPerKg != null ? Number(line.savingsPerKg) : undefined),
          installmentPerKg: dto.installmentPerKg ?? (line.installmentPerKg != null ? Number(line.installmentPerKg) : undefined),
          lineNotes: dto.lineNotes ?? line.lineNotes ?? undefined,
        };

        const stillHeld = await this.heldByOrderOnCoop(
          tx, order, line.realizationId, line.sourceProjectCoopId, lineId,
        );
        const toRelease = Math.min(merged.birdCount, stillHeld);

        await this.population.applyMovement(tx, {
          tenantId,
          projectCoopId: line.sourceProjectCoopId,
          movementType: 'OUT',
          movementSource: 'SALES_REALIZATION',
          quantity: merged.birdCount,
          movementDate: new Date(merged.pickupDate),
          sourceDocId: line.realizationId,
          sourceDocType: 'SalesRealization',
          createdBy: userId,
        });

        if (toRelease > 0) {
          await this.population.release(tx, tenantId, line.sourceProjectCoopId, toRelease);
        }

        await tx.salesRealizationLine.update({
          where: { id: lineId },
          data: { ...this.mapLine(merged), allocationReleased: toRelease },
        });

        return tx.salesRealization.findUniqueOrThrow({
          where: { id: line.realizationId },
          include: this.include,
        });
      })
      .catch((error) => this.rethrowDuplicate(error));
  }
```

- [ ] **Step 5: Tambahkan rute**

```ts
  @Patch('sales-realization-lines/:id')
  @ApiOperation({ summary: 'Ubah satu pengambilan' })
  updateLine(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
    @Body() dto: UpdateRealizationLineDto,
  ) {
    return this.service.updateLine(tenantId, id, dto, user.sub);
  }

  @Delete('sales-realization-lines/:id')
  @ApiOperation({ summary: 'Hapus satu pengambilan dan balikkan efeknya' })
  removeLine(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
  ) {
    return this.service.removeLine(tenantId, id, user.sub);
  }
```

Tambahkan `Delete` ke impor `@nestjs/common` dan impor `UpdateRealizationLineDto`.

- [ ] **Step 6: Build, hidupkan ulang preview, jalankan kedua skrip**

Run: `cd breeding-app && npm run build`, stop lalu start preview `api`, lalu
`python3 scratchpad/verify-realization-edit.py && python3 scratchpad/verify-realization-line.py`
Expected: 14/14 dan 17/17.

- [ ] **Step 7: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): edit and delete realization lines by reversal"
```

---

### Task 5: Menghapus realisasi

**Files:**
- Modify: `breeding-app/src/modules/sales/sales-realization.service.ts`
- Modify: `breeding-app/src/modules/sales/sales-realization.controller.ts`
- Test: `scratchpad/verify-realization-header.py` (pemeriksaan 12, sudah ditulis di Task 2)

**Interfaces:**
- Consumes: `reverseLine` dari Task 4.
- Produces: `SalesRealizationService.remove(tenantId, id, userId)`; rute `DELETE /sales-realizations/:id`.

- [ ] **Step 1: Jalankan skrip Task 2 dan lihat pemeriksaan 12 gagal**

Run: `python3 scratchpad/verify-realization-header.py`
Expected: pemeriksaan 1-11 lulus, pemeriksaan 12 gagal — `DELETE` belum ada sehingga realisasinya masih terbaca.

- [ ] **Step 2: Tambahkan `remove`**

```ts
  async remove(tenantId: string, id: string, userId?: string) {
    return this.prisma.$transaction(async (tx) => {
      const realization = await tx.salesRealization.findFirst({
        where: { id, tenantId, deletedAt: null },
        include: { lines: true },
      });
      if (!realization) throw new NotFoundException('Realisasi tidak ditemukan');

      const claimed = await tx.salesRealization.updateMany({
        where: { id, tenantId, deletedAt: null },
        data: { deletedAt: new Date() },
      });
      if (claimed.count !== 1) throw new NotFoundException('Realisasi tidak ditemukan');

      // Setiap baris dibalik satu per satu: ayamnya kembali, janjinya ditahan
      // lagi. Barisnya ikut dihapus supaya totalnya tidak menghitung bangkai.
      for (const line of realization.lines) {
        await this.reverseLine(tx, tenantId, line, userId);
      }
      await tx.salesRealizationLine.deleteMany({ where: { realizationId: id } });

      return tx.salesRealization.findUniqueOrThrow({ where: { id } });
    });
  }
```

- [ ] **Step 3: Tambahkan rute**

```ts
  @Delete('sales-realizations/:id')
  @Roles(SystemRole.SUPER_ADMIN, SystemRole.TENANT_ADMIN, SystemRole.MANAGER)
  @ApiOperation({ summary: 'Hapus realisasi dan balikkan seluruh barisnya' })
  remove(
    @CurrentTenant() tenantId: string,
    @CurrentUser() user: JwtPayload,
    @Param('id') id: string,
  ) {
    return this.service.remove(tenantId, id, user.sub);
  }
```

Tambahkan impor `Roles` dan `SystemRole`.

- [ ] **Step 4: Tulis pemeriksaan tambahan bahwa penghapusan membalik stok**

Tambahkan ke akhir `scratchpad/verify-realization-header.py`, sebelum blok pembersihan:

```python
# menghapus realisasi berisi baris harus membalik semuanya
o5 = make_order(80)
approve(o5["id"])
r5 = realize(o5["id"])["data"]
PCPOP = lambda: curl([API + "/coop-populations/" + PC] + H)["data"]["population"]
on0, al0 = PCPOP()["quantityOnHand"], PCPOP()["quantityAllocated"]
curl(["-X", "POST", API + "/sales-realizations/" + r5["id"] + "/lines"] + H +
     ["-d", json.dumps({"sourceProjectCoopId": PC, "pickupDate": "2026-12-06",
                        "dtpsNumber": "DTPS-H-" + RUN, "birdCount": 30,
                        "totalWeightKg": 66.0, "unitPrice": 25000})])
check("13 the line moved the balances", PCPOP()["quantityOnHand"], on0 - 30)
curl(["-X", "DELETE", API + "/sales-realizations/" + r5["id"]] + H)
check("14 deleting the realization restores stock", PCPOP()["quantityOnHand"], on0)
check("15 and restores the hold", PCPOP()["quantityAllocated"], al0)
```

- [ ] **Step 5: Build, hidupkan ulang preview, jalankan skrip**

Run: `cd breeding-app && npm run build`, stop lalu start preview `api`, lalu `python3 scratchpad/verify-realization-header.py`
Expected: `RESULT: all checks pass` — 15 dari 15.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): deleting a realization reverses every line"
```

---

### Task 6: Tipe dan i18n frontend

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts`
- Modify: `breeding-dashboard/messages/en.json`, `breeding-dashboard/messages/id.json`

**Interfaces:**
- Consumes: bentuk respons Task 2-5.
- Produces: tipe `SalesRealization`, `SalesRealizationLine`, `WeighingLocation`; namespace i18n `salesRealization`.

- [ ] **Step 1: Tambahkan tipe**

```ts
export type WeighingLocation = "ORIGIN_COOP" | "DESTINATION_CUSTOMER";

export interface SalesRealizationLine {
  id: string;
  sourceProjectCoopId: string;
  pickupDate: string;
  dtpsNumber: string;
  birdCount: number;
  totalWeightKg: string;
  avgWeightKg: string;
  isCulled: boolean;
  unitPrice: string | null;
  discountPerKg: string | null;
  adminFeePerKg: string | null;
  savingsPerKg: string | null;
  installmentPerKg: string | null;
  totalPrice: string | null;
  allocationReleased: number;
  lineNotes: string | null;
  sourceProjectCoop?: { id: string; coop: { code: string; name: string } } | null;
}

export interface SalesRealization {
  id: string;
  salesOrderId: string;
  realizationDate: string;
  weighingLocation: WeighingLocation;
  vehiclePlate: string | null;
  notes: string | null;
  createdAt: string;
  lines: SalesRealizationLine[];
}
```

- [ ] **Step 2: Tambahkan namespace i18n ke kedua berkas**

`en.json`:

```json
  "salesRealization": {
    "title": "Realization", "description": "Birds actually handed over, per pickup",
    "entity": "Realization", "open": "Realization", "start": "Open realization",
    "realizationDate": "Realization date", "weighingLocation": "Weighing location",
    "originCoop": "Weigh at coop", "destinationCustomer": "Weigh on delivery",
    "vehiclePlate": "Vehicle plate", "notes": "Notes",
    "lines": "Pickups", "addLine": "Add pickup", "pickupDate": "Pickup date",
    "sourceCoop": "Source coop", "dtpsNumber": "DTPS number", "birdCount": "Birds",
    "totalWeight": "Tonnage (kg)", "avgWeight": "Avg weight (kg)", "culled": "Culled",
    "unitPrice": "Sell price / kg", "discountPerKg": "Discount / kg",
    "adminFeePerKg": "Admin fee / kg", "savingsPerKg": "Savings / kg",
    "installmentPerKg": "Instalment / kg",
    "installmentDisabled": "Instalments are not built yet",
    "totalPrice": "Total", "lineNotes": "Notes",
    "totalRealized": "Total realized", "totalDiscount": "Total discount",
    "totalAdminFee": "Total admin fee", "totalSavings": "Total savings",
    "exceedsOrder": "Realized {birds} birds against {ordered} ordered",
    "lineRequired": "Fill in the coop, DTPS number, birds and tonnage",
    "notStarted": "This order has no realization yet"
  }
```

`id.json` memakai kunci yang sama persis dengan nilai bahasa Indonesia — misalnya `"originCoop": "Timbang Kandang"`, `"destinationCustomer": "Timbang Kirim"`, `"dtpsNumber": "No. DTPS"`, `"totalWeight": "Tonase (kg)"`, `"installmentDisabled": "Mekanisme cicilan belum dibangun"`, `"exceedsOrder": "Realisasi {birds} ekor dari {ordered} ekor yang dipesan"`.

- [ ] **Step 3: Periksa tipe, build, dan paritas kunci**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

Run: `cd breeding-dashboard && node -e "['en','id'].forEach(l=>{const m=require('./messages/'+l+'.json');console.log(l,Object.keys(m.salesRealization).length)})"`
Expected: dua baris dengan angka **yang sama**.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add src/types/api.ts messages
git commit -m "feat(sales): realization types and i18n keys"
```

---

### Task 7: Halaman realisasi

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/sales-orders/[id]/realization/page.tsx`

**Interfaces:**
- Consumes: tipe dan i18n Task 6; rute Task 2-5; `<ProjectCoopCombobox />` yang sudah ada.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tulis halaman**

Struktur seperti halaman detail pesanan: `PageHeader`, kartu, `Table`. Memuat pesanan dan realisasinya bersamaan:

```tsx
const { data: order } = useApi<SalesOrder>(`/sales-orders/${id}`);
const [realization, setRealization] = useState<SalesRealization | null>(null);
const [notStarted, setNotStarted] = useState(false);

const load = useCallback(() => {
  fetchApi<SalesRealization>(`/sales-orders/${id}/realization`)
    .then((r) => { setRealization(r); setNotStarted(false); })
    .catch(() => { setRealization(null); setNotStarted(true); });
}, [id]);
```

Kalau belum ada, halaman hanya menampilkan `t('notStarted')` dan tombol `t('start')` yang membuka dialog berisi tanggal realisasi dan pilihan lokasi timbang, lalu `POST /sales-orders/${id}/realization`.

Kalau sudah ada: header read-only dari pesanan (nomor DO, customer, alamat, penerima) plus bidang yang bisa diubah — tanggal realisasi, radio lokasi timbang, plat, catatan — yang disimpan lewat `PATCH /sales-realizations/${realization.id}`.

Tabel baris dengan kolom Tanggal ambil, Kandang, No. DTPS, Ekor, Tonase, AVG (read-only), Afkir, Harga/kg, Diskon/kg, Biaya admin/kg, Tabungan/kg, **Cicilan/kg yang dimatikan**, Total, Aksi.

Dialog tambah dan ubah baris memakai bidang yang sama. Kandang dibatasi yang ada di pesanan:

```tsx
// Backend menolak kandang yang tidak dijanjikan; batasi pilihannya agar
// penggunanya tidak menemukan aturan itu lewat pesan kesalahan.
const promisedCoops = useMemo(
  () => Array.from(new Set((order?.lines ?? []).map((l) => l.sourceProjectCoopId).filter(Boolean))),
  [order]
);
```

AVG ditampilkan terhitung dan hanya baca:

```tsx
const avg = (birds: string, tonnage: string) =>
  birds && tonnage && Number(birds) > 0 ? (Number(tonnage) / Number(birds)).toFixed(4) : "";
```

Cicilan dimatikan dengan keterangan:

```tsx
<Input id="line-installment" type="number" value={formInstallment} disabled />
<p className="text-xs text-muted-foreground">{t('installmentDisabled')}</p>
```

Payload **tidak menyertakan** `avgWeightKg`, `totalPrice` maupun `allocationReleased` — ketiganya diturunkan backend dan `forbidNonWhitelisted` menolak 400 kalau ikut terkirim.

Validasi klien sebelum mengirim:

```tsx
if (!formCoop || !formDtps.trim() || !(Number(formBirds) > 0) || !(Number(formTonnage) > 0)) {
  toast.error(t('lineRequired'));
  return;
}
```

Galat ditangkap dengan memunculkan pesan asli dari API:

```tsx
} catch (error) {
  toast.error(error instanceof Error ? error.message : tc('entityCreateFailed', { entity: t('entity') }));
}
```

Footer menjumlahkan dari `realization.lines`: total harga, total diskon (`discountPerKg × totalWeightKg`), PPN dari pesanan, total biaya admin, total tabungan. Di bawahnya, peringatan A2 kalau melebihi pesanan:

```tsx
const orderedBirds = (order?.lines ?? []).reduce((s, l) => s + (l.birdCount ?? 0), 0);
const realizedBirds = (realization?.lines ?? []).reduce((s, l) => s + l.birdCount, 0);
{realizedBirds > orderedBirds && (
  <p className="text-sm text-destructive">
    {t('exceedsOrder', { birds: realizedBirds, ordered: orderedBirds })}
  </p>
)}
```

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`, dengan rute `/sales-orders/[id]/realization` terdaftar.

- [ ] **Step 3: Smoke browser**

Siapkan satu DO yang sudah `APPROVED` dengan 100 ekor dari sebuah kandang, lalu buka halaman realisasinya. Buktikan, dan catat hasilnya:
1. Halaman menawarkan "Open realization"; setelah dibuka, plat terisi otomatis dari customer.
2. Pilihan kandang di dialog baris **hanya** berisi kandang yang ada di pesanan.
3. Mengisi 40 ekor dan 88 kg menampilkan AVG 2.2 sebagai read-only.
4. Menyimpan baris membuat halaman populasi kandang menunjukkan **Masuk** berkurang 40 dan **Dipesan** berkurang 40.
5. Bidang Cicilan/kg tampil dimatikan dengan keterangannya.
6. Mengubah baris jadi 25 ekor mengembalikan 15 ke kedua angka itu.
7. Menghapus baris memulihkan keduanya sepenuhnya.
8. Merealisasi 150 ekor pada pesanan 100 ekor **tersimpan** dan memunculkan peringatan di footer, tanpa memblokir.

Hapus data uji sesudahnya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/sales-orders/[id]/realization"
git commit -m "feat(sales): realization page"
```

---

### Task 8: Pintu masuk dari detail pesanan

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/sales-orders/[id]/page.tsx`

**Interfaces:**
- Consumes: tipe Task 6; halaman Task 7.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tambahkan tombol dan ringkasan**

Di blok aksi `PageHeader` (sebelum `<StatusAction …>`):

```tsx
<Button variant="outline" asChild>
  <Link href={`/sales-orders/${id}/realization`}>{tr('open')}</Link>
</Button>
```

dengan `const tr = useTranslations('salesRealization');`.

Di bawah tabel baris pesanan, kartu ringkasan yang memuat realisasinya sekali:

```tsx
const [realized, setRealized] = useState<SalesRealization | null>(null);
useEffect(() => {
  fetchApi<SalesRealization>(`/sales-orders/${id}/realization`)
    .then(setRealized)
    .catch(() => setRealized(null));
}, [id]);
```

Kartu menampilkan tanggal realisasi, lokasi timbang, plat, jumlah pengambilan, total ekor dan total tonase; kalau `realized` null, tampilkan `tr('notStarted')`.

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 3: Smoke browser**

Buka detail sebuah DO. Buktikan, dan catat hasilnya:
1. Tombol "Realization" tampil dan membuka halaman realisasi DO itu.
2. DO tanpa realisasi menampilkan keterangan "belum punya realisasi".
3. DO yang sudah punya realisasi menampilkan tanggal, lokasi timbang, jumlah pengambilan, total ekor dan tonase yang cocok dengan halaman realisasinya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/sales-orders/[id]/page.tsx"
git commit -m "feat(sales): realization entry point on the order detail"
```

---

### Task 9: Perbarui dokumen tahapan

**Files:**
- Modify: `docs/superpowers/specs/2026-09-27-sales-realization-design.md`
- Modify: `docs/superpowers/specs/2026-09-13-sales-module-finding.md`

**Interfaces:**
- Consumes: hasil Task 1-8.
- Produces: tidak ada.

- [ ] **Step 1: Tandai spec sudah diimplementasi**

Ubah baris `**Status:**` di `2026-09-27-sales-realization-design.md` menjadi:

```markdown
**Status:** Implemented (2026-09-27) — lihat plan `docs/superpowers/plans/2026-09-27-sales-realization.md`
```

- [ ] **Step 2: Tandai S-C selesai dan tutup K-S1**

Di `2026-09-13-sales-module-finding.md`, pada baris **S-C** di tabel tahapan, ganti kolom Status menjadi `Selesai (2026-09-27)`.

Di bagian **Konflik K-S1**, tambahkan di akhir:

```markdown
**Terselesaikan (2026-09-27).** Layar acuan hanya satu ("Modify Data Realisasi DO") dengan satu
header dan banyak baris, jadi "2 realisasi" pada contoh itu dua **baris** pengambilan berbeda DTPS.
Jawaban A1 dan data yang diamati sejalan. Dibangun sebagai header 1:1 ke DO dengan banyak baris.
```

Di tabel **Sub-pertanyaan yang belum terjawab**, ganti baris `A3` menjadi:

```markdown
| A3 | "Wajib diisi di setiap realisasi?" — dijawab "sudah ada di dalam form" (ambigu) | **Diputuskan (2026-09-27): wajib dan unik per tenant**, mengikuti preseden D1 |
```

- [ ] **Step 3: Periksa**

Run: `cd /Users/alva.e202511001/Desktop/project/breeding && grep -c "Implemented (2026-09-27)" docs/superpowers/specs/2026-09-27-sales-realization-design.md && grep -c "Terselesaikan (2026-09-27)" docs/superpowers/specs/2026-09-13-sales-module-finding.md`
Expected: `1` lalu `1`.

- [ ] **Step 4: Commit**

```bash
cd /Users/alva.e202511001/Desktop/project/breeding
git add docs/superpowers/specs
git commit -m "docs(sales): mark realization implemented and close K-S1"
```
