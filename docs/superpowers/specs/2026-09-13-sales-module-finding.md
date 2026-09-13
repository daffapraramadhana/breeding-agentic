# Modul Penjualan — Finding Alur Penuh

**Dibuat:** 2026-09-13
**Status:** Terbuka — alur pembanding sudah ditelusuri langsung (sesi login user), belum ada jawaban stakeholder
**Sumber pembanding:** `app1.programbroiler.com` — menu Marketing, Keuangan, Master (ditelusuri via browser 2026-09-13)
**Repo terdampak:** `breeding-app` (Prisma `SalesOrder`, `Customer`, `Delivery`, `SalesInvoice`, `SalesPayment`), `breeding-dashboard`
**Screenshot:** `docs/reference/programbroiler/2026-09-13-sales-*.png`, `2026-09-13-customer-*.png`

---

## Daftar isi

| Bagian | Isi |
|---|---|
| [Peta modul pembanding](#peta-modul-pembanding) | semua menu Logistik/Produksi/Marketing/Keuangan/Master |
| [Alur penjualan](#alur-penjualan-pembanding) | pesanan → realisasi → pengiriman → faktur → penerimaan uang |
| [S-1 Pesanan](#s-1--pesanan-penjualan-barang) | form + detail + list + status |
| [S-2 Realisasi](#s-2--realisasi-do) | timbang per pengambilan |
| [S-3 Pengiriman](#s-3--pengiriman) | armada/sopir |
| [S-4 Faktur](#s-4--faktur-do-gabungan) | faktur gabungan per customer |
| [S-5 Penerimaan uang](#s-5--penerimaan-uang-dari-customer) | pembayaran per customer |
| [S-6 Refund & Pembekuan](#s-6--refund--pembekuan-piutang) | |
| [S-7 Master Customer](#s-7--master-customer) | definisi resmi tiap field |
| [S-8 Master pendukung](#s-8--master-pendukung) | harga, umur DO, produk ayam besar |
| [Gap vs sistem kita](#gap-vs-sistem-kita) | tabel per entitas |
| [Pertanyaan stakeholder](#pertanyaan-stakeholder) | `Q-SO-*`, `Q-CU-*`, `Q-PAY-*` |
| [Opsi solusi](#opsi-solusi) | `S-*` |

---

## Peta modul pembanding

Diambil dari nav (`index.php/<controller>`):

| Menu | Controller | Modul kita |
|---|---|---|
| Logistik › Pembelian Barang Suplier | `pembelian_barang_supliers` | `purchase-order` |
| Logistik › Penerimaan Barang Suplier | `penerimaan_barang_supliers` | `goods-receipt` |
| Logistik › Pindah Barang | `pindah_barangs` | `goods-transfer` |
| Logistik › Pemakaian Barang Suplier | `pemakaian_barang_supliers` | `goods-consumption` |
| Logistik › Pengembalian Barang Suplier | `pengembalian_barang_supliers` | `goods-return` |
| Logistik › Trading Internal | `trading_internals` | `internal-trade` |
| Produksi › Data Harian | `data_harians` | daily recording |
| Produksi › Culling | `cullings` | — |
| **Marketing › Penjualan Barang Customer** | `penjualan_barang_customers` | `sales-order` |
| **Marketing › Pengiriman Barang Customer** | `pengiriman_barang_customers` | `delivery` |
| Keuangan › Pengeluaran OH | `pengeluarans` | expenses |
| Keuangan › Pembayaran Suplier & OH | `pembayaran_pembelian_supliers` | supplier payment |
| **Keuangan › Penerimaan Uang Dari Customer** | `penerimaan_uang_customers` | `sales-payment` |
| Keuangan › Pembayaran Trading Internal | `pembayaran_trading_internals` | — |
| **Keuangan › Pengembalian Uang Ke Customers** | `pengembalian_uang_customers` | — (refund) |
| **Keuangan › Pembekuan Piutang Customer** | `pembekuan_hutang_customers` | — (debt freeze) |
| Closing | `huts` | — |
| Master › Customer | `master_customers` | `customer` |
| Master › Type Kartu Peternak | `master_type_kartus` | `breeder-card-type` |
| Master › Produk Ayam Besar | `master_produk_ayam_besars` | `mature-bird-type` |
| Master › Range Harga Ayam Besar (Standarisasi Kontrak) | `master_ayambesars` | — |
| Master › Parameter Selisih Harga Pasar | `master_selisih_harga_pasars` | — |
| Master › Umur Berlaku DO | `master_berlaku_dos` | — |
| Master › Armada / Pegawai / Area / Farm / Blok / Kandang / Gudang | `master_*` | ada |
| Master › Bank / Kas / Default Bank Pembayaran | | `bank-account`, `cash-account` |
| Master Bonus › Susut & Bonus Pengiriman, Bonus Selisih Harga Pasar, dst. | | `bonus` (sebagian) |

Kesimpulan: **kerangka modul kita sudah dibangun dari pola app ini** — nama status, tipe penerima (Customer/Peternak), Delivery dengan armada+sopir+bonus semuanya cocok. Yang belum ada adalah **isi** form pesanan/realisasi dan mekanisme piutang.

---

## Alur penjualan pembanding

```
Pesanan (DO)  ──approval──▶  Telah Disetujui  ──▶  Realisasi (timbang per pengambilan, bisa >1×)
                                                        │
                                                        ├──▶ Pengiriman (armada/sopir) — opsional, kalau diantar
                                                        ├──▶ Faktur DO Gabungan (per customer per tgl realisasi)
                                                        └──▶ Penerimaan Uang (bayar piutang per customer, bukan per faktur)
                                                                 ├── Kelebihan / Kekurangan bayar
                                                                 ├── Tabungan customer (bisa dipakai bayar)
                                                                 └── Refund / Pembekuan piutang (Keuangan)
```

**Status pesanan** (dropdown filter list, urutan alfabet):

| Pembanding | `SalesStatus` kita | Catatan |
|---|---|---|
| Proses Persetujuan | `PENDING_APPROVAL` | default saat simpan |
| Telah Disetujui | `APPROVED` | |
| Approval Realisasi | `REALIZATION_APPROVAL` | |
| Proses Realisasi | `REALIZING` | |
| Proses Realisasi DO Limit Plafon Disetujui | `REALIZING_DO_LIMIT` | |
| Limit Plafon Diproses | `CREDIT_LIMIT_PROCESSING` | |
| Limit Plafon Ditolak | `CREDIT_LIMIT_REJECTED` | |
| Reject / **Reject (DO Lebih dari 3 hari)** | `REJECTED` | auto-reject kalau melewati *Umur Berlaku DO* |
| Dibatalkan | `CANCELLED` | |
| — | `CREDIT_LIMIT_APPROVED` | kita punya, mereka tidak tampak sebagai status terpisah |

Data nyata Feb 2025 (26 pesanan): 21 *Telah Disetujui*, 1 *Reject (DO Lebih dari 3 hari)*. Jadi alur harian: simpan → approve → realisasi; status plafon jarang.

Nomor DO: `DO.<Area>.<seq 6 digit>` — contoh `DO.Budidaya.001880`.

---

## S-1 · Pesanan (Penjualan Barang)

### Form input — header

| Field | Wajib | Sumber / perilaku |
|---|---|---|
| Area | ✓ | branch |
| Customer | ✓ | master customer; memicu autofill alamat, nama penerima, TOP, plafon, saldo |
| Alamat | ✓ | autofill dari customer, **bisa diedit** |
| Nama Penerima | ✓ | autofill nama customer, bisa diedit |
| Tanggal Pemesanan **s/d** | ✓ | rentang; lebar default = *Umur Berlaku DO* (master, saat ini 2 hari). Lewat rentang → auto **Reject (DO Lebih dari N hari)** |
| TOP | tampil | `N Hari` dari customer + link "klik disini" → popup: *"Customer belum melakukan pembayaran selama 759 hari dan batas dari TOP (3 hari)"* |
| Plafon Customer | tampil | `Rp (50.000.000)` merah = limit |
| Saldo Customer `?` | tampil | tooltip: **"Saldo estimasi (jumlah DO pesanan belum realisasi dan DO request)"** |
| PPN % | footer | default 0 |

### Form input — per baris

| Field | Catatan |
|---|---|
| Nama Produk | dari master produk (ayam besar) |
| Asal: Gudang Area / Gudang Farm / **Kandang Ownfarm** | radio → dropdown; untuk ayam hidup asalnya kandang |
| Sisa Stok | tampil setelah asal dipilih |
| Kuantitas: **AVG · Qty · Tonase** | tiga kotak; AVG × Qty = Tonase |
| Afkir | checkbox |
| Bobot Minimum | tampil, sumber belum terverifikasi (kemungkinan *Standarisasi Kontrak Ayam Besar*) |
| Harga Rekomendasi | tampil, sumber belum terverifikasi (kemungkinan range harga + *Selisih Harga Pasar*) |
| Harga Jual | input |
| Tabungan (Per Kg) | input, default dari customer |
| Keterangan Per Produk | |

Tabel: Asal Produk, Qty, Tonase, AVG, Afkir, Tabungan/Kg, Keterangan, Harga, Total Harga.

### Form "Jual Ke Peternak" (varian)

Header: Area, **Peternak**, Alamat, Nama Penerima, Tanggal s/d, **Tipe Kartu Saldo** (= `BreederCardType` kita). Baris: produk, asal (Gudang Area / **Kandang Kemitraan**), sisa stok, **Kuantitas, Harga Jual** saja — tanpa AVG/tonase/afkir/tabungan. Ini penjualan sapronak (pakan/OVK) ke mitra, bukan ayam.

### Detail pesanan (view)

Header + tabel *Detail Pesanan* (Produk, Ekor, Tonase, AVG, Afkir, Keterangan, **Harga Estimasi**) + blok *Data Realisasi* + tabel *Detail Realisasi* (lihat S-2). Aksi di list: Lihat Detail, **Edit Realisasi**, Hapus, **Cetak Faktur**.

### List

Filter: pencarian, Area, Status, **Penjualan Ke (Customer/Peternak)**, Periode Pesan. Kolom: Nomor DO (+ nomor DTPS di bawahnya), Tanggal (s/d), Realisasi, Customer, Asal, Kandang, Area, Status.

---

## S-2 · Realisasi DO

Layar **Modify Data Realisasi DO**. Satu DO bisa direalisasi **beberapa kali** (beberapa pengambilan/truk). Ini yang mengurangi stok ayam dan menjadi dasar faktur.

Header: Area, Nomor DO, Tanggal Pemesanan (s/d), **Tanggal Realisasi**, Keterangan DO, Customer, **No. Plat Mobil** (dari customer), Nama Penerima, Alamat, Keterangan Realisasi. Toggle **"Timbang Kandang"** (belum terverifikasi fungsinya — dugaan: ambil hasil timbang dari data produksi kandang).

Per baris realisasi:

| Field | Catatan |
|---|---|
| Tanggal Ambil | tanggal truk ambil |
| Project Kandang Farm | `Farm Pisang Sambo (Periode 18) \| Pisang Sambo Kandang - 4` — **project + kandang**, bukan gudang |
| **No. DTPS** | nomor dokumen timbang (Daftar Timbang…?) — belum terverifikasi kepanjangannya |
| Qty · Tonase · AVG (hitung) | AVG = Tonase/Qty |
| Afkir | checkbox |
| Sisa Stok | tampil |
| Harga Jual (per Kg) | |
| Diskon (per Kg) | |
| Biaya Admin (per Kg) | total tampil sebagai "Total Biaya Admin **Ke Peternak**" |
| Tabungan (per Kg) | |
| Cicilan Hutang (per Kg) `?` | tooltip: **"Cicilan bisa dimasukan jika bakul mempunyai hutang yang dibekukan"** — readonly kecuali ada piutang beku |
| Total Harga | hitung |

Footer: Total Harga Realisasi, Total Diskon, PPN %, Total Biaya Admin, Total Tabungan.

Contoh nyata: DO.Budidaya.000001 pesanan 1.000 ekor → 2 realisasi (07/09: 1.000 ekor, 09/09: 100 ekor — **boleh melebihi pesanan**), masing-masing DTPS berbeda.

---

## S-3 · Pengiriman

Form: Tanggal, Area, **Armada**, **Supir**, **Kota/Tujuan**, **Bonus Supir** (dropdown master), Customer Tujuan, DO Customer, Jumlah Ayam, Berat Keseluruhan. **Setara `Delivery` kita** (`vehicleId`, `driverEmployeeId`, `destinationCity`, `driverBonus`, `totalBirdCount`, `totalWeightKg`). Tidak ada gap berarti; kernet (`helperEmployeeId`) ekstra kita.

---

## S-4 · Faktur DO Gabungan

Tab di list Penjualan. Satu baris per **customer × tanggal realisasi**, kolom DO/SO (tipe), Area. Faktur = gabungan semua realisasi customer di hari itu. "Cetak Faktur" per DO juga ada. Kita: `SalesInvoice` 1:1 ke `SalesOrder` — **berbeda granularitas**.

---

## S-5 · Penerimaan Uang Dari Customer

Form: Area, Customer, Tanggal, Nomor Transaksi, Jenis (Cash / Transfer / Cek-Giro), **Bayar Piutang** (nominal), **Kelebihan Bayar** / **Kekurangan Bayar** (radio + nominal), **Tabungan** (nominal dipakai), Keterangan; sisi kanan: Kas Masuk / Bank Tujuan / Bank Pembayaran / Rekening / Nama Pengirim; info: Uang Masuk, **Saldo Customer**, Pelunasan Piutang, **Tabungan Customer**, Total Pembayaran.

Status: Sedang Dalam Proses Verifikasi / Telah Diverifikasi / Ditolak = `PaymentStatus` kita.

**Perbedaan struktural**: pembayaran diterapkan ke **saldo piutang customer** (running balance), bukan ke satu invoice. Kita: `SalesPayment.salesInvoiceId` wajib.

## S-6 · Refund & Pembekuan Piutang

- **Refund Pembayaran**: Area, Customer, Tanggal, Nomor Transaksi, Jenis (Cash/Transfer), Jumlah Refund, Kas/Bank Keluar → mengembalikan kelebihan bayar.
- **Pembekuan Hutang Customer**: daftar (kosong di tenant ini; user tidak punya hak tambah). Konsep: piutang lama "dibekukan", lalu dicicil per kg lewat *Cicilan Hutang (per Kg)* di realisasi.

---

## S-7 · Master Customer

Definisi resmi dari tooltip aplikasi:

| Field | Wajib | Tooltip / arti |
|---|---|---|
| Kode, Nama | ✓ | |
| Area Pendaftaran | ✓ | branch asal |
| Alamat, Kota | ✓ | |
| No KTP | ✓ | |
| NPWP | | |
| **Plat Nomor Mobil** | ✓ | muncul di realisasi sebagai "No. Plat Mobil" — customer (bakul) jemput sendiri |
| Keterangan | | |
| **Operasional** (checkbox multi: Berkah Budidaya / Berkah Pembelian / Breeding) | | *"Wilayah Operasional Customer"* → customer boleh transaksi di area mana |
| **Plafon** (toggle) | | *"Mengaktifkan pembatasan piutang Customer."* |
| **Limit Plafon** Rp | | *"Batas jumlah piutang yang diizinkan untuk Customer ini."* |
| **TOP Pembayaran** hari | | *"Pembatasan pembuatan DO berdasarkan pembayaran Customer, jika hari TOP telah dilewati maka akan di warning dan harus approval Pusat."* |
| **Cicilan /Kg** Rp | | *"Untuk menetapkan cicilan penambahan DO, untuk bakul yang memiliki hutang yang dibekukan oleh perusahaan."* |
| **Tabungan /Kg** Rp | | *"Untuk menetapkan tabungan penambahan DO"* |

List customer: Kode, Nama, Alamat, Kota, Limit Plafon, **Performa** ("Masih Baru"), **Jaminan** ("Tidak ada") — ini `performance` & `collateral` kita, jadi model `Customer` kita memang diturunkan dari sini.

## S-8 · Master pendukung

| Master | Isi | Relevansi |
|---|---|---|
| Umur Berlaku DO | 1 angka: jumlah hari (=2) | lebar rentang tanggal pesanan + auto-reject |
| Standarisasi Kontrak Ayam Besar (`master_ayambesars`) | Nama + daftar range *Nilai dari–sampai* | dugaan: band bobot → **Bobot Minimum** / harga kontrak |
| Parameter Selisih Harga Pasar | Rp global + per Area | dugaan: **Harga Rekomendasi** = harga pasar ± selisih |
| Produk Ayam Besar | tipe (AYAM BROILER) | = `MatureBirdType` kita |
| Type Kartu Peternak | | = `BreederCardType` kita |

---

## Gap vs sistem kita

### `SalesOrder` / `SalesOrderLine`

| Pembanding | Kita | Gap | Bobot |
|---|---|---|---|
| Baris: produk dari master + **asal kandang/gudang** + sisa stok | `productCode`/`productDescription` teks bebas, tanpa asal | **Struktural** — tanpa FK produk & asal, tidak ada stok, tidak ada telusur | tinggi |
| AVG · Qty · Tonase | `birdCount`, `totalWeightKg` | cukup, AVG dihitung | — |
| Afkir per baris | — | boolean | rendah |
| Bobot Minimum, Harga Rekomendasi (tampil) | `marketPrice` header | butuh master harga; bisa ditunda | sedang |
| Tabungan/Kg per baris | — | kolom + logika akumulasi tabungan customer | sedang |
| PPN % | — | kolom header | rendah |
| Alamat + Nama Penerima | — | 2 kolom header | rendah |
| Tanggal s/d + auto-reject | `orderDate` | `validUntil` + job/pengecekan | sedang |
| TOP warning + approval pusat | — | butuh `Customer.topDays` + last payment date | sedang |
| Saldo estimasi customer saat input | — | agregat DO belum realisasi | sedang |
| Nomor `DO.<Area>.<seq>` | `SO-…` | penomoran per area | rendah |
| **Realisasi per pengambilan** (S-2) | tidak ada entitas; `Delivery` ≠ realisasi | **Struktural** — entitas baru `SalesRealization(Line)` | tinggi |
| Jual ke peternak: Tipe Kartu Saldo | `breederId` ada, kartu tidak | FK `breederCardTypeId` | rendah |

### `Customer`

| Pembanding | Kita | Gap |
|---|---|---|
| Area Pendaftaran, No KTP, NPWP, **Plat Nomor** | — | 4 kolom |
| Operasional (multi-area) | — | M2M `CustomerBranch` |
| Plafon toggle + limit | `creditLimit` nullable | `creditLimitEnabled` atau null = off |
| TOP hari | — | `topDays Int?` |
| Cicilan/Kg, Tabungan/Kg | — | 2 kolom Decimal |
| Performa, Jaminan | `performance`, `collateral` ✅ | — |
| contactPerson/phone/email | ekstra kita | pertahankan |

### `SalesInvoice` / `SalesPayment`

| Pembanding | Kita | Gap |
|---|---|---|
| Faktur gabungan per customer × tanggal realisasi | invoice per SO | granularitas beda — perlu keputusan |
| Pembayaran ke saldo customer; kelebihan/kekurangan; pakai tabungan | payment per invoice | **Struktural** — ledger piutang per customer |
| Refund, Pembekuan piutang, Cicilan/Kg | — | modul baru |

### `Delivery`

Setara. Tidak ada gap berarti.

---

## Pertanyaan stakeholder

### Pesanan & realisasi — `Q-SO-*`

- **`Q-SO-1`** Realisasi: satu DO memang boleh direalisasi beberapa kali (beberapa truk / hari), dan totalnya boleh **melebihi** pesanan (contoh data: 1.100 dari 1.000)? Atau itu salah input?
- **`Q-SO-2`** "No. DTPS" kepanjangannya apa, siapa yang menerbitkan (timbangan kandang? mitra?), wajib?
- **`Q-SO-3`** "Timbang Kandang" (toggle di realisasi) fungsinya apa — mengambil angka dari recording harian kandang?
- **`Q-SO-4`** Harga Rekomendasi dan Bobot Minimum diambil dari mana? (dugaan: Standarisasi Kontrak Ayam Besar + Selisih Harga Pasar)
- **`Q-SO-5`** Afkir: pengaruhnya ke harga/stok apa? Harga afkir berbeda?
- **`Q-SO-6`** Umur Berlaku DO (2 hari): DO yang lewat langsung reject otomatis, atau perlu aksi manual?
- **`Q-SO-7`** Pengiriman (armada/sopir) dipakai untuk semua realisasi atau hanya kalau kita yang antar? Kalau bakul jemput sendiri (ada plat nomor di master), pengiriman tidak dibuat?
- **`Q-SO-8`** Biaya Admin (per Kg) "Ke Peternak" — ini potongan yang dibayarkan ke peternak mitra? Masuk ke mana di akuntansi?

### Customer & piutang — `Q-CU-*`

- **`Q-CU-1`** Plafon: kalau saldo estimasi + pesanan baru > limit — diblok, atau masuk status *Limit Plafon Diproses* menunggu approval?
- **`Q-CU-2`** TOP: warning "belum bayar selama N hari" — setelah warning, siapa yang approve ("Pusat")? Perlu role khusus?
- **`Q-CU-3`** Tabungan/Kg: uang yang dipotong per kg dari customer dan **disimpan atas nama customer**, bisa dipakai bayar piutang atau di-refund? Apakah berbunga/kadaluarsa?
- **`Q-CU-4`** Cicilan/Kg & Pembekuan Piutang: piutang macet dibekukan, lalu tiap kg pembelian baru dipotong cicilan? Siapa yang berhak membekukan?
- **`Q-CU-5`** Operasional (Budidaya/Pembelian/Breeding): customer hanya bisa dibuat DO di area yang dicentang?
- **`Q-CU-6`** Plat Nomor wajib — semua bakul jemput sendiri? Bagaimana kalau bakul punya >1 truk?

### Faktur & pembayaran — `Q-PAY-*`

- **`Q-PAY-1`** Faktur dibuat per DO (Cetak Faktur) atau gabungan per hari (Faktur DO Gabungan)? Mana yang jadi dasar tagihan resmi?
- **`Q-PAY-2`** Pembayaran diterapkan ke saldo customer (FIFO ke faktur tertua?) atau customer memilih faktur mana?
- **`Q-PAY-3`** Kelebihan bayar → otomatis jadi tabungan, atau refund?
- **`Q-PAY-4`** Verifikasi pembayaran: siapa yang verifikasi, apa yang dicek?

---

## Opsi solusi

Belum diputuskan. Urutan pengerjaan yang masuk akal:

| Tahap | Isi | Menjawab |
|---|---|---|
| **S-A** Master customer | kolom baru (KTP, NPWP, plat, TOP, cicilan/kg, tabungan/kg, plafon toggle, area pendaftaran) + `CustomerBranch` M2M | `Q-CU-5`, `Q-CU-6` |
| **S-B** Pesanan terstruktur | `SalesOrderLine.productId` + `sourceWarehouseId`/`sourceCoopId` + `isCulled` + `savingsPerKg`; header `recipientName`, `recipientAddress`, `validUntil`, `vatPercent`; nomor `DO.<area>.<seq>`; sisa stok tampil | `Q-SO-4/5/6` |
| **S-C** Realisasi | entitas `SalesRealization` + `SalesRealizationLine` (tanggalAmbil, project/kandang, dtpsNumber, qty, tonase, afkir, harga, diskon, biayaAdmin, tabungan, cicilan); stok ayam berkurang di sini; `Delivery` menempel ke realisasi | `Q-SO-1/2/3/7/8` |
| **S-D** Piutang customer | `CustomerLedger` (debit realisasi, kredit pembayaran, tabungan, refund, pembekuan); `SalesPayment` per customer bukan per invoice; faktur gabungan | `Q-CU-1..4`, `Q-PAY-*` |

S-A dan S-B tidak bergantung jawaban stakeholder; S-C butuh `Q-SO-1..3`; S-D butuh hampir semua `Q-CU`/`Q-PAY`.

---

## Belum terverifikasi

- Fungsi toggle "Timbang Kandang".
- Sumber pasti Harga Rekomendasi & Bobot Minimum (isi *Standarisasi Kontrak Ayam Besar* kosong di tenant ini).
- Form Pembekuan Piutang (user tidak punya hak tambah).
- Isi "Cetak Faktur" (PDF).
- Apakah realisasi mengurangi stok ayam di `data_harians`/recording kandang.

## Riwayat revisi

| Tanggal | Perubahan |
|---|---|
| 2026-09-13 | Dibuat. Alur ditelusuri langsung: list, detail, edit realisasi, jual ke peternak, pengiriman, penerimaan uang, refund, pembekuan, master customer (tooltip), master pendukung. |
