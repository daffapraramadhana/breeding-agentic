# Pesanan Penjualan Terstruktur (S-B)

**Dibuat:** 2026-09-27
**Status:** Design — menunggu plan
**Sumber keputusan:** `docs/stakeholder/2026-09-13-pertanyaan-modul-penjualan-jawaban.pdf`, [finding modul penjualan](2026-09-13-sales-module-finding.md), brainstorming 2026-09-27
**Repo terdampak:** `breeding-app` (Prisma, modul `sales`, modul `project`), `breeding-dashboard`
**Tahap:** **S-B** di [peta tahapan](2026-09-13-sales-module-finding.md#opsi-solusi) — prasyaratnya, [ledger populasi kandang](2026-09-27-coop-population-ledger-design.md), sudah selesai

---

## Tujuan

Hari ini baris pesanan penjualan adalah teks bebas. `SalesOrderLine` menyimpan
`productCode` dan `productDescription` sebagai string, tanpa kunci asing ke produk dan
tanpa asal stok; form frontend-nya mengetik deskripsi produk dengan tangan. Akibatnya
pesanan tidak bisa dihubungkan ke stok mana pun, tidak bisa divalidasi, dan tidak bisa
jadi dasar realisasi.

Spec ini membuat baris pesanan menunjuk produk dan kandang asal yang nyata, menahan ekor
yang dijanjikannya, dan membawa semua angka yang dipakai saat timbang: rata-rata bobot,
tonase, penanda afkir, tabungan per kg, dan bobot minimum yang disepakati.

Bukan tujuan spec ini: realisasi (S-C), piutang dan tabungan (S-D), faktur gabungan
(tidak dibangun sama sekali per C1=b), pengembangan `Delivery` (A8=c), dan penjualan ke
peternak — lihat K1.

## Keputusan yang mengikat

| # | Keputusan | Dasar |
|---|---|---|
| K1 | Hanya jalur **customer / ayam besar**; `RecipientType.BREEDER` dibiarkan apa adanya | Seluruh 19 jawaban stakeholder menyangkut customer. Satu-satunya pertanyaan yang menyebut peternak, `Q-SO-8`, justru tidak dijawab. |
| K2 | DO **menahan ekor sejak dibuat** (`PENDING_APPROVAL`), bukan sejak disetujui | Tooltip Saldo Customer di aplikasi acuan menghitung "DO pesanan belum realisasi **dan DO request**" |
| K3 | Kadaluarsa dijalankan **job harian** memakai `@nestjs/schedule` | A7=a "otomatis ditolak sistem"; dan karena K2, alokasi harus benar-benar dilepas — bukan sekadar terlihat ditolak |
| K4 | Baris **hanya boleh berasal dari kandang** | `InventoryStock.quantityAllocated` tidak pernah ditulis kode mana pun (kolom mati); finding mencatat "untuk ayam hidup asalnya kandang" |
| K5 | Harga rekomendasi = harga jual terakhir **per produk per customer** | Menutup sub-pertanyaan A5 yang terbuka |
| K6 | `totalWeightKg` **dihitung ulang backend** dari `birdCount × avgWeightKg` | Acuan menampilkan tiga kotak yang saling terikat; menyimpan ketiganya apa adanya mengundang ketidakcocokan |
| K7 | Bobot minimum **disimpan sebagai salinan** di baris | DO harus tetap utuh kalau master Standarisasi berubah. Mekanisme persisnya di acuan belum diverifikasi — lihat [Asumsi](#asumsi-yang-belum-terverifikasi) |
| K8 | Master "Umur Berlaku DO" **tidak dibangun**; `validUntil` jadi input | Master itu milik S-8 master pendukung, bukan S-B |

## Jawaban stakeholder yang dipakai

| Kode | Jawaban | Akibat di spec ini |
|---|---|---|
| A5 `Q-SO-4` | Harga Rekomendasi dari harga terakhir; Bobot Minimum dari Standarisasi Ayam Besar; Harga Jual bebas | `unitPrice` tanpa validasi minimum; rekomendasi dihitung, bukan disimpan; `minimumWeightKg` disalin |
| A6 `Q-SO-5` | Harga afkir berbeda — lebih murah; stok tidak dipisah | `isCulled` per baris, harga diketik manual, alokasi tetap dari kolam yang sama |
| A7 `Q-SO-6` | DO lewat rentang otomatis ditolak | `validUntil` + job harian |
| B5 `Q-CU-5` | Customer hanya bisa pesan di area yang dicentang | Dropdown customer di form disaring `CustomerBranch` |
| B1 `Q-CU-1` | Over-plafon masuk antrian approval | Plafon **ditampilkan** di sini; mekanisme approval-nya S-D |
| C1 `Q-PAY-1` | Faktur per DO | Tidak ada perubahan struktur — `SalesInvoice` sudah 1:1 |

## Perubahan data

Seluruh perubahan bersifat **aditif**: kolom baru semuanya nullable, tidak ada kolom
lama yang diubah atau dijatuhkan. `DeliveryLine.salesOrderLineId` tidak tersentuh, jadi
modul pengiriman yang sudah ada tetap jalan.

### `SalesOrder` — empat kolom header

```prisma
  recipientName    String?   @map("recipient_name")
  recipientAddress String?   @map("recipient_address") @db.Text
  validUntil       DateTime? @map("valid_until") @db.Date
  vatPercent       Decimal?  @default(0) @map("vat_percent") @db.Decimal(5, 2)
```

### `SalesOrderLine` — enam kolom baris

```prisma
  productId           String?  @map("product_id")
  sourceProjectCoopId String?  @map("source_project_coop_id")
  isCulled            Boolean  @default(false) @map("is_culled")
  avgWeightKg         Decimal? @map("avg_weight_kg") @db.Decimal(18, 4)
  savingsPerKg        Decimal? @map("savings_per_kg") @db.Decimal(18, 2)
  minimumWeightKg     Decimal? @map("minimum_weight_kg") @db.Decimal(18, 4)

  product           Product?     @relation(fields: [productId], references: [id])
  sourceProjectCoop ProjectCoop? @relation(fields: [sourceProjectCoopId], references: [id])
```

`productCode` dan `productDescription` **dipertahankan** supaya pesanan lama tetap
terbaca. Keduanya nullable dan tidak lagi diisi pesanan baru.

Kolom di schema nullable karena data lama tidak punya nilainya; **DTO mewajibkan**
`productId`, `sourceProjectCoopId`, `birdCount`, dan `avgWeightKg` untuk setiap baris
pesanan baru. Nullability adalah soal riwayat, bukan izin bagi pesanan baru untuk kosong.

### Relasi balik di `ProjectCoop`

`ProjectCoop` perlu `salesOrderLines SalesOrderLine[]`. Tidak ada kolomnya yang diubah.

## Siklus hidup dan alokasi

```
create ──► PENDING_APPROVAL  (allocate semua baris)
              │
              ├─► APPROVED ──► (S-C: realisasi melepas + menulis SALES_REALIZATION)
              ├─► REJECTED   (release)
              ├─► CANCELLED  (release)
              └─► kadaluarsa ──► REJECTED (release, oleh job harian)
```

Pembuatan DO menahan ekor per baris dari kandangnya lewat
`CoopPopulationService.allocate(tx, …)`, **di dalam transaksi yang sama** dengan
pembuatan pesanannya. Kalau baris ketiga gagal, dua tahanan pertama ikut dibatalkan —
inilah sebabnya `allocate` dibuat menerima `tx`.

Pengubahan DO yang masih `PENDING_APPROVAL` melepas seluruh alokasi lama lalu menahan
yang baru, juga dalam satu transaksi. Status selain itu tidak boleh mengubah baris.

**Batas terhadap S-C:** spec ini hanya melepas alokasi pada status akhir yang dimilikinya
sendiri — `REJECTED` dan `CANCELLED`. Realisasi adalah milik S-C: dia yang melepas
alokasi sekaligus menulis gerakan `SALES_REALIZATION`, dalam satu transaksi. S-B tidak
menulis gerakan populasi sama sekali.

### Aturan penolakan

| Kondisi | Respons |
|---|---|
| Baris tanpa `productId` atau `sourceProjectCoopId` | 400 di DTO |
| Kandang milik tenant lain | 400 — `resolveProjectCoop` sudah menanganinya |
| Customer tidak terdaftar di area pesanan (B5) | 400 — "Customer tidak beroperasi di area ini" |
| `birdCount` melebihi `quantityAvailable` kandang | 400 — pesan dari `allocate`, "Sisa populasi tidak mencukupi" |
| `birdCount` atau `avgWeightKg` nol, negatif, atau bukan angka wajar | 400 di DTO; `birdCount` wajib bilangan bulat positif |
| `validUntil` lebih awal dari `orderDate` | 400 |
| Mengubah baris pesanan yang bukan `PENDING_APPROVAL` | 409 |
| `vatPercent` di luar 0–100 | 400 |

Semua query difilter `tenantId`; soft delete lewat `deletedAt`.

## Job kadaluarsa

`@nestjs/schedule` dipasang; satu cron harian menolak setiap DO yang
`validUntil < hari ini` dan masih berstatus `PENDING_APPROVAL` atau `APPROVED`, lalu
melepas seluruh alokasinya — satu transaksi per pesanan, sehingga satu pesanan bermasalah
tidak menghentikan sisanya.

Job **hanya menyentuh pesanan yang punya `validUntil`**. Pesanan lama yang kolomnya
kosong tidak akan tersapu.

Job juga disediakan sebagai endpoint manual (`POST /sales-orders/expire-overdue`,
MANAGER ke atas) supaya bisa dijalankan dan diuji tanpa menunggu jadwal.

## Nomor DO

Acuan memakai `DO.<area>.<seq>` yang tidak reset tiap bulan, sementara
`ReferenceNumberGenerator` kita menghasilkan `PREFIX-YYYYMM-XXXX` dengan penghitung
berkunci `(tenantId, prefix, year, month)`.

Generator ditambah metode `generateSequential(tx, tenantId, prefix)` yang memakai tabel
`ReferenceCounter` yang sama dengan `year = 0, month = 0` sebagai penanda "tidak reset per
periode", dan mengembalikan `${prefix}.${seq}`. Dipanggil dengan prefix `DO.<kode cabang>`.

Pesanan lama yang `doNumber`-nya berformat `SO-YYYYMM-XXXX` tidak diubah.

## Harga rekomendasi dan bobot minimum

**Harga rekomendasi** dicari saat form membutuhkannya, bukan disimpan: `unitPrice` dari
baris penjualan terakhir untuk `productId` + `customerId` itu, diurutkan dari pesanan
terbaru, mengabaikan pesanan yang dihapus dan yang `REJECTED`/`CANCELLED`. Customer baru
tidak punya riwayat, dan formnya **menampilkan tanda hubung, bukan angka** — menampilkan
harga dari customer lain akan menyesatkan orang yang sedang menawar.

Endpoint: `GET /sales-orders/recommended-price?productId=&customerId=`.

**Bobot minimum** diisi dari master `MatureBirdStandard` yang dipilih di form dan disalin
ke `minimumWeightKg` di baris. Salinan, bukan kunci asing, supaya perubahan master tidak
mengubah DO yang sudah disepakati.

## Frontend

| Bagian | Isi |
|---|---|
| Header | Area; Customer **tersaring oleh area** (`CustomerBranch`, lewat `branchId` di `CustomerCombobox` yang sudah ada); Alamat dan Nama Penerima terisi otomatis dari customer tapi bisa diedit; Tanggal pesan; Masa berlaku (default `orderDate + 2 hari`); PPN % |
| Panel customer | Plafon dan TOP dari master. **Saldo ditampilkan sebagai tanda hubung** dengan keterangan bahwa piutang belum dibangun — itu S-D |
| Baris | Produk; Kandang asal dengan sisa stoknya; AVG, Qty, Tonase (tonase terhitung, hanya baca); Afkir; Bobot minimum; Harga rekomendasi (hanya baca); Harga jual; Tabungan/kg; Keterangan |
| Detail | Header + tabel baris dengan kolom yang sama, plus total |
| Daftar | Filter area, status, periode pesan; kolom Nomor DO, Tanggal s/d, Customer, Area, Status |

Kunci i18n masuk ke `messages/en.json` **dan** `id.json`, termasuk namespace `navigation`.

## Asumsi yang belum terverifikasi

| Asumsi | Kalau salah |
|---|---|
| Bobot minimum = batas bawah range Standarisasi yang dipilih | Formulanya di acuan ditandai "sumber belum terverifikasi" di finding. Kalau ternyata diturunkan dari AVG secara otomatis, yang berubah hanya cara form mengisinya — kolom salinannya tetap benar |
| "Harga terakhir" mengabaikan pesanan yang ditolak dan dibatalkan | Kalau ternyata dihitung apa adanya, satu klausa `where` berubah |
| `validUntil` default 2 hari | Nilai di acuan saat ditelusuri. Begitu master Umur Berlaku DO dibangun di S-8, default-nya pindah ke sana |

## Verifikasi

Repo belum punya jest yang jalan (`breeding-app/src/app.controller.spec.ts` gagal parse
sejak sebelum cabang mana pun). Verifikasi memakai pola dua plan terakhir: skrip curl per
tugas yang **dijalankan sampai gagal dulu** sebelum implementasi, lalu lagi sesudahnya,
ditambah smoke browser.

Yang wajib dibuktikan, bukan diasumsikan:

1. Membuat DO menahan ekor persis sebanyak yang dipesan di setiap kandang.
2. DO yang gagal di baris terakhir **tidak menyisakan tahanan** di baris sebelumnya.
3. Dua DO tidak bisa menahan ekor yang sama — yang kedua ditolak 400.
4. Menolak dan membatalkan DO melepas seluruh alokasinya, tepat sekali.
5. Mengubah DO `PENDING_APPROVAL` melepas yang lama dan menahan yang baru; selisihnya benar.
6. Job kadaluarsa menolak dan melepas DO yang lewat, dan **tidak menyentuh** pesanan tanpa `validUntil`.
7. `totalWeightKg` tidak pernah berbeda dari `birdCount × avgWeightKg`, termasuk saat klien mengirim nilai yang bertentangan.
8. Customer yang tidak terdaftar di area pesanan ditolak.
9. Kandang milik tenant lain ditolak di setiap rute.
10. Harga rekomendasi kosong untuk customer baru, dan tidak pernah mengambil harga customer lain.

## Yang dibuka spec ini

Setelah ini selesai, **S-C realisasi** punya baris pesanan yang bisa dirujuk per kandang
beserta alokasinya, dan **S-D piutang** punya nilai pesanan yang terstruktur untuk jadi
dasar faktur per DO.
