# Dashboard Analytics Redesign

## Problem

The current dashboard page shows basic stat cards with several placeholder values (bird population, FCR, mortality rate show "-"). It lacks charts, trends, and actionable insights that farm managers need for daily decision-making.

## Solution

Build a purpose-built analytics dashboard with two new backend endpoints and a redesigned frontend with charts and KPI indicators.

## Backend

### New Module: `dashboard`

Create a `DashboardModule` with `DashboardController` and `DashboardService`.

### `GET /dashboard/stats`

Returns all dashboard KPIs in a single call. Tenant-scoped, accessible by all roles.

Response shape:

```typescript
{
  // Production KPIs
  activeProjects: number;          // COUNT of projects with active status
  birdPopulation: number;          // SUM of ProjectChickIn.population for active projects
  avgFcr: number | null;           // Weighted average FCR across active projects
  mortalityRate: number | null;    // Percentage mortality across active projects

  // Operations
  openPurchaseOrders: number;      // POs with status IN (ORDERED, PROCESSING)
  pendingSalesOrders: number;      // SOs with status IN (PENDING_APPROVAL, APPROVED)
  criticalStockAlerts: number;     // InventoryStock with status IN (LOW, CRITICAL)

  // Financial
  pendingInvoiceAmount: string;    // SUM of unpaid/partial invoice totals (Decimal serialized as string)
  overdueInvoiceCount: number;     // COUNT of overdue invoices

  // Project phases
  projectsByPhase: {
    rearing: number;
    harvest: number;
    cleaning: number;
    preparation: number;
  };
}
```

**Computation details:**
- `birdPopulation`: `SUM(ProjectChickIn.population)` WHERE `Project.isActive = true` and chick-in is within active date range. Join path: `ProjectChickIn -> ProjectCoop -> Project (tenantId)`. Filter `deletedAt: null` on all models.
- `avgFcr`: Since no daily recording model exists yet, compute estimated FCR from `FcrStandardDetail` based on current day-of-project (days since rearingStartDate). Weighted average across active projects by population.
- `mortalityRate`: Since no daily recording model exists yet, return `null` for now. Will be populated when a recording module is built. (Show "-" on frontend when null.)
- `projectsByPhase`: Determined by comparing current date against `ProjectChickIn` date ranges (`rearingStartDate`, `harvestStartDate`, `cleaningStartDate`, `prepStartDate`). If a project has multiple chick-ins, use the earliest active one. Join path for tenant scoping: `ProjectChickIn -> ProjectCoop -> Project (tenantId)`.
- `criticalStockAlerts`: Join `InventoryStock -> Warehouse -> Branch (tenantId)` or `Warehouse -> Farm -> Branch (tenantId)` to scope by tenant.
- `pendingInvoiceAmount`: Use `Decimal` type from Prisma for financial calculations, serialize as string in JSON response to preserve precision.
- All queries must include `deletedAt: null` filter for soft-delete compliance.

### `GET /dashboard/trends?days=30`

Returns time-series data for charts. Default 30 days, max 90.

Response shape:

```typescript
{
  mortalityTrend: Array<{
    date: string;      // ISO date
    rate: number;      // percentage
  }>;

  fcrTrend: Array<{
    date: string;
    value: number;     // actual FCR
    standard: number;  // standard FCR for comparison
  }>;

  salesTrend: Array<{
    week: string;      // ISO week start date
    revenue: number;
    orderCount: number;
  }>;
}
```

**Computation details:**
- `mortalityTrend`: No daily recording model exists yet. Return empty array for now. Will be populated when a recording module is built. Frontend should show "No data available" state.
- `fcrTrend`: Compute estimated FCR from `FcrStandardDetail` benchmarks based on day-of-project for each active project. Shows the standard curve rather than actual performance until a recording module exists.
- `salesTrend`: Weekly aggregation of `SalesInvoice.totalAmount` (using Decimal) and `SalesOrder` counts. Group by ISO week. All queries include `deletedAt: null` filter. Note: weekly granularity differs from daily for other trends — frontend charts handle different X-axis scales.

### Backend implementation notes

- Standard NestJS module pattern: module, controller, service
- Use Prisma `aggregate()` and `groupBy()` for counts and sums
- Use raw SQL (`prisma.$queryRaw`) for date-based trend aggregations where Prisma's API is insufficient
- Both endpoints are tenant-scoped via `@CurrentTenant()` decorator (from `src/common/decorators/current-tenant.decorator.ts`)
- Both endpoints are guarded by `@UseGuards(JwtAuthGuard)`
- Add `@ApiTags('dashboard')`, `@ApiBearerAuth()`, and `@ApiOperation()` Swagger decorators following existing controller patterns
- Add `QueryDashboardTrendsDto` for the `days` query param: `@IsOptional()`, `@IsInt()`, `@Min(1)`, `@Max(90)`, default 30
- Response types defined as interfaces for Swagger `@ApiResponse()` documentation

## Frontend

### Page Structure

The dashboard page (`src/app/(dashboard)/page.tsx`) is redesigned with this layout:

```
Row 1: KPI Banner (full width)
  - Bird Population, FCR vs Standard, Mortality Rate, Active Projects
  - Each KPI shows value + trend indicator

Row 2: Two charts side by side (50/50)
  - Left: Mortality & FCR Trend (LineChart, dual Y-axis, 30 days)
  - Right: Projects by Phase (PieChart/DonutChart)

Row 3: Two panels side by side (50/50)
  - Left: Weekly Sales (BarChart, revenue + order count)
  - Right: Stock Alerts (list of critical/low items with links)

Row 4: Three small stat cards (33/33/33)
  - Open POs, Pending Sales, Invoice Aging (overdue count + amount)
```

### Components

All dashboard-specific components in `src/app/(dashboard)/_components/`:

| Component | Chart Type | Data Source |
|-----------|-----------|-------------|
| `DashboardKpiBanner` | Stat display | `/dashboard/stats` |
| `MortalityFcrChart` | Recharts LineChart (dual axis) | `/dashboard/trends` |
| `ProjectPhaseChart` | Recharts PieChart | `/dashboard/stats` |
| `SalesTrendChart` | Recharts BarChart | `/dashboard/trends` |
| `StockAlertsList` | List with badges | `/inventory-stocks?stockStatus=LOW&stockStatus=CRITICAL&limit=5` |
| `OperationalCards` | Stat cards | `/dashboard/stats` |

### Data Fetching

Use the existing `useApi<T>` hook for `/dashboard/stats` and `/dashboard/trends?days=30`. Use `usePaginated` for the stock alerts list. Each component handles its own data fetching via these hooks, which provide built-in loading/error states and cancellation.

```typescript
// In page.tsx - use useApi for dashboard-specific endpoints
const { data: stats, isLoading: statsLoading } = useApi<DashboardStats>("/dashboard/stats");
const { data: trends, isLoading: trendsLoading } = useApi<DashboardTrends>("/dashboard/trends?days=30");

// In StockAlertsList - use usePaginated
const { data: stocks, isLoading } = usePaginated<InventoryStock>("/inventory-stocks", {
  limit: 5,
  extra: { stockStatus: "LOW" },
});
```

Each chart component receives data as props and handles `null` gracefully (shows "No data available" state).

### Chart Configuration

- **Mortality & FCR Chart**: Dual Y-axis LineChart. Left axis: mortality % (red line). Right axis: FCR value (blue line) with standard comparison (dashed gray line). Uses ChartContainer and ChartTooltip from existing chart.tsx wrapper.
- **Projects by Phase Chart**: PieChart with 4 segments (rearing=green, harvest=amber, cleaning=blue, preparation=gray). Shows count labels.
- **Sales Trend Chart**: BarChart with revenue bars (green) and order count line overlay. Weekly grouping.

### Color Scheme

Follows existing theme variables. Key colors:
- Production/birds: `text-blue-400` / blue chart colors
- FCR: `text-amber-400` / amber chart colors
- Mortality: `text-red-400` / red chart colors
- Revenue/positive: `text-green-400` / green chart colors
- Alerts/warnings: `text-orange-500`

### i18n Keys

Add to both `messages/en.json` and `messages/id.json` under `"dashboard"`:

```json
{
  "dashboard": {
    "title": "Dashboard",
    "description": "Overview of your farm operations",
    "birdPopulation": "Bird Population",
    "avgFcr": "Avg FCR",
    "mortalityRate": "Mortality Rate",
    "activeProjects": "Active Projects",
    "mortalityFcrTrend": "Mortality & FCR Trend",
    "projectsByPhase": "Projects by Phase",
    "salesTrend": "Weekly Sales",
    "stockAlerts": "Stock Alerts",
    "openPOs": "Open POs",
    "pendingSales": "Pending Sales",
    "invoiceAging": "Invoice Aging",
    "overdueInvoices": "Overdue Invoices",
    "pendingAmount": "Pending Amount",
    "rearing": "Rearing",
    "harvest": "Harvest",
    "cleaning": "Cleaning",
    "preparation": "Preparation",
    "revenue": "Revenue",
    "orders": "Orders",
    "viewAll": "View All",
    "noData": "No data available",
    "last30Days": "Last 30 Days",
    "vsStandard": "vs Standard",
    "fcrValue": "FCR Value",
    "mortalityPercent": "Mortality %",
    "standardFcr": "Standard FCR",
    "date": "Date",
    "week": "Week"
  }
}
```

### Responsive Behavior

- Desktop (>=1024px): 2-column grid for charts, 3-column for stat cards
- Tablet (>=768px): 2-column grid, cards stack to 2 columns
- Mobile (<768px): Single column, all components stack

### Loading State

Single full-page skeleton while data loads (reuse existing `CardSkeleton`). Show partial data if one endpoint fails (use `Promise.allSettled`).

## Files to Create/Modify

### New files:
- `breeding-app/src/modules/dashboard/dashboard.module.ts`
- `breeding-app/src/modules/dashboard/dashboard.controller.ts`
- `breeding-app/src/modules/dashboard/dashboard.service.ts`
- `breeding-app/src/modules/dashboard/dto/query-dashboard-trends.dto.ts`
- `breeding-dashboard/src/app/(dashboard)/_components/dashboard-kpi-banner.tsx`
- `breeding-dashboard/src/app/(dashboard)/_components/mortality-fcr-chart.tsx`
- `breeding-dashboard/src/app/(dashboard)/_components/project-phase-chart.tsx`
- `breeding-dashboard/src/app/(dashboard)/_components/sales-trend-chart.tsx`
- `breeding-dashboard/src/app/(dashboard)/_components/stock-alerts-list.tsx`
- `breeding-dashboard/src/app/(dashboard)/_components/operational-cards.tsx`

### Modified files:
- `breeding-app/src/app.module.ts` — register DashboardModule
- `breeding-dashboard/src/app/(dashboard)/page.tsx` — rewrite with new components
- `breeding-dashboard/src/types/api.ts` — add DashboardStats and DashboardTrends types
- `breeding-dashboard/messages/en.json` — add dashboard i18n keys
- `breeding-dashboard/messages/id.json` — add dashboard i18n keys

## Out of Scope

- Detailed P&L analytics (exists in AI Insights page)
- Employee/driver performance metrics (better as dedicated pages)
- Real-time updates / WebSocket (future enhancement)
- Date range picker for dashboard (future enhancement — hardcoded to 30 days for now)
- Export/download functionality
