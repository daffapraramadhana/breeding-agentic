# Ekspedisi & Surat Jalan — Klasifikasi Supplier, Goods Transfer, Goods Receipt

**Dibuat:** 2026-09-13
**Status:** Implemented (2026-09-13) — lihat plan `docs/superpowers/plans/2026-09-13-logistics-expedition.md`
**Menggantikan:** `2026-04-19-goods-transfer-expedition-design.md` (superseded — asumsi armada internal terbantah oleh jawaban stakeholder `Q-GT-1`=(b))
**Sumber keputusan:** `docs/stakeholder/2026-09-06-pertanyaan-ekspedisi-pengiriman-jawaban.docx`, `docs/superpowers/specs/2026-09-05-logistics-expedition-finding.md`, `docs/reference/programbroiler/`
**Repo terdampak:** `breeding-app` (Prisma, master-data, transfer, procurement), `breeding-dashboard`

---

## Ringkasan keputusan stakeholder yang dipakai

| Kode | Jawaban | Konsekuensi di spec ini |
|---|---|---|
| `Q-GEN-1` (A1) | Semua pengangkut sudah terdaftar sebagai supplier | Pakai `Supplier`, tidak ada tabel carrier baru |
| `Q-GEN-2` (A2) | Nomor surat jalan (+ nama ekspedisi, asumsi) | Tidak ada plat, sopir, tanggal kirim, lead time |
| `Q-GEN-3` (A3) | Tidak perlu evaluasi vendor | Tidak ada laporan; relasi tetap terstruktur karena A1 |
| `Q-GR-1` (B1) | Ongkir barang masuk kita yang tanggung | Ekspedisi perlu ada di goods-receipt |
| `Q-GR-2` (B2) | Jarang >1 ekspedisi per penerimaan | Field di **header**, bukan per baris |
| `Q-GR-3` (B3) | Ongkir = biaya operasional | Tidak ada landed cost |
| `Q-GT-1` (C1) | Pindah barang diangkut vendor ekspedisi | `carrierSupplierId`, bukan `vehicleId`/driver/helper |
| `Q-GT-2` (C2) | Satu pindah barang = satu truk | Ekspedisi + surat jalan di **header** |
| `Q-GT-3` (C3) | **Tidak terjawab** | Asumsi: input manual, opsional, tidak unik — lihat [Asumsi terbuka](#asumsi-terbuka) |
| `Q-GT-4` (C4) | Perlu telusur asal BPB | `GoodsTransferLine.goodsReceiptLineId` |
| `Q-GT-5` (C5) | Dua langkah (disiapkan → berangkat) | Status `PREPARING` + aksi `dispatch` |

Temuan tambahan dari master supplier pembanding (`docs/reference/programbroiler/README.md`): supplier diklasifikasikan ke satu atau lebih **kategori produk**, dan `JASA EXPEDISI` adalah salah satu kategori. Spec ini mengikuti esensi itu: penanda jasa angkut bukan kolom boolean di supplier, melainkan klasifikasi kategori.

## Tujuan

1. Supplier bisa diklasifikasikan per kategori produk; kategori tertentu ditandai sebagai jasa ekspedisi.
2. `GoodsTransfer` mencatat ekspedisi + nomor surat jalan di header, punya tahap `PREPARING` sebelum `IN_TRANSIT`, dan tiap barisnya bisa merujuk BPB asal.
3. `GoodsReceipt` mencatat ekspedisi + nomor surat jalan di header.

## Non-goal

- **Stock movement untuk goods-transfer.** Saat ini transfer tidak menggerakkan stok sama sekali (bukan di create, bukan di receive — `MovementSource.TRANSFER` belum dipakai). Itu gap lama yang dikerjakan sebagai spec terpisah; spec ini tidak menyentuhnya. `PREPARING` di sini bernilai audit (siapa mengizinkan barang keluar), bukan kontrol stok.
- Landed cost / alokasi ongkir ke HPP (`Q-GR-3`=b).
- Evaluasi kinerja vendor (`Q-GEN-3`=b).
- Plat nomor, sopir, kernet, tanggal kirim vs terima (`Q-GEN-2`).
- Ekspedisi di `goods-return`, `internal-trade`, `delivery`. Retur pembelian di pembanding tidak punya ekspedisi.
- Klasifikasi supplier sampai level produk (modal "Kategori Produk" di pembanding). Berhenti di level kategori.
- `LogisticsShippingCost.carrierSupplierId` (R-1) — carrier bisa diturunkan dari dokumen yang ditautkan; tidak disimpan dua kali.
- Ekspedisi per baris (`Q-GR-2`=c, `Q-GT-2`=a).

---

## 1 · Master data — klasifikasi supplier

### Schema

```prisma
model ProductCategory {
  // ... kolom yang ada
  isExpedition Boolean @default(false) @map("is_expedition")

  supplierCategories SupplierCategory[]
}

model Supplier {
  // ... kolom yang ada
  categories       SupplierCategory[]
  goodsTransfersAsCarrier GoodsTransfer[] @relation("TransferCarrier")
  goodsReceiptsAsCarrier  GoodsReceipt[]  @relation("ReceiptCarrier")
}

model SupplierCategory {
  supplierId String @map("supplier_id")
  categoryId String @map("category_id")

  supplier Supplier        @relation(fields: [supplierId], references: [id])
  category ProductCategory @relation(fields: [categoryId], references: [id])

  @@id([supplierId, categoryId])
  @@index([categoryId])
  @@map("supplier_categories")
}
```

`isExpedition` adalah kunci "kategori mana yang jasa angkut". Nama kategori tidak dijadikan kunci karena teks bebas per tenant. `purchasePurpose` yang sudah ada juga teks bebas (`'Production'`, `'Health'`, …) — tidak dipakai.

**Definisi "supplier ekspedisi"**: supplier yang punya minimal satu kategori dengan `isExpedition = true`, `deletedAt = null` pada keduanya.

### Migration

1. Tambah kolom `is_expedition` (default false).
2. Buat tabel `supplier_categories`.
3. Tidak ada backfill. Supplier lama tanpa kategori tetap valid; tidak ada yang otomatis jadi ekspedisi.

### Seed

- Kategori `Jasa Ekspedisi` (`isExpedition: true`).
- Dua supplier baru berkategori `Jasa Ekspedisi` — satu badan usaha, satu perorangan (meniru komposisi daftar pembanding). Nama bebas, kode mengikuti pola seed yang ada.
- Supplier seed lama mendapat kategori sesuai produk yang dipasoknya (Feed, Medicine, …) supaya klasifikasi tidak kosong di data demo.

### Backend — `master-data`

**`ProductCategory`**: `CreateProductCategoryDto` / `UpdateProductCategoryDto` + `isExpedition?: boolean` (`@IsBoolean`, `@IsOptional`). Service meneruskan ke Prisma.

**`Supplier`**:

- `CreateSupplierDto` / `UpdateSupplierDto` + `categoryIds?: string[]` (`@IsArray`, `@IsUUID('4', { each: true })`, `@IsOptional`).
- `create`: validasi semua `categoryIds` milik tenant dan tidak terhapus (satu `findMany` + bandingkan jumlah; kalau kurang → `BadRequestException`). Tulis supplier + `categories.create[]` dalam satu `$transaction`.
- `update`: kalau `categoryIds` dikirim (termasuk `[]`), replace-all: `deleteMany` lalu `createMany`. Kalau tidak dikirim, klasifikasi tidak disentuh.
- `findAll` menerima query tambahan `expedition?: boolean` (DTO baru `QuerySupplierDto extends PaginationDto`, `@IsBooleanString`-style transform). Kalau `true`, tambahkan `where.categories = { some: { category: { isExpedition: true, deletedAt: null } } }`.
- `findAll` dan `findOne` `include: { categories: { include: { category: { select: { id, name, isExpedition } } } } }`.
- `remove` (soft delete) tidak menyentuh join table.

Helper yang dipakai modul lain: `SupplierService.assertExpeditionSupplier(tenantId, supplierId)` — throw `BadRequestException('Supplier is not classified as expedition')` kalau tidak memenuhi definisi di atas, `NotFoundException` kalau tidak ada di tenant. `MasterDataModule` sudah meng-export `SupplierService`; `TransferModule` dan `ProcurementModule` tinggal mengimpor `MasterDataModule`.

### Frontend — `breeding-dashboard`

- **`/product-categories`**: checkbox "Kategori jasa ekspedisi" di dialog create/edit. Kolom tabel kecil (ikon/badge) untuk yang `isExpedition`.
- **`/suppliers`**: bagian "Klasifikasi kategori" di dialog — daftar checkbox dari `/product-categories` (fetch sekali saat dialog dibuka). Kolom tabel menampilkan nama kategori sebagai badge.
- **`SupplierCombobox`** + prop `expeditionOnly?: boolean`. Kalau true: query `expedition=true`, placeholder `t('selectExpedition')`, teks kosong `t('noExpeditionFound')`. Prop opsional supaya pemakai lama tidak berubah.
- `types/api.ts`: `Supplier.categories?: { category: { id; name; isExpedition } }[]`, `ProductCategory.isExpedition: boolean`.
- i18n (`en.json` + `id.json`): `productCategory.isExpedition`, `supplier.categories`, `supplier.selectExpedition`, `supplier.noExpeditionFound`.

---

## 2 · Goods transfer

### Schema

```prisma
enum TransferStatus {
  PREPARING        // baru — default
  IN_TRANSIT
  RECEIVED
  PARTIAL
  DAMAGED_IN_TRANSIT
  CANCELLED
}

model GoodsTransfer {
  // ... kolom yang ada; status default berubah:
  status             TransferStatus @default(PREPARING)
  carrierSupplierId  String?   @map("carrier_supplier_id")
  deliveryNoteNumber String?   @map("delivery_note_number")
  dispatchedAt       DateTime? @map("dispatched_at")
  dispatchedById     String?   @map("dispatched_by_id")

  carrierSupplier Supplier? @relation("TransferCarrier", fields: [carrierSupplierId], references: [id])
  dispatchedBy    User?     @relation("TransferDispatchedBy", fields: [dispatchedById], references: [id])

  @@index([carrierSupplierId])
}

model GoodsTransferLine {
  // ... kolom yang ada
  goodsReceiptLineId String? @map("goods_receipt_line_id")
  goodsReceiptLine   GoodsReceiptLine? @relation(fields: [goodsReceiptLineId], references: [id])

  @@index([goodsReceiptLineId])
}
```

Back-relation: `User.goodsTransfersDispatched GoodsTransfer[] @relation("TransferDispatchedBy")`, `GoodsReceiptLine.transferLines GoodsTransferLine[]` (mirror `consumptionLines`).

**Dibuang dari spec `2026-04-19`**: `vehicleId`, `driverEmployeeId`, `helperEmployeeId`, semua back-relation ke `Vehicle`/`Employee`.

### Migration

1. Tambah enum value `PREPARING`.
2. Ubah default `status` → `PREPARING`. **Record lama tidak diubah** — yang sudah `IN_TRANSIT` tetap `IN_TRANSIT`, field ekspedisi null.
3. Tambah 4 kolom nullable di `goods_transfers`, 1 kolom nullable + index di `goods_transfer_lines`.

### Lifecycle

```
PREPARING ──dispatch──▶ IN_TRANSIT ──receive──▶ RECEIVED
    │                       │                  ▲
    │                       ├──receive──▶ PARTIAL ──▶ RECEIVED
    │                       ├──receive──▶ DAMAGED_IN_TRANSIT
    │                       └──cancel───▶ CANCELLED
    └──cancel──▶ CANCELLED
```

`TRANSFER_STATUS_TRANSITIONS`:

```ts
[TransferStatus.PREPARING]: [TransferStatus.IN_TRANSIT, TransferStatus.CANCELLED],
// baris lain tidak berubah
```

`transitionStatus` generik (`POST /:id/status`) **menolak** target `IN_TRANSIT` dengan `BadRequestException('Use dispatch endpoint')` — satu-satunya jalan ke `IN_TRANSIT` adalah `dispatch`, supaya `dispatchedAt`/`dispatchedById` selalu terisi.

### API — `/api/goods-transfers`

| Method | Path | Perubahan |
|---|---|---|
| `POST` | `/` | Status hasil = `PREPARING`. DTO + `carrierSupplierId?`, `deliveryNoteNumber?`; `lines[].goodsReceiptLineId?`. |
| `PATCH` | `/:id` | Hanya saat `PREPARING`, selain itu `BadRequestException('Transfer can only be edited while preparing')`. Bisa mengubah `carrierSupplierId`, `deliveryNoteNumber`. |
| `POST` | `/:id/dispatch` | **Baru.** Lihat di bawah. |
| `POST` | `/:id/status` | Tambah guard: target `IN_TRANSIT` ditolak. |
| `GET` | `/`, `/:id` | Include `carrierSupplier { id, name }`, `dispatchedBy { id, name }`, `lines.goodsReceiptLine { id, goodsReceipt { receiptNumber } }`. List menerima filter `status` (sudah ada pola pagination; tambah kalau belum). |

**`DispatchGoodsTransferDto`**

```ts
{
  carrierSupplierId?: string;   // @IsUUID, opsional — menimpa yang di record
  deliveryNoteNumber?: string;  // @IsString @MaxLength(100), opsional — menimpa yang di record
}
```

**Logika `dispatch(tenantId, id, dto, user)`**, dalam satu `$transaction`:

1. `findOne` (tenant-scoped). Assert `status === PREPARING`, else `BadRequestException`.
2. Assert `lines.length >= 1`.
3. `carrierSupplierId = dto.carrierSupplierId ?? transfer.carrierSupplierId`. Kalau null → `BadRequestException('Expedition is required to dispatch')`.
4. `SupplierService.assertExpeditionSupplier(tenantId, carrierSupplierId)`.
5. Update: `status = IN_TRANSIT`, `carrierSupplierId`, `deliveryNoteNumber = dto.deliveryNoteNumber ?? transfer.deliveryNoteNumber`, `dispatchedAt = now()`, `dispatchedById = user.sub`.
6. **Tidak ada stock movement** (lihat Non-goal).

Validasi `carrierSupplierId` di `create`/`update` memakai helper yang sama — supplier non-ekspedisi ditolak sejak awal, bukan baru saat dispatch.

Validasi `lines[].goodsReceiptLineId` di `create`: mirror `GoodsConsumptionService` — line harus ada, tenant sama, dan `goodsReceipt.warehouseId === dto.fromWarehouseId`. Kalau consumption belum memvalidasi gudang, tambahkan di transfer saja; jangan ubah consumption di spec ini.

### Permissions

| Aksi | STAFF | MANAGER | TENANT_ADMIN | SUPER_ADMIN |
|---|---|---|---|---|
| Create / edit `PREPARING` | ✅ | ✅ | ✅ | ✅ |
| Dispatch | ❌ | ✅ | ✅ | ✅ |
| Receive / cancel | mengikuti guard `transitionStatus` yang ada | | | |

Guard dispatch memakai `@Roles(...)` yang sudah dipakai controller lain.

### Frontend

- **`/goods-transfers/new`**: card header + `SupplierCombobox expeditionOnly` (label "Ekspedisi") + `Input` "No. surat jalan"; keduanya opsional. `LineItemsField` diaktifkan `showGoodsReceiptRef showLineNotes warehouseId={form.fromWarehouseId}`. Kalau `fromWarehouseId` berubah, `goodsReceiptLineId` di semua baris di-reset.
- **`/goods-transfers/[id]`**: card **Ekspedisi** di antara header dan tabel baris.
  - `PREPARING`: tampilkan ekspedisi & surat jalan saat ini (atau "—"), tombol **Edit** (buka form edit) dan **Berangkatkan** (primary, hanya MANAGER+). Modal dispatch: `SupplierCombobox expeditionOnly` (prefilled) + input surat jalan (prefilled). Submit → `POST /:id/dispatch` → refresh.
  - `IN_TRANSIT` ke atas: read-only — ekspedisi, surat jalan, dispatched at, dispatched by.
  - Tabel baris: kolom "BPB asal" (nomor penerimaan) kalau ada. Read-only saat status ≠ `PREPARING`.
  - Tombol Receive hanya muncul saat `IN_TRANSIT`/`PARTIAL`.
- **`/goods-transfers`**: filter status + badge `PREPARING` (warna mengikuti `Delivery.PREPARING` kalau sudah ada komponen badge status; kalau belum, pakai variant `secondary`).
- i18n: `goodsTransfer.status.preparing`, `goodsTransfer.expedition.{title,carrier,deliveryNote,dispatchedAt,dispatchedBy,none}`, `goodsTransfer.action.dispatch`, `goodsTransfer.dispatchModal.{title,submit,carrierRequired}`, `goodsTransfer.line.sourceReceipt`.
- `types/api.ts`: `TransferStatus` + `'PREPARING'`; `GoodsTransfer` + `carrierSupplierId`, `carrierSupplier?`, `deliveryNoteNumber?`, `dispatchedAt?`, `dispatchedBy?`; `GoodsTransferLine` + `goodsReceiptLineId?`, `goodsReceiptLine?`.

---

## 3 · Goods receipt

### Schema

```prisma
model GoodsReceipt {
  // ... kolom yang ada
  carrierSupplierId  String? @map("carrier_supplier_id")
  deliveryNoteNumber String? @map("delivery_note_number")

  carrierSupplier Supplier? @relation("ReceiptCarrier", fields: [carrierSupplierId], references: [id])

  @@index([carrierSupplierId])
}
```

Keduanya opsional: record lama harus tetap valid, dan ada kasus barang dijemput sendiri tanpa vendor.

### Migration

Dua kolom nullable + satu index. Tidak ada backfill.

### Backend — `procurement`

- `CreateGoodsReceiptDto` + `carrierSupplierId?` (`@IsUUID`), `deliveryNoteNumber?` (`@IsString @MaxLength(100)`).
- `create`: kalau `carrierSupplierId` ada → `SupplierService.assertExpeditionSupplier`. Simpan keduanya.
- `findAll` / `findOne`: include `carrierSupplier { id, name }`.
- `LogisticsShippingCost` **tidak berubah**.

### Frontend

- **`/goods-receipts/new`**: di card header, setelah Tanggal Terima: `SupplierCombobox expeditionOnly` (label "Ekspedisi") + input "No. surat jalan". Keduanya opsional.
- **`/goods-receipts/[id]`**: tampilkan ekspedisi & surat jalan di header detail (atau "—").
- i18n: `goodsReceipt.carrier`, `goodsReceipt.deliveryNote`.
- `types/api.ts`: `GoodsReceipt` + `carrierSupplierId?`, `carrierSupplier?`, `deliveryNoteNumber?`.

---

## 4 · Pengujian

### Backend (`breeding-app`)

Supplier / kategori:
- `GET /suppliers?expedition=true` hanya mengembalikan supplier berkategori `isExpedition`; supplier dengan kategori ekspedisi yang sudah soft-deleted tidak ikut.
- `POST /suppliers` dengan `categoryIds` berisi id tenant lain → 400.
- `PATCH /suppliers/:id` dengan `categoryIds: []` mengosongkan klasifikasi; tanpa `categoryIds` tidak mengubah.

Goods transfer:
- Create → status `PREPARING`; `carrierSupplierId` non-ekspedisi → 400.
- Create dengan `goodsReceiptLineId` dari gudang lain → 400.
- Dispatch happy path: status `IN_TRANSIT`, `dispatchedAt`/`dispatchedById` terisi, `carrierSupplierId` dari body menimpa record.
- Dispatch ditolak: status ≠ `PREPARING`; tanpa line; tanpa carrier (record & body kosong); carrier non-ekspedisi; carrier tenant lain; role STAFF.
- `POST /:id/status` dengan target `IN_TRANSIT` → 400.
- `PATCH /:id` setelah dispatch → 400.
- Cancel dari `PREPARING` → `CANCELLED`.
- Tidak ada `InventoryMovement` tercipta di create/dispatch/receive (mengunci non-goal).

Goods receipt:
- Create dengan `carrierSupplierId` non-ekspedisi → 400; tanpa carrier → OK.

### Frontend

- `SupplierCombobox expeditionOnly` tidak menampilkan supplier pakan (mock API).
- Tombol Berangkatkan hanya saat `PREPARING` dan role MANAGER+.
- Card Ekspedisi read-only setelah dispatch.
- Reset `goodsReceiptLineId` saat gudang asal berubah.

---

## 5 · Dampak ke dokumen lain

| Dokumen | Perubahan |
|---|---|
| `2026-04-19-goods-transfer-expedition-design.md` | Status → **Superseded**, pointer ke spec ini. Tidak dihapus. |
| `2026-09-05-logistics-expedition-finding.md` | Jawaban diisi, status 10/11, F-1 direvisi (penanda lewat kategori), M-3 `goods-return` ditutup. |
| `docs/reference/programbroiler/README.md` | Sudah memuat temuan master supplier. |

## Asumsi terbuka

Dua asumsi menunggu konfirmasi stakeholder. Keduanya **tidak mengubah schema** kalau ternyata beda — hanya validasi dan UI.

| Asumsi | Kalau ternyata beda |
|---|---|
| `deliveryNoteNumber` input manual, opsional, tidak unik (C3 kosong) | Wajib → tambah validasi `@IsNotEmpty` di dispatch + required di UI. Unik → tambah `@@unique([tenantId, deliveryNoteNumber])` — satu migration kecil. Auto-generate → pakai `ReferenceNumberGenerator` prefix `SJ`; kolom sama. |
| Nama ekspedisi tetap dicatat meski A2 hanya mencentang surat jalan | Kalau tidak perlu → `carrierSupplierId` dibiarkan null, combobox disembunyikan. Kolom tidak dibuang. |

Item follow-up lain ke stakeholder (kolom `Transport`, `Pakai Di Kandang`, master Armada, klasifikasi `BERKAH BREEDING FARM`) tercatat di finding doc dan `docs/reference/programbroiler/README.md`; tidak memblokir spec ini.

## Urutan implementasi

1. Bagian 1 (master data) — prasyarat keduanya.
2. Bagian 2 (goods transfer).
3. Bagian 3 (goods receipt).

Tiap bagian: schema + migration → backend + test → frontend → `npm run build` backend sebelum commit. Sub-repo `breeding-app` dan `breeding-dashboard` di-commit terpisah dengan scope `feat(master-data):`, `feat(transfer):`, `feat(procurement):`.
