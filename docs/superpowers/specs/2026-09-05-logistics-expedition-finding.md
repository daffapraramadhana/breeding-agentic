# Ekspedisi & Dokumen Pengiriman — Finding Lintas Modul

**Dibuat:** 2026-09-05
**Status:** 10/11 terjawab (2026-09-13). Keputusan diturunkan ke `2026-09-13-logistics-expedition-design.md`. Sisa: `Q-GT-3` + item follow-up di [Belum diperiksa](#belum-diperiksa).
**Jawaban stakeholder:** `docs/stakeholder/2026-09-06-pertanyaan-ekspedisi-pengiriman-jawaban.docx`
**Screenshot pembanding:** `docs/reference/programbroiler/`
**Sumber pembanding:** `app1.programbroiler.com`
**Repo terdampak:** `breeding-app` (Prisma, procurement/logistics), `breeding-dashboard`

> **Dokumen hidup.** Modul baru ditambahkan sebagai bagian `M-x` tanpa mengubah bagian yang sudah ada. Cara menambah: lihat [Lampiran A](#lampiran-a--cara-menambah-modul-baru).

---

## Daftar isi

| Bagian | Isi |
|---|---|
| [Status](#status) | ringkasan progres & modul tercakup |
| [Temuan lintas modul](#temuan-lintas-modul) | fakta yang berlaku di lebih dari satu modul |
| [M-1 · `goods-receipt`](#m-1--goods-receipt-penerimaan-po) | penerimaan PO |
| [M-2 · `goods-transfer`](#m-2--goods-transfer-pindah-barang) | pindah barang antar gudang |
| [M-3+ · kandidat](#m-3--kandidat-belum-ditelusuri) | modul yang belum ditelusuri |
| [Pertanyaan](#pertanyaan-stakeholder) | `Q-GEN-*`, `Q-GR-*`, `Q-GT-*` |
| [Opsi solusi](#opsi-solusi) | `R-*` (receipt), `T-*` (transfer) |
| [Dampak ke spec lain](#dampak-ke-spec-lain) | konsekuensi tiap jawaban |
| [Lampiran A](#lampiran-a--cara-menambah-modul-baru) | template modul baru |
| [Lampiran B](#lampiran-b--riwayat-revisi) | riwayat revisi |

---

## Status

| Modul | Ditelusuri | Pertanyaan | Terjawab | Bisa jalan tanpa jawaban |
|---|---|---|---|---|
| `goods-receipt` | ✅ | `Q-GR-1` … `Q-GR-3` | 3/3 | — |
| `goods-transfer` | ✅ | `Q-GT-1` … `Q-GT-5` | 4/5 (`Q-GT-3` kosong) | **`T-1`** (referensi BPB) |
| lintas modul | ✅ | `Q-GEN-1` … `Q-GEN-3` | 3/3 | — |
| `goods-return` | ✅ ditutup | — | — | form pembanding tidak punya ekspedisi/surat jalan (`docs/reference/programbroiler/2026-09-13-goods-return-input-retur-pembelian.png`) |
| `internal-trade` | ❌ | — | — | — |
| `delivery` | ❌ (sudah punya pola) | — | — | — |

**Total pertanyaan terbuka: 1** (`Q-GT-3`). Follow-up non-blocking di [Belum diperiksa](#belum-diperiksa).

---

## Temuan lintas modul

Fakta di bawah berlaku di **lebih dari satu modul**. Temuan khusus satu modul ada di bagian modulnya masing-masing.

### F-1 · Master ekspedisi = master supplier (terverifikasi)

Dropdown `Ekspedisi` di aplikasi pembanding bersumber dari **master supplier**, bukan master kendaraan atau master carrier tersendiri. Berlaku di `goods-receipt` (placeholder "Pilih Suplier") maupun `goods-transfer` (isi dropdown terkonfirmasi).

Isi dropdown yang terlihat:

| Entri | Kode | Jenis |
|---|---|---|
| ACA / SATAM EXPEDISI | `S-049` | perusahaan ekspedisi |
| ACIN | `S-045` | perorangan |
| ADUNG | `S-018` | perorangan |
| Agus Purwanto | `S-047` | perorangan |
| **BERKAH BREEDING FARM** | `S-011` | **entitas internal grup** |
| BIRD LINE | `S-031` | perusahaan |
| BSA | `S-041` | perusahaan |
| BUDIANTO | `S-048` | perorangan |
| CIS NATAWIJAYA | `S-038` | perusahaan |
| CV RESTU IBU | `S-032` | perusahaan |
| PT BERKAH UTAMA SATWA (MOBIL PAKAN) | — | perusahaan, terpilih |

Kode tertinggi terlihat `S-049` → puluhan entri.

**Yang dibuktikan:**

1. Master ekspedisi memakai tabel supplier yang sama dengan supplier pakan/obat.
2. Bukan armada internal — tidak ada plat nomor atau nama sopir sebagai entitas.
3. Bercampur: perusahaan ekspedisi, pemilik truk perorangan, dan entitas internal grup dalam satu daftar.
4. ~~**Tidak difilter kategori** — `BERKAH BREEDING FARM` adalah farm, bukan jasa angkut, tapi tetap muncul.~~ **Direvisi 2026-09-13**: master supplier pembanding punya *Klasifikasi Kategori Produk* (10 kategori, termasuk `JASA EXPEDISI`) — lihat `docs/reference/programbroiler/2026-09-13-supplier-input-data-suplier.png`. Jadi penanda **ada**, lewat klasifikasi kategori. Kemungkinan `BERKAH BREEDING FARM` memang diklasifikasikan `JASA EXPEDISI` (armada farm sendiri ditagihkan sebagai supplier). Belum terverifikasi — perlu buka edit supplier itu di master mereka.

### F-2 · Nama master dipakai menyimpan atribut (anti-pola)

`PT BERKAH UTAMA SATWA (MOBIL PAKAN)` — jenis kendaraan ditempel ke dalam nama vendor. Indikasi tidak ada kolom jenis armada.

Gejala sejenis: satu tabel `Supplier` dipakai untuk beberapa peran (pemasok barang, jasa angkut, entitas internal) tanpa penanda peran.

**Jangan ditiru.** Atribut = kolom sendiri.

### F-3 · Pola `Delivery` sudah ada di sistem kita

`Delivery` (pengiriman ke customer) sudah punya `vehicleId` + `driverEmployeeId` + `helperEmployeeId` + status default `PREPARING` (`prisma/schema.prisma:1561`). Ini pola **armada internal** yang sudah berjalan.

Relevan karena: kalau modul lain memilih armada internal, polanya tinggal ditiru. Kalau memilih carrier eksternal, sistem kita akan punya **dua pola berbeda** untuk hal yang mirip — perlu disadari, bukan otomatis salah.

### F-4 · Kondisi ongkos angkut sekarang

| Hal | Status | Lokasi |
|---|---|---|
| `LogisticsShippingCost` | **ada** — `goodsReceiptId` / `goodsTransferId` / `internalTradeId` semua nullable | `prisma/schema.prisma:1463` |
| Komponen biaya yang tercatat | `shippingCost`, `insuranceCost`, `handlingCost`, `otherCost`, `totalCost` | `prisma/schema.prisma:1463` |
| Kolom carrier di dalamnya | **tidak ada** — biaya tercatat, vendor tidak diketahui | — |
| Alokasi ongkir ke HPP (landed cost) | **tidak ada** | `src/modules/inventory/services/inventory-stock.service.ts` |
| Halaman `/logistics-shipping-costs` | CRUD berdiri sendiri, tidak tersambung dari form manapun | `src/app/(dashboard)/logistics-shipping-costs/page.tsx` |

Celah terbesar saat ini: **biaya angkut tercatat tanpa diketahui dibayar ke siapa.**

---

## M-1 · `goods-receipt` (Penerimaan PO)

Form pembanding: **INPUT DATA PENERIMAAN PO**

### Yang diamati

| Observasi | Detail |
|---|---|
| Letak field `Ekspedisi` | Card **INFORMASI DATA TAMBAH PRODUK** — satu grup dengan Nama Produk, Kuantitas Sisa PO, Kuantitas |
| Kardinalitas | **Per baris produk**, bukan per header |
| Bukti kardinalitas | Tabel `LIST PRODUK` punya kolom `Ekspedisi` sejajar `Nama Produk`, `Kuantitas` |
| Sumber dropdown | Placeholder "Pilih Suplier" → lihat [F-1](#f-1--master-ekspedisi--master-supplier-terverifikasi) |
| Wajib diisi | Ya (`*` merah) |
| Section terpisah | Header `EKSPEDISI` di atas `LIST PRODUK`, isinya tidak terlihat pada state kosong |
| Kolom lain di tabel | `Transport`, `Tipe`, `Pakai Di Kandang` |

**Belum terverifikasi:**

- Arti kolom `Transport` — dugaan: nominal ongkir atau moda angkut.
- Arti kolom `Tipe` — dugaan: PO vs Bonus, mengikuti radio di atas form.
- Isi section `EKSPEDISI` saat ada data.

### Dugaan intensi

Urut dari yang paling mungkin. **Ketiganya dugaan**, belum dikonfirmasi ke pengguna aplikasi pembanding.

1. **Atribusi ongkos angkut per produk.** Satu PO bisa datang terpisah dengan ekspedisi berbeda (pakan lewat truk A, obat lewat kurir B). Tagihan datang per ekspedisi, harus dicocokkan ke barang mana.
2. **Landed cost.** HPP = harga beli + ongkir. Tanpa tahu ekspedisi per baris, ongkir tidak bisa dialokasikan.
3. **Evaluasi vendor angkut.** Melacak keterlambatan atau kerusakan per ekspedisi.

### Kondisi sistem kita

| Hal | Status | Lokasi |
|---|---|---|
| `GoodsReceipt` — ekspedisi/carrier | **tidak ada** | `prisma/schema.prisma:1160` |
| `GoodsReceiptLine` — ekspedisi/carrier | **tidak ada** | `prisma/schema.prisma:1188` |
| Form penerimaan | field: PO, Warehouse, Tanggal Terima, Notes, Line Items | `src/app/(dashboard)/goods-receipts/new/page.tsx` |

Sisanya lihat [F-4](#f-4--kondisi-ongkos-angkut-sekarang).

---

## M-2 · `goods-transfer` (Pindah Barang)

Form pembanding: **INPUT PINDAH BARANG**
URL: `app1.programbroiler.com/.../pindah_barangs/addnew/...`

### Yang diamati

| Field | Nilai / placeholder | Level |
|---|---|---|
| `Tanggal Pindah Barang` | `06/09/2026` | header |
| `Area` | `Berkah Budidaya` (wajib) | header |
| Gudang Asal | radio 3 opsi: Area / **Farm** / Kandang Ownfarm → `Keranding - Gudang Keranding` | header |
| Gudang Tujuan | radio 3 opsi sama → `Gili-Gili - Gudang Gili Gili` | header |
| `Pengambilan barang` | radio `Persediaan` (satu-satunya opsi terlihat) | per baris |
| `Nama Produk` | `Pilih Produk` | per baris |
| `No. Referensi` | **`Pilih BPB`** — ada ikon bantuan `?` | per baris |
| `Kuantitas` | kosong | per baris |
| **`Ekspedisi`** | `PT BERKAH UTAMA SATWA (MOBIL PAKAN)` | per baris |
| **`No Surat Jalan`** | kosong | per baris |
| `Keterangan Per Barang` | kosong | per baris |

Kolom tabel: `Nama Produk`, `Kuantitas`, `Ekspedisi`, `Transport`, `No Surat Jalan`, `Keterangan`, `Pakai Di Kandang`, `Hapus`.

**Belum terverifikasi:**

- Arti kolom `Transport` (sama dengan M-1).
- Apakah radio `Pengambilan barang` punya opsi selain `Persediaan`.

### Kondisi sistem kita

| Hal | Status | Lokasi |
|---|---|---|
| `GoodsTransfer` — ekspedisi / kendaraan / sopir | **tidak ada** | `prisma/schema.prisma:1212` |
| `GoodsTransfer` — nomor surat jalan | **tidak ada** | — |
| `GoodsTransferLine` — `goodsReceiptLineId` (BPB) | **tidak ada** | `prisma/schema.prisma:1240` |
| `GoodsTransferLine` — `notes` | **ada** | `prisma/schema.prisma:1249` |
| `GoodsConsumptionLine` — `goodsReceiptLineId` | **ada** (sudah di `main`) | `prisma/schema.prisma:1293` |
| `TransferStatus` | `IN_TRANSIT` (default) / `RECEIVED` / `PARTIAL` / `DAMAGED_IN_TRANSIT` / `CANCELLED` — **tidak ada `PREPARING`** | `prisma/schema.prisma:78` |
| `LogisticsShippingCost.goodsTransferId` | **ada**, nullable — belum terpakai dari form | `prisma/schema.prisma:1468` |
| `LineItemsField` prop `showGoodsReceiptRef` | **sudah ada**, dipakai di `goods-consumptions` | `src/components/forms/line-items-field.tsx:40` |
| `GoodsReceiptLineCombobox` | **sudah ada** | `src/components/forms/goods-receipt-line-combobox.tsx` |
| Form pindah barang | field: Gudang Asal, Gudang Tujuan, Tanggal, Notes, Line Items | `src/app/(dashboard)/goods-transfers/new/page.tsx` |

**Pemilihan gudang kita sudah setara — tidak perlu diubah.** `WarehouseGroupedCombobox` mengelompokkan otomatis jadi Cabang / Kandang Ownfarm / Kandang Kemitraan; satu dropdown menggantikan 3 radio + 3 dropdown milik aplikasi pembanding.

### Konflik dengan spec `2026-04-19`

`docs/superpowers/specs/2026-04-19-goods-transfer-expedition-design.md` (status `Design`, belum diimplementasi) bertentangan di tiga titik:

| # | Spec kita | Pembanding | Tingkat |
|---|---|---|---|
| **K-1** | Ekspedisi = armada internal: `vehicleId` + `driverEmployeeId` + `helperEmployeeId` | master supplier (`S-xxx`) berisi vendor, perorangan, dan entitas internal — bukan kendaraan | **Terkonfirmasi.** Dugaan "PT afiliasi" sudah terbantah oleh isi dropdown ([F-1](#f-1--master-ekspedisi--master-supplier-terverifikasi)) |
| **K-2** | Non-goal eksplisit: *"No surat jalan, resi, AWB, or tracking-number fields"* | ada field `No Surat Jalan` | **Konflik langsung** — non-goal kemungkinan keliru |
| **K-3** | Ekspedisi diisi di header lewat aksi `dispatch` (`PREPARING` → `IN_TRANSIT`) | ekspedisi diisi per baris saat input awal | **Konflik alur** |

Spec `2026-04-19` juga mengusulkan lifecycle `PREPARING` yang **tidak ada** di aplikasi pembanding — form mereka langsung simpan. Ini tidak otomatis berarti spec kita salah; kontrol dua tahap punya nilai audit sendiri, dan `Delivery` kita sudah memakainya ([F-3](#f-3--pola-delivery-sudah-ada-di-sistem-kita)). Tapi perlu disadari itu keputusan kita, bukan sesuatu yang dicontoh.

---

## M-3+ · Kandidat belum ditelusuri

Modul yang kemungkinan punya kebutuhan serupa. Belum dibuka form pembandingnya.

| Modul | Kenapa kandidat | Petunjuk awal |
|---|---|---|
| ~~`goods-return`~~ | **Ditutup 2026-09-13** — form retur pembanding tidak punya ekspedisi/surat jalan, hanya PO → BPB → barang | `docs/reference/programbroiler/2026-09-13-goods-return-input-retur-pembelian.png` |
| `internal-trade` | jual-beli antar cabang, barang fisik berpindah | model `InternalTrade` (`prisma/schema.prisma:1357`) tidak punya ekspedisi; `LogisticsShippingCost.internalTradeId` **sudah ada** tapi belum terpakai — `prisma/schema.prisma:1469` |
| `delivery` | sudah punya vehicle+driver+helper | perlu dicek apakah pembanding punya field ekspedisi eksternal juga — kalau ya, `Delivery` kita mungkin kurang |

Kalau salah satu ditelusuri, tambahkan sebagai bagian `M-3`, `M-4`, dst. Lihat [Lampiran A](#lampiran-a--cara-menambah-modul-baru).

---

## Pertanyaan stakeholder

Penomoran berprefix modul supaya penambahan modul baru tidak menggeser nomor lama.

> **Pemetaan dari versi sebelumnya** (bila sudah terlanjur dibagikan): `P1`→`Q-GR-1`, `P2`→`Q-GR-2`, `P3`→`Q-GR-3`, `P4`→`Q-GEN-1`, `P5`→`Q-GEN-2`, `P6`→`Q-GEN-3`, `P7`→`Q-GT-1`, `P8`→`Q-GT-2`, `P9`→`Q-GT-3`, `P10`→`Q-GT-4`, `P11`→`Q-GT-5`.

### Lintas modul — `Q-GEN-*`

#### `Q-GEN-1` — Master ekspedisi disimpan di mana?

> **Jawaban: (a)** — semua sudah terdaftar sebagai supplier. Penanda kategori: mengikuti pembanding, lewat klasifikasi `Supplier ↔ ProductCategory` + `ProductCategory.isExpedition`.

- **(a) Pakai `Supplier` yang sudah ada** → nol tabel baru. **Wajib disertai penanda kategori** — lihat catatan.
- **(b) Tabel master `Carrier` sendiri** → lebih bersih secara domain; satu tabel + CRUD + halaman baru.
- **(c) Pakai `Vehicle` yang sudah ada** → hanya masuk akal kalau semua pengangkutan pakai armada sendiri (lihat `Q-GT-1`).

> **Bukti lapangan:** pembanding memilih (a) **tanpa** penanda kategori, akibatnya sebuah farm (`BERKAH BREEDING FARM`) muncul di daftar ekspedisi. Kalau kita ambil (a), penanda kategori bukan opsional.
>
> Daftar mereka juga mencampur badan usaha dan perorangan. Kalau kita juga menyewa truk perorangan, opsi (c) gugur — `Vehicle` menyimpan kendaraan, bukan pihak yang disewa.

#### `Q-GEN-2` — Data apa yang perlu dilacak per pengiriman?

> **Jawaban:** hanya "Nomor Surat jalan" yang ditulis. Nama ekspedisi tidak dicentang tapi diasumsikan tetap dicatat (konsisten dengan `Q-GEN-1`/`Q-GR-1`/`Q-GT-1`). Plat, sopir, tanggal kirim: **tidak**.

Menentukan seberapa jauh scope. Pilih yang berlaku:

- [ ] Nama ekspedisi saja
- [ ] Plus nomor kendaraan / nama sopir
- [ ] Plus nomor surat jalan / resi
- [ ] Plus tanggal kirim vs tanggal terima (untuk hitung lead time)

#### `Q-GEN-3` — Perlu evaluasi vendor angkut?

> **Jawaban: (b)** — tidak.

- **(a) Ya** — perlu laporan keterlambatan / kerusakan per ekspedisi → field harus relasi terstruktur, bukan teks bebas.
- **(b) Tidak** — cukup untuk pencatatan biaya.

### `goods-receipt` — `Q-GR-*`

#### `Q-GR-1` — Ongkir barang masuk dari supplier dibayar siapa?

> **Jawaban: (b)** — kita yang tanggung. Bagian B tidak gugur.

**Penentu utama** — menentukan apakah field ekspedisi perlu ada sama sekali di modul ini.

- **(a) Supplier yang tanggung** (DDP / franco gudang) → bukan urusan kita, tidak perlu ditambah.
- **(b) Kita yang tanggung** (FOB / loco gudang) → perlu; `LogisticsShippingCost` sekarang mencatat angka tanpa tahu tagihan dari siapa.
- **(c) Campuran** tergantung supplier atau jenis barang → perlu, plus penanda per transaksi.

#### `Q-GR-2` — Satu penerimaan bisa dibagi ke berapa ekspedisi?

> **Jawaban: (c)** — bisa, tapi jarang. Header saja.

Menentukan level field: header atau per baris.

- **(a) Selalu satu** → cukup satu field di header. Form tetap pendek.
- **(b) Bisa lebih dari satu, dan sering** → per baris, seperti pembanding.
- **(c) Bisa lebih dari satu, tapi jarang** → header saja; kasus langka dicatat di notes.

> Pembanding memilih per-baris. Kalau operasional kita 1 penerimaan = 1 truk, meniru per-baris hanya memperpanjang form tanpa manfaat.

#### `Q-GR-3` — Ongkir harus masuk HPP barang?

> **Jawaban: (b)** — biaya operasional. R-4 gugur.

**Masalah terpisah, mungkin lebih mendesak daripada field ekspedisi.**

- **(a) Ya, terserap ke harga pokok persediaan** → butuh mekanisme alokasi biaya. Menambah dropdown ekspedisi **tidak** menyelesaikan ini.
- **(b) Tidak, dibebankan sebagai biaya operasional periode berjalan** → cukup pencatatan.

Kalau (a), lanjutan: dasar alokasinya apa — nilai, kuantitas, atau berat?

### `goods-transfer` — `Q-GT-*`

#### `Q-GT-1` — Pindah barang antar gudang diangkut siapa?

> **Jawaban: (b)** — vendor jasa ekspedisi. Spec `2026-04-19` **superseded** oleh `2026-09-13-logistics-expedition-design.md`.

**Menentukan `Vehicle` vs carrier — dan apakah spec `2026-04-19` perlu direvisi.**

- **(a) Truk milik sendiri** → spec `2026-04-19` **tetap valid**, konflik `K-1` gugur.
- **(b) Vendor pihak ketiga** → spec `2026-04-19` **harus direvisi**, pakai master carrier.
- **(c) Pemilik truk perorangan yang disewa** → sama seperti (b): yang dicatat pihak penyedia, bukan kendaraan.
- **(d) Campuran** → butuh keduanya; spec **diperluas**, bukan diganti.

> Pembanding menjawab (b)+(c): daftar mereka berisi perusahaan ekspedisi **dan** perorangan, semuanya sebagai supplier. Tidak ada jejak armada internal di form itu. Yang perlu dipastikan: apakah operasional **kita** sama.

#### `Q-GT-2` — Satu pindah barang = satu kendaraan?

> **Jawaban: (a)** — selalu satu truk. Header.

Menentukan header vs per-baris untuk ekspedisi dan surat jalan.

- **(a) Selalu satu truk** → ekspedisi + surat jalan di **header**. Operator tidak mengetik nomor yang sama berulang.
- **(b) Bisa dipecah ke beberapa kendaraan** → per baris, seperti pembanding.

> Satu surat jalan biasanya mencakup seluruh muatan satu truk, bukan per item. Menaruh `No Surat Jalan` per baris berarti nomor yang sama diketik ulang untuk tiap produk. Dugaan kenapa pembanding begitu: mengakomodasi kasus (b).

#### `Q-GT-3` — Nomor surat jalan: diterbitkan siapa, wajib tercatat?

> **Jawaban: kosong** — kotak jawaban berisi salinan pertanyaan. **Perlu ditanya ulang.** Asumsi sementara di spec: input manual, opsional, tidak unik.

- Auto-generate sistem, atau input manual dari dokumen fisik?
- Wajib atau opsional?
- Perlu unik per tenant?

> Non-goal di spec `2026-04-19` menolak field ini. Kalau jawabannya "wajib tercatat", non-goal itu harus dicabut (`K-2`).

#### `Q-GT-4` — Pindah barang perlu telusur asal BPB?

> **Jawaban: (a)** — ya.

- **(a) Ya** → tambah `goodsReceiptLineId` di `GoodsTransferLine`, mirroring `GoodsConsumptionLine`.
- **(b) Tidak** → cukup produk + kuantitas.

> Kemungkinan besar (a). `goods-consumption` sudah punya telusur BPB. Kalau pindah barang tidak punya, rantai telusur putus di tengah — barang pindah gudang lalu kehilangan asal-usulnya.

#### `Q-GT-5` — `GoodsTransfer` perlu status `PREPARING` (draft)?

> **Jawaban: (a)** — dua langkah. Catatan: transfer saat ini **tidak menggerakkan stok sama sekali**; stock movement transfer dipisah jadi spec tersendiri (lihat non-goal di spec 2026-09-13).

- **(a) Ya** → lanjutkan spec `2026-04-19`; ada tahap draft, stok baru berkurang saat dispatch.
- **(b) Tidak** → sederhanakan spec; sekali simpan langsung `IN_TRANSIT` seperti pembanding.

> Pembanding tidak punya tahap draft. Ini murni keputusan kita. `Delivery` kita sudah memakai `PREPARING` sebagai default ([F-3](#f-3--pola-delivery-sudah-ada-di-sistem-kita)) — memilih (b) berarti dua modul serupa punya perilaku berbeda.

---

## Opsi solusi

**Diputuskan 2026-09-13**: `R-2` (+ `deliveryNoteNumber`), `T-1`, `T-2`, `T-3b`, `T-4` tanpa stock movement. Gugur: `R-0`, `R-1`, `R-3`, `R-4`, `T-3a`. Detail di `2026-09-13-logistics-expedition-design.md`. Tabel di bawah dipertahankan sebagai riwayat.

### `goods-receipt` — `R-*`

| Opsi | Perubahan | Menjawab | Biaya |
|---|---|---|---|
| **R-0** Tidak ada | — | `Q-GR-1`=(a) | nol |
| **R-1** Carrier di biaya angkut | `LogisticsShippingCost.carrierSupplierId` nullable → `Supplier` | `Q-GR-1`=(b/c), `Q-GR-2`=(a/c), `Q-GEN-3`=(a) | 1 kolom + 1 relasi + 1 migration. **Tidak menyentuh `GoodsReceipt`** — record biaya sudah punya `goodsReceiptId` |
| **R-2** Carrier di header penerimaan | `GoodsReceipt.carrierSupplierId` + field di form | `Q-GR-2`=(a) | 1 kolom + DTO + 1 combobox |
| **R-3** Carrier per baris | `GoodsReceiptLine.carrierSupplierId` | `Q-GR-2`=(b) | 1 kolom + ubah `LineItemsField` (dipakai bersama modul lain — prop harus opsional) |
| **R-4** Alokasi landed cost | mekanisme serap ongkir ke nilai persediaan | `Q-GR-3`=(a) | Paling besar. **Independen** dari R-1…R-3 |

Rekomendasi awal bila `Q-GR-1`=(b) dan `Q-GR-2`=(a): **R-1**. Paling sedikit menyentuh kode, langsung menutup celah terbesar di [F-4](#f-4--kondisi-ongkos-angkut-sekarang).

### `goods-transfer` — `T-*`

| Opsi | Perubahan | Level | Menjawab | Biaya |
|---|---|---|---|---|
| **T-1** Referensi BPB + catatan per baris | `GoodsTransferLine.goodsReceiptLineId` + aktifkan `showGoodsReceiptRef` / `showLineNotes` | per baris | `Q-GT-4`=(a) | **Rendah** — komponen sudah ada. Migration 1 kolom + 1 index |
| **T-2** Nomor surat jalan | `GoodsTransfer.deliveryNoteNumber` | header | `Q-GT-3`, `Q-GT-2`=(a) | 1 kolom string + 1 input |
| **T-3a** Ekspedisi = armada internal | ikuti spec `2026-04-19`: `vehicleId` + `driverEmployeeId` + `helperEmployeeId` + `dispatchedAt` + `dispatchedById` | header | `Q-GT-1`=(a) | Sedang — sesuai spec yang sudah ditulis |
| **T-3b** Ekspedisi = carrier eksternal | `GoodsTransfer.carrierSupplierId` (atau `carrierId` bila `Q-GEN-1`=(b)) | header | `Q-GT-1`=(b/c) | Sedang — **spec `2026-04-19` harus direvisi** |
| **T-4** Lifecycle `PREPARING` | tambah enum + aksi `dispatch` + pindahkan stock movement | — | `Q-GT-5`=(a) | Besar — menyentuh logika stok |

#### Kenapa T-1 bisa jalan duluan

**Tidak bergantung jawaban stakeholder manapun.** Alasannya konsistensi internal, bukan mencontoh pembanding:

- `GoodsConsumptionLine` sudah punya `goodsReceiptLineId` (sudah di `main`).
- `GoodsTransferLine` belum.
- Rantai telusur putus: barang bisa ditelusuri saat dipakai, tidak saat dipindah.

Diff frontend satu blok:

```diff
  <LineItemsField
    lines={lines}
    onChange={setLines}
    showPrice={false}
+   showGoodsReceiptRef
+   showLineNotes
+   warehouseId={form.fromWarehouseId}
  />
```

Backend: mirror `GoodsConsumptionLine` — `goodsReceiptLineId String?`, relasi opsional ke `GoodsReceiptLine`, `@@index([goodsReceiptLineId])`.

**Konfirmasi sebelum dikerjakan:** apakah `GoodsReceiptLineCombobox` memfilter BPB berdasarkan gudang. Di consumption ia menerima prop `warehouseId`; untuk transfer harus `fromWarehouseId`, bukan tujuan.

---

## Dampak ke spec lain

| Jawaban | Konsekuensi |
|---|---|
| `Q-GT-1`=(b) atau (c) | Spec `2026-04-19` **harus direvisi** — ganti `Vehicle` dengan carrier |
| `Q-GT-1`=(d) | Spec `2026-04-19` **diperluas** — dukung armada internal **dan** carrier |
| `Q-GT-3` = wajib tercatat | Non-goal *"No surat jalan"* di spec `2026-04-19` **harus dicabut** |
| `Q-GT-5`=(b) | Spec `2026-04-19` disederhanakan — buang lifecycle `PREPARING` |
| `Q-GR-1`=(a) | Semua opsi `R-*` gugur; tidak ada pekerjaan di `goods-receipt` |
| `Q-GEN-1`=(b) | Semua opsi yang menyebut `carrierSupplierId` berubah jadi `carrierId` → tabel `Carrier` |

### Belum diperiksa

Follow-up ke stakeholder / screenshot pembanding, **tidak memblokir** spec 2026-09-13:

| # | Item | Menutup apa |
|---|---|---|
| 1 | `Q-GT-3` — surat jalan: sistem/manual, wajib/opsional, unik? | validasi + UI `deliveryNoteNumber` |
| 2 | Konfirmasi `Q-GEN-2`: nama ekspedisi tetap dicatat? | apakah combobox ekspedisi ditampilkan |
| 3 | Menu **Master** pembanding dibuka | ada master Armada/Ekspedisi terpisah? |
| 4 | Master › Suplier › edit `BERKAH BREEDING FARM` | verifikasi revisi F-1 |
| 5 | Master › Kategori Produk (kalau ada) | 10 kategori fixed atau editable |
| 6 | Input Pindah Barang / Penerimaan PO **setelah 1 baris ditambah** | isi section `EKSPEDISI`, kolom `Transport`, `Tipe`, `Pakai Di Kandang` |
| 7 | Dropdown Ekspedisi di Penerimaan PO dibuka | sama dengan di Pindah Barang? |
| 8 | Detail Pindah Barang / Penerimaan tersimpan | ada tahapan status? |
| 9 | Di mana biaya angkut dicatat di pembanding | alur `LogisticsShippingCost` |
| 10 | Menu **Logistik** dibuka | ada internal-trade? |

Internal (bukan stakeholder): apakah `/logistics-shipping-costs` dilebur ke form penerimaan; modul kandidat di [M-3+](#m-3--kandidat-belum-ditelusuri) (`goods-return` sudah ditutup).

### Referensi

- `docs/superpowers/specs/2026-04-19-goods-transfer-expedition-design.md` — status `Design`, **perlu revisi tergantung `Q-GT-1`, `Q-GT-3`, `Q-GT-5`**
- Aplikasi pembanding: `app1.programbroiler.com` — form **INPUT DATA PENERIMAAN PO**, **INPUT PINDAH BARANG**

---

## Lampiran A — Cara menambah modul baru

Saat modul lain ditelusuri, tambahkan tanpa mengubah bagian yang sudah ada:

1. **Tambah bagian `M-x`** setelah modul terakhir, sebelum [M-3+ kandidat](#m-3--kandidat-belum-ditelusuri). Pakai struktur yang sama: *Yang diamati* → *Belum terverifikasi* → *Dugaan intensi* (kalau ada) → *Kondisi sistem kita* → *Konflik dengan spec* (kalau ada).
2. **Fakta yang berlaku >1 modul** masuk ke [Temuan lintas modul](#temuan-lintas-modul) sebagai `F-x`, bukan diulang di tiap modul. Rujuk dengan link.
3. **Pertanyaan baru** pakai prefix modul: `Q-GRT-*` (goods-return), `Q-IT-*` (internal-trade), dst. **Jangan menyisipkan nomor di tengah deret yang sudah ada** — pertanyaan mungkin sudah dibagikan ke stakeholder.
4. **Opsi solusi** pakai prefix huruf sendiri, konsisten dengan `R-*` / `T-*`.
5. **Update tabel [Status](#status)** dan hapus baris modul dari [M-3+ kandidat](#m-3--kandidat-belum-ditelusuri).
6. **Catat di [Lampiran B](#lampiran-b--riwayat-revisi).**

Kalau modul baru **membantah** temuan yang sudah ada, jangan hapus temuan lamanya — tandai terbantah dan sebutkan buktinya, seperti yang dilakukan pada `K-1`.

### Template bagian modul

```markdown
## M-x · `nama-modul` (Judul Indonesia)

Form pembanding: **NAMA FORM**
URL: `...`

### Yang diamati

| Field | Nilai / placeholder | Level |
|---|---|---|

**Belum terverifikasi:**
-

### Kondisi sistem kita

| Hal | Status | Lokasi |
|---|---|---|
```

---

## Lampiran B — Riwayat revisi

| Tanggal | Perubahan |
|---|---|
| 2026-09-05 | Dibuat untuk `goods-receipt` (form INPUT DATA PENERIMAAN PO). |
| 2026-09-05 | Ditambah `goods-transfer` (form INPUT PINDAH BARANG); ditemukan konflik `K-1`…`K-3` dengan spec `2026-04-19`. |
| 2026-09-05 | Isi dropdown `Ekspedisi` terverifikasi → `F-1`. Dugaan "PT afiliasi" **terbantah**; `K-1` naik jadi konflik terkonfirmasi. |
| 2026-09-13 | Jawaban stakeholder masuk (10/11). F-1 direvisi: penanda ekspedisi ada lewat klasifikasi kategori supplier. `goods-return` ditutup. Opsi solusi diputuskan → spec `2026-09-13-logistics-expedition-design.md`; spec `2026-04-19` superseded. Ditambah 8 screenshot pembanding di `docs/reference/programbroiler/`. |
| 2026-09-05 | Restrukturisasi jadi format unified. Penomoran `P1`–`P11` → `Q-GEN/GR/GT-*` (pemetaan ada di bagian Pertanyaan). Ditambah `F-3` (pola `Delivery`), `F-4` (kondisi ongkos angkut), `M-3+` kandidat, Lampiran A & B. |
