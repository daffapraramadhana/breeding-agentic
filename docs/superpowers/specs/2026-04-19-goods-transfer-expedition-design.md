# Goods Transfer — Expedition Data & Dispatch Lifecycle

**Date:** 2026-04-19
**Status:** Design
**Scope:** `breeding-app` (backend), `breeding-dashboard` (frontend), Prisma schema

---

## Problem

`GoodsTransfer` currently starts in `IN_TRANSIT` with no record of who/what is shipping it. There is no vehicle, driver, dispatch time, or dispatching user captured — so once a transfer is `IN_TRANSIT`, the system cannot answer *"which truck, which driver, when did it leave, who authorized it?"*.

`Delivery` (sales-side) already solves this: it has `vehicleId`, `driverEmployeeId`, `helperEmployeeId`, `destinationCity`, `DeliveryShippingCost`, and a `PREPARING → IN_DELIVERY → RECEIVED` lifecycle. `GoodsTransfer` should be consistent.

## Goal

Add **internal-fleet expedition tracking** to `GoodsTransfer` via a dedicated dispatch step, mirroring `Delivery`'s pattern.

## Non-goals

- No third-party-logistics (3PL) / carrier / forwarder master data.
- No surat jalan, resi, AWB, or tracking-number fields.
- No GPS / live tracking.
- No multi-transfer-per-truck batching (one truck = one transfer).
- Driver bonus stays on the existing manual `DriverBonus` flow.

## Lifecycle

```
PREPARING (new, default for new transfers)
  │
  ├── dispatch ──▶ IN_TRANSIT
  │                  │
  │                  ├── receive ──▶ RECEIVED
  │                  ├── receive ──▶ PARTIAL ──▶ RECEIVED
  │                  ├── receive ──▶ DAMAGED_IN_TRANSIT (terminal)
  │                  └── cancel  ──▶ CANCELLED (terminal)
  │
  └── cancel ─────▶ CANCELLED (terminal)
```

`PREPARING` is the new default status for newly created transfers. `IN_TRANSIT` can no longer be set directly by create — only by the dispatch action.

## Schema changes

### Enum
Add `PREPARING` to `TransferStatus`.

### `GoodsTransfer` — new fields

| Field | Type | Nullable | Required when |
|---|---|---|---|
| `vehicleId` | `String` → `Vehicle.id` | yes | must be set to transition `PREPARING` → `IN_TRANSIT` |
| `driverEmployeeId` | `String` → `Employee.id` | yes | must be set to transition `PREPARING` → `IN_TRANSIT` |
| `helperEmployeeId` | `String` → `Employee.id` | yes | never required (optional helper) |
| `dispatchedAt` | `DateTime` | yes | set to `now()` on dispatch |
| `dispatchedById` | `String` → `User.id` | yes | set to current user on dispatch |

All nullable so legacy rows remain valid. Unique constraints: none new.

### `Vehicle` and `Employee` — reverse relations
Add back-relations `Vehicle.goodsTransfers GoodsTransfer[]`, `Employee.goodsTransfersAsDriver GoodsTransfer[]`, `Employee.goodsTransfersAsHelper GoodsTransfer[]`.

### Migration
- Add enum value `PREPARING`.
- Add five nullable columns on `goods_transfer` table.
- No data backfill — existing rows keep current status and null expedition fields.

## Backend API (`/api/goods-transfers`)

All endpoints tenant-scoped via existing guards.

| Method | Path | Purpose | Status transition | Required role |
|---|---|---|---|---|
| `POST` | `/` | Create draft transfer | → `PREPARING` | `STAFF`, `MANAGER`, `TENANT_ADMIN` |
| `GET` | `/` | List with filters (status, warehouses, dates) | — | any authed |
| `GET` | `/:id` | Detail incl. lines, expedition, costs | — | any authed |
| `PATCH` | `/:id` | Edit draft only | stays `PREPARING` | `STAFF`, `MANAGER`, `TENANT_ADMIN` |
| `POST` | `/:id/dispatch` | Assign vehicle/driver, ship it | `PREPARING` → `IN_TRANSIT` | `MANAGER`, `TENANT_ADMIN` |
| `POST` | `/:id/receive` | Record receipt | `IN_TRANSIT` → `RECEIVED` / `PARTIAL` / `DAMAGED_IN_TRANSIT` | `MANAGER`, `TENANT_ADMIN` |
| `POST` | `/:id/cancel` | Cancel | any non-terminal → `CANCELLED` | `MANAGER`, `TENANT_ADMIN` |

### `DispatchGoodsTransferDto`

```ts
{
  vehicleId: string;          // required, must exist in tenant, not soft-deleted
  driverEmployeeId: string;   // required, must exist in tenant, not soft-deleted
  helperEmployeeId?: string;  // optional
}
```

Service logic for dispatch:
1. Load transfer; assert `status === PREPARING` and `tenantId` match.
2. Validate `vehicleId`, `driverEmployeeId`, `helperEmployeeId` all resolve within tenant (active, not deleted).
3. If `helperEmployeeId` is provided, assert `helperEmployeeId !== driverEmployeeId`.
4. Assert transfer has at least one line.
5. Within a transaction: update transfer (`status = IN_TRANSIT`, `vehicleId`, `driverEmployeeId`, `helperEmployeeId`, `dispatchedAt = now()`, `dispatchedById = user.id`) **and** fire the outbound stock movement (see Stock-movement section).

### `PATCH /:id` guardrail
Rejects the update if `status !== PREPARING`. Prevents editing lines or warehouses after dispatch.

### `POST /:id/receive` — unchanged contract
Keeps current payload (`lines[].quantityReceived`, `lines[].quantityDamaged`, `actualReceiptDate`, optional `notes`). Only the guard changes: must be `IN_TRANSIT` or `PARTIAL` (not `PREPARING`).

## Stock-movement timing (critical)

Current behavior **must be audited during implementation**. The new flow requires:

| Event | Stock effect |
|---|---|
| Create (`PREPARING`) | None — this is a plan. |
| Dispatch (`IN_TRANSIT`) | Decrement `fromWarehouse` stock. Optionally record in an in-transit bucket. |
| Receive (`RECEIVED`/`PARTIAL`) | Increment `toWarehouse` stock by received qty; write-off damaged qty. |
| Cancel from `PREPARING` | None. |
| Cancel from `IN_TRANSIT` | Reverse the dispatch decrement (restore `fromWarehouse` stock). |

**Open item for implementation plan:** inspect `GoodsTransferService` and any stock-movement/`InventoryLedger` writer to confirm whether stock currently moves on create. If it does, that call has to migrate to the dispatch handler.

## Frontend (Next.js dashboard)

### List page — `/goods-transfers`
- Status filter includes new `PREPARING`.
- Status badge colors follow existing Delivery palette (reuse component if one exists).
- Row-level quick action "Dispatch" visible when status = `PREPARING` and user has permission.

### Create page — `/goods-transfers/new`
Unchanged fields. Submit saves as `PREPARING`. No vehicle/driver pickers on this page.

### Detail page — `/goods-transfers/[id]`
New **Expedition card** between the header and the lines table:
- `PREPARING`: card shows placeholder text + **Dispatch** button (primary).
- `IN_TRANSIT` or later: card is read-only and shows Vehicle name + plate, Driver name, Helper name (or "—"), Dispatched at, Dispatched by.

Lines table becomes read-only when status ≠ `PREPARING`.

### Dispatch modal
Triggered by the Dispatch button. Three comboboxes (reuse existing shared combobox components under `breeding-dashboard/src/components/forms/`):
- **Vehicle** — lists tenant's active vehicles, shows `name · plateNumber`.
- **Driver** — lists tenant's active employees.
- **Helper** — same list, optional; blocks selecting the same employee as driver.

Submit calls `POST /:id/dispatch`. On success: close modal, refresh detail page, status flips to `IN_TRANSIT`.

### Receive flow
Already exists. Only change: entry point becomes visible only when status is `IN_TRANSIT` or `PARTIAL`.

### i18n
New keys in both `en.json` and `id.json`:
- `goodsTransfer.status.preparing`
- `goodsTransfer.expedition.title`
- `goodsTransfer.expedition.vehicle`, `.driver`, `.helper`, `.dispatchedAt`, `.dispatchedBy`
- `goodsTransfer.action.dispatch`
- `goodsTransfer.dispatchModal.*` (title, vehicle label, driver label, helper label, submit, validation messages)

### API types
Sync `breeding-dashboard/src/types/api.ts` with new DTOs and extended `GoodsTransfer` shape.

## Permissions

Reuse `Delivery` module's guard logic:

| Action | STAFF | MANAGER | TENANT_ADMIN | SUPER_ADMIN |
|---|---|---|---|---|
| Create draft / edit `PREPARING` | ✅ | ✅ | ✅ | ✅ |
| Dispatch | ❌ | ✅ | ✅ | ✅ |
| Receive | ❌ | ✅ | ✅ | ✅ |
| Cancel | ❌ | ✅ | ✅ | ✅ |

## Testing

Backend (`breeding-app`):
- Create → defaults to `PREPARING`, no stock movement.
- Dispatch happy path: status flips, stock decrements, `dispatchedAt`/`dispatchedById` set.
- Dispatch rejected when status ≠ `PREPARING`.
- Dispatch rejected when vehicle/driver missing, cross-tenant, or soft-deleted.
- Dispatch rejected when driver === helper.
- Receive rejected when status = `PREPARING`.
- Cancel from `PREPARING` → no stock movement.
- Cancel from `IN_TRANSIT` → stock restored on `fromWarehouse`.
- PATCH rejected after dispatch.

Frontend:
- Dispatch button visible only in `PREPARING`.
- Expedition card renders placeholder in `PREPARING`, read-only values in `IN_TRANSIT`+.
- Lines table read-only after dispatch.
- Dispatch modal blocks submit when required fields empty or driver === helper.

## Open items for the implementation plan

1. Audit `GoodsTransferService.create` and any inventory-ledger writers to confirm where stock movement currently fires, and move it to the dispatch handler.
2. Confirm which role guards / decorators the `Delivery` module uses so the transfer module can reuse them.
3. Confirm whether there is a shared `StatusBadge` component for delivery status that can accept transfer status.
4. Decide whether legacy transfers (already `IN_TRANSIT` with null vehicle/driver) should surface any warning in the UI or silently display "—".
