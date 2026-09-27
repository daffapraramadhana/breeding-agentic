# Pesanan Penjualan Terstruktur (S-B) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Mengubah baris pesanan penjualan dari teks bebas menjadi baris terstruktur yang menunjuk produk dan kandang asal yang nyata, menahan ekor yang dijanjikannya, dan membawa angka yang dipakai saat timbang.

**Architecture:** Kolom baru ditambahkan aditif ke `SalesOrder` dan `SalesOrderLine`; kolom teks lama dipertahankan agar pesanan lama tetap terbaca. Pembuatan dan pengubahan pesanan memanggil `CoopPopulationService.allocate(tx, …)`/`release(tx, …)` di dalam transaksi yang sama dengan pesanannya, sehingga kegagalan di baris mana pun tidak menyisakan tahanan. Penolakan, pembatalan, dan job kadaluarsa harian melepas kembali.

**Tech Stack:** NestJS 11, Prisma 7 (client di `generated/prisma/`), PostgreSQL 15, `@nestjs/schedule` (dipasang di Task 8), Next.js 16, shadcn/ui, next-intl.

**Spec:** `docs/superpowers/specs/2026-09-27-sales-structured-order-design.md`

## Global Constraints

- Lingkup hanya `RecipientType.CUSTOMER`; jalur `BREEDER` tidak disentuh (K1).
- DO menahan ekor sejak `PENDING_APPROVAL`, bukan sejak `APPROVED` (K2).
- Baris hanya boleh berasal dari kandang; pilihan gudang tidak ditawarkan (K4).
- `totalWeightKg` selalu dihitung backend dari `birdCount × avgWeightKg`; nilai dari klien diabaikan (K6).
- Bobot minimum disimpan sebagai salinan `minimumWeightKg`, bukan kunci asing ke master (K7).
- Semua kolom baru nullable di schema karena data lama tidak punya nilainya; **DTO tetap mewajibkan** `productId`, `sourceProjectCoopId`, `birdCount`, `avgWeightKg` untuk baris pesanan baru.
- S-B tidak pernah menulis gerakan populasi. Realisasi milik S-C.
- Semua query difilter `tenantId`; soft delete lewat `deletedAt`.
- Respons dibungkus `{ data, statusCode, timestamp }` oleh `TransformInterceptor` — skrip verifikasi membaca `r["data"]`.
- `ValidationPipe` memakai `whitelist: true, forbidNonWhitelisted: true, transform: true` — properti yang tidak dideklarasikan menghasilkan 400.
- `RolesGuard` mencocokkan peran **persis**, tanpa hierarki: tulis `@Roles(SystemRole.SUPER_ADMIN, SystemRole.TENANT_ADMIN, SystemRole.MANAGER)`, bukan `@Roles(SystemRole.MANAGER)`.
- `JwtPayload` diimpor dari `common/interfaces/request-with-user.interface.js` dengan `import type`; id pengguna ada di `user.sub`.
- Kunci i18n ditambahkan ke `messages/en.json` **dan** `id.json`, termasuk namespace `navigation`.
- Jalankan `cd breeding-app && npm run build` sebelum commit backend. **Peringatan:** build menghapus `dist/` dan mematikan `npm run start:dev` yang sedang jalan — hidupkan lagi sesudahnya.
- `npx prisma migrate dev` **tidak** meregenerasi client di setup ini; jalankan `npx prisma generate` sendiri setelah mengubah schema.
- Repo tidak punya jest yang jalan (`src/app.controller.spec.ts` gagal parse di base cabang juga). Verifikasi memakai skrip Python+curl yang dijalankan sampai **gagal dulu** sebelum implementasi.
- Perintah curl diawali `rtk proxy`. Skrip Node dijalankan sebagai `.mts` lewat `npx tsx` dari dalam `breeding-app/.verify/` (tidak dilacak git, dihapus di akhir), dan membangun `new PrismaClient({ adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }) })` seperti `PrismaService`.

## Review Focus

Lima kelas masukan yang spec implikasikan tetapi langkah fitur tidak melatihnya. Tiap baris punya tes yang ditempelkan ke task pemiliknya.

1. **Dua DO bersamaan merebut ekor terakhir sebuah kandang** — tepat satu boleh lolos; yang kalah 400 dan tidak menyisakan pesanan. Diuji di Task 4.
2. **Satu DO dengan beberapa baris dari kandang yang sama** — alokasi harus dijumlahkan dulu; tiga baris @400 dari kandang berisi 1000 harus ditolak, bukan lolos karena tiap barisnya sendiri muat. Diuji di Task 4.
3. **Pengubahan DO yang gagal di tengah** — kalau alokasi baru tidak muat, alokasi lama harus utuh kembali, bukan hilang. Diuji di Task 5.
4. **Job kadaluarsa berjalan saat pesanan sudah ditolak orang lain** — tidak boleh melepas alokasi dua kali. Diuji di Task 8.
5. **Klien mengirim `totalWeightKg` yang bertentangan dengan `birdCount × avgWeightKg`** — nilai backend yang menang. Diuji di Task 4.

---

## Struktur berkas

**Backend (`breeding-app`)** — modul `sales` memakai berkas datar; ikuti pola itu.

| Berkas | Tanggung jawab |
|---|---|
| `prisma/schema.prisma` | 4 kolom header, 6 kolom baris, 2 relasi, relasi balik |
| `prisma/migrations/<ts>_add_structured_sales_order/migration.sql` | migrasi aditif |
| `src/common/utils/reference-number.generator.ts` | **diubah** — `generateSequential` |
| `src/modules/sales/dto/create-sales-order.dto.ts` | **diubah** — kolom baru + validasi |
| `src/modules/sales/dto/expire-sales-order.dto.ts` | — tidak ada; job tidak menerima payload |
| `src/modules/sales/sales-order.service.ts` | **diubah** — create/update/transition/expire/recommendedPrice |
| `src/modules/sales/sales-order.controller.ts` | **diubah** — 2 rute baru |
| `src/modules/sales/sales-order-expiry.scheduler.ts` | cron harian lintas tenant |
| `src/modules/sales/sales.module.ts` | **diubah** — `ProjectModule`, `ScheduleModule` |
| `src/common/constants/status-transitions.constant.ts` | **diubah** — `APPROVED → REJECTED` |

**Frontend (`breeding-dashboard`)**

| Berkas | Tanggung jawab |
|---|---|
| `src/types/api.ts` | **diubah** — tipe baris & header |
| `messages/en.json`, `messages/id.json` | kunci i18n |
| `src/components/forms/mature-bird-standard-combobox.tsx` | pemilih standar bobot |
| `src/app/(dashboard)/sales-orders/new/page.tsx` | **ditulis ulang** — form terstruktur |
| `src/app/(dashboard)/sales-orders/[id]/page.tsx` | **diubah** — kolom baru di detail |
| `src/app/(dashboard)/sales-orders/page.tsx` | **diubah** — filter & kolom |

---

### Task 1: Schema dan migrasi

**Files:**
- Modify: `breeding-app/prisma/schema.prisma`
- Create: `breeding-app/prisma/migrations/<timestamp>_add_structured_sales_order/migration.sql` (dibuat `prisma migrate dev`)

**Interfaces:**
- Consumes: model `SalesOrder`, `SalesOrderLine`, `Product`, `ProjectCoop` yang sudah ada.
- Produces: kolom `SalesOrder.recipientName|recipientAddress|validUntil|vatPercent`; kolom `SalesOrderLine.productId|sourceProjectCoopId|isCulled|avgWeightKg|savingsPerKg|minimumWeightKg`; relasi `SalesOrderLine.product`, `SalesOrderLine.sourceProjectCoop`, `ProjectCoop.salesOrderLines`, `Product.salesOrderLines`.

- [ ] **Step 1: Tambahkan kolom header**

Di `model SalesOrder`, tepat setelah baris `paymentMethod String? @map("payment_method")`:

```prisma
  recipientName    String?   @map("recipient_name")
  recipientAddress String?   @map("recipient_address") @db.Text
  validUntil       DateTime? @map("valid_until") @db.Date
  vatPercent       Decimal?  @default(0) @map("vat_percent") @db.Decimal(5, 2)
```

- [ ] **Step 2: Tambahkan kolom baris dan relasinya**

Di `model SalesOrderLine`, setelah baris `totalPrice Decimal? @map("total_price") @db.Decimal(18, 2)`:

```prisma
  productId           String?  @map("product_id")
  sourceProjectCoopId String?  @map("source_project_coop_id")
  isCulled            Boolean  @default(false) @map("is_culled")
  avgWeightKg         Decimal? @map("avg_weight_kg") @db.Decimal(18, 4)
  savingsPerKg        Decimal? @map("savings_per_kg") @db.Decimal(18, 2)
  minimumWeightKg     Decimal? @map("minimum_weight_kg") @db.Decimal(18, 4)
```

dan di blok relasinya, tepat setelah `salesOrder SalesOrder @relation(fields: [salesOrderId], references: [id])`:

```prisma
  product           Product?     @relation(fields: [productId], references: [id])
  sourceProjectCoop ProjectCoop? @relation(fields: [sourceProjectCoopId], references: [id])
```

dan di akhir model, sebelum `@@map("sales_order_lines")`:

```prisma
  @@index([productId])
  @@index([sourceProjectCoopId])
```

- [ ] **Step 3: Tambahkan relasi balik**

Di `model ProjectCoop`, setelah baris `birdTransfersIn`:

```prisma
  salesOrderLines SalesOrderLine[]
```

Di `model Product`, di blok daftar relasi (setelah `goodsConsumptionLines GoodsConsumptionLine[]`):

```prisma
  salesOrderLines SalesOrderLine[]
```

- [ ] **Step 4: Buat migrasi**

Run: `cd breeding-app && npx prisma migrate dev --name add_structured_sales_order`
Expected: migrasi dibuat dan diterapkan.

- [ ] **Step 5: Regenerasi client**

`migrate dev` tidak menjalankannya di setup ini.

Run: `cd breeding-app && npx prisma generate`
Expected: `Generated Prisma Client`.

- [ ] **Step 6: Periksa migrasi aditif**

Run: `cd breeding-app && grep -cE '^\s*(UPDATE|DELETE|DROP|TRUNCATE|ALTER TABLE [^ ]+ (DROP|ALTER) COLUMN)' prisma/migrations/*_add_structured_sales_order/migration.sql`
Expected: `0`. Jangan pakai pola longgar `grep 'UPDATE'` — itu ikut mencocokkan `ON UPDATE CASCADE` di klausa foreign key.

- [ ] **Step 7: Build**

Run: `cd breeding-app && npm run build`
Expected: keluar dengan status 0.

- [ ] **Step 8: Commit**

```bash
cd breeding-app
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(sales): add structured sales order columns"
```

---

### Task 2: Nomor DO berurutan

**Files:**
- Modify: `breeding-app/src/common/utils/reference-number.generator.ts`
- Test: `breeding-app/.verify/do-number.mts`

**Interfaces:**
- Consumes: model `ReferenceCounter` (`@@unique([tenantId, prefix, year, month])`).
- Produces: `ReferenceNumberGenerator.generateSequential(tx, tenantId, prefix): Promise<string>` yang mengembalikan `${prefix}.${seq}` tanpa reset per periode.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `breeding-app/.verify/do-number.mts`:

```ts
import { PrismaClient } from '../generated/prisma/client.js';
import { PrismaPg } from '@prisma/adapter-pg';
import { config } from 'dotenv';
config({ path: new URL('../.env', import.meta.url).pathname });

const prisma: any = new PrismaClient({ adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }) });
const fails: string[] = [];
const check = (label: string, actual: unknown, expected: unknown) => {
  const ok = JSON.stringify(actual) === JSON.stringify(expected);
  if (!ok) fails.push(label);
  console.log(`${ok ? 'PASS' : 'FAIL'} ${label}: got ${JSON.stringify(actual)}, expected ${JSON.stringify(expected)}`);
};

const { ReferenceNumberGenerator } = await import('../src/common/utils/reference-number.generator.js');
const gen: any = new ReferenceNumberGenerator();

const tenant = await prisma.tenant.findFirstOrThrow({ where: { deletedAt: null } });
const RUN = String(Date.now()).slice(-6);
const PREFIX = `DO.TEST${RUN}`;

const a = await gen.generateSequential(prisma, tenant.id, PREFIX);
const b = await gen.generateSequential(prisma, tenant.id, PREFIX);
check('1 first number', a, `${PREFIX}.1`);
check('2 second number increments', b, `${PREFIX}.2`);

const rows = await prisma.referenceCounter.findMany({ where: { tenantId: tenant.id, prefix: PREFIX } });
check('3 exactly one counter row', rows.length, 1);
check('4 counter is period-less', [rows[0].year, rows[0].month], [0, 0]);

// prefix berbeda punya deret sendiri
const other = await gen.generateSequential(prisma, tenant.id, `${PREFIX}X`);
check('5 a different prefix starts its own sequence', other, `${PREFIX}X.1`);

// deret bulanan yang lama tidak terganggu
const monthly = await gen.generate(prisma, tenant.id, `T${RUN}`);
check('6 the monthly generator still works', /^T\d+-\d{6}-\d{4}$/.test(monthly), true);

await prisma.referenceCounter.deleteMany({ where: { tenantId: tenant.id, prefix: { startsWith: PREFIX } } });
await prisma.referenceCounter.deleteMany({ where: { tenantId: tenant.id, prefix: `T${RUN}` } });

console.log('\nRESULT:', fails.length ? `${fails.length} FAILING: ${fails}` : 'all checks pass');
await prisma.$disconnect();
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `cd breeding-app && npx tsx .verify/do-number.mts`
Expected: gagal dengan `TypeError: gen.generateSequential is not a function`.

- [ ] **Step 3: Tambahkan metodenya**

Di `src/common/utils/reference-number.generator.ts`, di dalam kelas, setelah metode `generate`:

```ts
  /**
   * Nomor berurutan yang TIDAK reset per periode — dipakai nomor DO
   * (`DO.<kode cabang>.<urut>`) yang di aplikasi acuan terus bertambah.
   *
   * Memakai tabel penghitung yang sama dengan `year = 0, month = 0` sebagai
   * penanda "tanpa periode"; kunci uniknya `(tenantId, prefix, year, month)`
   * sehingga deret ini tidak pernah bertabrakan dengan deret bulanan.
   */
  async generateSequential(
    tx: any,
    tenantId: string,
    prefix: string,
  ): Promise<string> {
    const counter = await tx.referenceCounter.upsert({
      where: { tenantId_prefix_year_month: { tenantId, prefix, year: 0, month: 0 } },
      update: { lastValue: { increment: 1 } },
      create: { tenantId, prefix, year: 0, month: 0, lastValue: 1 },
    });

    return `${prefix}.${counter.lastValue}`;
  }
```

- [ ] **Step 4: Jalankan dan pastikan lulus**

Run: `cd breeding-app && npx tsx .verify/do-number.mts`
Expected: `RESULT: all checks pass` — 6 dari 6.

- [ ] **Step 5: Commit**

```bash
cd breeding-app
git add src/common/utils/reference-number.generator.ts
git commit -m "feat(sales): add period-less sequential reference numbers"
```

---

### Task 3: DTO pesanan terstruktur

**Files:**
- Modify: `breeding-app/src/modules/sales/dto/create-sales-order.dto.ts`

**Interfaces:**
- Consumes: kolom dari Task 1.
- Produces: `CreateSalesOrderLineDto` dengan `productId`, `sourceProjectCoopId`, `birdCount`, `avgWeightKg` **wajib**, plus `isCulled`, `savingsPerKg`, `minimumWeightKg`, `unitPrice`, `notes` opsional; `CreateSalesOrderDto` dengan `validUntil` wajib dan `recipientName`, `recipientAddress`, `vatPercent` opsional. `UpdateSalesOrderDto` mewarisi lewat `PartialType` yang sudah ada.

- [ ] **Step 1: Tulis ulang `CreateSalesOrderLineDto`**

Ganti seluruh kelas `CreateSalesOrderLineDto` dengan:

```ts
export class CreateSalesOrderLineDto {
  @ApiProperty({ format: 'uuid', description: 'Produk ayam besar' })
  @IsUUID()
  productId: string;

  @ApiProperty({ format: 'uuid', description: 'Siklus kandang asal' })
  @IsUUID()
  sourceProjectCoopId: string;

  @ApiProperty({ description: 'Jumlah ekor', example: 500 })
  @IsInt()
  @IsPositive()
  birdCount: number;

  @ApiProperty({ description: 'Rata-rata bobot per ekor (kg)', example: 2.1 })
  @IsNumber({ maxDecimalPlaces: 4 })
  @IsPositive()
  @Type(() => Number)
  avgWeightKg: number;

  @ApiPropertyOptional({ description: 'Ayam afkir — harganya diketik manual', default: false })
  @IsOptional()
  @IsBoolean()
  isCulled?: boolean;

  @ApiPropertyOptional({ description: 'Bobot minimum yang disepakati (kg)' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 4 })
  @Min(0)
  @Type(() => Number)
  minimumWeightKg?: number;

  @ApiPropertyOptional({ description: 'Tabungan per kg' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Type(() => Number)
  savingsPerKg?: number;

  @ApiPropertyOptional({ description: 'Harga jual per kg — bebas, tanpa batas minimum (A5)' })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 4 })
  @Min(0)
  @Type(() => Number)
  unitPrice?: number;

  @ApiPropertyOptional({ description: 'Keterangan per produk' })
  @IsOptional()
  @IsString()
  lineNotes?: string;
}
```

`totalWeightKg` dan `totalPrice` **sengaja dihapus dari DTO**: keduanya dihitung backend (K6), dan `forbidNonWhitelisted` akan menolak 400 kalau klien tetap mengirimnya — itulah yang membuktikan Review Focus 5.

`lineNotes` adalah nama baru; `SalesOrderLine` belum punya kolom catatan, jadi Task 4 menyimpannya ke `productDescription` yang sudah ada dan kini tak terpakai.

- [ ] **Step 2: Tambahkan kolom header ke `CreateSalesOrderDto`**

Sisipkan sebelum properti `notes`:

```ts
  @ApiPropertyOptional({ description: 'Nama penerima — default nama customer, bisa diedit' })
  @IsOptional()
  @IsString()
  recipientName?: string;

  @ApiPropertyOptional({ description: 'Alamat kirim — default alamat customer, bisa diedit' })
  @IsOptional()
  @IsString()
  recipientAddress?: string;

  @ApiProperty({ example: '2026-09-29', description: 'Batas akhir masa berlaku DO' })
  @IsDateString()
  validUntil: string;

  @ApiPropertyOptional({ description: 'PPN persen', default: 0 })
  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @Min(0)
  @Max(100)
  @Type(() => Number)
  vatPercent?: number;
```

- [ ] **Step 3: Rapikan impor**

Baris impor `class-validator` menjadi:

```ts
import { IsString, IsDateString, IsOptional, IsArray, ValidateNested, IsUUID, IsEnum, IsNumber, IsInt, IsPositive, IsBoolean, Min, Max } from 'class-validator';
```

- [ ] **Step 4: Build**

Run: `cd breeding-app && npm run build`
Expected: keluar dengan status 0. Service belum memakai kolom baru, jadi kompilasi tetap lolos.

- [ ] **Step 5: Commit**

```bash
cd breeding-app
git add src/modules/sales/dto/create-sales-order.dto.ts
git commit -m "feat(sales): structured sales order line DTO"
```

---

### Task 4: Membuat pesanan dengan alokasi

**Files:**
- Modify: `breeding-app/src/modules/sales/sales-order.service.ts`
- Modify: `breeding-app/src/modules/sales/sales.module.ts`
- Test: `scratchpad/verify-so-create.py`

**Interfaces:**
- Consumes: `ReferenceNumberGenerator.generateSequential(tx, tenantId, prefix)` (Task 2); `CoopPopulationService.allocate(tx, tenantId, projectCoopId, quantity)` dan `.resolveProjectCoop(tx, projectCoopId, tenantId)` dari modul `project`; DTO Task 3.
- Produces: `SalesOrderService.create(tenantId, dto)` yang menyimpan baris terstruktur dan menahan ekor; helper privat `sumPerCoop(lines)` dan `releaseLines(tx, tenantId, lines)` yang dipakai Task 5 dan 6.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-so-create.py`:

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

pop = curl([API + "/coop-populations?limit=5"] + H)["data"]["data"]
PC = pop[0]["id"]
BRANCH = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]

def avail(pid):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]["quantityAvailable"]

def adjust(pid, qty, direction):
    return curl(["-X", "POST", API + "/coop-populations/" + pid + "/adjustments"] + H +
                ["-d", json.dumps({"quantity": qty, "direction": direction,
                                   "reason": "so " + RUN, "movementDate": "2026-09-27"})])

# customer yang terdaftar di area pesanan
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "SO Cust " + RUN, "code": "SOC-" + RUN,
                               "operatingBranchIds": [BRANCH["id"]]})])["data"]
outsider = curl(["-X", "POST", API + "/customers"] + H +
                ["-d", json.dumps({"name": "SO Outsider " + RUN, "code": "SOO-" + RUN})])["data"]

adjust(PC, 1000, "IN")
start = avail(PC)

def order(lines, **over):
    body = {"branchId": BRANCH["id"], "customerId": cust["id"], "recipientType": "CUSTOMER",
            "orderDate": "2026-09-27", "validUntil": "2026-09-29", "lines": lines}
    body.update(over)
    return curl(["-X", "POST", API + "/sales-orders"] + H + ["-d", json.dumps(body)])

def line(qty, **over):
    l = {"productId": product["id"], "sourceProjectCoopId": PC,
         "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}
    l.update(over)
    return l

r = order([line(100)])
so = r["data"]
check("1 order created", so["status"], "PENDING_APPROVAL")
check("2 do number uses the branch code", so["doNumber"].startswith("DO." + BRANCH["code"] + "."), True)
check("3 line keeps the product link", so["lines"][0]["productId"], product["id"])
check("4 allocation reduced availability", avail(PC), start - 100)
check("5 on-hand untouched by allocation",
      curl([API + "/coop-populations/" + PC] + H)["data"]["population"]["quantityOnHand"], start)

# Review Focus 5: backend menghitung tonase; nilai klien ditolak mentah
check("6 totalWeightKg is computed", float(so["lines"][0]["totalWeightKg"]), 200.0)
check("7 client-sent totalWeightKg is rejected",
      order([line(10, totalWeightKg="999")]).get("statusCode"), 400)

# Review Focus 2: beberapa baris dari kandang yang sama dijumlahkan dulu
left = avail(PC)
over = order([line(left // 2 + 10), line(left // 2 + 10)])
check("8 lines on one coop are summed before allocating", over.get("statusCode"), 400)
check("9 the rejected order left no allocation", avail(PC), left)

# validasi lain
check("10 customer outside the order area rejected",
      order([line(10)], customerId=outsider["id"]).get("statusCode"), 400)
check("11 validUntil before orderDate rejected",
      order([line(10)], validUntil="2026-09-26").get("statusCode"), 400)
check("12 zero birdCount rejected", order([line(0)]).get("statusCode"), 400)
check("13 fractional birdCount rejected", order([line(1.5)]).get("statusCode"), 400)
check("14 missing sourceProjectCoopId rejected",
      order([{"productId": product["id"], "birdCount": 5, "avgWeightKg": 2.0}]).get("statusCode"), 400)
check("15 vatPercent above 100 rejected", order([line(10)], vatPercent=101).get("statusCode"), 400)

# kandang milik tenant lain ditolak (Verifikasi #9 di spec)
BOGUS = "00000000-0000-4000-8000-000000000000"
before_foreign = avail(PC)
check("15b a coop that does not belong to this tenant is rejected",
      order([line(10, sourceProjectCoopId=BOGUS)]).get("statusCode"), 400)
check("15c the rejected order took nothing from the valid coop", avail(PC), before_foreign)

# Review Focus 1: dua DO bersamaan merebut ekor terakhir
rest = avail(PC)
with ThreadPoolExecutor(max_workers=2) as ex:
    a, b = [f.result() for f in [ex.submit(order, [line(rest)]), ex.submit(order, [line(rest)])]]
won = sum(1 for x in (a, b) if x.get("statusCode") != 400)
check("16 exactly one concurrent order wins the last birds", won, 1)
check("17 availability is exactly zero afterwards", avail(PC), 0)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-so-create.py`
Expected: FAIL pada pemeriksaan 2 — `doNumber` masih berformat `SO-YYYYMM-XXXX`, dan pemeriksaan 3 `KeyError`/`None` karena baris belum menyimpan `productId`.

- [ ] **Step 3: Suntikkan ketergantungan baru**

Di `src/modules/sales/sales.module.ts`, impor `ProjectModule` dan masukkan ke `imports`:

```ts
import { ProjectModule } from '../project/project.module.js';

@Module({
  imports: [ProjectModule],
  controllers: [SalesOrderController, DeliveryController],
  // providers dan exports tidak berubah
})
```

`ProjectModule` sudah mengekspor `CoopPopulationService`, jadi tidak ada perubahan di sisi sana.

- [ ] **Step 4: Tulis ulang `create`**

Di `sales-order.service.ts`, tambahkan impor:

```ts
import { BadRequestException, ConflictException } from '@nestjs/common';
import { CoopPopulationService } from '../project/coop-population.service.js';
```

suntikkan di konstruktor:

```ts
    private population: CoopPopulationService,
```

dan ganti `create` beserta dua helper baru:

```ts
  /**
   * Beberapa baris boleh menunjuk kandang yang sama. Alokasi harus dijumlahkan
   * dulu — menahan per baris membuat tiga baris @400 dari kandang berisi 1000
   * lolos satu per satu padahal totalnya tidak muat.
   */
  private sumPerCoop(lines: { sourceProjectCoopId?: string | null; birdCount?: number | null }[]) {
    const perCoop = new Map<string, number>();
    for (const line of lines) {
      if (!line.sourceProjectCoopId || !line.birdCount) continue;
      perCoop.set(
        line.sourceProjectCoopId,
        (perCoop.get(line.sourceProjectCoopId) ?? 0) + line.birdCount,
      );
    }
    return perCoop;
  }

  private async releaseLines(
    tx: Prisma.TransactionClient,
    tenantId: string,
    lines: { sourceProjectCoopId: string | null; birdCount: number | null }[],
  ) {
    for (const [projectCoopId, qty] of this.sumPerCoop(lines)) {
      await this.population.release(tx, tenantId, projectCoopId, qty);
    }
  }

  async create(tenantId: string, dto: CreateSalesOrderDto) {
    return this.prisma.$transaction(async (tx) => {
      const branch = await tx.branch.findFirst({
        where: { id: dto.branchId, tenantId, deletedAt: null },
        select: { id: true, code: true },
      });
      if (!branch) throw new BadRequestException('Area tidak ditemukan untuk tenant ini');

      const orderDate = new Date(dto.orderDate);
      const validUntil = new Date(dto.validUntil);
      if (validUntil < orderDate) {
        throw new BadRequestException('Masa berlaku tidak boleh lebih awal dari tanggal pesan');
      }

      // B5: customer hanya boleh dipesankan di area yang dicentang di masternya.
      if (dto.customerId) {
        const operates = await tx.customerBranch.findFirst({
          where: {
            customerId: dto.customerId,
            branchId: dto.branchId,
            customer: { tenantId, deletedAt: null },
          },
        });
        if (!operates) {
          throw new BadRequestException('Customer tidak beroperasi di area ini');
        }
      }

      for (const line of dto.lines) {
        await this.population.resolveProjectCoop(tx, line.sourceProjectCoopId, tenantId);
      }

      for (const [projectCoopId, qty] of this.sumPerCoop(dto.lines)) {
        await this.population.allocate(tx, tenantId, projectCoopId, qty);
      }

      const doNumber = await this.refGenerator.generateSequential(
        tx, tenantId, `DO.${branch.code}`,
      );

      const lines = dto.lines.map((line) => {
        // K6: tonase selalu diturunkan, tidak pernah diterima dari klien.
        const totalWeightKg = new Decimal(line.birdCount).mul(new Decimal(line.avgWeightKg));
        return {
          productId: line.productId,
          sourceProjectCoopId: line.sourceProjectCoopId,
          birdCount: line.birdCount,
          avgWeightKg: new Decimal(line.avgWeightKg),
          totalWeightKg,
          isCulled: line.isCulled ?? false,
          minimumWeightKg: line.minimumWeightKg != null ? new Decimal(line.minimumWeightKg) : null,
          savingsPerKg: line.savingsPerKg != null ? new Decimal(line.savingsPerKg) : null,
          unitPrice: line.unitPrice != null ? new Decimal(line.unitPrice) : null,
          totalPrice: line.unitPrice != null ? totalWeightKg.mul(new Decimal(line.unitPrice)) : null,
          productDescription: line.lineNotes ?? null,
        };
      });

      return tx.salesOrder.create({
        data: {
          tenantId,
          doNumber,
          branchId: dto.branchId,
          projectId: dto.projectId,
          customerId: dto.customerId,
          breederId: dto.breederId,
          recipientType: dto.recipientType,
          recipientName: dto.recipientName,
          recipientAddress: dto.recipientAddress,
          orderDate,
          validUntil,
          vatPercent: dto.vatPercent != null ? new Decimal(dto.vatPercent) : new Decimal(0),
          contractPrice: dto.contractPrice ? new Decimal(dto.contractPrice) : null,
          marketPrice: dto.marketPrice ? new Decimal(dto.marketPrice) : null,
          paymentMethod: dto.paymentMethod,
          notes: dto.notes,
          lines: { create: lines },
        },
        include: { lines: true },
      });
    });
  }
```

Tambahkan impor `Prisma` kalau belum ada:

```ts
import { Prisma } from '../../../generated/prisma/client.js';
```

- [ ] **Step 4b: Sertakan relasi baris di setiap pembacaan**

Frontend menampilkan kode produk dan kode kandang, jadi `lines: true` saja tidak cukup.
Tambahkan konstanta di kelasnya dan pakai di `create`, `findAll`, `findOne`, `update`,
`transitionStatus`, dan `remove` — ganti setiap `include: { lines: true }` dengannya:

```ts
  private readonly orderInclude = {
    lines: {
      include: {
        product: { select: { id: true, code: true, name: true } },
        sourceProjectCoop: {
          select: { id: true, coop: { select: { code: true, name: true } } },
        },
      },
    },
  } as const;
```

`findOne` dan `findAll` juga perlu `customer: { select: { id: true, name: true, code: true } }`
dan `branch: { select: { id: true, code: true, name: true } }` supaya daftar dan detail
tidak perlu memanggil balik satu per satu.

- [ ] **Step 5: Build, hidupkan lagi dev server, jalankan skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi preview `api`, lalu `python3 scratchpad/verify-so-create.py`
Expected: `RESULT: all checks pass` — 17 dari 17.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): structured order creation allocates birds per coop"
```

---

### Task 5: Mengubah pesanan dan realokasi

**Files:**
- Modify: `breeding-app/src/modules/sales/sales-order.service.ts`
- Modify: `breeding-app/src/modules/sales/dto/update-sales-order.dto.ts`
- Test: `scratchpad/verify-so-update.py`

**Interfaces:**
- Consumes: `sumPerCoop`, `releaseLines` dari Task 4.
- Produces: `SalesOrderService.update(tenantId, id, dto)` yang mengganti baris beserta alokasinya, hanya pada `PENDING_APPROVAL`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-so-update.py`:

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

pop = curl([API + "/coop-populations?limit=5"] + H)["data"]["data"]
PC = pop[0]["id"]
BRANCH = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "Upd Cust " + RUN, "code": "UPC-" + RUN,
                               "operatingBranchIds": [BRANCH["id"]]})])["data"]

def avail(pid):
    return curl([API + "/coop-populations/" + pid] + H)["data"]["population"]["quantityAvailable"]

curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 1000, "direction": "IN", "reason": "upd " + RUN, "movementDate": "2026-09-27"})])
start = avail(PC)

def line(qty):
    return {"productId": product["id"], "sourceProjectCoopId": PC,
            "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}

so = curl(["-X", "POST", API + "/sales-orders"] + H +
          ["-d", json.dumps({"branchId": BRANCH["id"], "customerId": cust["id"],
                             "recipientType": "CUSTOMER", "orderDate": "2026-09-27",
                             "validUntil": "2026-09-29", "lines": [line(300)]})])["data"]
check("1 initial allocation", avail(PC), start - 300)

def patch(body):
    return curl(["-X", "PATCH", API + "/sales-orders/" + so["id"]] + H + ["-d", json.dumps(body)])

r = patch({"lines": [line(120)]})
check("2 lowering the order frees the difference", avail(PC), start - 120)
check("3 the order now has one line of 120", r["data"]["lines"][0]["birdCount"], 120)

r = patch({"lines": [line(500)]})
check("4 raising the order takes more", avail(PC), start - 500)

# header-only edits must not touch allocation
r = patch({"notes": "catatan " + RUN})
check("5 a header-only edit leaves allocation alone", avail(PC), start - 500)
check("6 lines survive a header-only edit", len(r["data"]["lines"]), 1)

# Review Focus 3: kalau alokasi baru tidak muat, yang lama harus utuh kembali
before = avail(PC)
check("7 an over-sized update is rejected", patch({"lines": [line(999999)]}).get("statusCode"), 400)
check("8 the old allocation survived the failed update", avail(PC), before)
check("9 the order still has its 500-bird line",
      curl([API + "/sales-orders/" + so["id"]] + H)["data"]["lines"][0]["birdCount"], 500)

# status selain PENDING_APPROVAL tidak boleh mengubah baris
curl(["-X", "PATCH", API + "/sales-orders/" + so["id"] + "/status"] + H +
     ["-d", json.dumps({"status": "APPROVED"})])
check("10 editing lines of an approved order is refused", patch({"lines": [line(10)]}).get("statusCode"), 409)

curl(["-X", "DELETE", API + "/sales-orders/" + so["id"]] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-so-update.py`
Expected: FAIL pada pemeriksaan 2 — `update` belum menyentuh baris sama sekali, jadi ketersediaan tetap `start - 300`.

- [ ] **Step 3: Izinkan `lines` di DTO update**

`UpdateSalesOrderDto` memakai `PartialType(CreateSalesOrderDto)`, jadi `lines` sudah ikut opsional. Pastikan berkasnya berisi:

```ts
import { PartialType } from '@nestjs/swagger';
import { CreateSalesOrderDto } from './create-sales-order.dto.js';

export class UpdateSalesOrderDto extends PartialType(CreateSalesOrderDto) {}
```

- [ ] **Step 4: Tulis ulang `update`**

Ganti metode `update` di `sales-order.service.ts`:

```ts
  async update(tenantId: string, id: string, dto: UpdateSalesOrderDto) {
    return this.prisma.$transaction(async (tx) => {
      const current = await tx.salesOrder.findFirst({
        where: { id, tenantId, deletedAt: null },
        include: { lines: true },
      });
      if (!current) throw new NotFoundException('Sales order not found');

      if (dto.lines !== undefined) {
        if (current.status !== SalesStatus.PENDING_APPROVAL) {
          throw new ConflictException(
            'Baris pesanan hanya bisa diubah selama masih menunggu persetujuan',
          );
        }

        // Lepas dulu yang lama, lalu tahan yang baru. Kalau yang baru tidak
        // muat, seluruh transaksi dibatalkan dan alokasi lama kembali utuh.
        await this.releaseLines(tx, tenantId, current.lines);

        for (const line of dto.lines) {
          await this.population.resolveProjectCoop(tx, line.sourceProjectCoopId, tenantId);
        }
        for (const [projectCoopId, qty] of this.sumPerCoop(dto.lines)) {
          await this.population.allocate(tx, tenantId, projectCoopId, qty);
        }

        await tx.salesOrderLine.deleteMany({ where: { salesOrderId: id } });
        await tx.salesOrderLine.createMany({
          data: dto.lines.map((line) => {
            const totalWeightKg = new Decimal(line.birdCount).mul(new Decimal(line.avgWeightKg));
            return {
              salesOrderId: id,
              productId: line.productId,
              sourceProjectCoopId: line.sourceProjectCoopId,
              birdCount: line.birdCount,
              avgWeightKg: new Decimal(line.avgWeightKg),
              totalWeightKg,
              isCulled: line.isCulled ?? false,
              minimumWeightKg: line.minimumWeightKg != null ? new Decimal(line.minimumWeightKg) : null,
              savingsPerKg: line.savingsPerKg != null ? new Decimal(line.savingsPerKg) : null,
              unitPrice: line.unitPrice != null ? new Decimal(line.unitPrice) : null,
              totalPrice: line.unitPrice != null ? totalWeightKg.mul(new Decimal(line.unitPrice)) : null,
              productDescription: line.lineNotes ?? null,
            };
          }),
        });
      }

      const data: Record<string, any> = {};
      if (dto.projectId !== undefined) data.projectId = dto.projectId;
      if (dto.recipientName !== undefined) data.recipientName = dto.recipientName;
      if (dto.recipientAddress !== undefined) data.recipientAddress = dto.recipientAddress;
      if (dto.orderDate !== undefined) data.orderDate = new Date(dto.orderDate);
      if (dto.validUntil !== undefined) data.validUntil = new Date(dto.validUntil);
      if (dto.vatPercent !== undefined) data.vatPercent = new Decimal(dto.vatPercent);
      if (dto.contractPrice !== undefined)
        data.contractPrice = dto.contractPrice ? new Decimal(dto.contractPrice) : null;
      if (dto.marketPrice !== undefined)
        data.marketPrice = dto.marketPrice ? new Decimal(dto.marketPrice) : null;
      if (dto.paymentMethod !== undefined) data.paymentMethod = dto.paymentMethod;
      if (dto.notes !== undefined) data.notes = dto.notes;

      const orderDate = data.orderDate ?? current.orderDate;
      const validUntil = data.validUntil ?? current.validUntil;
      if (validUntil && validUntil < orderDate) {
        throw new BadRequestException('Masa berlaku tidak boleh lebih awal dari tanggal pesan');
      }

      if (Object.keys(data).length > 0) {
        await tx.salesOrder.update({ where: { id }, data });
      }

      return tx.salesOrder.findUniqueOrThrow({ where: { id }, include: { lines: true } });
    });
  }
```

`branchId`, `customerId`, `breederId`, dan `recipientType` sengaja tidak lagi bisa diubah lewat `update`: memindahkan pesanan ke area lain akan melewati pemeriksaan B5 dan memindahkan alokasi tanpa jejak. Hapus pesanan lalu buat baru.

- [ ] **Step 5: Build, hidupkan lagi dev server, jalankan skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi preview `api`, lalu `python3 scratchpad/verify-so-update.py`
Expected: `RESULT: all checks pass` — 10 dari 10.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): re-allocate birds when a pending order changes"
```

---

### Task 6: Melepas alokasi saat ditolak dan dibatalkan

**Files:**
- Modify: `breeding-app/src/modules/sales/sales-order.service.ts`
- Modify: `breeding-app/src/common/constants/status-transitions.constant.ts:63`
- Test: `scratchpad/verify-so-release.py`

**Interfaces:**
- Consumes: `releaseLines` dari Task 4.
- Produces: `transitionStatus` yang melepas alokasi saat pindah ke `REJECTED`/`CANCELLED`; `remove` yang melepas saat pesanan dihapus; matriks `APPROVED → REJECTED`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-so-release.py`:

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

PC = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]["id"]
BRANCH = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "Rel Cust " + RUN, "code": "RLC-" + RUN,
                               "operatingBranchIds": [BRANCH["id"]]})])["data"]

def avail():
    return curl([API + "/coop-populations/" + PC] + H)["data"]["population"]["quantityAvailable"]

curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 1000, "direction": "IN", "reason": "rel " + RUN, "movementDate": "2026-09-27"})])
start = avail()

def make(qty=200):
    return curl(["-X", "POST", API + "/sales-orders"] + H +
                ["-d", json.dumps({"branchId": BRANCH["id"], "customerId": cust["id"],
                                   "recipientType": "CUSTOMER", "orderDate": "2026-09-27",
                                   "validUntil": "2026-09-29",
                                   "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                              "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}]})])["data"]

def status(oid, s):
    return curl(["-X", "PATCH", API + "/sales-orders/" + oid + "/status"] + H + ["-d", json.dumps({"status": s})])

a = make()
check("1 allocated on create", avail(), start - 200)
status(a["id"], "REJECTED")
check("2 rejecting releases the allocation", avail(), start)
check("3 rejecting twice does not release twice",
      [status(a["id"], "REJECTED").get("statusCode"), avail()][1], start)

b = make()
status(b["id"], "CANCELLED")
check("4 cancelling releases the allocation", avail(), start)

c = make()
curl(["-X", "DELETE", API + "/sales-orders/" + c["id"]] + H)
check("5 deleting a pending order releases the allocation", avail(), start)

d = make()
status(d["id"], "APPROVED")
check("6 an approved order keeps its allocation", avail(), start - 200)
check("7 an approved order may be rejected", status(d["id"], "REJECTED").get("statusCode", 200), 200)
check("8 rejecting an approved order releases it", avail(), start)

print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-so-release.py`
Expected: FAIL pada pemeriksaan 2 — penolakan belum melepas apa pun, ketersediaan tetap `start - 200`.

- [ ] **Step 3: Izinkan `APPROVED → REJECTED`**

Matriks saat ini hanya mengizinkan `APPROVED → REALIZATION_APPROVAL`, sehingga DO yang sudah disetujui lalu kadaluarsa tidak bisa ditolak sama sekali. Di `src/common/constants/status-transitions.constant.ts` baris 63:

```ts
  [SalesStatus.APPROVED]: [SalesStatus.REALIZATION_APPROVAL, SalesStatus.REJECTED, SalesStatus.CANCELLED],
```

- [ ] **Step 4: Lepaskan alokasi di `transitionStatus` dan `remove`**

Ganti kedua metode di `sales-order.service.ts`:

```ts
  private static readonly RELEASING_STATUSES: SalesStatus[] = [
    SalesStatus.REJECTED,
    SalesStatus.CANCELLED,
  ];

  async transitionStatus(
    tenantId: string,
    id: string,
    targetStatus: SalesStatus,
    userRole: SystemRole,
    userId?: string,
  ) {
    return this.prisma.$transaction(async (tx) => {
      const so = await tx.salesOrder.findFirst({
        where: { id, tenantId, deletedAt: null },
        include: { lines: true },
      });
      if (!so) throw new NotFoundException('Sales order not found');

      this.statusTransition.validateTransition(
        so.status, targetStatus, userRole, SALES_STATUS_TRANSITIONS,
      );

      const data: Record<string, any> = { status: targetStatus };
      if (targetStatus === SalesStatus.APPROVED && userId) {
        data.approvalDate = new Date();
        data.approvedBy = userId;
      }
      if (
        (targetStatus === SalesStatus.REALIZING ||
          targetStatus === SalesStatus.REALIZING_DO_LIMIT) && userId
      ) {
        data.realizationDate = new Date();
        data.realizedBy = userId;
      }

      // Syarat pada status lama: dua permintaan bersamaan tidak bisa sama-sama
      // menang, jadi alokasinya tidak pernah dilepas dua kali.
      const moved = await tx.salesOrder.updateMany({
        where: { id, tenantId, status: so.status },
        data,
      });
      if (moved.count !== 1) {
        throw new ConflictException('Status pesanan ini baru saja diubah orang lain');
      }

      if (SalesOrderService.RELEASING_STATUSES.includes(targetStatus)) {
        await this.releaseLines(tx, tenantId, so.lines);
      }

      return tx.salesOrder.findUniqueOrThrow({ where: { id }, include: { lines: true } });
    });
  }

  async remove(tenantId: string, id: string) {
    return this.prisma.$transaction(async (tx) => {
      const so = await tx.salesOrder.findFirst({
        where: { id, tenantId, deletedAt: null },
        include: { lines: true },
      });
      if (!so) throw new NotFoundException('Sales order not found');

      const claimed = await tx.salesOrder.updateMany({
        where: { id, tenantId, deletedAt: null },
        data: { deletedAt: new Date() },
      });
      if (claimed.count !== 1) throw new NotFoundException('Sales order not found');

      // Pesanan yang sudah ditolak atau dibatalkan tidak lagi menahan apa pun.
      if (!SalesOrderService.RELEASING_STATUSES.includes(so.status)) {
        await this.releaseLines(tx, tenantId, so.lines);
      }

      return tx.salesOrder.findUniqueOrThrow({ where: { id } });
    });
  }
```

- [ ] **Step 5: Build, hidupkan lagi dev server, jalankan ketiga skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi preview `api`, lalu
`python3 scratchpad/verify-so-release.py && python3 scratchpad/verify-so-create.py && python3 scratchpad/verify-so-update.py`
Expected: ketiganya `RESULT: all checks pass` — 8/8, 17/17, 10/10.

- [ ] **Step 6: Commit**

```bash
cd breeding-app
git add src/modules/sales src/common/constants/status-transitions.constant.ts
git commit -m "feat(sales): release allocations when an order is rejected or cancelled"
```

---

### Task 7: Harga rekomendasi

**Files:**
- Modify: `breeding-app/src/modules/sales/sales-order.service.ts`
- Modify: `breeding-app/src/modules/sales/sales-order.controller.ts`
- Create: `breeding-app/src/modules/sales/dto/query-recommended-price.dto.ts`
- Test: `scratchpad/verify-so-price.py`

**Interfaces:**
- Consumes: baris terstruktur dari Task 4.
- Produces: `GET /sales-orders/recommended-price?productId=&customerId=` yang mengembalikan `{ unitPrice: string | null, lastOrderDate: string | null, lastDoNumber: string | null }`.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-so-price.py`:

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

PC = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]["id"]
BRANCH = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 1000, "direction": "IN", "reason": "prc " + RUN, "movementDate": "2026-09-27"})])

def customer(tag):
    return curl(["-X", "POST", API + "/customers"] + H +
                ["-d", json.dumps({"name": tag + " " + RUN, "code": tag + "-" + RUN,
                                   "operatingBranchIds": [BRANCH["id"]]})])["data"]

a, b = customer("PRA"), customer("PRB")

def order(cust, price, order_date="2026-09-27"):
    return curl(["-X", "POST", API + "/sales-orders"] + H +
                ["-d", json.dumps({"branchId": BRANCH["id"], "customerId": cust["id"],
                                   "recipientType": "CUSTOMER", "orderDate": order_date,
                                   "validUntil": "2026-09-30",
                                   "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                              "birdCount": 10, "avgWeightKg": 2.0,
                                              "unitPrice": price}]})])["data"]

def rec(cust):
    return curl([API + "/sales-orders/recommended-price?productId=" + product["id"] +
                 "&customerId=" + cust["id"]] + H)["data"]

check("1 a brand-new customer has no recommendation", rec(a)["unitPrice"], None)

o1 = order(a, 20000, "2026-09-20")
check("2 the first order becomes the recommendation", float(rec(a)["unitPrice"]), 20000.0)

o2 = order(a, 23000, "2026-09-25")
check("3 the newest order wins", float(rec(a)["unitPrice"]), 23000.0)
check("4 the recommendation names its source", rec(a)["lastDoNumber"], o2["doNumber"])

check("5 another customer does not inherit the price", rec(b)["unitPrice"], None)

curl(["-X", "PATCH", API + "/sales-orders/" + o2["id"] + "/status"] + H +
     ["-d", json.dumps({"status": "REJECTED"})])
check("6 a rejected order is not used as a recommendation", float(rec(a)["unitPrice"]), 20000.0)

check("7 a missing customerId is rejected",
      curl([API + "/sales-orders/recommended-price?productId=" + product["id"]] + H).get("statusCode"), 400)

for o in (o1, o2):
    curl(["-X", "DELETE", API + "/sales-orders/" + o["id"]] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-so-price.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 1 — rutenya belum ada.

- [ ] **Step 3: Tulis DTO query**

Create `breeding-app/src/modules/sales/dto/query-recommended-price.dto.ts`:

```ts
import { ApiProperty } from '@nestjs/swagger';
import { IsUUID } from 'class-validator';

export class QueryRecommendedPriceDto {
  @ApiProperty({ format: 'uuid' })
  @IsUUID()
  productId: string;

  @ApiProperty({ format: 'uuid' })
  @IsUUID()
  customerId: string;
}
```

- [ ] **Step 4: Tambahkan metodenya**

Di `sales-order.service.ts`:

```ts
  /**
   * K5: harga jual terakhir untuk produk itu kepada customer itu. Pesanan yang
   * ditolak atau dibatalkan tidak dipakai — harga yang tidak jadi bukan harga.
   * Customer baru mengembalikan null, dan form menampilkan tanda hubung; angka
   * dari customer lain akan menyesatkan orang yang sedang menawar.
   */
  async recommendedPrice(tenantId: string, productId: string, customerId: string) {
    const line = await this.prisma.salesOrderLine.findFirst({
      where: {
        productId,
        unitPrice: { not: null },
        salesOrder: {
          tenantId,
          customerId,
          deletedAt: null,
          status: { notIn: [SalesStatus.REJECTED, SalesStatus.CANCELLED] },
        },
      },
      orderBy: [{ salesOrder: { orderDate: 'desc' } }, { salesOrder: { createdAt: 'desc' } }],
      select: {
        unitPrice: true,
        salesOrder: { select: { orderDate: true, doNumber: true } },
      },
    });

    return {
      unitPrice: line?.unitPrice ?? null,
      lastOrderDate: line?.salesOrder.orderDate ?? null,
      lastDoNumber: line?.salesOrder.doNumber ?? null,
    };
  }
```

- [ ] **Step 5: Tambahkan rutenya**

Di `sales-order.controller.ts`, **sebelum** `@Get(':id')` — kalau diletakkan sesudahnya, Nest akan mencocokkan `recommended-price` sebagai sebuah id:

```ts
  @Get('recommended-price')
  @ApiOperation({ summary: 'Harga jual terakhir untuk produk ini ke customer ini' })
  recommendedPrice(
    @CurrentTenant() tenantId: string,
    @Query() query: QueryRecommendedPriceDto,
  ) {
    return this.service.recommendedPrice(tenantId, query.productId, query.customerId);
  }
```

Tambahkan impor `QueryRecommendedPriceDto`.

- [ ] **Step 6: Build, hidupkan lagi dev server, jalankan skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi preview `api`, lalu `python3 scratchpad/verify-so-price.py`
Expected: `RESULT: all checks pass` — 7 dari 7.

- [ ] **Step 7: Commit**

```bash
cd breeding-app
git add src/modules/sales
git commit -m "feat(sales): recommended price from the last order to this customer"
```

---

### Task 8: Job kadaluarsa

**Files:**
- Modify: `breeding-app/package.json` (dependensi `@nestjs/schedule`)
- Modify: `breeding-app/src/app.module.ts`
- Create: `breeding-app/src/modules/sales/sales-order-expiry.scheduler.ts`
- Modify: `breeding-app/src/modules/sales/sales-order.service.ts`
- Modify: `breeding-app/src/modules/sales/sales-order.controller.ts`
- Modify: `breeding-app/src/modules/sales/sales.module.ts`
- Test: `scratchpad/verify-so-expiry.py`

**Interfaces:**
- Consumes: `releaseLines` dari Task 4; matriks `APPROVED → REJECTED` dari Task 6.
- Produces: `SalesOrderService.expireOverdue(tenantId): Promise<{ expired: number; failed: number }>`; rute `POST /sales-orders/expire-overdue` (MANAGER ke atas); `SalesOrderExpiryScheduler` dengan cron harian.

- [ ] **Step 1: Tulis skrip verifikasi yang gagal**

Create `scratchpad/verify-so-expiry.py`:

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

PC = curl([API + "/coop-populations?limit=1"] + H)["data"]["data"][0]["id"]
BRANCH = curl([API + "/branches?limit=1"] + H)["data"]["data"][0]
product = curl([API + "/products?limit=1"] + H)["data"]["data"][0]
cust = curl(["-X", "POST", API + "/customers"] + H +
            ["-d", json.dumps({"name": "Exp Cust " + RUN, "code": "EXC-" + RUN,
                               "operatingBranchIds": [BRANCH["id"]]})])["data"]

def avail():
    return curl([API + "/coop-populations/" + PC] + H)["data"]["population"]["quantityAvailable"]

curl(["-X", "POST", API + "/coop-populations/" + PC + "/adjustments"] + H +
     ["-d", json.dumps({"quantity": 1000, "direction": "IN", "reason": "exp " + RUN, "movementDate": "2026-09-27"})])
start = avail()

def make(order_date, valid_until, qty=100):
    return curl(["-X", "POST", API + "/sales-orders"] + H +
                ["-d", json.dumps({"branchId": BRANCH["id"], "customerId": cust["id"],
                                   "recipientType": "CUSTOMER", "orderDate": order_date,
                                   "validUntil": valid_until,
                                   "lines": [{"productId": product["id"], "sourceProjectCoopId": PC,
                                              "birdCount": qty, "avgWeightKg": 2.0, "unitPrice": 25000}]})])["data"]

stale = make("2026-01-01", "2026-01-03")
fresh = make("2026-09-27", "2099-12-31")
check("1 both orders allocated", avail(), start - 200)

r = curl(["-X", "POST", API + "/sales-orders/expire-overdue"] + H + ["-d", "{}"])["data"]
check("2 exactly one order expired", r["expired"], 1)
check("3 the stale order is rejected",
      curl([API + "/sales-orders/" + stale["id"]] + H)["data"]["status"], "REJECTED")
check("4 its allocation is released", avail(), start - 100)
check("5 the fresh order is untouched",
      curl([API + "/sales-orders/" + fresh["id"]] + H)["data"]["status"], "PENDING_APPROVAL")

# Review Focus 4: menjalankan job lagi tidak boleh melepas dua kali
again = curl(["-X", "POST", API + "/sales-orders/expire-overdue"] + H + ["-d", "{}"])["data"]
check("6 a second run expires nothing", again["expired"], 0)
check("7 and releases nothing twice", avail(), start - 100)

# pesanan lama tanpa validUntil tidak boleh tersapu — dibandingkan sebelum/sesudah
def legacy_statuses():
    rows = curl([API + "/sales-orders?limit=100"] + H)["data"]["data"]
    return {o["id"]: o["status"] for o in rows if o.get("validUntil") is None}

before_legacy = legacy_statuses()
curl(["-X", "POST", API + "/sales-orders/expire-overdue"] + H + ["-d", "{}"])
check("8 orders without validUntil are left untouched", legacy_statuses(), before_legacy)

curl(["-X", "DELETE", API + "/sales-orders/" + fresh["id"]] + H)
print()
print("RESULT:", "all checks pass" if not fails else f"{len(fails)} FAILING: {fails}")
```

- [ ] **Step 2: Jalankan dan pastikan gagal**

Run: `python3 scratchpad/verify-so-expiry.py`
Expected: gagal dengan `KeyError: 'data'` pada pemeriksaan 2 — rute `expire-overdue` belum ada.

- [ ] **Step 3: Pasang `@nestjs/schedule`**

Run: `cd breeding-app && npm install @nestjs/schedule`
Expected: terpasang tanpa galat peer-dependency.

- [ ] **Step 4: Daftarkan `ScheduleModule`**

Di `src/app.module.ts`, tambahkan impor dan masukkan ke `imports` root:

```ts
import { ScheduleModule } from '@nestjs/schedule';
// …
    ScheduleModule.forRoot(),
```

- [ ] **Step 5: Tambahkan `expireOverdue` ke service**

```ts
  /**
   * A7: DO yang lewat masa berlaku ditolak otomatis. Karena K2 membuat DO
   * menahan ekor sejak dibuat, penolakan ini HARUS melepas alokasinya — kalau
   * tidak, stok kandang beku selamanya.
   *
   * Satu transaksi per pesanan: satu pesanan bermasalah tidak menghentikan
   * sisanya. Klaim status bersyarat membuat dua kali jalan tidak melepas dua kali.
   */
  async expireOverdue(tenantId: string) {
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    const due = await this.prisma.salesOrder.findMany({
      where: {
        tenantId,
        deletedAt: null,
        validUntil: { not: null, lt: today },
        status: { in: [SalesStatus.PENDING_APPROVAL, SalesStatus.APPROVED] },
      },
      include: { lines: true },
    });

    let expired = 0;
    let failed = 0;

    for (const so of due) {
      try {
        const done = await this.prisma.$transaction(async (tx) => {
          const claimed = await tx.salesOrder.updateMany({
            where: { id: so.id, tenantId, status: so.status, deletedAt: null },
            data: { status: SalesStatus.REJECTED },
          });
          if (claimed.count !== 1) return false;
          await this.releaseLines(tx, tenantId, so.lines);
          return true;
        });
        if (done) expired++;
      } catch (error) {
        failed++;
        this.logger.error(
          `Gagal menolak DO kadaluarsa ${so.doNumber ?? so.id}: ${(error as Error).message}`,
        );
      }
    }

    return { expired, failed };
  }
```

Tambahkan logger di kelasnya:

```ts
  private readonly logger = new Logger(SalesOrderService.name);
```

dan impor `Logger` dari `@nestjs/common`.

- [ ] **Step 6: Tambahkan rute manual**

Di `sales-order.controller.ts`, **sebelum** `@Get(':id')` dan `@Patch(':id')` tidak relevan karena ini `@Post`, tetapi tetap letakkan di atas rute ber-parameter agar terbaca berurutan:

```ts
  @Post('expire-overdue')
  @Roles(SystemRole.SUPER_ADMIN, SystemRole.TENANT_ADMIN, SystemRole.MANAGER)
  @ApiOperation({ summary: 'Tolak semua DO yang lewat masa berlaku dan lepaskan alokasinya' })
  expireOverdue(@CurrentTenant() tenantId: string) {
    return this.service.expireOverdue(tenantId);
  }
```

Pastikan `RolesGuard` ada di `@UseGuards` tingkat kelas dan `Roles`/`SystemRole` terimpor.

- [ ] **Step 7: Tulis scheduler**

Create `breeding-app/src/modules/sales/sales-order-expiry.scheduler.ts`:

```ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PrismaService } from '../../prisma/prisma.service.js';
import { SalesOrderService } from './sales-order.service.js';

@Injectable()
export class SalesOrderExpiryScheduler {
  private readonly logger = new Logger(SalesOrderExpiryScheduler.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly salesOrders: SalesOrderService,
  ) {}

  /** Berjalan per tenant supaya satu tenant bermasalah tidak menghentikan yang lain. */
  @Cron(CronExpression.EVERY_DAY_AT_1AM)
  async expireOverdueOrders() {
    const tenants = await this.prisma.tenant.findMany({
      where: { deletedAt: null },
      select: { id: true },
    });

    for (const tenant of tenants) {
      try {
        const { expired, failed } = await this.salesOrders.expireOverdue(tenant.id);
        if (expired > 0 || failed > 0) {
          this.logger.log(`Tenant ${tenant.id}: ${expired} DO kadaluarsa ditolak, ${failed} gagal`);
        }
      } catch (error) {
        this.logger.error(`Tenant ${tenant.id}: job kadaluarsa gagal — ${(error as Error).message}`);
      }
    }
  }
}
```

Daftarkan `SalesOrderExpiryScheduler` di `providers` `sales.module.ts`.

- [ ] **Step 8: Build, hidupkan lagi dev server, jalankan skrip**

Run: `cd breeding-app && npm run build`, hidupkan lagi preview `api`, lalu `python3 scratchpad/verify-so-expiry.py`
Expected: `RESULT: all checks pass` — 8 dari 8.

- [ ] **Step 9: Commit**

```bash
cd breeding-app
git add package.json package-lock.json src/app.module.ts src/modules/sales
git commit -m "feat(sales): expire overdue orders daily and release their allocations"
```

---

### Task 9: Tipe, i18n, dan combobox standar bobot

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts`
- Modify: `breeding-dashboard/messages/en.json`, `breeding-dashboard/messages/id.json`
- Create: `breeding-dashboard/src/components/forms/mature-bird-standard-combobox.tsx`

**Interfaces:**
- Consumes: bentuk respons dari Task 4, 6, 7.
- Produces: tipe `SalesOrderLine` (diperluas), `SalesOrder` (diperluas), `RecommendedPrice`; komponen `<MatureBirdStandardCombobox value onChange disabled />`; namespace i18n `salesOrders` diperluas.

- [ ] **Step 1: Perluas tipe**

Di `breeding-dashboard/src/types/api.ts`, tambahkan di bawah tipe yang ada:

```ts
export interface RecommendedPrice {
  unitPrice: string | null;
  lastOrderDate: string | null;
  lastDoNumber: string | null;
}

export interface StructuredSalesOrderLine {
  id: string;
  productId: string | null;
  sourceProjectCoopId: string | null;
  birdCount: number | null;
  avgWeightKg: string | null;
  totalWeightKg: string | null;
  isCulled: boolean;
  minimumWeightKg: string | null;
  savingsPerKg: string | null;
  unitPrice: string | null;
  totalPrice: string | null;
  productDescription: string | null;
  product?: { id: string; code: string; name: string } | null;
  sourceProjectCoop?: { id: string; coop: { code: string; name: string } } | null;
}
```

Pada interface `SalesOrder` yang sudah ada, tambahkan:

```ts
  recipientName: string | null;
  recipientAddress: string | null;
  validUntil: string | null;
  vatPercent: string | null;
```

- [ ] **Step 2: Tambahkan kunci i18n ke kedua berkas**

Ke namespace `salesOrders` di `en.json`:

```json
    "validUntil": "Valid until",
    "recipientName": "Recipient name",
    "recipientAddress": "Delivery address",
    "vatPercent": "VAT %",
    "creditLimit": "Credit limit",
    "topDays": "Terms (days)",
    "balance": "Balance",
    "balanceNotBuilt": "Receivables are not built yet",
    "sourceCoop": "Source coop",
    "available": "Available",
    "avgWeight": "Avg weight (kg)",
    "birdCount": "Birds",
    "totalWeight": "Tonnage (kg)",
    "culled": "Culled",
    "minimumWeight": "Minimum weight",
    "recommendedPrice": "Recommended price",
    "noRecommendation": "No earlier price for this customer",
    "unitPrice": "Sell price",
    "savingsPerKg": "Savings / kg",
    "lineNotes": "Line notes",
    "lineRequired": "Add at least one line with a product, a coop and a bird count",
    "customerAreaHint": "Only customers operating in the selected area are listed",
    "expireOverdue": "Expire overdue orders"
```

Di `id.json` kunci yang sama dengan nilai bahasa Indonesia — misalnya `"validUntil": "Berlaku sampai"`, `"sourceCoop": "Kandang asal"`, `"birdCount": "Ekor"`, `"totalWeight": "Tonase (kg)"`, `"culled": "Afkir"`, `"minimumWeight": "Bobot minimum"`, `"recommendedPrice": "Harga rekomendasi"`, `"noRecommendation": "Belum ada harga sebelumnya untuk customer ini"`, `"unitPrice": "Harga jual"`, `"balanceNotBuilt": "Piutang belum dibangun"`.

- [ ] **Step 3: Tulis combobox standar bobot**

Create `breeding-dashboard/src/components/forms/mature-bird-standard-combobox.tsx`, meniru struktur `src/components/forms/project-coop-combobox.tsx` persis, dengan perbedaan:

```tsx
import { MatureBirdStandard } from "@/types/api";

interface MatureBirdStandardComboboxProps {
  value: string;
  onChange: (standardId: string, minimumWeightKg: number | null) => void;
  disabled?: boolean;
}

// di dalam komponen:
  useEffect(() => {
    setIsLoading(true);
    fetchPaginated<MatureBirdStandard>("/mature-bird-standards", { limit: 50, search })
      .then((res) => setRows(res.data))
      .catch(() => {})
      .finally(() => setIsLoading(false));
  }, [search]);

  // bobot minimum = batas bawah terendah dari range standar itu
  function minimumOf(row: MatureBirdStandard): number | null {
    if (row.ranges.length === 0) return null;
    return Math.min(...row.ranges.map((r) => Number(r.valueFrom)));
  }
```

`onSelect` memanggil `onChange(row.id, minimumOf(row))`, dan label item menampilkan `row.name` dengan bobot minimumnya sebagai teks kecil di kanan.

- [ ] **Step 4: Periksa tipe, build, dan hitung kunci**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

Run: `cd breeding-dashboard && node -e "['en','id'].forEach(l=>{const m=require('./messages/'+l+'.json');console.log(l,Object.keys(m.salesOrders).length)})"`
Expected: dua baris dengan **angka yang sama** di kedua bahasa.

- [ ] **Step 5: Commit**

```bash
cd breeding-dashboard
git add src/types/api.ts messages src/components/forms/mature-bird-standard-combobox.tsx
git commit -m "feat(sales): structured order types, i18n keys and standard combobox"
```

---

### Task 10: Form pesanan

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/sales-orders/new/page.tsx` (ditulis ulang)

**Interfaces:**
- Consumes: tipe dan combobox Task 9; rute Task 4 dan 7; `<CustomerCombobox branchId>` dan `<ProjectCoopCombobox>` yang sudah ada.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Tulis ulang halaman**

Pertahankan kerangka halaman yang ada (`PageHeader`, kartu, `Button` simpan). Yang berubah:

Header — `BranchCombobox` untuk area; `<CustomerCombobox value={customerId} onChange={…} branchId={branchId} />` sehingga hanya customer yang beroperasi di area itu muncul (B5); `recipientName` dan `recipientAddress` terisi otomatis saat customer dipilih tetapi tetap bisa diedit; `orderDate`; `validUntil` dengan default dua hari setelah `orderDate`; `vatPercent`.

```tsx
const [branchId, setBranchId] = useState("");
const [customerId, setCustomerId] = useState("");
const [orderDate, setOrderDate] = useState(new Date().toISOString().slice(0, 10));
const [validUntil, setValidUntil] = useState(() => {
  const d = new Date();
  d.setDate(d.getDate() + 2);
  return d.toISOString().slice(0, 10);
});

// ganti area → customer yang terpilih bisa jadi tidak lagi sah di area baru
useEffect(() => {
  setCustomerId("");
  setRecipientName("");
  setRecipientAddress("");
}, [branchId]);

useEffect(() => {
  if (!customerId) return;
  fetchApi<Customer>(`/customers/${customerId}`).then((c) => {
    setRecipientName(c.contactPerson ?? c.name);
    setRecipientAddress(c.address ?? "");
    setCreditLimit(c.creditLimitEnabled ? c.creditLimit : null);
    setTopDays(c.topDays);
  }).catch(() => {});
}, [customerId]);
```

Panel customer menampilkan `creditLimit` dan `topDays`; **saldo ditulis sebagai `"-"`** dengan teks kecil `t('balanceNotBuilt')`.

Baris — tiap baris punya `<ProductCombobox>`, `<ProjectCoopCombobox>` (menampilkan sisa populasi), input `birdCount` dan `avgWeightKg`, tonase hanya-baca, checkbox afkir, `<MatureBirdStandardCombobox>` yang mengisi `minimumWeightKg`, harga rekomendasi hanya-baca, `unitPrice`, `savingsPerKg`, `lineNotes`.

```tsx
// tonase selalu turunan — sama seperti backend (K6)
const tonnage = (line: SalesLine) =>
  line.birdCount && line.avgWeightKg
    ? (Number(line.birdCount) * Number(line.avgWeightKg)).toFixed(2)
    : "";

// harga rekomendasi dimuat saat produk dan customer dua-duanya ada
useEffect(() => {
  if (!customerId || !line.productId) { setRecommended(null); return; }
  fetchApi<RecommendedPrice>(
    `/sales-orders/recommended-price?productId=${line.productId}&customerId=${customerId}`
  ).then(setRecommended).catch(() => setRecommended(null));
}, [customerId, line.productId]);
```

Kalau `recommended.unitPrice` null, tampilkan `t('noRecommendation')`, **bukan angka**.

Validasi klien sebelum kirim:

```tsx
const filled = lines.filter((l) => l.productId && l.sourceProjectCoopId && Number(l.birdCount) > 0);
if (filled.length === 0) {
  toast.error(t('lineRequired'));
  return;
}
if (filled.length !== lines.filter((l) => l.productId || l.sourceProjectCoopId || l.birdCount).length) {
  toast.error(t('lineRequired'));
  return;
}
if (new Date(validUntil) < new Date(orderDate)) {
  toast.error(tc('required', { field: t('validUntil') }));
  return;
}
```

Payload **tidak menyertakan** `totalWeightKg` maupun `totalPrice` — backend yang menghitung, dan `forbidNonWhitelisted` akan menolak 400 kalau ikut terkirim:

```tsx
body: JSON.stringify({
  branchId, customerId, recipientType: "CUSTOMER",
  recipientName: recipientName.trim() || undefined,
  recipientAddress: recipientAddress.trim() || undefined,
  orderDate, validUntil,
  vatPercent: Number(vatPercent || 0),
  notes: notes.trim() || undefined,
  lines: filled.map((l) => ({
    productId: l.productId,
    sourceProjectCoopId: l.sourceProjectCoopId,
    birdCount: Number(l.birdCount),
    avgWeightKg: Number(l.avgWeightKg),
    isCulled: l.isCulled,
    minimumWeightKg: l.minimumWeightKg ?? undefined,
    savingsPerKg: l.savingsPerKg ? Number(l.savingsPerKg) : undefined,
    unitPrice: l.unitPrice ? Number(l.unitPrice) : undefined,
    lineNotes: l.lineNotes?.trim() || undefined,
  })),
})
```

Galat ditangkap dengan memunculkan pesan asli dari API:

```tsx
} catch (error) {
  toast.error(error instanceof Error ? error.message : tc('entityCreateFailed', { entity: t('entity') }));
}
```

- [ ] **Step 2: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`, dengan rute `/sales-orders/new` terdaftar.

- [ ] **Step 3: Smoke browser**

Buka `http://localhost:3005/sales-orders/new`. Buktikan, dan catat hasilnya:
1. Memilih area membuat daftar customer hanya berisi yang beroperasi di area itu; customer yang tidak terdaftar tidak muncul.
2. Memilih customer mengisi nama penerima dan alamat, dan keduanya masih bisa diedit.
3. Plafon dan TOP tampil; saldo tampil sebagai `-` dengan keterangan piutang belum dibangun.
4. Memilih kandang menampilkan sisa populasinya; mengisi ekor dan AVG mengisi tonase otomatis.
5. Memilih standar bobot mengisi bobot minimum.
6. Harga rekomendasi berbunyi "belum ada harga sebelumnya" untuk customer baru.
7. Menyimpan menghasilkan nomor `DO.<kode area>.<urut>`, dan halaman populasi kandang menunjukkan **Tersedia** berkurang sebanyak ekor yang dipesan sementara **Masuk** tidak berubah.
8. Menyimpan dengan ekor melebihi sisa populasi memunculkan toast berisi pesan dari API dan form tetap terbuka.

Hapus pesanan uji sesudahnya.

- [ ] **Step 4: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/sales-orders/new"
git commit -m "feat(sales): structured sales order form"
```

---

### Task 11: Detail dan daftar pesanan

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/sales-orders/[id]/page.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/sales-orders/page.tsx`

**Interfaces:**
- Consumes: tipe Task 9; rute Task 4, 6, 8.
- Produces: tidak ada; halaman daun.

- [ ] **Step 1: Perluas halaman detail**

Header menampilkan nomor DO, tanggal pesan **s/d** masa berlaku, customer, area, status, nama dan alamat penerima, PPN.

Tabel baris menampilkan: Produk, Kandang asal, Ekor, AVG, Tonase, Afkir (badge kalau `isCulled`), Bobot minimum, Tabungan/kg, Harga jual, Total. Baris lama yang `productId`-nya kosong menampilkan `productDescription` apa adanya di kolom Produk dan tanda hubung di kolom lainnya — pesanan sebelum S-B tetap terbaca:

```tsx
const productLabel = (line: StructuredSalesOrderLine) =>
  line.product ? `${line.product.code} — ${line.product.name}` : (line.productDescription ?? "-");
```

- [ ] **Step 2: Perluas daftar**

Kolom: Nomor DO, Tanggal (s/d masa berlaku), Customer, Area, Status. Filter status memakai `StatusBadge` yang sudah ada. Tambahkan tombol **Expire overdue orders** (`t('expireOverdue')`) yang memanggil `POST /sales-orders/expire-overdue` lalu `refetch()`, dan memunculkan toast berisi jumlah yang ditolak:

```tsx
const res = await fetchApi<{ expired: number; failed: number }>("/sales-orders/expire-overdue", {
  method: "POST",
  body: JSON.stringify({}),
});
toast.success(`${res.expired} DO kadaluarsa ditolak`);
refetch();
```

- [ ] **Step 3: Periksa tipe dan build**

Run: `cd breeding-dashboard && npx tsc --noEmit && npm run build`
Expected: `No errors found` lalu `Compiled successfully`.

- [ ] **Step 4: Smoke browser**

Buka `http://localhost:3005/sales-orders`. Buktikan, dan catat hasilnya:
1. Pesanan yang dibuat di Task 10 muncul dengan nomor `DO.…` dan rentang tanggalnya.
2. Detailnya menampilkan kandang asal, AVG, tonase, dan bobot minimum di barisnya.
3. Pesanan lama yang belum terstruktur tetap terbuka tanpa galat dan menampilkan deskripsi teksnya.
4. Menolak pesanan lewat aksi status mengembalikan **Tersedia** di halaman populasi kandang.
5. Tombol Expire overdue orders menghasilkan toast berisi angka.

Bersihkan data uji sesudahnya.

- [ ] **Step 5: Commit**

```bash
cd breeding-dashboard
git add "src/app/(dashboard)/sales-orders"
git commit -m "feat(sales): structured order detail and list"
```

---

### Task 12: Perbarui dokumen tahapan

**Files:**
- Modify: `docs/superpowers/specs/2026-09-27-sales-structured-order-design.md`
- Modify: `docs/superpowers/specs/2026-09-13-sales-module-finding.md`

**Interfaces:**
- Consumes: hasil Task 1-11.
- Produces: tidak ada.

- [ ] **Step 1: Tandai spec sudah diimplementasi**

Ubah baris `**Status:**` di `2026-09-27-sales-structured-order-design.md` menjadi:

```markdown
**Status:** Implemented (2026-09-27) — lihat plan `docs/superpowers/plans/2026-09-27-sales-structured-order.md`
```

- [ ] **Step 2: Tandai S-B selesai di tabel tahapan**

Di `2026-09-13-sales-module-finding.md`, pada baris **S-B**, ganti kolom Status menjadi:

```
Selesai (2026-09-27)
```

- [ ] **Step 3: Periksa**

Run: `cd /Users/alva.e202511001/Desktop/project/breeding && grep -c "Implemented (2026-09-27)" docs/superpowers/specs/2026-09-27-sales-structured-order-design.md && grep -c "Selesai (2026-09-27)" docs/superpowers/specs/2026-09-13-sales-module-finding.md`
Expected: `1` lalu `1`.

- [ ] **Step 4: Commit**

```bash
cd /Users/alva.e202511001/Desktop/project/breeding
git add docs/superpowers/specs
git commit -m "docs(sales): mark structured sales order implemented"
```
