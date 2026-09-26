# Master Data Penjualan — Customer Lengkap & Standarisasi Ayam Besar

**Dibuat:** 2026-09-26
**Status:** Design
**Sumber keputusan:** `docs/stakeholder/2026-09-13-pertanyaan-modul-penjualan-jawaban.pdf`, `docs/superpowers/specs/2026-09-13-sales-module-finding.md`
**Repo terdampak:** `breeding-app` (Prisma, master-data), `breeding-dashboard`
**Tahap:** S-A + S-B2 di [peta tahapan](2026-09-13-sales-module-finding.md#opsi-solusi) — prasyarat untuk pesanan terstruktur (S-B)

---

## Tujuan

Melengkapi master data yang dibutuhkan modul penjualan, supaya tahap berikutnya (pesanan → realisasi → piutang) punya pijakan:

1. `Customer` menyimpan data yang dipakai saat pesanan & realisasi: identitas resmi, plat kendaraan, plafon, TOP, tabungan/kg.
2. Satu customer bisa beroperasi di beberapa area, dan **itu membatasi** di area mana dia boleh dibuatkan pesanan.
3. Ada master **Standarisasi Ayam Besar** sebagai sumber *Bobot Minimum* di form pesanan.

## Jawaban stakeholder yang mengikat

| Kode | Jawaban | Konsekuensi di spec ini |
|---|---|---|
| B5 `Q-CU-5` | (a) Customer hanya bisa pesan di area yang dicentang | `CustomerBranch` M2M **dipakai untuk memfilter**, bukan sekadar catatan |
| B6 `Q-CU-6` | (a) Satu plat per customer cukup | `vehiclePlate` di master; realisasi menampilkannya read-only |
| B2 `Q-CU-2` | TOP dihitung dari **pembayaran terakhir**; approval berlaku sampai customer bayar | `topDays` + `lastPaymentDate` disiapkan di sini; logikanya di S-D |
| B3 `Q-CU-3` | Tabungan milik customer | `savingsPerKg` sebagai nilai default per customer |
| B1 `Q-CU-1` | Over-plafon → antrian approval | `creditLimitEnabled` memisahkan "tanpa plafon" dari "plafon Rp 0" |
| A5 `Q-SO-4` | Bobot Minimum dari **Standarisasi Ayam Besar** | Master baru `MatureBirdStandard` + range |
| B4 `Q-CU-4` | **Tidak dijawab** | `installmentPerKg` **disiapkan kolomnya**, mekanismenya tidak dibangun |

## Non-goal

- Mekanisme pembekuan piutang & pemotongan cicilan (`Q-CU-4` belum dijawab). Hanya kolom nilainya yang ada.
- Perhitungan saldo piutang, tabungan berjalan, dan TOP warning — itu S-D.
- Perubahan pada `SalesOrder` / `SalesOrderLine` — itu S-B.
- Mengisi `lastPaymentDate` secara otomatis; kolom disiapkan, pengisiannya di S-D.
- Master "Parameter Selisih Harga Pasar" — tidak dipakai karena A5 menjawab Harga Rekomendasi = harga terakhir, bukan harga pasar.

---

## 1 · Schema

### `Customer` — kolom baru

```prisma
model Customer {
  // ... kolom yang ada (code, name, contactPerson, phone, email, address,
  //     city, creditLimit, performance, collateral, timestamps)
  registrationBranchId String?   @map("registration_branch_id")
  idCardNumber         String?   @map("id_card_number")
  taxNumber            String?   @map("tax_number")
  vehiclePlate         String?   @map("vehicle_plate")
  creditLimitEnabled   Boolean   @default(false) @map("credit_limit_enabled")
  topDays              Int?      @map("top_days")
  savingsPerKg         Decimal?  @map("savings_per_kg") @db.Decimal(18, 2)
  installmentPerKg     Decimal?  @map("installment_per_kg") @db.Decimal(18, 2)
  lastPaymentDate      DateTime? @map("last_payment_date") @db.Date

  registrationBranch Branch?          @relation("CustomerRegistrationBranch", fields: [registrationBranchId], references: [id])
  operatingBranches  CustomerBranch[]

  @@index([registrationBranchId])
}
```

Semua nullable supaya baris lama tetap valid. `creditLimitEnabled` default `false` — artinya customer lama dianggap **tanpa** pembatasan piutang sampai seseorang menyalakannya, yang sesuai perilaku sekarang (`creditLimit` nullable dan tidak pernah dicek).

### `CustomerBranch` — area operasional

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
```

Bentuk dan perilakunya sama persis dengan `SupplierCategory` yang sudah jalan: baris tautan, bukan entitas domain, jadi tanpa `deletedAt` dan diganti secara replace-all saat update.

`Branch` mendapat dua back-relation: `registeredCustomers Customer[] @relation("CustomerRegistrationBranch")` dan `operatingCustomers CustomerBranch[] @relation("BranchOperatingCustomers")`.

### `MatureBirdStandard` — standarisasi ayam besar

```prisma
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

  standard MatureBirdStandard @relation(fields: [standardId], references: [id])

  @@index([standardId])
  @@map("mature_bird_standard_ranges")
}
```

Meniru bentuk di aplikasi acuan: satu nama standarisasi berisi daftar *Nilai dari – Nilai sampai*. Baris range adalah bagian dari induknya (tidak dirujuk entitas lain), jadi diganti replace-all saat update dan ikut terhapus saat induknya di-soft-delete.

**Yang sengaja tidak dibuat sekarang**: kaitan standarisasi ke produk/customer dan cara memilih range mana yang jadi "Bobot Minimum" untuk satu baris pesanan. Itu keputusan form pesanan (S-B); di sini master-nya dulu supaya datanya bisa diisi.

### Migration

Satu migration: 9 kolom nullable + 1 index di `customers`, tabel `customer_branches`, `mature_bird_standards`, `mature_bird_standard_ranges`. Semua aditif, tanpa backfill.

---

## 2 · Backend

### `CustomerService`

- `CreateCustomerDto` / `UpdateCustomerDto` bertambah: `registrationBranchId?` (`@IsUUID`), `idCardNumber?`, `taxNumber?`, `vehiclePlate?` (`@IsString @MaxLength(50)`), `creditLimitEnabled?` (`@IsBoolean`), `topDays?` (`@IsInt @Min(0)`), `savingsPerKg?`, `installmentPerKg?` (`@Type(() => Number) @IsNumber @Min(0)`), `operatingBranchIds?: string[]` (`@IsArray @IsUUID('4', { each: true })`).
- `create`: `registrationBranchId` dan setiap `operatingBranchIds` divalidasi milik tenant & tidak terhapus (satu `count` + bandingkan jumlah, pola `validateCategoryIds` di `SupplierService`). Tulis customer + `operatingBranches.create[]` dalam `$transaction` yang sudah ada.
- `update`: kalau `operatingBranchIds` dikirim (termasuk `[]`) → replace-all `deleteMany` + `createMany`; kalau tidak dikirim, tautan tidak disentuh.
- `findAll` menerima `QueryCustomerDto extends PaginationDto` dengan `branchId?: string`. Kalau diisi: `where.operatingBranches = { some: { branchId } }`.
- `findAll` / `findOne` include `operatingBranches: { include: { branch: { select: { id, code, name } } } }` dan `registrationBranch: { select: { id, code, name } }`.
- `remove` (soft delete) tidak menyentuh tabel tautan.

Catatan pelajaran dari `SupplierService`: include tautan **harus** menyaring induk yang sudah di-soft-delete (`where: { branch: { deletedAt: null } }`), supaya dialog edit tidak mengirim balik id area yang sudah dihapus dan kena 400. Ini bug yang sempat terjadi di supplier dan diperbaiki di `1adf940`.

### `MatureBirdStandardService` — modul baru di `master-data`

CRUD standar mengikuti `MatureBirdTypeService`: `create`, `findAll` (paginated + search nama), `findOne`, `update`, `remove` (soft delete). `ranges` ikut DTO induk (`@ValidateNested`), ditulis dengan `create[]` saat create dan replace-all saat update. Validasi: minimal satu range, dan `valueTo >= valueFrom` pada setiap baris.

Route: `/api/mature-bird-standards`. Service didaftarkan di `MasterDataModule` (providers + exports, mengikuti 8 service yang sudah ada).

### Seed

- Tiga customer seed lama mendapat `registrationBranchId` + `operatingBranches` ke cabang itu, plus contoh `topDays: 3`, `creditLimitEnabled: true`, `savingsPerKg: 100`, `vehiclePlate`.
- Satu `MatureBirdStandard` contoh ("Standar Broiler") dengan 3 range.

---

## 3 · Frontend

### `/customers`

Dialog bertambah, dikelompokkan supaya tidak jadi satu kolom panjang:

| Grup | Field |
|---|---|
| Identitas | Kode (auto kalau kosong), Nama, Area Pendaftaran (`BranchCombobox`), Alamat, Kota |
| Dokumen | No. KTP, NPWP, Plat Nomor Mobil |
| Kontak | Contact person, Telepon, Email *(tetap — ekstra kita, tidak dibuang)* |
| Syarat & ketentuan | Plafon (switch `creditLimitEnabled`) → Limit Plafon (disabled saat switch mati), TOP Pembayaran (hari), Tabungan/Kg, Cicilan/Kg *(dengan catatan "mekanisme belum aktif")* |
| Operasional | Checkbox daftar cabang = area operasional |

Kolom tabel bertambah: Area Pendaftaran dan badge area operasional. Kolom `performance` & `collateral` tetap.

### `CustomerCombobox`

Tambah prop opsional `branchId?: string`. Kalau diisi → `extra: { branchId }`; kalau kosong → perilaku lama persis. Pemakai sekarang (`sales-orders/new`, `deliveries/new`) tidak berubah di spec ini — penyambungan filter ke form pesanan terjadi di S-B, saat form itu memang dikerjakan ulang.

### `/mature-bird-standards`

Halaman master baru mengikuti pola `/product-categories`: tabel + dialog. Dialog berisi nama + daftar range yang bisa ditambah/hapus baris (dua input angka per baris). Menu masuk ke grup Master Data di sidebar.

### Types & i18n

`types/api.ts`: `Customer` bertambah field baru + `operatingBranches?: { branchId; branch: Pick<Branch,"id"|"code"|"name"> }[]` + `registrationBranch?`; tipe baru `MatureBirdStandard` & `MatureBirdStandardRange`. Semua label baru masuk `messages/en.json` dan `id.json`.

---

## 4 · Pengujian

Repo belum punya test suite service-level; verifikasi mengikuti praktik yang berlaku: `npm run build` + curl untuk backend, `tsc --noEmit` + build + smoke browser untuk frontend.

Backend (curl):
- Create customer tanpa field baru → tetap berhasil (kompatibilitas mundur).
- Create dengan `operatingBranchIds` berisi id cabang tenant lain → 400.
- `PATCH` dengan `operatingBranchIds: []` mengosongkan; tanpa field itu tidak mengubah.
- `GET /customers?branchId=<A>` hanya mengembalikan customer yang beroperasi di A.
- Soft-delete satu cabang → customer yang tertaut tetap bisa di-`PATCH` (regresi yang pernah terjadi di supplier).
- `POST /mature-bird-standards` tanpa range → 400; dengan `valueTo < valueFrom` → 400.

Frontend (browser):
- Dialog customer: switch plafon mati → input limit disabled; simpan & buka lagi → nilai kembali utuh.
- Centang dua area → badge muncul di tabel; hapus centang → badge hilang.
- Halaman standarisasi: tambah 2 range, simpan, edit, hapus satu range.

---

## 5 · Urutan implementasi

1. Schema + migration + seed (satu commit, backend hijau).
2. `CustomerService` + DTO + controller (`branchId` filter, replace-all tautan).
3. `MatureBirdStandard` modul baru.
4. Frontend types + i18n + dialog customer.
5. Frontend halaman standarisasi + prop `branchId` pada `CustomerCombobox`.

Tiap langkah: `npm run build` backend / `tsc --noEmit && npm run build` frontend sebelum commit. Sub-repo di-commit terpisah dengan scope `feat(master-data):`.

## Asumsi terbuka

| Asumsi | Kalau ternyata beda |
|---|---|
| Cicilan/Kg hanya disimpan, tidak dipakai (B4 belum dijawab) | Mekanismenya jadi spec tersendiri; kolom sudah siap |
| Bobot Minimum dipilih dari range standarisasi berdasarkan bobot rata-rata baris pesanan | Kaitan standar↔produk/customer ditentukan di S-B; master tidak berubah |
| Plat nomor cukup satu dan read-only di realisasi (B6=a) | Kalau ternyata per realisasi, tambah kolom di realisasi; master tetap |
