# Dashboard & Layout Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the entire Breeding Dashboard UI from a traditional shadcn template to a sleek glassmorphic editorial bento design with light/dark theme support.

**Architecture:** CSS-first approach — update theme variables and base component styles so most pages inherit the new look automatically. Then redesign layout shell (sidebar, header) and dashboard-specific components. No structural/routing changes.

**Tech Stack:** Next.js 16, Tailwind CSS, shadcn/ui primitives, Recharts, next-themes, next-intl, lucide-react

**Spec:** `docs/superpowers/specs/2026-03-23-dashboard-redesign-design.md`

---

## Task 1: Theme Foundation — globals.css

**Files:**
- Modify: `breeding-dashboard/src/app/globals.css`

This is the foundation. Every subsequent task depends on these CSS variables being correct.

- [ ] **Step 1: Replace the `:root` light theme variables**

Replace the entire `:root` block with hex-based colors and glass tokens:

```css
:root {
  --radius: 0.625rem;
  --background: #f0eeeb;
  --background-gradient: linear-gradient(135deg, #f0eeeb 0%, #e8e6f0 40%, #f2eff5 70%, #eef0eb 100%);
  --foreground: #1a1a1a;
  --card: rgba(255,255,255,0.75);
  --card-foreground: #1a1a1a;
  --popover: rgba(255,255,255,0.85);
  --popover-foreground: #1a1a1a;
  --primary: #3d8c5c;
  --primary-foreground: #ffffff;
  --secondary: rgba(0,0,0,0.04);
  --secondary-foreground: #1a1a1a;
  --muted: rgba(0,0,0,0.03);
  --muted-foreground: #aaaaaa;
  --accent: rgba(61,140,92,0.1);
  --accent-foreground: #2d6b44;
  --destructive: #c4432b;
  --border: rgba(255,255,255,0.6);
  --input: rgba(0,0,0,0.06);
  --ring: #3d8c5c;
  --chart-1: #3d8c5c;
  --chart-2: #b8860b;
  --chart-3: #c4432b;
  --chart-4: #7c3aed;
  --chart-5: #1a1a1a;
  --sidebar: rgba(255,255,255,0.75);
  --sidebar-foreground: #1a1a1a;
  --sidebar-primary: #3d8c5c;
  --sidebar-primary-foreground: #ffffff;
  --sidebar-accent: rgba(61,140,92,0.1);
  --sidebar-accent-foreground: #2d6b44;
  --sidebar-border: rgba(0,0,0,0.06);
  --sidebar-ring: #3d8c5c;
  /* Glass tokens */
  --glass-bg: rgba(255,255,255,0.75);
  --glass-border: rgba(255,255,255,0.6);
  --glass-shadow: 0 4px 20px rgba(0,0,0,0.04);
  --glass-blur: 20px;
  /* Semantic accent colors */
  --accent-green: #3d8c5c;
  --accent-red: #c4432b;
  --accent-amber: #b8860b;
  --accent-purple: #7c3aed;
}
```

- [ ] **Step 2: Replace the `.dark` theme variables**

```css
.dark {
  --background: #0f1114;
  --background-gradient: linear-gradient(135deg, #0f1114 0%, #141425 35%, #1a1228 60%, #111518 100%);
  --foreground: #e8e8e8;
  --card: rgba(255,255,255,0.04);
  --card-foreground: #e8e8e8;
  --popover: rgba(255,255,255,0.08);
  --popover-foreground: #e8e8e8;
  --primary: #5cb87a;
  --primary-foreground: #ffffff;
  --secondary: rgba(255,255,255,0.05);
  --secondary-foreground: #e8e8e8;
  --muted: rgba(255,255,255,0.04);
  --muted-foreground: #555555;
  --accent: rgba(92,184,122,0.12);
  --accent-foreground: #5cb87a;
  --destructive: #ef6b5b;
  --border: rgba(255,255,255,0.06);
  --input: rgba(255,255,255,0.06);
  --ring: #5cb87a;
  --chart-1: #5cb87a;
  --chart-2: #d4a017;
  --chart-3: #ef6b5b;
  --chart-4: #a78bfa;
  --chart-5: #e8e8e8;
  --sidebar: rgba(255,255,255,0.04);
  --sidebar-foreground: #e8e8e8;
  --sidebar-primary: #5cb87a;
  --sidebar-primary-foreground: #ffffff;
  --sidebar-accent: rgba(92,184,122,0.12);
  --sidebar-accent-foreground: #5cb87a;
  --sidebar-border: rgba(255,255,255,0.06);
  --sidebar-ring: #5cb87a;
  /* Glass tokens */
  --glass-bg: rgba(255,255,255,0.04);
  --glass-border: rgba(255,255,255,0.06);
  --glass-shadow: 0 4px 20px rgba(0,0,0,0.15);
  --glass-blur: 20px;
  /* Semantic accent colors */
  --accent-green: #5cb87a;
  --accent-red: #ef6b5b;
  --accent-amber: #d4a017;
  --accent-purple: #a78bfa;
}
```

- [ ] **Step 3: Remove the `.emerald` theme block entirely**

Delete lines 119-152 (the entire `.emerald { ... }` block).

- [ ] **Step 4: Verify the `@theme inline` block and `@layer base` remain unchanged**

The `@theme inline` block (lines 7-48) maps CSS vars to Tailwind — keep as-is. The `@layer base` block (lines 154-161) — keep as-is.

- [ ] **Step 5: Run the dev server and verify no build errors**

```bash
cd breeding-dashboard && npm run dev
```

Open browser — colors should already shift since variables changed. Some things will look broken until we update components. That's expected.

- [ ] **Step 6: Commit**

```bash
git add breeding-dashboard/src/app/globals.css
git commit -m "feat(ui): replace oklch theme with hex/rgba glassmorphic tokens"
```

---

## Task 2: Providers & Theme Toggle

**Files:**
- Modify: `breeding-dashboard/src/components/providers.tsx`
- Modify: `breeding-dashboard/src/components/layout/theme-toggle.tsx`

- [ ] **Step 1: Update providers.tsx**

Change ThemeProvider config — remove emerald theme, disable system detection, set default to light:

```tsx
<ThemeProvider
  attribute="class"
  defaultTheme="light"
  themes={["light", "dark"]}
>
```

Remove `enableSystem` prop entirely.

- [ ] **Step 2: Update Toaster styling in providers.tsx**

Update the Toaster component with glass-like styling:

```tsx
<Toaster
  richColors
  position="top-right"
  toastOptions={{
    className: "!rounded-[14px] !border-[var(--glass-border)] !shadow-[var(--glass-shadow)]",
  }}
/>
```

- [ ] **Step 3: Rewrite theme-toggle.tsx as a pill toggle**

Replace the dropdown with a single button that toggles between light and dark:

```tsx
"use client";

import { Moon, Sun } from "lucide-react";
import { useTheme } from "next-themes";
import { Button } from "@/components/ui/button";

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();

  return (
    <Button
      variant="ghost"
      size="icon"
      onClick={() => setTheme(theme === "dark" ? "light" : "dark")}
      className="h-8 w-8 rounded-[10px] bg-[var(--secondary)]"
    >
      <Sun className="h-4 w-4 rotate-0 scale-100 transition-transform dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-4 w-4 rotate-90 scale-0 transition-transform dark:rotate-0 dark:scale-100" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add breeding-dashboard/src/components/providers.tsx breeding-dashboard/src/components/layout/theme-toggle.tsx
git commit -m "feat(ui): simplify to light/dark toggle, remove emerald theme"
```

---

## Task 3: UI Primitives — Card, Button, Badge, Input, Table, Skeleton

**Files:**
- Modify: `breeding-dashboard/src/components/ui/card.tsx`
- Modify: `breeding-dashboard/src/components/ui/button.tsx`
- Modify: `breeding-dashboard/src/components/ui/badge.tsx`
- Modify: `breeding-dashboard/src/components/ui/input.tsx`
- Modify: `breeding-dashboard/src/components/ui/select.tsx`
- Modify: `breeding-dashboard/src/components/ui/textarea.tsx`
- Modify: `breeding-dashboard/src/components/ui/table.tsx`
- Modify: `breeding-dashboard/src/components/ui/skeleton.tsx`
- Modify: `breeding-dashboard/src/components/ui/tooltip.tsx`
- Modify: `breeding-dashboard/src/components/ui/popover.tsx`
- Modify: `breeding-dashboard/src/components/ui/dialog.tsx`
- Modify: `breeding-dashboard/src/components/ui/sheet.tsx`

- [ ] **Step 1: Update Card component**

In `card.tsx`, change the Card's className from:
```
"bg-card text-card-foreground flex flex-col gap-6 rounded-xl border py-6 shadow-sm"
```
To:
```
"bg-[var(--glass-bg)] text-card-foreground flex flex-col gap-6 rounded-[18px] border border-[var(--glass-border)] py-6 shadow-[var(--glass-shadow)] backdrop-blur-[var(--glass-blur)]"
```

- [ ] **Step 2: Update Button variants**

In `button.tsx`, update the `buttonVariants` cva base class — change `rounded-md` to `rounded-[10px]`. Then update variant styles:
- `default`: `"bg-gradient-to-br from-[var(--accent-green)] to-[#2d6b44] text-white shadow-[0_2px_8px_rgba(61,140,92,0.3)] hover:shadow-[0_4px_12px_rgba(61,140,92,0.4)] hover:brightness-110"`
- `secondary`: `"bg-[var(--secondary)] text-[var(--secondary-foreground)] border border-[var(--glass-border)] hover:bg-[var(--secondary)]/80"`
- `destructive`: `"bg-[var(--destructive)] text-white hover:bg-[var(--destructive)]/90"`
- `outline`: `"border border-[var(--glass-border)] bg-[var(--glass-bg)] hover:bg-[var(--secondary)]"`
- `ghost` and `link`: keep as-is

- [ ] **Step 3: Update Badge variants**

In `badge.tsx`, change base class `rounded-full` to `rounded-[6px]`. Keep the variant structure but all existing code that uses Badge with specific color classes from `STATUS_COLORS` will continue working.

- [ ] **Step 4: Update Input**

In `input.tsx`, change `rounded-md` to `rounded-xl` (12px). Update border/background:
```
"border-[var(--glass-border)] bg-[var(--input)] focus:ring-[var(--accent-green)] focus:ring-2 focus:border-transparent placeholder:text-[var(--muted-foreground)]"
```

- [ ] **Step 5: Update Select, Textarea similarly**

Apply the same `rounded-xl` and glass border/bg treatment to `select.tsx` and `textarea.tsx`.

- [ ] **Step 6: Update Table**

In `table.tsx`:
- `TableHeader`: add `text-[10px] uppercase tracking-[1.5px] text-[var(--muted-foreground)]`
- `TableRow`: change border to `border-b border-[rgba(0,0,0,0.03)] dark:border-[rgba(255,255,255,0.03)]`
- `TableRow` hover: `hover:bg-[var(--muted)]`

- [ ] **Step 7: Update Skeleton**

In `skeleton.tsx`, change `rounded-md` to `rounded-[18px]` and ensure background uses `var(--muted)`.

- [ ] **Step 8: Update Tooltip**

In `tooltip.tsx`, update TooltipContent className to include `rounded-[10px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] text-[var(--foreground)]`.

- [ ] **Step 9: Update Popover**

In `popover.tsx`, update PopoverContent className to include `rounded-[14px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)]`.

- [ ] **Step 10: Update Dialog**

In `dialog.tsx`, update DialogContent className — change `rounded-lg` to `rounded-[20px]`, add `bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)]`. Update DialogOverlay to `bg-black/40 backdrop-blur-sm`.

- [ ] **Step 11: Update Sheet**

In `sheet.tsx`, update SheetContent className — add `bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border-[var(--glass-border)]`.

- [ ] **Step 12: Run build to verify no TypeScript errors**

```bash
cd breeding-dashboard && npm run build
```

- [ ] **Step 13: Commit**

```bash
git add breeding-dashboard/src/components/ui/
git commit -m "feat(ui): update all UI primitives with glassmorphic styling"
```

---

## Task 4: Shared Components — StatusBadge, PageHeader, LoadingSkeleton, EmptyState

**Files:**
- Modify: `breeding-dashboard/src/components/shared/status-badge.tsx`
- Modify: `breeding-dashboard/src/components/shared/page-header.tsx`
- Modify: `breeding-dashboard/src/components/shared/loading-skeleton.tsx`
- Modify: `breeding-dashboard/src/components/shared/empty-state.tsx`
- Modify: `breeding-dashboard/src/lib/constants.ts` (STATUS_COLORS)

- [ ] **Step 1: Update STATUS_COLORS to tinted fill style**

In `constants.ts`, replace the `STATUS_COLORS` map. The new pattern is: `bg-{color}/8 text-{color}` (low-opacity fill, full color text, no border). Update all entries:

```ts
export const STATUS_COLORS: Record<string, string> = {
  // Positive
  ACTIVE: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  APPROVED: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  VERIFIED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  COMPLETE: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  RECEIVED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  CONSUMED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  RETURNED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  PAID: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  FULLY_RECEIVED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  CREDIT_LIMIT_APPROVED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  SUPPLIER_APPROVED: "bg-[var(--accent-green)]/10 text-[var(--accent-green)]",
  // In-progress
  PROCESSING: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  IN_TRANSIT: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  IN_DELIVERY: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  PREPARING: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  ORDERED: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  PENDING: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  PENDING_APPROVAL: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  PENDING_VERIFICATION: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  REALIZING: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  REALIZING_DO_LIMIT: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  REALIZATION_APPROVAL: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  CREDIT_LIMIT_PROCESSING: "bg-[var(--accent-purple)]/10 text-[var(--accent-purple)]",
  UNPAID: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  // Partial
  PARTIAL: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  PARTIAL_RECEIVED: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  OVERPAID: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  // Warning
  DAMAGED_IN_TRANSIT: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  LOW: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  OVER_STOCK: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  ON_LEAVE: "bg-[var(--accent-amber)]/10 text-[var(--accent-amber)]",
  // Inactive
  INACTIVE: "bg-[var(--muted-foreground)]/10 text-[var(--muted-foreground)]",
  MAINTENANCE: "bg-[var(--muted-foreground)]/10 text-[var(--muted-foreground)]",
  NORMAL: "bg-[var(--muted-foreground)]/10 text-[var(--muted-foreground)]",
  TERMINATED: "bg-[var(--muted-foreground)]/10 text-[var(--muted-foreground)]",
  // Negative
  REJECTED: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
  CANCELLED: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
  SUPPLIER_REJECTED: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
  CREDIT_LIMIT_REJECTED: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
  OUT_OF_STOCK: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
  CRITICAL: "bg-[var(--accent-red)]/10 text-[var(--accent-red)]",
};
```

- [ ] **Step 2: Update status-badge.tsx**

Update the Badge usage to remove border and add uppercase micro styling:

```tsx
<Badge
  variant="ghost"
  className={cn(
    "rounded-[6px] px-2 py-0.5 text-[10px] font-medium uppercase tracking-[0.3px] border-0",
    STATUS_COLORS[status] ?? "bg-[var(--muted)] text-[var(--muted-foreground)]"
  )}
>
```

- [ ] **Step 3: Update page-header.tsx**

Add optional `sectionLabel` prop. Keep the existing `actions` prop for backward compatibility — all existing pages pass `actions={...}`. Update styling:

```tsx
interface PageHeaderProps {
  title: string;
  description?: string;
  sectionLabel?: string;
  actions?: React.ReactNode;
}

export function PageHeader({ title, sectionLabel, actions }: PageHeaderProps) {
  return (
    <div className="flex items-start justify-between">
      <div>
        {sectionLabel && (
          <p className="text-[10px] uppercase tracking-[2px] text-[var(--muted-foreground)] mb-1">
            {sectionLabel}
          </p>
        )}
        <h1 className="text-[24px] font-light tracking-[-0.5px] text-[var(--foreground)]">
          {title}
        </h1>
      </div>
      {actions && (
        <div className="flex items-center gap-2">{actions}</div>
      )}
    </div>
  );
}
```

Note: Keep accepting `description` prop in the interface for backward compat but don't render it. The `actions` prop name is unchanged so no call sites need updating.

- [ ] **Step 4: Update loading-skeleton.tsx**

Update CardSkeleton to use glass-style rounded corners:

```tsx
export function CardSkeleton() {
  return (
    <div className="rounded-[18px] bg-[var(--glass-bg)] border border-[var(--glass-border)] p-6 space-y-4">
      <Skeleton className="h-4 w-1/3 rounded-[8px]" />
      <Skeleton className="h-8 w-1/2 rounded-[8px]" />
      <Skeleton className="h-4 w-1/4 rounded-[8px]" />
    </div>
  );
}
```

Update TableSkeleton similarly with glass card wrapper.

- [ ] **Step 5: Update empty-state.tsx**

Wrap in glass card, add colored icon background circle:

```tsx
export function EmptyState({ title, description, action }: EmptyStateProps) {
  return (
    <div className="flex flex-col items-center justify-center rounded-[18px] bg-[var(--glass-bg)] border border-[var(--glass-border)] py-16 px-6 text-center backdrop-blur-[var(--glass-blur)]">
      <div className="mb-4 flex h-12 w-12 items-center justify-center rounded-full bg-[var(--muted)]">
        <Package className="h-6 w-6 text-[var(--muted-foreground)]" />
      </div>
      <h3 className="text-[15px] font-medium text-[var(--foreground)]">{title || "No data found"}</h3>
      {description && <p className="mt-1 text-[13px] text-[var(--muted-foreground)]">{description}</p>}
      {action && <div className="mt-4">{action}</div>}
    </div>
  );
}
```

- [ ] **Step 6: Commit**

```bash
git add breeding-dashboard/src/components/shared/ breeding-dashboard/src/lib/constants.ts
git commit -m "feat(ui): update shared components with glassmorphic styling"
```

---

## Task 5: Data Table Redesign

**Files:**
- Modify: `breeding-dashboard/src/components/shared/data-table.tsx`

- [ ] **Step 1: Wrap table in glass card, update search and pagination**

Rewrite the DataTable component. Key changes:
- Wrap the entire table in a glass card (`rounded-[18px]`, glass bg, blur, border)
- Search input gets glass pill styling
- Table border wrapper (`rounded-md border`) is removed — the glass card handles it
- Pagination uses rounded number pills

```tsx
return (
  <div className="space-y-4">
    {onSearchChange && (
      <div className="relative max-w-sm">
        <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-[var(--muted-foreground)]" />
        <Input
          placeholder={searchPlaceholder}
          value={search || ""}
          onChange={(e) => onSearchChange(e.target.value)}
          className="pl-9 rounded-xl bg-[var(--glass-bg)] border-[var(--glass-border)]"
        />
      </div>
    )}

    {isLoading ? (
      <TableSkeleton cols={columns.length} />
    ) : data.length === 0 ? (
      <EmptyState title={emptyTitle} description={emptyDescription} action={emptyAction} />
    ) : (
      <div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] overflow-hidden">
        <Table>
          <TableHeader>
            <TableRow className="border-b border-[rgba(0,0,0,0.05)] dark:border-[rgba(255,255,255,0.05)] hover:bg-transparent">
              {columns.map((col) => (
                <TableHead
                  key={col.header}
                  className={cn("text-[10px] uppercase tracking-[1.2px] text-[var(--muted-foreground)] font-medium py-3 px-5", col.className)}
                >
                  {col.header}
                </TableHead>
              ))}
            </TableRow>
          </TableHeader>
          <TableBody>
            {data.map((row, idx) => (
              <TableRow
                key={idx}
                className={cn(
                  "border-b border-[rgba(0,0,0,0.03)] dark:border-[rgba(255,255,255,0.03)] hover:bg-[var(--muted)]",
                  onRowClick && "cursor-pointer"
                )}
                onClick={() => onRowClick?.(row)}
              >
                {columns.map((col) => (
                  <TableCell key={col.header} className={cn("py-3.5 px-5 text-[13px]", col.className)}>
                    {col.cell ? col.cell(row) : col.accessorKey ? String(row[col.accessorKey] ?? "") : ""}
                  </TableCell>
                ))}
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </div>
    )}

    {totalPages > 1 && onPageChange && (
      <div className="flex items-center justify-between px-1">
        <p className="text-[11px] text-[var(--muted-foreground)]">
          {total !== undefined ? `${total} total records` : `Page ${page} of ${totalPages}`}
        </p>
        <div className="flex items-center gap-1">
          <Button
            variant="ghost"
            size="icon-sm"
            onClick={() => onPageChange(page - 1)}
            disabled={page <= 1}
            className="rounded-[8px]"
          >
            <ChevronLeft className="h-4 w-4" />
          </Button>
          {Array.from({ length: Math.min(totalPages, 5) }, (_, i) => {
            const pageNum = i + 1;
            return (
              <button
                key={pageNum}
                onClick={() => onPageChange(pageNum)}
                className={cn(
                  "h-8 w-8 rounded-[8px] text-[11px] font-medium transition-colors",
                  pageNum === page
                    ? "bg-[var(--foreground)] text-[var(--background)]"
                    : "text-[var(--muted-foreground)] hover:bg-[var(--muted)]"
                )}
              >
                {pageNum}
              </button>
            );
          })}
          <Button
            variant="ghost"
            size="icon-sm"
            onClick={() => onPageChange(page + 1)}
            disabled={page >= totalPages}
            className="rounded-[8px]"
          >
            <ChevronRight className="h-4 w-4" />
          </Button>
        </div>
      </div>
    )}
  </div>
);
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/components/shared/data-table.tsx
git commit -m "feat(ui): redesign DataTable with glass card and rounded pagination"
```

---

## Task 6: Layout Shell — Dashboard Layout & Sidebar

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/layout.tsx`
- Modify: `breeding-dashboard/src/components/layout/app-sidebar.tsx`
- Modify: `breeding-dashboard/src/components/layout/collapsible-sidebar-group.tsx`
- Modify: `breeding-dashboard/src/components/ui/sidebar.tsx` (minimal style overrides)

- [ ] **Step 1: Update layout.tsx**

Add gradient background wrapper. Restyle the shell:

```tsx
return (
  <SidebarProvider>
    <div className="flex min-h-screen w-full" style={{ background: 'var(--background-gradient)' }}>
      <AppSidebar />
      <SidebarInset className="flex-1 border-0 bg-transparent shadow-none">
        <Header />
        <main className="flex-1 overflow-auto p-2 md:p-4 lg:p-6">{children}</main>
      </SidebarInset>
    </div>
    <ChatWidget />
  </SidebarProvider>
);
```

- [ ] **Step 2: Redesign app-sidebar.tsx**

Complete rewrite of the sidebar component. Key elements:
- Glass background with `backdrop-filter: blur(20px)`
- Rounded shape with `border-radius: 20px` and `margin: 14px`
- Logo with green gradient
- Search placeholder with ⌘K badge
- Updated nav items with rounded active states
- User card at bottom

The sidebar uses shadcn's `<Sidebar>` primitive but we override its styling via className. Here is the full component structure:

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { Search } from "lucide-react";
import {
  Sidebar, SidebarContent, SidebarGroup, SidebarHeader,
  SidebarMenu, SidebarMenuButton, SidebarMenuItem,
  SidebarMenuSubButton, SidebarMenuSubItem,
} from "@/components/ui/sidebar";
import { NAV_ITEMS } from "@/lib/constants";
import type { NavItem } from "@/lib/constants";
import { CollapsibleSidebarGroup } from "./collapsible-sidebar-group";
import { useTranslations } from "next-intl";
import { useAuth } from "@/hooks/use-auth";

export function AppSidebar() {
  const pathname = usePathname();
  const t = useTranslations("navigation");
  const { user } = useAuth();
  const initials = user?.name?.split(" ").map((n) => n[0]).join("").toUpperCase().slice(0, 2) || "U";

  return (
    <Sidebar
      variant="sidebar"
      collapsible="icon"
      className="!bg-[var(--glass-bg)] !backdrop-blur-[20px] !border !border-[var(--glass-border)] !rounded-[20px] !m-[14px] !shadow-[var(--glass-shadow)] !h-[calc(100vh-28px)]"
    >
      <SidebarHeader className="border-0 px-4 py-4">
        {/* Logo */}
        <Link href="/" className="flex items-center gap-2.5">
          <div className="flex h-[34px] w-[34px] shrink-0 items-center justify-center rounded-[10px] bg-gradient-to-br from-[#3d8c5c] to-[#2d6b44] text-white text-[13px] font-semibold shadow-[0_2px_8px_rgba(61,140,92,0.3)]">
            B
          </div>
          <span className="truncate text-[15px] font-medium group-data-[collapsible=icon]:hidden">
            Breeding
          </span>
        </Link>
        {/* Search placeholder */}
        <div className="mt-3 flex items-center gap-2 rounded-xl bg-[var(--muted)] px-3 py-2.5 text-[12px] text-[var(--muted-foreground)] group-data-[collapsible=icon]:hidden">
          <Search className="h-3.5 w-3.5 opacity-40" />
          <span>Search...</span>
          <kbd className="ml-auto text-[9px] bg-[var(--secondary)] px-1.5 py-0.5 rounded">⌘K</kbd>
        </div>
      </SidebarHeader>
      <SidebarContent>
        <SidebarGroup>
          <SidebarMenu>
            {/* Nav items — same iteration logic as current, but with updated styling */}
            {NAV_ITEMS.map((section) => {
              if ("url" in section) {
                const Icon = section.icon;
                return (
                  <SidebarMenuItem key={section.title}>
                    <SidebarMenuButton
                      asChild
                      isActive={pathname === section.url}
                      tooltip={t(section.title)}
                      className="rounded-xl data-[active=true]:bg-[var(--accent)] data-[active=true]:text-[var(--accent-foreground)]"
                    >
                      <Link href={section.url}><Icon /><span>{t(section.title)}</span></Link>
                    </SidebarMenuButton>
                  </SidebarMenuItem>
                );
              }
              const isSectionActive = section.items.some(
                (item) => pathname === item.url || pathname.startsWith(item.url + "/")
              );
              return (
                <CollapsibleSidebarGroup key={section.title} title={t(section.title)} isActive={isSectionActive}>
                  {section.items.map((item) => {
                    const ItemIcon = item.icon;
                    return (
                      <SidebarMenuSubItem key={item.title}>
                        <SidebarMenuSubButton
                          asChild
                          isActive={pathname === item.url || pathname.startsWith(item.url + "/")}
                          className="rounded-xl data-[active=true]:bg-[var(--accent)] data-[active=true]:text-[var(--accent-foreground)]"
                        >
                          <Link href={item.url}>{ItemIcon && <ItemIcon />}<span>{t(item.title)}</span></Link>
                        </SidebarMenuSubButton>
                      </SidebarMenuSubItem>
                    );
                  })}
                </CollapsibleSidebarGroup>
              );
            })}
          </SidebarMenu>
        </SidebarGroup>
        {/* User card at bottom */}
        <div className="mt-auto p-3 group-data-[collapsible=icon]:p-2">
          <div className="flex items-center gap-2.5 rounded-xl bg-[var(--muted)] p-2.5 group-data-[collapsible=icon]:justify-center">
            <div className="flex h-8 w-8 shrink-0 items-center justify-center rounded-[9px] bg-[var(--foreground)] text-[10px] font-medium text-[var(--background)]">
              {initials}
            </div>
            <div className="group-data-[collapsible=icon]:hidden">
              <p className="text-[12px] font-medium">{user?.name}</p>
              <p className="text-[10px] text-[var(--muted-foreground)]">{user?.role?.replace("_", " ")}</p>
            </div>
          </div>
        </div>
      </SidebarContent>
    </Sidebar>
  );
}
```

- [ ] **Step 3: Update collapsible-sidebar-group.tsx**

Update section labels to uppercase micro style:
```
className="text-[9px] uppercase tracking-[1.5px] text-[var(--muted-foreground)] px-3"
```

Update sub-items to use `rounded-xl` and green active fill.

- [ ] **Step 4: Override sidebar.tsx base styles**

In `sidebar.tsx`, make these specific changes:

1. Find the CSS variables at the top of the file (or in the SidebarProvider). Set:
   - `--sidebar-width: 240px`
   - `--sidebar-width-icon: 72px`

2. Find the root `<aside>` element in the Sidebar component. Override its default border and background by adding these to its className: `!border-0 !bg-transparent`. The glass styling is applied via the className prop from `app-sidebar.tsx`.

3. The `SidebarInset` component typically has `border` and `bg-background` — we override those in `layout.tsx` via className (`border-0 bg-transparent shadow-none`).

- [ ] **Step 5: Run dev server and visually verify sidebar**

```bash
cd breeding-dashboard && npm run dev
```

Check: sidebar floats with rounded corners, gradient shows through, nav items highlight correctly, collapse works.

- [ ] **Step 6: Commit**

```bash
git add breeding-dashboard/src/app/\(dashboard\)/layout.tsx breeding-dashboard/src/components/layout/ breeding-dashboard/src/components/ui/sidebar.tsx
git commit -m "feat(ui): redesign layout shell with floating glass sidebar"
```

---

## Task 7: Header & Breadcrumbs Redesign

**Files:**
- Modify: `breeding-dashboard/src/components/layout/header.tsx`
- Modify: `breeding-dashboard/src/components/layout/breadcrumbs.tsx`
- Modify: `breeding-dashboard/src/components/layout/language-toggle.tsx`

- [ ] **Step 1: Redesign header.tsx**

Remove the bar layout. Make it a transparent flex row that sits inside the content area:

```tsx
export function Header() {
  const { user, logout } = useAuth();
  const t = useTranslations('common');
  const initials = user?.name?.split(" ").map((n) => n[0]).join("").toUpperCase().slice(0, 2) || "U";

  return (
    <header className="flex items-center gap-4 px-6 pt-4 pb-2">
      <div className="ml-auto flex items-center gap-2">
        <LanguageToggle />
        <ThemeToggle />
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="ghost" className="relative h-8 w-8 rounded-[10px] bg-[var(--secondary)]">
              <span className="text-[10px] font-medium">{initials}</span>
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end" className="rounded-[14px]">
            <DropdownMenuLabel>
              <div className="flex flex-col">
                <span className="text-[13px] font-medium">{user?.name}</span>
                <span className="text-[11px] text-[var(--muted-foreground)]">{user?.email}</span>
              </div>
            </DropdownMenuLabel>
            <DropdownMenuSeparator />
            <DropdownMenuItem onClick={logout} className="rounded-lg">
              <LogOut className="mr-2 h-4 w-4" />
              {t('logout')}
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      </div>
    </header>
  );
}
```

Remove SidebarTrigger (collapse is in sidebar), remove Breadcrumbs, remove Separator.

- [ ] **Step 2: Simplify breadcrumbs.tsx**

Keep the component but simplify to only show on nested pages (detail/edit). For top-level list pages, return null. For nested pages, show a minimal path:

```tsx
export function DashboardBreadcrumbs() {
  const pathname = usePathname();
  const segments = pathname.split("/").filter(Boolean);

  // Only show on nested pages (more than 1 segment)
  if (segments.length <= 1) return null;

  const parentSegment = segments[0];
  const parentLabel = ROUTE_LABELS[parentSegment] || parentSegment;

  return (
    <nav className="flex items-center gap-1.5 text-[11px] text-[var(--muted-foreground)] mb-1">
      <Link href={`/${parentSegment}`} className="hover:text-[var(--foreground)] transition-colors">
        {parentLabel}
      </Link>
      <span>/</span>
      <span className="text-[var(--foreground)]">
        {ROUTE_LABELS[segments[segments.length - 1]] || segments[segments.length - 1]}
      </span>
    </nav>
  );
}
```

- [ ] **Step 3: Update language-toggle.tsx**

Style as a glass pill button — same approach as theme toggle:
```
className="h-8 rounded-[10px] bg-[var(--secondary)] px-2.5 text-[11px]"
```

- [ ] **Step 4: Commit**

```bash
git add breeding-dashboard/src/components/layout/header.tsx breeding-dashboard/src/components/layout/breadcrumbs.tsx breeding-dashboard/src/components/layout/language-toggle.tsx
git commit -m "feat(ui): redesign header as transparent bar, simplify breadcrumbs"
```

---

## Task 8: i18n Keys

**Files:**
- Modify: `breeding-dashboard/messages/en.json`
- Modify: `breeding-dashboard/messages/id.json`

- [ ] **Step 1: Add new keys to en.json**

Add under the `"dashboard"` section:
```json
"greeting": {
  "morning": "Good morning",
  "afternoon": "Good afternoon",
  "evening": "Good evening"
},
"timeRange": {
  "1d": "1D",
  "7d": "7D",
  "1m": "1M",
  "all": "All"
},
"viewAll": "View all"
```

- [ ] **Step 2: Add corresponding keys to id.json**

```json
"greeting": {
  "morning": "Selamat pagi",
  "afternoon": "Selamat siang",
  "evening": "Selamat malam"
},
"timeRange": {
  "1d": "1H",
  "7d": "7H",
  "1m": "1B",
  "all": "Semua"
},
"viewAll": "Lihat semua"
```

- [ ] **Step 3: Commit**

```bash
git add breeding-dashboard/messages/
git commit -m "feat(i18n): add greeting and time range keys for dashboard redesign"
```

---

## Task 9: Dashboard Page & KPI Banner

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/page.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/components/dashboard-kpi-banner.tsx`
- Create: `breeding-dashboard/src/app/(dashboard)/components/mini-sparkline.tsx`

- [ ] **Step 1: Create mini-sparkline.tsx utility component**

A small pure component that renders a static SVG sparkline from an array of numbers:

```tsx
interface MiniSparklineProps {
  data: number[];
  width?: number;
  height?: number;
  color: string;
}

export function MiniSparkline({ data, width = 40, height = 16, color }: MiniSparklineProps) {
  if (data.length < 2) return null;
  const min = Math.min(...data);
  const max = Math.max(...data);
  const range = max - min || 1;
  const points = data.map((v, i) => {
    const x = (i / (data.length - 1)) * width;
    const y = height - ((v - min) / range) * (height - 2) - 1;
    return `${x},${y}`;
  });
  return (
    <svg width={width} height={height} viewBox={`0 0 ${width} ${height}`}>
      <polyline points={points.join(" ")} fill="none" stroke={color} strokeWidth="1.5" opacity="0.5" />
    </svg>
  );
}
```

- [ ] **Step 2: Redesign dashboard-kpi-banner.tsx**

Replace the dark gradient banner with 4 glass KPI cards. Add greeting with time-of-day logic. Each card gets an icon badge, large value, delta, and mini sparkline.

The greeting uses browser local time:
```tsx
function getGreeting(t: (key: string) => string): string {
  const hour = new Date().getHours();
  if (hour >= 5 && hour < 12) return t("greeting.morning");
  if (hour >= 12 && hour < 17) return t("greeting.afternoon");
  return t("greeting.evening");
}
```

Each KPI card structure:
```tsx
<div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] p-[18px]">
  <div className="flex justify-between items-start">
    <span className="text-[9px] uppercase tracking-[1.5px] text-[var(--muted-foreground)]">{label}</span>
    <div className="flex h-7 w-7 items-center justify-center rounded-lg" style={{ background: iconBgGradient }}>
      <Icon className="h-3.5 w-3.5" style={{ color: iconColor }} />
    </div>
  </div>
  <p className="text-[28px] font-light tracking-[-0.5px] mt-2" style={{ color: valueColor }}>{value}</p>
  <div className="flex items-center gap-1 mt-1">
    <span className="text-[11px] font-medium" style={{ color: deltaColor }}>{delta}</span>
    <MiniSparkline data={sparkData} color={deltaColor} />
  </div>
</div>
```

The component now accepts both `stats` and `trends` props to extract sparkline data.

The greeting + KPI grid wrapper (rendered by the banner component):
```tsx
<div>
  {/* Greeting */}
  <div className="flex items-center justify-between mb-6 px-1">
    <div>
      <p className="text-[10px] uppercase tracking-[2px] text-[var(--muted-foreground)] mb-1">Overview</p>
      <h1 className="text-[24px] font-light tracking-[-0.5px]">{greeting}, {firstName}</h1>
    </div>
    {/* Date + controls could go here */}
  </div>
  {/* Responsive KPI grid: 4 cols -> 2 cols -> 1 col */}
  <div className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-4">
    {kpis.map(kpi => (
      <KpiCard key={kpi.label} {...kpi} />
    ))}
  </div>
</div>
```

Sparkline data mapping: use the last 7 entries from `trends.fcrTrend` for FCR, `trends.mortalityTrend` for mortality, `trends.salesTrend` for sales. For bird population and active projects (no trend data), omit the sparkline.

- [ ] **Step 3: Update page.tsx layout**

Update the dashboard page to pass trends data to the KPI banner and use the new bento grid layout:

```tsx
<div className="space-y-4">
  {/* PageHeader is now handled by the KPI banner greeting */}
  {stats && trends && <DashboardKpiBanner stats={stats} trends={trends} user={user} />}

  <div className="grid grid-cols-1 gap-4 md:grid-cols-[1.6fr_1fr]">
    <MortalityFcrChart trends={trends} />
    <StockAlertsList />
  </div>

  <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
    <SalesTrendChart trends={trends} />
    {stats && <OperationalCards stats={stats} />}
  </div>

  {stats && <ProjectPhaseChart stats={stats} />}
</div>
```

Note: Remove `<PageHeader>` from the dashboard — the greeting replaces it. Import `useAuth` to get user name.

- [ ] **Step 4: Commit**

```bash
git add breeding-dashboard/src/app/\(dashboard\)/
git commit -m "feat(dashboard): redesign KPI banner with glass cards, sparklines, and greeting"
```

---

## Task 10: Dashboard Charts Redesign

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/components/mortality-fcr-chart.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/components/project-phase-chart.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/components/sales-trend-chart.tsx`

- [ ] **Step 1: Redesign mortality-fcr-chart.tsx**

Replace `<Card>` wrapper with a glass div. Add time-range filter pills (visual only for now — the API already returns 30 days). Add inline legend. Update chart colors to use new palette.

Glass card wrapper:
```tsx
<div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] p-5">
```

Time-range pills in header:
```tsx
<div className="flex gap-1.5">
  {["1d", "7d", "1m", "all"].map((range) => (
    <button
      key={range}
      className={cn(
        "px-2.5 py-1 rounded-md text-[10px] font-medium transition-colors",
        range === "1m"
          ? "bg-[var(--foreground)] text-[var(--background)]"
          : "bg-[var(--muted)] text-[var(--muted-foreground)] hover:bg-[var(--secondary)]"
      )}
    >
      {t(`timeRange.${range}`)}
    </button>
  ))}
</div>
```

Update Recharts colors: FCR line uses `var(--foreground)`, mortality uses `var(--accent-red)`, standard dashed uses `var(--muted-foreground)`.

- [ ] **Step 2: Redesign sales-trend-chart.tsx**

Simplify to a compact card with large value + area sparkline. Replace ComposedChart with a simple AreaChart:

```tsx
<div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] p-[18px]">
  <p className="text-[9px] uppercase tracking-[1.5px] text-[var(--muted-foreground)] mb-2">
    {t("salesTrend")}
  </p>
  <p className="text-[22px] font-light text-[var(--foreground)]">
    Rp {totalRevenue}
  </p>
  {/* Area chart with gradient fill */}
  <ChartContainer config={chartConfig} className="h-[60px] w-full mt-2">
    <AreaChart data={chartData}>
      <defs>
        <linearGradient id="salesFill" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stopColor="var(--accent-green)" stopOpacity={0.12} />
          <stop offset="100%" stopColor="var(--accent-green)" stopOpacity={0} />
        </linearGradient>
      </defs>
      <Area type="monotone" dataKey="revenue" stroke="var(--accent-green)" strokeWidth={1.5} fill="url(#salesFill)" />
    </AreaChart>
  </ChartContainer>
  <p className="text-[11px] text-[var(--accent-green)] mt-1">{deltaText}</p>
</div>
```

- [ ] **Step 3: Redesign project-phase-chart.tsx**

Replace Card wrapper with glass div. Update donut chart colors to match new palette. Style legend as rounded pills:

```tsx
<div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] p-5">
```

- [ ] **Step 4: Commit**

```bash
git add breeding-dashboard/src/app/\(dashboard\)/components/mortality-fcr-chart.tsx breeding-dashboard/src/app/\(dashboard\)/components/sales-trend-chart.tsx breeding-dashboard/src/app/\(dashboard\)/components/project-phase-chart.tsx
git commit -m "feat(dashboard): redesign all chart components with glass cards"
```

---

## Task 11: Stock Alerts & Operational Cards

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/components/stock-alerts-list.tsx`
- Modify: `breeding-dashboard/src/app/(dashboard)/components/operational-cards.tsx`

- [ ] **Step 1: Redesign stock-alerts-list.tsx**

Replace Card wrapper with glass div. Update alert rows to use status dots and semantic tinting:

Each alert row:
```tsx
<div className={cn(
  "flex items-center gap-3 rounded-[14px] p-3",
  stock.stockStatus === "CRITICAL" ? "bg-[var(--accent-red)]/[0.04]" : "bg-[var(--accent-amber)]/[0.04]"
)}>
  <div className={cn(
    "h-2 w-2 shrink-0 rounded-full",
    stock.stockStatus === "CRITICAL" ? "bg-[var(--accent-red)] dark:shadow-[0_0_8px_rgba(239,107,91,0.35)]" : "bg-[var(--accent-amber)]"
  )} />
  <div className="flex-1 min-w-0">
    <p className="text-[13px] truncate">{stock.product?.name}</p>
    <p className="text-[10px] text-[var(--muted-foreground)]">{stock.warehouse?.name} · {qty}</p>
  </div>
  <span className={cn(
    "px-2 py-0.5 rounded-[6px] text-[9px] uppercase tracking-[0.3px] font-medium",
    stock.stockStatus === "CRITICAL"
      ? "bg-[var(--accent-red)] text-white"
      : "bg-[var(--accent-amber)] text-white"
  )}>
    {stock.stockStatus === "CRITICAL" ? t("critical") : t("low")}
  </span>
</div>
```

Add "View all" link in header.

- [ ] **Step 2: Redesign operational-cards.tsx**

Replace Card wrapper with glass divs. Each card now spans a single cell in a 3-col grid (the grid is defined in page.tsx). Add segmented progress bars:

```tsx
<div className="rounded-[18px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] p-[18px]">
  <p className="text-[9px] uppercase tracking-[1.5px] text-[var(--muted-foreground)] mb-2">{label}</p>
  <p className="text-[22px] font-light" style={{ color: valueColor }}>{value}</p>
  <p className="text-[11px] text-[var(--muted-foreground)] mt-1">{subtitle}</p>
  {/* Segmented progress bar */}
  <div className="flex gap-1 mt-3">
    {[0,1,2,3].map(i => (
      <div
        key={i}
        className="h-1 flex-1 rounded-full"
        style={{
          background: i < filledSegments ? `${accentColor}` : 'var(--muted)',
          opacity: i < filledSegments ? 1 - (i * 0.15) : 1,
        }}
      />
    ))}
  </div>
</div>
```

Note: The OperationalCards component should now return just the cards (not the grid wrapper) since page.tsx handles the grid. Or keep the grid inside if that's simpler.

- [ ] **Step 3: Commit**

```bash
git add breeding-dashboard/src/app/\(dashboard\)/components/stock-alerts-list.tsx breeding-dashboard/src/app/\(dashboard\)/components/operational-cards.tsx
git commit -m "feat(dashboard): redesign stock alerts and operational cards"
```

---

## Task 12: Chart Tooltip & ChatWidget Styling

**Files:**
- Modify: `breeding-dashboard/src/components/ui/chart.tsx`
- Modify: `breeding-dashboard/src/components/chat/chat-widget.tsx`

- [ ] **Step 1: Update chart.tsx tooltip styling**

Find the `ChartTooltipContent` component and update its container className to use glass treatment:

```
"rounded-[12px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)] border border-[var(--glass-border)] shadow-[var(--glass-shadow)] px-3 py-2"
```

- [ ] **Step 2: Update chat-widget.tsx**

Add glass treatment to the chat panel container and the floating trigger button. The button gets `rounded-[14px] bg-[var(--glass-bg)] backdrop-blur-[var(--glass-blur)]`. The chat panel gets the same glass card styling.

- [ ] **Step 3: Commit**

```bash
git add breeding-dashboard/src/components/ui/chart.tsx breeding-dashboard/src/components/chat/chat-widget.tsx
git commit -m "feat(ui): apply glass styling to chart tooltips and chat widget"
```

---

## Task 13: Build Verification & Final Cleanup

**Files:**
- All modified files

- [ ] **Step 1: Run full build**

```bash
cd breeding-dashboard && npm run build
```

Fix any TypeScript or build errors.

- [ ] **Step 2: Run lint**

```bash
cd breeding-dashboard && npm run lint
```

Fix any lint errors.

- [ ] **Step 3: Visual smoke test**

Start dev server, verify these pages look correct:
1. Dashboard home page (light + dark mode)
2. Any table page (e.g., `/products`, `/inventory`)
3. A detail/edit page (e.g., `/projects/1`)
4. Sidebar collapse/expand
5. Mobile responsive (narrow browser window)

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix(ui): address build errors and visual polish from redesign"
```
