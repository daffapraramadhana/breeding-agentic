# Realisasi DO (S-C)

**Dibuat:** 2026-09-27
**Status:** Design — menunggu plan
**Sumber keputusan:** `docs/stakeholder/2026-09-13-pertanyaan-modul-penjualan-jawaban.pdf`, [finding modul penjualan §S-2](2026-09-13-sales-module-finding.md#s-2--realisasi-do), brainstorming 2026-09-27
**Repo terdampak:** `breeding-app` (Prisma, modul `sales`), `breeding-dashboard`
**Tahap:** **S-C** di [peta tahapan](2026-09-13-sales-module-finding.md#opsi-solusi) — prasyaratnya, [ledger populasi](2026-09-27-coop-population-ledger-design.md) dan [pesanan terstruktur](2026-09-27-sales-structured-order-design.md), sudah selesai

---

## Tujuan

Pesanan menjanjikan ayam; realisasi menyerahkannya. Ini peristiwa yang **mengurangi stok ayam** dan jadi dasar faktur — dan sampai sekarang belum ada entitasnya sama sekali.

Sebuah DO direalisasi dalam beberapa pengambilan: truk datang, ayam dihitung dan ditimbang, slip timbang dicatat. Spec ini membangun dokumen itu, menghubungkannya ke kandang yang benar, memotong populasinya, dan melepaskan janji yang sudah ditunaikan.

Bukan tujuan spec ini: faktur (`SalesInvoice` sudah 1:1 ke DO — milik S-D), piutang dan tabungan (S-D), pembekuan piutang dan cicilan (S-E, `Q-CU-4` belum dijawab), pengembangan `Delivery` (A8=c).

## Keputusan yang mengikat

Ketiga keputusan pertama didelegasikan oleh pemilik produk pada 2026-09-27 dengan instruksi mengambil praktik terbaik. Dasarnya dicatat agar bisa ditinjau ulang.

| # | Keputusan | Dasar |
|---|---|---|
| R1 | **Satu dokumen realisasi per DO, banyak baris** — `@@unique` pada `salesOrderId` | Menutup konflik `K-S1`. Layar acuan **satu** ("Modify Data Realisasi DO") dengan satu header dan banyak baris; contoh DO.Budidaya.000001 yang punya "2 realisasi" adalah dua **baris** pengambilan berbeda DTPS. Jawaban A1 "selalu satu kali" dan data yang diamati ternyata sejalan, bukan bertentangan. |
| R2 | **`dtpsNumber` wajib dan unik per tenant** | Menutup sub-pertanyaan `A3`. Baris realisasi adalah peristiwa timbang; timbang tanpa nomor slipnya tidak bisa ditelusuri. Mengikuti preseden `D1`, di mana nomor surat jalan disalin dari dokumen fisik dan wajib sebelum berangkat. |
| R3 | **Tidak ada kebijakan baru untuk kandang kelebihan alokasi** — realisasi dibatasi stok nyata | Menolak deplesi berarti menolak mencatat kenyataan. Membatalkan DO tertua otomatis mengambil keputusan komersial sebagai efek samping entri data. Yang tersisa adalah prinsip yang sudah berlaku: alokasi itu janji, ledger itu kebenaran, realisasi menyelesaikan janji terhadap kebenaran. |
| R4 | **`avgWeightKg` dihitung backend dari `totalWeightKg ÷ birdCount`** | Arahnya terbalik dari pesanan. Di DO, AVG × Qty = Tonase — rata-rata direncanakan. Di realisasi, truk ditimbang dan ayam dihitung, rata-ratanya turunan. Cerminan aturan K6 di S-B. |
| R5 | **Realisasi boleh melebihi pesanan, tanpa diblok dan tanpa kanal peringatan baru di API** | A2=b. Peringatannya dihitung di form dari total pesanan versus total realisasi — datanya sudah ada di layar. |
| R6 | **Cicilan dan biaya admin disimpan tapi tidak diposting** | `Q-CU-4` (mekanisme cicilan) dan `Q-SO-8` (perlakuan akuntansi biaya admin) belum dijawab. Perlakuan sama seperti `installmentPerKg` di S-A: kolomnya disiapkan, mekanismenya tidak. |

## Jawaban stakeholder yang dipakai

| Kode | Jawaban | Akibat di spec ini |
|---|---|---|
| A1 `Q-SO-1` | (b) Selalu satu kali | Header 1:1 ke DO (R1) |
| A2 `Q-SO-1` | (b) Boleh melebihi, dengan peringatan | Tidak diblok; peringatan di form (R5) |
| A3 `Q-SO-2` | DTPS dari Numerator Form Timbangan, unik | `dtpsNumber` wajib, unik per tenant, input manual — numeratornya di luar sistem ini (R2) |
| A4 `Q-SO-3` | Radio 2 pilihan: Timbang Kandang / Timbang Kirim | Enum `WeighingLocation { ORIGIN_COOP, DESTINATION_CUSTOMER }` di header |
| A6 `Q-SO-5` | Harga afkir berbeda, stok tidak dipisah | `isCulled` per baris, harga diketik manual |
| B6 `Q-CU-6` | Satu plat per customer cukup | `vehiclePlate` disalin dari master saat realisasi |
| A8 `Q-SO-7` | Bakul jemput sendiri | `Delivery` tidak disentuh |

## Perubahan data

Seluruhnya aditif: satu enum baru, dua tabel baru, satu relasi balik di `SalesOrder` dan satu di `ProjectCoop`. Tidak ada kolom lama yang diubah.

### Enum

```prisma
enum WeighingLocation {
  ORIGIN_COOP          // Timbang Kandang
  DESTINATION_CUSTOMER // Timbang Kirim
}
```

### `SalesRealization` — header

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
```

Tidak punya nomor sendiri: dokumen ini dikenali lewat DO-nya, sama seperti di aplikasi acuan. `vehiclePlate` adalah **salinan**, bukan kunci asing — plat yang berubah di master tidak boleh mengubah dokumen yang sudah ditandatangani.

### `SalesRealizationLine` — satu baris per pengambilan

```prisma
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

`tenantId` sengaja diduplikasi dari header. Menurunkannya lewat relasi lebih "bersih", tetapi A3 menyatakan nomor DTPS **unik**, dan aturan keunikan yang penting harus ditegakkan basis data — pemeriksaan di service bisa dilewati dua permintaan bersamaan, yang persis kelas kesalahan yang ditemukan di dua review terakhir. Kolom ini diisi dari header saat baris dibuat dan tidak pernah berubah. Preseden yang sama sudah ada di repo: `CoopPopulationMovement` dan `CoopDepletionEntry` keduanya membawa `tenantId` walaupun bisa diturunkan lewat `projectCoop.project`.

Pelanggarannya dipetakan ke 409 berpesan manusiawi, bukan dibiarkan bocor sebagai P2002.

Baris tidak punya `deletedAt`: menghapus baris realisasi membalik gerakan populasinya, dan menyimpan bangkainya hanya membuat total salah. Riwayatnya hidup di ledger populasi, yang memang append-only.

### Relasi balik

`SalesOrder` dapat `realization SalesRealization?`. `ProjectCoop` dapat `salesRealizationLines SalesRealizationLine[]`.

## Alur

```
DO (APPROVED …) ──► buat realisasi (header)
                      │
                      └─► tambah baris ──► OUT / SALES_REALIZATION sebesar ekornya
                                            └─► lepas alokasi sebanyak
                                                min(ekor baris, sisa yang masih ditahan
                                                    DO ini di kandang tersebut)
```

Setiap baris ditulis dalam **satu transaksi**: gerakan populasi dulu, pelepasan alokasi sesudahnya. Urutan itu disengaja — gerakan `OUT` adalah pemeriksaan fisik yang menentukan, dan kalau ayamnya tidak ada seluruh transaksi batal sebelum alokasi tersentuh.

Menghapus baris membalik urutannya: gerakan `IN` dulu, baru alokasi ditahan kembali. Kebalikannya bisa gagal menahan karena ketersediaan belum naik, dan barisnya jadi tidak bisa dihapus.

### Sisa yang masih ditahan

Jumlah yang dilepas per baris adalah `min(ekor baris, sisa tahanan DO di kandang itu)`, di mana sisa tahanan = total ekor baris **pesanan** di kandang tersebut dikurangi total ekor baris **realisasi** yang sudah tercatat di kandang yang sama.

Ini yang membuat R5 bekerja tanpa perlakuan khusus: kelebihan realisasi tidak punya alokasi untuk dilepas, jadi `min` menghasilkan nol dan tidak ada yang perlu dihitung.

Jumlah yang benar-benar dilepas **disimpan di barisnya** sebagai `allocationReleased`. Saat baris dihapus, yang ditahan kembali adalah angka itu, bukan hasil hitung ulang dari total. Menghitung ulang berarti menurunkan angka lama dari keadaan baru, dan setiap baris lain yang berubah di antaranya membuat hasilnya meleset — kelas kesalahan yang sama dengan snapshot baris basi yang ditemukan di review S-B.

## Aturan penolakan

| Kondisi | Respons |
|---|---|
| Ekor melebihi stok kandang | 400 — pesan dari `applyMovement`, "Populasi kandang tidak mencukupi" |
| `dtpsNumber` sudah dipakai di tenant ini | 409 — "Nomor DTPS ini sudah dipakai" |
| Realisasi kedua untuk DO yang sama | 409 — "Pesanan ini sudah punya realisasi" |
| Status DO di luar `APPROVED`, `REALIZATION_APPROVAL`, `REALIZING`, `REALIZING_DO_LIMIT` | 409 — "Pesanan ini belum siap direalisasi" |
| DO milik tenant lain, atau kandang milik tenant lain | 400 |
| `birdCount` atau `totalWeightKg` nol, negatif, atau bukan angka wajar | 400 di DTO; `birdCount` wajib bilangan bulat positif |
| Kandang yang tidak ada di baris pesanan DO ini | 400 — realisasi hanya boleh dari kandang yang dijanjikan |
| Harga, diskon, biaya admin, tabungan negatif | 400 di DTO |

Semua query difilter `tenantId`; header pakai `deletedAt`.

Membuat atau mengubah realisasi **tidak** menggerakkan status DO. Transisi status tetap milik aksi status yang sudah ada dan matriksnya — realisasi memeriksa status, tidak mengubahnya. Memisahkan keduanya berarti satu peristiwa tidak diam-diam memicu peristiwa lain.

Pemeriksaan "kandang harus ada di pesanan" layak dijelaskan: tanpa itu, seseorang bisa merealisasi DO dari kandang mana pun dan memotong stok yang tidak pernah dijanjikan. Kalau di lapangan ternyata ayam sering diambil dari kandang pengganti, aturan ini yang pertama dilonggarkan — dan itu perubahan satu baris.

## API

| Metode | Rute | Peran | Isi |
|---|---|---|---|
| GET | `/sales-orders/:id/realization` | semua | header + baris + total; 404 kalau belum ada |
| POST | `/sales-orders/:id/realization` | STAFF+ | buat header |
| PATCH | `/sales-realizations/:id` | STAFF+ | ubah header (lokasi timbang, plat, catatan, tanggal) |
| DELETE | `/sales-realizations/:id` | MANAGER+ | hapus realisasi beserta seluruh barisnya, membalik semua gerakan |
| POST | `/sales-realizations/:id/lines` | STAFF+ | tambah baris |
| PATCH | `/sales-realization-lines/:id` | STAFF+ | ubah baris — dibalik lalu diterapkan ulang |
| DELETE | `/sales-realization-lines/:id` | STAFF+ | hapus baris, membalik gerakannya |

Respons dibungkus `{ data, statusCode, timestamp }` oleh `TransformInterceptor`. Peran ditulis eksplisit tiga-tiganya (`SUPER_ADMIN, TENANT_ADMIN, MANAGER`) karena `RolesGuard` mencocokkan keanggotaan persis tanpa hierarki.

Mengubah baris diperlakukan sebagai **balik lalu terapkan ulang** dalam satu transaksi, bukan sebagai selisih. Selisih pada empat besaran sekaligus (ekor, kandang, tonase, harga) lebih mudah salah daripada dibalik utuh, dan ledger populasi memang dirancang menerima gerakan pembalik.

## Frontend

| Bagian | Isi |
|---|---|
| Akses | Tombol di detail pesanan: "Realisasi" — membuka halaman realisasi DO itu |
| Header | DO, tanggal pesan s/d berlaku, customer, alamat, nama penerima — semua read-only dari DO. Yang bisa diisi: tanggal realisasi, **Lokasi timbang** (radio: Timbang Kandang / Timbang Kirim), plat yang terisi otomatis dari customer tapi bisa diedit, catatan |
| Baris | Tanggal ambil; kandang asal **dibatasi kandang yang ada di pesanan**, dengan sisa stoknya; No. DTPS; Qty; Tonase; AVG read-only terhitung; Afkir; Harga jual/kg; Diskon/kg; Biaya admin/kg; Tabungan/kg; **Cicilan/kg dimatikan** dengan keterangan bahwa mekanismenya belum dibangun; Total read-only |
| Footer | Total harga realisasi, total diskon, PPN dari DO, total biaya admin, total tabungan |
| Peringatan | Kalau total ekor atau tonase realisasi melebihi pesanan, peringatan terlihat di footer — tidak memblokir penyimpanan (A2) |

Kunci i18n masuk ke `messages/en.json` **dan** `id.json`, termasuk namespace `navigation` kalau ada entri menu baru.

## Asumsi yang belum terverifikasi

| Asumsi | Kalau salah |
|---|---|
| DTPS = "Data Timbangan PS", nomornya dari Numerator Form Timbangan di luar sistem ini | Kalau ternyata harus dibangkitkan sistem, tambah satu deret di `ReferenceNumberGenerator` dan bidangnya jadi read-only |
| Realisasi hanya boleh dari kandang yang ada di pesanan | Kalau ayam boleh diambil dari kandang pengganti, satu pemeriksaan dihapus |
| Toggle "Timbang Kandang" di acuan adalah pilihan lokasi timbang, bukan penarik data recording | Finding menandainya "belum terverifikasi fungsinya"; A4 mengonfirmasi ini radio dua pilihan, yang mendukung pembacaan ini. Kalau ternyata dia menarik hasil timbang dari produksi, itu fitur tambahan, bukan perubahan bentuk data |

## Verifikasi

Repo belum punya jest yang jalan (`breeding-app/src/app.controller.spec.ts` gagal parse sejak sebelum cabang mana pun). Verifikasi memakai pola tiga plan terakhir: skrip curl per tugas yang **dijalankan sampai gagal dulu** sebelum implementasi, lalu lagi sesudahnya, ditambah smoke browser.

Yang wajib dibuktikan, bukan diasumsikan:

1. Menambah baris memotong stok kandang tepat sebanyak ekornya, sekali.
2. Menambah baris melepas alokasi DO tepat sebanyak yang masih ditahannya, tidak lebih.
3. Baris yang ekornya melebihi stok ditolak dan **tidak menyisakan gerakan** di ledger.
4. Menghapus baris memulihkan stok **dan** menahan kembali persis `allocationReleased`, bukan hasil hitung ulang.
5. Mengubah baris membalik yang lama lalu menerapkan yang baru; selisih saldonya benar.
6. `avgWeightKg` tidak pernah berbeda dari `totalWeightKg ÷ birdCount`, termasuk saat klien mengirim nilai yang bertentangan.
7. DTPS kembar di tenant yang sama ditolak 409, termasuk lintas realisasi.
8. Realisasi kedua untuk DO yang sama ditolak 409.
9. Realisasi yang melebihi pesanan diterima, dan alokasi yang dilepas berhenti di nol — tidak negatif.
10. Kandang yang tidak ada di pesanan ditolak.
11. Dua permintaan bersamaan pada baris yang sama tidak memotong stok dua kali.
12. Kandang dan pesanan milik tenant lain ditolak di setiap rute.

## Yang dibuka spec ini

Setelah ini selesai, **S-D** punya angka yang dibutuhkan faktur per DO: tonase dan harga yang benar-benar terserah, bukan yang direncanakan, beserta diskon, biaya admin, dan tabungan per kg.
