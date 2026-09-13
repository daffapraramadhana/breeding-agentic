# Referensi — aplikasi pembanding `app1.programbroiler.com`

Screenshot form dari aplikasi yang dipakai di lapangan. **Bukan untuk ditiru plek** — yang diambil esensinya: field apa yang ada, di level mana (header / per baris), dan alur apa yang mereka anggap penting.

Konteks analisis: `docs/superpowers/specs/2026-09-05-logistics-expedition-finding.md`.
Jawaban stakeholder: `docs/stakeholder/`.

## Indeks

| File | Menu | Modul kita | Esensi |
|---|---|---|---|
| `2026-09-13-po-input-detail-pemesanan.png` | Logistik › Input Detail Data Pemesanan | `purchase-order` | Header: Area, nomor otomatis, tanggal, supplier, keterangan. **Tujuan pengiriman per baris produk** (radio Area / Farm / Kandang Ownfarm → gudang). Satu PO bisa dikirim ke beberapa gudang. |
| `2026-09-13-goods-receipt-list-data-penerimaan-produk.png` | Logistik › Data Penerimaan Produk (list) | `goods-receipt` | Filter: pencarian, Area, Gudang Tujuan, Periode Terima, Kategori — masing-masing dengan checkbox "Seluruh". Default periode ±2 bulan ke belakang. |
| `2026-09-13-goods-transfer-list-pindah-barang.png` | Logistik › Pindah Barang (list) | `goods-transfer` | Filter sama dengan penerimaan minus Kategori. Label periode "Distribusi". |
| `2026-09-13-goods-transfer-input-pindah-barang.png` | Logistik › Input Pindah Barang | `goods-transfer` | Header: tanggal, Area, gudang asal & tujuan (radio 3 jenis). **Per baris**: Pengambilan barang (radio `Persediaan`), produk, **No. Referensi = BPB**, kuantitas, **Ekspedisi** (placeholder "Pilih Armada/Ekspedisi"), **No Surat Jalan**, keterangan. Section `EKSPEDISI` kosong di atas tabel. Kolom tabel: Nama Produk, Kuantitas, Ekspedisi, Transport, No Surat Jalan, Keterangan, Pakai Di Kandang. |
| `2026-09-13-goods-consumption-input-pemakaian-barang.png` | Logistik › Input Data Pemakaian Barang | `goods-consumption` | Header: Area, nomor otomatis, tanggal, keterangan, **tujuan pemakaian** (radio Area / Farm / Kandang Ownfarm). Per baris: produk, **No. Referensi = BPB**, kuantitas, keterangan. Tidak ada ekspedisi — barang dipakai di tempat. |
| `2026-09-13-supplier-input-data-suplier.png` | Master › Input Data Suplier | `supplier` | Kode, nama, alamat, no. rekening + atas nama, keterangan, **hari jatuh tempo** (default 30). **Klasifikasi Kategori Produk**: 10 kategori (DOC BROILER, DOC JANTAN, OVK, PAKAN, PERLENGKAPAN PRODUKSI, PENGELUARAN OPERASIONAL, FIXED ASSET, AYAM BESAR, JASA UMUM, **JASA EXPEDISI**) — supplier diklasifikasikan ke satu atau lebih kategori. |
| `2026-09-13-supplier-input-modal-kategori-produk.png` | Master › Input Data Suplier › modal Kategori Produk | `supplier` | Klik `+` pada kategori (di sini OVK) membuka modal daftar produk per sub-kategori (OBAT, VAKSIN) dengan checkbox + "Pilih Semua". Jadi klasifikasi supplier turun sampai **produk mana saja yang dipasok** supplier ini. |
| `2026-09-13-goods-return-input-retur-pembelian.png` | Logistik › Input Data Retur Pembelian | `goods-return` | Header: Area, tanggal, keterangan, **PO** (hanya yang belum ada pembayaran supplier), **BPB** (hanya yang belum ada pemakaian), nama barang. Tidak ada ekspedisi & surat jalan di form ini. |

## Pola yang berulang di semua form

- **Pemilihan gudang selalu radio 3 jenis** (Area / Farm / Kandang Ownfarm) + dropdown. Di sistem kita sudah setara lewat `WarehouseGroupedCombobox` — tidak perlu ditiru.
- **Referensi BPB per baris** muncul di pindah barang, pemakaian, dan retur. Rantai telusur: PO → BPB → (pindah | pakai | retur).
- **Nomor dokumen otomatis** ("Otomatis") — sama dengan `ReferenceNumberGenerator` kita.
- **Ekspedisi + surat jalan hanya ada di pindah barang** (dan penerimaan PO, lihat finding doc). Retur dan pemakaian tidak punya.

## Temuan penting dari master supplier

`JASA EXPEDISI` adalah salah satu **kategori produk** yang bisa ditempelkan ke supplier. Jadi pembanding sebenarnya *punya* penanda jasa angkut — lewat klasifikasi kategori, bukan kolom khusus. Ini merevisi pembacaan F-1 di finding doc: kemungkinan dropdown Ekspedisi memang difilter kategori `JASA EXPEDISI`, dan `BERKAH BREEDING FARM` muncul karena **sengaja** diklasifikasikan sebagai jasa ekspedisi (armada farm sendiri ditagihkan sebagai supplier). Belum terverifikasi — perlu dicek isi klasifikasi `BERKAH BREEDING FARM` di master mereka.

## Hal yang belum jelas dari screenshot

Terkumpul di finding doc bagian *Belum terverifikasi*; ringkasnya: arti kolom `Transport` dan `Pakai Di Kandang`, isi section `EKSPEDISI` saat ada data, opsi lain radio `Pengambilan barang`.

## Menambah screenshot

Nama file: `YYYY-MM-DD-<modul-kita>-<input|list|detail>-<judul-form-pembanding>.png`. Tambahkan baris ke tabel indeks.
