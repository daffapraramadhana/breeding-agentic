# Ledger Populasi Kandang

**Dibuat:** 2026-09-27
**Status:** Design — menunggu plan
**Sumber keputusan:** brainstorming 2026-09-27 (keputusan pengguna dicatat di [Keputusan yang mengikat](#keputusan-yang-mengikat))
**Repo terdampak:** `breeding-app` (Prisma, modul `project`), `breeding-dashboard`
**Tahap:** prasyarat **S-B** pesanan terstruktur di [peta tahapan modul penjualan](2026-09-13-sales-module-finding.md#opsi-solusi)

---

## Tujuan

Form pesanan penjualan (S-B) harus menampilkan **sisa stok** saat penggunanya memilih asal
barang. Untuk asal gudang angka itu sudah ada — `InventoryStock.quantityAvailable`. Untuk
asal **kandang**, yaitu jalur mayoritas karena yang dijual adalah ayam hidup, sistem tidak
punya jawabannya: populasi hanya tercatat sebagai `ProjectChickIn.population` (jumlah DOC
masuk) dan tidak ada satu pun transaksi yang menguranginya.

Spec ini membangun sumber angka itu: **populasi berjalan per siklus pemeliharaan**, dengan
riwayat gerakan yang bisa diaudit.

Bukan tujuan spec ini: form Data Harian yang lengkap (pakan, bobot sampel, air, obat) dan
turunan FCR/IP. Yang dibangun hanya yang menggerakkan jumlah ekor. Form Data Harian penuh
nanti bisa menumpang ledger yang sama tanpa mengubah schema.

## Konteks yang ditemukan di kode

| Temuan | Berkas | Akibat pada desain |
|---|---|---|
| Populasi ayam hidup hanya ada sebagai jumlah DOC masuk | `prisma/schema.prisma` model `ProjectChickIn` | Butuh ledger baru; chick-in jadi gerakan masuk |
| `CoopFloor.population` **bukan** hitungan hidup — dipakai sebagai `population × area = maxPopulation` | `src/modules/farm/farm.service.ts:60` | Kolom itu tidak disentuh dan tidak dipakai sebagai saldo |
| Tidak ada recording harian mortalitas di schema; `FcrStandardDetail.mortality` dan `RecordingDeviation.mortalityPct` adalah tabel master/standar | `prisma/schema.prisma` | Mortalitas harus punya dokumen input sendiri |
| Aplikasi acuan punya Produksi › Data Harian (`data_harians`) dan Produksi › Culling (`cullings`) | [peta modul pembanding](2026-09-13-sales-module-finding.md#peta-modul-pembanding) | Subsistem ini adalah bagian populasi dari dua menu itu |
| Pola ledger + saldo sudah mapan di codebase | `InventoryMovement` + `InventoryStock` | Ditiru bentuknya, bukan bikin pola baru |
| `InventoryStock.quantityAllocated` sudah ada | `prisma/schema.prisma:1501` | Alokasi bukan konsep asing di sistem ini |

## Keputusan yang mengikat

| # | Keputusan | Akibat di spec ini |
|---|---|---|
| K1 | Lingkup **ledger populasi saja** | Pakan, bobot, air, obat, FCR/IP tidak dibangun |
| K2 | Grain = **`ProjectCoop`** (siklus × kandang) | Kandang yang dipakai ulang di siklus berikutnya dapat saldo baru; riwayat antar siklus tidak tercampur |
| K3 | DO aktif **menahan** ekor | Kolom `quantityAllocated` + metode `allocate()`/`release()` |
| K4 | Ada **koreksi manual / opname** | Gerakan `ADJUSTMENT`, wajib beralasan, MANAGER ke atas |
| K5 | Ada **pindah ayam antar kandang** | Dokumen `CoopBirdTransfer`, dua gerakan dalam satu transaksi |
| K6 | **Tidak** memisahkan afkir-dibuang dari afkir-dijual | Satu gerakan `CULLING`; sejalan dengan jawaban stakeholder A6 "stok tidak dipisah" |
| K7 | **Tidak** ada backfill saldo awal dari chick-in lama | Migrasi tidak menerbitkan gerakan untuk baris lama; lihat [Data lama](#data-lama) |

## Arsitektur

### Tabel saldo — `CoopPopulation`

Satu baris per project-coop. Denormalisasi saldo supaya form pesanan tidak perlu menjumlahkan
ledger setiap kali dibuka, persis alasan `InventoryStock` ada.

```prisma
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
```

Ekor adalah bilangan bulat, jadi `Int` — bukan `Decimal` seperti stok barang.

### Tabel ledger — `CoopPopulationMovement`

Append-only. Tidak pernah diubah, tidak pernah dihapus.

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
```

`MovementType { IN, OUT }` yang sudah ada dipakai ulang. `BirdMovementSource` dibuat baru
karena `MovementSource` yang ada berisi sumber dokumen barang, bukan ayam.

### Dokumen input — `CoopDepletionEntry`

Satu entri per kandang per tanggal. Inilah bagian populasi dari "Data Harian" di aplikasi acuan.

```prisma
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
```

### Dokumen input — `CoopBirdTransfer`

```prisma
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

Nomor dibuat oleh `ReferenceNumberGenerator` dengan prefiks `CBT`, mengikuti pola dokumen lain.

Kolom `deletedAt` tetap ada mengikuti konvensi repo, tetapi **tidak ada rute yang
menghapusnya** — lihat [Cara saldo bergerak](#cara-saldo-bergerak).

### Relasi balik di `ProjectCoop`

Model `ProjectCoop` yang sudah ada perlu ditambah relasi balik untuk keempat tabel di atas:
`population CoopPopulation?`, `populationMovements CoopPopulationMovement[]`,
`depletionEntries CoopDepletionEntry[]`, serta `birdTransfersOut` dan `birdTransfersIn`
dengan nama relasi `BirdTransferSource` dan `BirdTransferDestination`. Tidak ada kolom
`ProjectCoop` yang diubah.

## Cara saldo bergerak

Semua penulisan gerakan **dan** pembaruan saldo terjadi di dalam satu `$transaction`.
Tidak ada jalur lain yang boleh menyentuh `CoopPopulation` — satu service memegang
pintu itu, supaya `quantityAfter` di ledger selalu cocok dengan saldo.

| Peristiwa | Gerakan yang ditulis |
|---|---|
| Chick-in dibuat | `IN` / `CHICK_IN` sejumlah populasi |
| Chick-in diubah | gerakan selisih: `IN` kalau naik, `OUT` kalau turun, tetap bersumber `CHICK_IN` |
| Chick-in dihapus | `OUT` / `CHICK_IN` sejumlah populasi terakhir |
| Entri deplesi dibuat | `OUT` / `MORTALITY` dan/atau `OUT` / `CULLING`; yang bernilai 0 tidak menulis gerakan |
| Entri deplesi diubah | gerakan **selisih** per jenis — mortalitas 10 → 7 menulis `IN` / `MORTALITY` sebanyak 3 |
| Entri deplesi dihapus | gerakan pembalik penuh per jenis |
| Pindah ayam dibuat | `OUT` / `TRANSFER_OUT` di asal **dan** `IN` / `TRANSFER_IN` di tujuan, satu transaksi |
| Koreksi manual | `IN` atau `OUT` / `ADJUSTMENT` sesuai arah, `notes` wajib |
| Realisasi penjualan | `OUT` / `SALES_REALIZATION` — **disediakan metodenya, dipanggil di S-C** |

Ledger tidak pernah diedit. Koreksi selalu berupa gerakan baru, sehingga saldo bisa
direkonstruksi dari ledger dan riwayat pembetulan tetap terlihat — sama seperti buku besar.

Dokumen `CoopBirdTransfer` tidak bisa diedit atau dihapus setelah dibuat; pembetulan
dilakukan lewat koreksi manual. Ini menghindari pembalikan dua sisi yang mudah salah,
dan koreksi manual memang sudah ada sebagai jalan keluar (K4).

## Alokasi

```
quantityAvailable = quantityOnHand − quantityAllocated
```

Spec ini membangun **mekanismenya saja**: kolom `quantityAllocated` beserta metode
`allocate(projectCoopId, qty, doc)` dan `release(projectCoopId, qty, doc)` di service.
Belum ada pemanggilnya sampai S-B dibangun.

Batas itu disengaja. Yang mana DO menahan ekor, kapan tahanannya lepas (ditolak,
dibatalkan, kadaluarsa) adalah logika pesanan, dan itu milik S-B. Membangunnya di sini
berarti menebak bentuk DO yang belum dirancang.

Alokasi tidak menulis gerakan ledger — dia bukan perpindahan ayam, hanya pemesanan
terhadap saldo yang sama. Ledger tetap mencatat kejadian fisik saja.

## Aturan penolakan

| Kondisi | Respons |
|---|---|
| Gerakan `OUT` melebihi `quantityOnHand` | 400 — "Populasi kandang tidak mencukupi" |
| `allocate()` melebihi `quantityAvailable` | 400 — "Sisa populasi tidak mencukupi" |
| Entri deplesi kedua untuk kandang + tanggal yang sama | 409 — pelanggaran `@@unique` dipetakan jadi pesan manusiawi, bukan string Prisma |
| Nomor transfer bentrok | 409 — sama perlakuannya |
| Pindah ke project-coop yang sama dengan asalnya | 400 |
| `mortalityCount` / `cullingCount` / `quantity` negatif atau bukan bilangan bulat | 400 di DTO |
| Koreksi manual tanpa alasan | 400 |
| `projectCoopId` milik tenant lain atau sudah dihapus | 400 |

Semua query difilter `tenantId`; penghapusan lewat `deletedAt`. Peran: **STAFF** boleh
mengisi entri deplesi; **MANAGER ke atas** untuk koreksi manual dan pindah ayam.

Pelanggaran `@@unique` dipetakan ke 409 berpesan manusiawi, bukan dibiarkan bocor sebagai
P2002 — pelajaran dari review master data penjualan, di mana kunci unik yang bertahan
melewati soft delete memunculkan 500 berisi string Prisma.

## API

| Metode | Rute | Peran | Isi |
|---|---|---|---|
| GET | `/coop-populations` | semua | saldo per project-coop; filter `projectId`, `coopId`, `branchId`, `farmId`, `includeInactive`; menyertakan nama kandang/farm/area |
| GET | `/coop-populations/:projectCoopId` | semua | satu saldo |
| GET | `/coop-populations/:projectCoopId/movements` | semua | ledger, terbaru dulu, paginasi |
| POST | `/coop-populations/:projectCoopId/adjustments` | MANAGER+ | koreksi manual, `{ quantity: number, direction: 'IN' \| 'OUT', reason: string, movementDate: string }` — `reason` tersimpan di `notes` gerakan |
| GET/POST/PATCH/DELETE | `/coop-depletions` | STAFF+ | entri deplesi harian |
| GET/POST | `/coop-bird-transfers` | MANAGER+ | pindah ayam |

Respons tetap dibungkus `{ data, statusCode, timestamp }` oleh `TransformInterceptor`.

### Arah baca daftar

`GET /coop-populations` membaca dari **`ProjectCoop`** dan menempelkan saldo sebagai relasi
opsional, bukan sebaliknya. Baris saldo dibuat malas — baru ada saat gerakan pertama ditulis —
sehingga kandang yang belum pernah bergerak tetap muncul dengan nilai nol, bukan hilang dari
daftar. Ini yang membuat kandang lama terdampak K7 tetap terlihat dan bisa dibetulkan lewat
koreksi manual.

## Frontend

| Halaman | Isi |
|---|---|
| **Populasi Kandang** | Daftar saldo kandang yang siklusnya aktif — kolom kandang, farm, area, masuk, terpakai, tersedia. Filter area/farm/proyek. Klik baris → panel riwayat gerakan (tanggal, jenis, sumber, jumlah, saldo sesudah, catatan). |
| **Deplesi Harian** | Form: kandang, tanggal, mortalitas, afkir, catatan. Daftar entri dengan filter tanggal dan kandang. Entri yang sudah ada untuk tanggal itu dibuka sebagai edit, bukan ditolak diam-diam. |
| **Pindah Ayam** | Form: kandang asal, kandang tujuan, jumlah, tanggal, catatan. Sisa populasi asal tampil begitu kandang dipilih. |
| **Dialog Koreksi** | Dari halaman populasi: arah (tambah/kurang), jumlah, alasan wajib. |

Kunci i18n ditambahkan ke `messages/en.json` **dan** `messages/id.json`, termasuk namespace
`navigation` yang dibaca sidebar — tanpa itu judul menu muncul sebagai kunci mentah.

## Data lama

Migrasi bersifat aditif: empat tabel baru, satu enum baru, tidak ada kolom yang diubah atau
dijatuhkan, **tidak ada `UPDATE`**. Aman dijalankan di staging tanpa menyentuh data yang ada.

Konsekuensi K7 yang harus diterima sadar: setiap `ProjectChickIn` yang dibuat sebelum migrasi
tidak menghasilkan gerakan, jadi kandangnya terbaca **0** ekor. Kandang begitu baru bisa
dipakai menjual setelah dibetulkan lewat koreksi manual. Seed diperbarui supaya data
pengembangan tetap koheren — chick-in di seed melewati jalur baru sehingga saldonya terisi.

## Verifikasi

Repo ini belum punya jest yang jalan: satu-satunya berkas spesifikasi
(`breeding-app/src/app.controller.spec.ts`) gagal parse sejak sebelum cabang mana pun, jadi
menjalankannya tidak membuktikan apa pun. Verifikasi memakai pola yang sudah dipakai dua plan
terakhir: skrip curl per tugas yang **dijalankan sampai gagal dulu** sebelum implementasi,
lalu dijalankan lagi sesudahnya, ditambah smoke browser untuk halaman.

Yang wajib dibuktikan, bukan sekadar diasumsikan:

1. Saldo sesudah serangkaian gerakan sama dengan `quantityAfter` gerakan terakhir.
2. Entri deplesi yang diubah menghasilkan gerakan selisih, bukan gerakan penuh yang kedua.
3. Gerakan `OUT` yang melebihi saldo ditolak dan **tidak** menyisakan baris ledger.
4. Pindah ayam yang gagal di sisi tujuan tidak menyisakan gerakan di sisi asal.
5. Dua entri deplesi untuk kandang + tanggal yang sama menghasilkan 409 berpesan manusiawi.
6. `allocate()` melebihi `quantityAvailable` ditolak; `release()` mengembalikannya utuh.
7. Kandang milik tenant lain tidak terbaca lewat rute mana pun.

Memperbaiki setup jest adalah pekerjaan tersendiri dan **tidak** termasuk spec ini.

## Yang dibuka spec ini

Setelah ini selesai, **S-B** punya pijakan: `quantityAvailable` per kandang untuk ditampilkan
sebagai sisa stok, dan `allocate()`/`release()` untuk menahan ekor. **S-C** punya
`SALES_REALIZATION` untuk memotong stok saat timbang.
