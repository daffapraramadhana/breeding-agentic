# Warehouse Form Redesign

**Date:** 2026-03-29
**Status:** Approved

## Summary

Redesign the warehouse CRUD UI from dialog-based to dedicated full-page forms, add an `address` field to the schema, and replace the owner type dropdown with radio buttons — matching the legacy system's "Input Data Gudang" screen.

## Changes

### 1. Backend — Prisma Schema

Add `address` field to the Warehouse model:

```prisma
model Warehouse {
  // ... existing fields
  address   String?            @map("address")
  // ... rest
}
```

Run migration after schema change.

### 2. Backend — DTO

Update `CreateWarehouseDto` to include optional `address`:

```typescript
@IsOptional()
@IsString()
address?: string;
```

`UpdateWarehouseDto` already extends `PartialType(CreateWarehouseDto)`, so no change needed there.

### 3. Backend — Service

No changes needed — the service already spreads `...dto` on update, and the create method just needs `address` added to the `data` object.

### 4. Frontend — Type Update

Add `address?: string` to the `Warehouse` interface in `src/types/api.ts`.

### 5. Frontend — New Page: `/warehouses/new/page.tsx`

Dedicated full-page create form with:

- **PageHeader** with title "Create Warehouse" and back button to `/warehouses`
- **Card** containing the form:
  - **Code** — text input, required
  - **Name** — text input, required
  - **Address** — text input, optional
  - **Owner Type** — radio button group (Branch / Farm), not a dropdown
    - Only show BRANCH and FARM options (COOP is rare, can be added later)
    - When radio selection changes, clear the owner combobox value
  - **Owner** — combobox that dynamically loads branches or farms based on radio selection
    - Uses `EntityCombobox` with endpoint `/branches` or `/farms`
    - The selected owner also determines `branchId`:
      - If owner type = BRANCH, `branchId` = the selected branch ID
      - If owner type = FARM, `branchId` = the farm's `branchId` (fetched from API)
- **Submit button** — POSTs to `/warehouses`, redirects to `/warehouses` list on success

### 6. Frontend — Edit Page: `/warehouses/[id]/page.tsx`

Detail/edit page for existing warehouses:

- Fetches warehouse by ID via `GET /warehouses/:id`
- Same form layout as create page, pre-populated with existing values
- Submit button PATCHes to `/warehouses/:id`
- Back button returns to `/warehouses`

### 7. Frontend — List Page Update

Modify existing `/warehouses/page.tsx`:

- Remove the create/edit Dialog entirely
- "Add Warehouse" button links to `/warehouses/new`
- Table row edit button links to `/warehouses/[id]`
- Keep delete confirmation dialog
- Add `address` column to the table (optional, show if present)

### 8. Frontend — i18n

Add to both `en.json` and `id.json`:

```json
// en.json additions
"address": "Address",
"addressPlaceholder": "Warehouse address",
"selectOwnerType": "Select owner type",
"backToList": "Back to warehouses"

// id.json additions
"address": "Alamat",
"addressPlaceholder": "Alamat gudang",
"selectOwnerType": "Pilih tipe pemilik",
"backToList": "Kembali ke daftar gudang"
```

## Out of Scope

- COOP as an owner type radio option (can be added later if needed)
- Warehouse detail/view page (the `[id]` page is edit-only for now)
- Changes to `WarehouseCombobox` used in other modules
