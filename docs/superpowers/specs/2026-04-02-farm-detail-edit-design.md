# Farm Detail Page — Edit with Coops & Floors

## Problem

When editing a farm, users cannot see or manage the related coops (kandang) and floors (blok) from the same page. They must navigate to separate `/coops` and `/coop-floors` pages. This breaks the mental model: a farm is a unit that contains coops, which contain floors.

## Solution

Enhance the existing `/farms/[id]` detail page to become the edit hub for a farm and all its child entities. Users can view, edit, add, and delete coops and floors directly from this page.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Where to edit | Enhanced detail page (`/farms/[id]`) | Users already navigate here. No extra route needed. |
| Layout | Accordion pattern | Farm → Coop → Floor is a tree. Accordion maps naturally to it and scales well with many coops. |
| Edit interaction | Modal dialogs | Consistent with existing coop/floor edit patterns across the app. |
| Add interaction | Modal dialogs | Same pattern as editing — "+ Add" button opens a create modal. |
| Delete interaction | Confirmation dialog with dependency warnings | Soft-delete. Shows warning if entity is referenced by active projects. |
| Save strategy | Per-action (immediate) | Each edit/add/delete saves immediately via existing API endpoints. No batch save needed. |

## Page Structure

### 1. Farm Info Card (top)

Always visible at the top of the page. Shows:

- Farm name (large heading)
- Address, branch name, status badge (`OWN`/`COOP`), farm type
- **"Edit Farm" button** — opens modal with: Branch, Name, Address, Farm Type, Status
- Summary stats row:
  - Total kandang count
  - Total floor count
  - Total capacity (sum of all coop capacities)

The existing `PageHeader` with back button remains. The info card replaces the current 3 small cards (Address, Coops count, Created date) with a single, richer card.

**Edit Farm Modal fields:** Branch (BranchCombobox), Name (required), Address, Farm Type, Status (OWN/COOP). Same fields as current edit dialog on `/farms` list page. Saves via `PATCH /farms/:id`.

### 2. Coops Section

Below the farm info card. Contains:

- Section header: "Kandang" with a **"+ Tambah Kandang"** button on the right
- List of coop accordion items

#### Coop Accordion Item (collapsed)

Shows in a single row:
- Expand/collapse chevron icon
- Coop code (bold) + name
- Status badge (ACTIVE/INACTIVE/MAINTENANCE)
- Capacity
- Floor count (e.g., "3 floors")
- Edit button — opens coop edit modal
- Delete button — opens delete confirmation

#### Coop Accordion Item (expanded)

When expanded, shows the coop's floors as a table below the header row:

| Column | Source |
|--------|--------|
| Kode | `code` |
| Nama | `name` |
| Luas (m²) | `area` |
| Populasi/m² | `population` |
| Max Populasi | `maxPopulation` (auto-calculated) |
| Status | Status badge |
| Aksi | Edit + Delete buttons |

Below the table: **"+ Tambah Floor"** button (dashed outline, full width).

Only one coop should be expanded at a time (single-expand accordion).

**Add Coop Modal fields:** Code (required), Name (required), Capacity (required), Status, Description. Branch and Farm are inherited from the parent farm (not shown in the form). Saves via `POST /coops` with `farmId` and `branchId` auto-populated.

**Edit Coop Modal fields:** Same as add, pre-filled with existing values. Saves via `PATCH /coops/:id`.

### 3. Floor Modals

**Add Floor Modal fields:** Code (required), Name (required), Populasi/m² (required), Luas/Area m² (required), Max Populasi (auto-calculated, read-only), Status, Description. Coop, Farm, and Branch are inherited. Saves via `POST /coop-floors` with `coopId`, `farmId`, and `branchId` auto-populated.

**Edit Floor Modal fields:** Same as add, pre-filled. Saves via `PATCH /coop-floors/:id`.

### 4. Delete Behavior

**Simple case (no dependencies):**
- Confirmation dialog: "Are you sure you want to delete [entity name]?"
- For coops: "This kandang has X floors that will also be deleted."
- Soft-delete via `DELETE /coops/:id` or `DELETE /coop-floors/:id`

**With dependencies (active projects):**
- Warning dialog: "This kandang is used in X active projects. Deleting it may affect ongoing operations."
- Requires extra confirmation or blocks deletion depending on backend response
- The backend should return a 409 Conflict or similar if the entity cannot be deleted

## Data Fetching

### Backend Changes

The `GET /farms/:id` endpoint needs to include coops with their floors. Currently it returns coops but not nested floors.

**Option:** Add a query parameter `?include=coops.floors` or always include floors when fetching a single farm. The response shape:

```typescript
{
  data: {
    id: string;
    name: string;
    // ... other farm fields
    coops: Array<{
      id: string;
      code: string;
      name: string;
      capacity: number;
      status: CoopStatus;
      // ... other coop fields
      floors: Array<{
        id: string;
        code: string;
        name: string;
        area: number;
        population: number;
        maxPopulation: number;
        status: CoopStatus;
        // ... other floor fields
      }>;
    }>;
  }
}
```

### Frontend Data Flow

- Page fetches farm data via `useApi<Farm>('/farms/${id}')` (existing)
- After any mutation (add/edit/delete coop or floor), call `refetch()` to reload the full farm with updated coops/floors
- No separate fetches for coops or floors — all nested in the farm response

## Components to Create/Modify

### New Components

1. **`FarmInfoCard`** — farm details display with edit button and summary stats
2. **`CoopAccordion`** — accordion list of coops, each expandable to show floors
3. **`CoopAccordionItem`** — single coop row with expand/collapse, showing floors table when expanded
4. **`FloorTable`** — table of floors inside an expanded coop
5. **`CoopFormDialog`** — modal for creating/editing a coop (no farm/branch selection needed)
6. **`FloorFormDialog`** — modal for creating/editing a floor (no coop/farm/branch selection needed)

### Modified Files

- **`breeding-dashboard/src/app/(dashboard)/farms/[id]/page.tsx`** — replace current layout with new component composition
- **`breeding-app/src/modules/farm/farm.service.ts`** — ensure `findOne` includes `coops` with nested `floors`
- **`breeding-dashboard/src/types/api.ts`** — ensure `Farm` type includes `coops` with nested `floors` (may already be correct)

### Existing Components Reused

- `Dialog`, `DialogContent`, etc. from shadcn/ui
- `ConfirmDialog` from `@/components/shared/confirm-dialog`
- `StatusBadge` from `@/components/shared/status-badge`
- `PageHeader` from `@/components/shared/page-header`
- `Input`, `Label`, `Select`, `Button` from shadcn/ui

## API Endpoints Used

| Action | Method | Endpoint | Notes |
|--------|--------|----------|-------|
| Fetch farm + coops + floors | GET | `/farms/:id` | Backend must include nested coops.floors |
| Edit farm | PATCH | `/farms/:id` | Existing |
| Add coop | POST | `/coops` | Auto-populate farmId, branchId |
| Edit coop | PATCH | `/coops/:id` | Existing |
| Delete coop | DELETE | `/coops/:id` | Existing, soft-delete |
| Add floor | POST | `/coop-floors` | Auto-populate coopId, farmId, branchId |
| Edit floor | PATCH | `/coop-floors/:id` | Existing |
| Delete floor | DELETE | `/coop-floors/:id` | Existing, soft-delete |

No new API endpoints needed. Only the `GET /farms/:id` response needs to be enriched with nested data.

## Out of Scope

- Drag-and-drop reordering of coops or floors
- Moving a coop to a different farm from this page
- Bulk operations (delete multiple, status change multiple)
- The create farm wizard — it stays as-is
- The `/coops` and `/coop-floors` list pages — they stay as-is
