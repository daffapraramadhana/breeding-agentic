# Dashboard & Layout Redesign — Glassmorphic Editorial Bento

**Date:** 2026-03-23
**Status:** Draft
**Scope:** Full shell (sidebar, header) + dashboard home page + all shared page components + dark mode

## Summary

Redesign the entire Breeding Dashboard UI from a traditional shadcn template look to a sleek, modern glassmorphic editorial design. Inspired by Tubik Studio's trading dashboard aesthetic, combined with editorial typography and a bento grid layout. The design applies globally — not just the dashboard home page.

## Design Direction

**Hybrid of three influences:**
- **Editorial (A):** Light font weights (300), uppercase micro-labels with wide letter-spacing, generous whitespace, warm muted tones
- **Bento Grid (C):** Cards in a structured grid layout, rounded corners (18px), soft borders
- **Tubik Reference:** Frosted glass cards (`backdrop-filter: blur(20px)`), soft gradient backgrounds, mini sparklines on KPI cards, floating card depth via soft shadows, color-coded icon badges

## Theme System

### Two Themes Only (remove Emerald)

**Light Mode:**
- Background gradient: `linear-gradient(135deg, #f0eeeb 0%, #e8e6f0 40%, #f2eff5 70%, #eef0eb 100%)`
- Card surface: `rgba(255,255,255,0.75)` with `backdrop-filter: blur(20px)`
- Card border: `1px solid rgba(255,255,255,0.6)`
- Card shadow: `0 4px 20px rgba(0,0,0,0.04)`
- Card border-radius: `18px`
- Text primary: `#1a1a1a`
- Text secondary: `#aaa`
- Accent green: `#3d8c5c` (positive values, active states, CTAs)
- Accent red: `#c4432b` (alerts, negative values)
- Accent amber: `#b8860b` (warning/low states)
- Accent purple: `#7c3aed` (neutral accent — project counts, etc.)

**Dark Mode:**
- Background gradient: `linear-gradient(135deg, #0f1114 0%, #141425 35%, #1a1228 60%, #111518 100%)`
- Card surface: `rgba(255,255,255,0.04)` with `backdrop-filter: blur(20px)`
- Card border: `1px solid rgba(255,255,255,0.06)`
- Card shadow: `0 4px 20px rgba(0,0,0,0.15)`
- Card border-radius: `18px`
- Text primary: `#e8e8e8`
- Text secondary: `#555`
- Accent green: `#5cb87a` (brighter for dark contrast)
- Accent red: `#ef6b5b` (brighter for dark contrast)
- Accent amber: `#d4a017` (brighter for dark contrast)
- Accent purple: `#a78bfa` (brighter for dark contrast)
- Alert dots get glow: `box-shadow: 0 0 8px rgba(color, 0.35)`

### Color System Approach
- **Replace oklch with hex/rgba.** The current oklch variables are removed. All colors defined in hex. Glass properties use rgba for transparency. This simplifies the theme and avoids mixing two color systems.
- The `--background` CSS variable becomes a solid fallback color (`#f0eeeb` light / `#0f1114` dark). The gradient is applied via a dedicated `--background-gradient` variable on the layout's root element.
- Tailwind's `bg-background` still works for components that need a solid background. The gradient is only on the page layout.

### CSS Variable Updates
- Update `globals.css` to define only `light` and `dark` themes (remove `.emerald` class)
- Define new CSS variables:
  - `--background` (solid fallback), `--background-gradient` (gradient value)
  - `--glass-bg`, `--glass-border`, `--glass-shadow` (as full values, not individual parts)
  - `--glass-blur: 20px` (sidebar and large cards only — see performance note)
  - `--accent-green`, `--accent-red`, `--accent-amber`, `--accent-purple`
- Update ThemeProvider in `providers.tsx` to only support `light` and `dark` with `defaultTheme="light"` (remove `enableSystem`)
- Update `theme-toggle.tsx` to a pill toggle with sun/moon icons (not a dropdown)

### Performance: backdrop-filter Usage
- Apply `backdrop-filter: blur(20px)` only to **large container elements**: sidebar, main content cards (KPI cards, chart cards, table card, dialog)
- **Do NOT apply blur** to small elements: buttons, input fields, filter pills, badges, pagination pills. These use solid semi-transparent rgba backgrounds instead (no blur)
- This limits GPU compositing to ~5-8 blur layers max per page, which performs well on modern hardware

## Typography

- Keep Geist Sans and Geist Mono fonts
- Page titles: `font-weight: 300`, `font-size: 24px`, `letter-spacing: -0.5px`
- Section labels (uppercase micro): `font-size: 10px`, `text-transform: uppercase`, `letter-spacing: 1.5px`, `color: secondary`
- KPI values: `font-weight: 300`, `font-size: 28px`
- Card metric values: `font-weight: 300`, `font-size: 22px`
- Body text: `font-size: 13px`, `font-weight: 400`
- Small/caption text: `font-size: 11px`
- Nav items: `font-size: 13px`
- Section group labels in sidebar: `font-size: 9px`, `letter-spacing: 1.5px`

## Component Redesign Spec

### 1. Layout Shell (`src/app/(dashboard)/layout.tsx`)

**Current:** `SidebarProvider` → `AppSidebar` + `SidebarInset` (Header + main with `p-6`)

**New:** Keep using `SidebarProvider` for responsive collapse/expand behavior, but restyle the shell. The `SidebarInset` wrapper is kept but restyled — remove its default border and background, and apply the gradient background via a wrapping `div` on the layout root. The main content area uses `padding: 6px 8px` to let cards handle their own internal padding. The gradient is set on the layout's outermost container so it covers the full viewport behind both sidebar and content.

### 2. Sidebar (`src/components/layout/app-sidebar.tsx`)

**Current:** Standard shadcn sidebar, dark `BA` logo square, collapsible icon mode, border-right edge-to-edge.

**New — Expandable Rail:**
- White frosted glass in light mode (`rgba(255,255,255,0.75)` + blur), dark glass in dark mode (`rgba(255,255,255,0.04)` + blur)
- `border-radius: 20px` — fully rounded as a floating shape
- Detached from edges: `margin: 14px` all around (floats within the gradient)
- `box-shadow: 0 4px 24px rgba(0,0,0,0.04)` (light) / `rgba(0,0,0,0.2)` (dark)
- Width: `240px` expanded, `72px` icon-only collapsed
- Logo: green gradient square (`linear-gradient(135deg, #3d8c5c, #2d6b44)`) with glow shadow, letter "B"
- Collapse toggle button: small rounded square in header row
- **Search bar**: Visual-only placeholder with `⌘K` shortcut badge — clicking it triggers the existing browser search or a future command palette. No new search feature is built; this is a styled placeholder that currently does nothing on click. Rounded input (`border-radius: 12px`), subtle background
- **Nav items**: `border-radius: 12px`, active item gets soft green fill (`rgba(61,140,92,0.1)` light / `rgba(61,140,92,0.12)` dark), icons from lucide-react
- **Section labels**: uppercase, `letter-spacing: 1.5px`, `font-size: 9px`
- **User card at bottom**: avatar (dark rounded square with initials), name, role, ellipsis menu — all inside a subtle background card with `border-radius: 12px`
- Hover states: items get subtle background on hover

### 3. Header (`src/components/layout/header.tsx`)

**Current:** Fixed 14px height bar with border-bottom, sidebar trigger, breadcrumbs, theme/language toggles, avatar dropdown.

**New:** No separate header bar. The header elements are integrated directly into the main content area's top section:
- Left side: Section label (uppercase, tiny) + page title (light weight, 24px)
- Right side: Action buttons (glass pills), theme toggle, language toggle, avatar
- All controls use glass treatment: `rgba(255,255,255,0.7)` + blur + rounded (`10px`)
- Breadcrumbs removed from header — the section label above the title serves as location context (e.g., "Overview", "Operations", "Master Data")
- The sidebar trigger button moves into the sidebar itself (the collapse toggle)

### 4. Breadcrumbs (`src/components/layout/breadcrumbs.tsx`)

**Current:** Horizontal breadcrumb trail in header.

**New:** For top-level pages, replaced by contextual section labels above page titles (e.g., "OPERATIONS" on inventory page). For nested/detail pages (e.g., `/purchase-orders/123`, `/projects/5/edit`), keep a minimal breadcrumb showing: section label > parent page link > current page. This ensures users can navigate back from detail views. The breadcrumb component is simplified (no full path, max 2 levels deep) and styled without separators — just subtle linked text.

### 5. Page Header (`src/components/shared/page-header.tsx`)

**Current:** Flex row with title + description on left, actions slot on right.

**New:**
- Title: `font-weight: 300`, `font-size: 24px`, `letter-spacing: -0.5px`
- Above title: section label in uppercase micro text
- Description: removed (the section label provides enough context)
- Action buttons on right use glass pill treatment
- Primary CTA: green gradient fill with glow shadow
- Secondary actions: glass background

### 6. Dashboard KPI Banner (`src/app/(dashboard)/components/dashboard-kpi-banner.tsx`)

**Current:** Dark gradient banner (`bg-gradient-to-r from-slate-900 to-slate-700`) with centered icon + label + value.

**New — 4 glass KPI cards in a grid:**
- Each card is a separate frosted glass card (`border-radius: 18px`)
- Layout per card:
  - Top row: label (uppercase micro) + colored icon badge (28x28 rounded square with gradient background matching the metric's semantic color)
  - Middle: large value (`font-weight: 300`, `font-size: 28px`)
  - Bottom: delta indicator (e.g., `+3.2%`) + mini inline sparkline SVG (~40x16px)
  - Sparklines: rendered as static `<svg><path>` elements using the last 7 data points from the trends API. Simple polyline path, no Recharts dependency. Data comes from the existing `/dashboard/trends` endpoint.
- Icon badge colors: green for birds, amber for FCR, red for mortality, purple for active projects
- Personalized greeting above the KPI row: "Good morning/afternoon/evening, {firstName}"
  - Morning: 5:00–11:59, Afternoon: 12:00–16:59, Evening: 17:00–4:59 (browser local time)
  - i18n keys: `dashboard.greeting.morning`, `dashboard.greeting.afternoon`, `dashboard.greeting.evening`

### 7. Mortality FCR Chart (`src/app/(dashboard)/components/mortality-fcr-chart.tsx`)

**Current:** shadcn Card wrapper, Recharts LineChart.

**New:**
- Frosted glass card container (no shadcn Card)
- Time-range filter pills in the card header (1D / 7D / 1M / All) — styled as small rounded pills, active gets dark fill (light mode) or light fill (dark mode)
- Legend as inline row below the header (colored line indicators + labels)
- Chart area: subtle horizontal grid lines (`rgba` based), gradient fill under FCR line, active data point indicator (dot + outer ring)
- Standard FCR as dashed reference line

### 8. Project Phase Chart (`src/app/(dashboard)/components/project-phase-chart.tsx`)

**Current:** Donut/pie chart in shadcn Card.

**New:**
- Frosted glass card container
- Donut chart with softer colors that match the new palette
- Legend items styled as small rounded pills below the chart
- Keep the donut chart but with updated palette colors

### 9. Sales Trend Chart (`src/app/(dashboard)/components/sales-trend-chart.tsx`)

**Current:** ComposedChart (Bar + Line) in shadcn Card.

**New:**
- Frosted glass card
- Compact layout: large current value at top (e.g., "Rp 142M"), delta below it
- Area/line sparkline chart instead of bar chart — gradient fill under the line
- The mini sparkline approach from the KPI card, but larger (spanning the card width)

### 10. Stock Alerts List (`src/app/(dashboard)/components/stock-alerts-list.tsx`)

**Current:** Card with bordered rows showing product, warehouse, quantity, status badge.

**New:**
- Frosted glass card
- Each alert row: rounded inner card (`border-radius: 14px`) with subtle semantic background tint
- Left: colored status dot (8px circle, critical = red with glow in dark mode)
- Center: product name + warehouse/quantity as subtext
- Right: status pill badge (filled background for critical, subtle for low)
- "View all" link in card header (green accent color)

### 11. Operational Cards (`src/app/(dashboard)/components/operational-cards.tsx`)

**Current:** 3 shadcn Cards with icon + label + value.

**New:**
- 3 frosted glass cards in a grid
- Each card: label (uppercase micro), large value, subtitle, and a small progress bar (segmented, colored by status)
- Progress bar segments: 4 rounded segments showing relative fill level

### 12. Data Table (`src/components/shared/data-table.tsx`)

**Current:** Standard table with shadcn Table component, search input, pagination.

**New:**
- Table sits inside a single frosted glass card (`border-radius: 18px`, overflow hidden)
- Column headers: uppercase micro labels, subtle bottom border
- Rows: clean separators (`rgba` based), no heavy borders
- Row hover: very subtle background change
- Search bar above the table: glass pill input
- Filter buttons: glass pill style with dropdown chevron
- Pagination: rounded number pills, active page gets dark fill
- Status badges in rows: rounded pills with semantic tinted backgrounds (not outlined)
- Action column: subtle ellipsis (⋯) button

### 13. Status Badge (`src/components/shared/status-badge.tsx`)

**Current:** Outlined badge variants.

**New:** Filled tinted badges — semantic color as background at low opacity, text in the full semantic color. `border-radius: 6px`, `padding: 3px 8px`, `font-size: 10px`, `text-transform: uppercase`, `letter-spacing: 0.3px`. No outline/border.
- Normal/Active: green tinted bg + green text
- Low/Warning: amber tinted bg + amber text
- Critical/Overdue: red tinted bg + red text
- Completed/Paid: green tinted bg + green text
- Draft/Pending: grey tinted bg + grey text

### 14. Loading Skeletons (`src/components/shared/loading-skeleton.tsx`)

**Current:** Standard shadcn skeleton rectangles.

**New:** Skeleton shapes should match the new card layouts:
- CardSkeleton: glass card shape with rounded corners (18px)
- TableSkeleton: glass card with shimmer rows inside
- PageSkeleton: full page layout with sidebar + content skeletons
- Skeleton pulse color adapts to theme (subtle on glass background)

### 15. Empty State (`src/components/shared/empty-state.tsx`)

**Current:** Centered with Package icon, title, description.

**New:** Same centered layout but inside a glass card. Icon gets a soft colored background circle. Description text in secondary color. Action button uses green gradient CTA style.

### 16. Button Variants (`src/components/ui/button.tsx`)

Update button variants to match the new design language:
- **Primary/default:** Green gradient fill (`linear-gradient(135deg, #3d8c5c, #2d6b44)`) with glow shadow, white text, `border-radius: 10px`
- **Secondary:** Glass background (`rgba(255,255,255,0.7)` light / `rgba(255,255,255,0.05)` dark) with border, `border-radius: 10px`
- **Ghost:** No background, text only, subtle hover background
- **Destructive:** Red fill, white text

### 17. Card (`src/components/ui/card.tsx`)

Update the base Card component's default styling to use glass treatment:
- `background: var(--glass-bg)`, `backdrop-filter: blur(var(--glass-blur))`, `border: 1px solid var(--glass-border)`, `box-shadow: var(--glass-shadow)`, `border-radius: 18px`
- This ensures all existing Card usages automatically get the new look

### 18. Input / Select / Textarea

Update form inputs to match:
- `border-radius: 12px`
- Glass background: subtle `rgba` based
- Focus ring: green accent
- Placeholder text: secondary color

### 19. Dialog / Sheet

Modal and side-panel overlays:
- Glass background with blur
- `border-radius: 20px` for dialogs
- Overlay: subtle dark tint with blur

### 20. Combobox Components (`src/components/forms/*.tsx`)

All 14+ entity combobox selectors inherit from Popover + Command:
- Popover content: glass card with rounded corners
- Command input: rounded, glass background
- Command items: rounded hover states, subtle active indicator
- No structural changes needed — styling flows from updated primitives

### 21. Theme Toggle (`src/components/layout/theme-toggle.tsx`)

**Current:** Dropdown with 4 options (Light, Dark, Emerald, System).

**New:** Pill toggle with sun/moon icons — not a dropdown. Single button that toggles between light and dark. Semi-transparent glass background (solid rgba, no blur).

### 22. Language Toggle (`src/components/layout/language-toggle.tsx`)

Same as current but styled as a glass pill.

### 23. Chart Configuration (`src/components/ui/chart.tsx`)

Update chart tooltip and legend styling:
- Tooltip: glass card appearance (blur, rounded, subtle shadow)
- Legend: inline rounded pills
- Chart colors: update palette to match new accent colors

## Dashboard Page Layout

**Greeting row:** "Good morning/afternoon/evening, {firstName}" with date and action buttons

**Row 1 — KPIs:** 4-column grid of glass KPI cards (birds, FCR, mortality, active projects)

**Row 2 — Charts + Alerts:** 2-column grid (60/40 split)
- Left: FCR & Mortality trend chart with time-range pills
- Right: Stock alerts list with status dots

**Row 3 — Operations:** 3-column grid
- Sales trend (sparkline + value)
- Open POs (value + progress bar)
- Overdue invoices (value + progress bar)

**Row 4 — Project Phases:** Project phase donut chart (always shown, below operations row)

## Table Page Layout (all CRUD pages)

**Header row:** Section label + page title on left, primary CTA + secondary actions on right

**Filter bar:** Search input (glass pill) + filter dropdowns (glass pills) in a horizontal row

**Table card:** Single frosted glass card containing:
- Column headers (uppercase micro)
- Data rows with clean separators
- Pagination at the bottom

## Responsive Behavior

- Sidebar collapses to icon-only rail on tablets
- Sidebar becomes a sheet/drawer on mobile
- KPI grid: 4 cols → 2 cols on tablet → 1 col on mobile
- Chart + alerts row: 2 cols → stacked on mobile
- Operations row: 3 cols → stacked on mobile
- Table: horizontal scroll on small screens

## Files to Modify

### Core Theme
1. `src/app/globals.css` — CSS variables, gradient backgrounds, glass tokens, remove emerald theme
2. `src/components/providers.tsx` — Remove emerald from ThemeProvider

### Layout
3. `src/app/(dashboard)/layout.tsx` — Gradient background, new shell structure
4. `src/components/layout/app-sidebar.tsx` — Complete redesign to expandable rail
5. `src/components/layout/collapsible-sidebar-group.tsx` — Update styling for rounded items
6. `src/components/layout/header.tsx` — Integrate into content area, remove bar
7. `src/components/layout/breadcrumbs.tsx` — Replace with section labels (may deprecate)
8. `src/components/layout/theme-toggle.tsx` — Light/Dark only toggle
9. `src/components/layout/language-toggle.tsx` — Glass pill styling

### Shared Components
10. `src/components/shared/page-header.tsx` — New editorial header with section label
11. `src/components/shared/data-table.tsx` — Glass card wrapper, updated row styling
12. `src/components/shared/status-badge.tsx` — Filled tinted badges
13. `src/components/shared/loading-skeleton.tsx` — Glass-aware skeletons
14. `src/components/shared/empty-state.tsx` — Glass card with styled icon

### UI Primitives
15. `src/components/ui/card.tsx` — Glass defaults
16. `src/components/ui/button.tsx` — Updated variants
17. `src/components/ui/badge.tsx` — Tinted fill style
18. `src/components/ui/input.tsx` — Rounded glass input
19. `src/components/ui/select.tsx` — Rounded glass select
20. `src/components/ui/textarea.tsx` — Rounded glass textarea
21. `src/components/ui/dialog.tsx` — Glass overlay
22. `src/components/ui/sheet.tsx` — Glass side panel
23. `src/components/ui/chart.tsx` — Updated tooltip/legend styling
24. `src/components/ui/table.tsx` — Clean row separators
25. `src/components/ui/skeleton.tsx` — Theme-aware pulse

### Dashboard Components
26. `src/app/(dashboard)/page.tsx` — New bento layout with greeting
27. `src/app/(dashboard)/components/dashboard-kpi-banner.tsx` — Glass KPI cards with sparklines
28. `src/app/(dashboard)/components/mortality-fcr-chart.tsx` — Glass card, time pills, gradient fill
29. `src/app/(dashboard)/components/project-phase-chart.tsx` — Glass card, updated colors
30. `src/app/(dashboard)/components/sales-trend-chart.tsx` — Compact sparkline style
31. `src/app/(dashboard)/components/stock-alerts-list.tsx` — Status dots, semantic tints
32. `src/app/(dashboard)/components/operational-cards.tsx` — Progress bars, glass cards

## Additional Components

### ChatWidget (`src/components/chat/chat-widget.tsx`)
- Keep functionality as-is
- Restyle the floating button and chat panel to use glass treatment (rounded corners, glass background)

### Confirm Dialog (`src/components/shared/confirm-dialog.tsx`)
- Inherits updated Dialog and Button styling — no dedicated changes needed

### Status Action (`src/components/shared/status-action.tsx`)
- Inherits updated DropdownMenu and Button styling — no dedicated changes needed

### Sonner/Toaster
- Update the Toaster in `providers.tsx` to use glass-like toast styling: rounded corners (14px), semi-transparent background, subtle border

### Tooltip (`src/components/ui/tooltip.tsx`)
- Update tooltip styling: glass background, rounded corners (10px), no heavy border

### Popover (`src/components/ui/popover.tsx`)
- Update popover content styling: glass card appearance, rounded corners (14px)

## i18n New Keys

Add the following keys to both `messages/en.json` and `messages/id.json`:
- `dashboard.greeting.morning` — "Good morning"
- `dashboard.greeting.afternoon` — "Good afternoon"
- `dashboard.greeting.evening` — "Good evening"
- `dashboard.timeRange.1d` / `7d` / `1m` / `all`
- `dashboard.viewAll` — "View all"

## Files to Modify (additional)

33. `src/components/chat/chat-widget.tsx` — Glass treatment on chat panel
34. `src/components/ui/tooltip.tsx` — Glass tooltip
35. `src/components/ui/popover.tsx` — Glass popover
36. `src/components/providers.tsx` — Toaster styling update

## Non-Goals

- No structural changes to routing or data fetching
- No backend API changes
- No changes to form validation logic
- No changes to auth flow
- No changes to login/auth pages (`(auth)` route group) — these remain as-is
- Combobox components inherit styling from updated primitives — no individual redesign needed
- No new dependencies (backdrop-filter is natively supported in all modern browsers)
- The sidebar ⌘K search is visual-only placeholder — no global search feature is built
