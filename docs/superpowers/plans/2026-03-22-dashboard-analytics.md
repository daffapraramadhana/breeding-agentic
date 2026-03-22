# Dashboard Analytics Redesign — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the placeholder dashboard with a data-rich analytics page featuring KPI banners, trend charts, and operational alerts.

**Architecture:** Two new backend endpoints (`GET /dashboard/stats`, `GET /dashboard/trends`) in a dedicated NestJS module aggregate data from Projects, Sales, Inventory, and Invoices. The frontend consumes these via `useApi` hooks and renders Recharts-based visualizations inside shadcn/ui cards.

**Tech Stack:** NestJS 11, Prisma 7, Next.js 16, Recharts 2.15, shadcn/ui, next-intl

**Spec:** `docs/superpowers/specs/2026-03-22-dashboard-analytics-design.md`

---

## File Structure

### New Files (Backend)
| File | Responsibility |
|------|---------------|
| `breeding-app/src/modules/dashboard/dashboard.module.ts` | Module registration |
| `breeding-app/src/modules/dashboard/dashboard.controller.ts` | Two GET endpoints |
| `breeding-app/src/modules/dashboard/dashboard.service.ts` | All aggregation queries |
| `breeding-app/src/modules/dashboard/dto/query-dashboard-trends.dto.ts` | `days` query param validation |

### New Files (Frontend)
| File | Responsibility |
|------|---------------|
| `breeding-dashboard/src/app/(dashboard)/components/dashboard-kpi-banner.tsx` | Hero KPI row |
| `breeding-dashboard/src/app/(dashboard)/components/mortality-fcr-chart.tsx` | Dual-axis line chart |
| `breeding-dashboard/src/app/(dashboard)/components/project-phase-chart.tsx` | Donut chart |
| `breeding-dashboard/src/app/(dashboard)/components/sales-trend-chart.tsx` | Bar chart |
| `breeding-dashboard/src/app/(dashboard)/components/stock-alerts-list.tsx` | Critical stock list |
| `breeding-dashboard/src/app/(dashboard)/components/operational-cards.tsx` | PO / Sales / Invoice cards |

### Modified Files
| File | Change |
|------|--------|
| `breeding-app/src/app.module.ts` | Import DashboardModule |
| `breeding-dashboard/src/app/(dashboard)/page.tsx` | Rewrite with new components |
| `breeding-dashboard/src/types/api.ts` | Add DashboardStats, DashboardTrends types |
| `breeding-dashboard/messages/en.json` | Add `dashboard` i18n keys |
| `breeding-dashboard/messages/id.json` | Add `dashboard` i18n keys |

---

## Task 1: Backend — DTO and Module Shell

**Files:**
- Create: `breeding-app/src/modules/dashboard/dto/query-dashboard-trends.dto.ts`
- Create: `breeding-app/src/modules/dashboard/dashboard.module.ts`
- Create: `breeding-app/src/modules/dashboard/dashboard.controller.ts`
- Create: `breeding-app/src/modules/dashboard/dashboard.service.ts`
- Modify: `breeding-app/src/app.module.ts`

- [ ] **Step 1: Create the trends query DTO**

```typescript
// breeding-app/src/modules/dashboard/dto/query-dashboard-trends.dto.ts
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class QueryDashboardTrendsDto {
  @ApiPropertyOptional({ description: 'Number of days for trend data (1-90)', default: 30 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(90)
  days: number = 30;
}
```

- [ ] **Step 2: Create the dashboard service (empty shell)**

```typescript
// breeding-app/src/modules/dashboard/dashboard.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service.js';

@Injectable()
export class DashboardService {
  constructor(private readonly prisma: PrismaService) {}

  async getStats(tenantId: string) {
    return {
      activeProjects: 0,
      birdPopulation: 0,
      avgFcr: null,
      mortalityRate: null,
      openPurchaseOrders: 0,
      pendingSalesOrders: 0,
      criticalStockAlerts: 0,
      pendingInvoiceAmount: '0',
      overdueInvoiceCount: 0,
      projectsByPhase: { rearing: 0, harvest: 0, cleaning: 0, preparation: 0 },
    };
  }

  async getTrends(tenantId: string, days: number) {
    return {
      mortalityTrend: [],
      fcrTrend: [],
      salesTrend: [],
    };
  }
}
```

- [ ] **Step 3: Create the dashboard controller**

```typescript
// breeding-app/src/modules/dashboard/dashboard.controller.ts
import { Controller, Get, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation } from '@nestjs/swagger';
import { DashboardService } from './dashboard.service.js';
import { JwtAuthGuard } from '../../common/guards/jwt-auth.guard.js';
import { CurrentTenant } from '../../common/decorators/current-tenant.decorator.js';
import { QueryDashboardTrendsDto } from './dto/query-dashboard-trends.dto.js';

@ApiTags('Dashboard')
@ApiBearerAuth()
@Controller('dashboard')
@UseGuards(JwtAuthGuard)
export class DashboardController {
  constructor(private readonly dashboardService: DashboardService) {}

  @ApiOperation({ summary: 'Get dashboard statistics' })
  @Get('stats')
  getStats(@CurrentTenant() tenantId: string) {
    return this.dashboardService.getStats(tenantId);
  }

  @ApiOperation({ summary: 'Get dashboard trend data' })
  @Get('trends')
  getTrends(
    @CurrentTenant() tenantId: string,
    @Query() query: QueryDashboardTrendsDto,
  ) {
    return this.dashboardService.getTrends(tenantId, query.days);
  }
}
```

- [ ] **Step 4: Create the dashboard module**

```typescript
// breeding-app/src/modules/dashboard/dashboard.module.ts
import { Module } from '@nestjs/common';
import { DashboardController } from './dashboard.controller.js';
import { DashboardService } from './dashboard.service.js';

@Module({
  controllers: [DashboardController],
  providers: [DashboardService],
})
export class DashboardModule {}
```

- [ ] **Step 5: Register DashboardModule in app.module.ts**

Add import at top:
```typescript
import { DashboardModule } from './modules/dashboard/dashboard.module.js';
```

Add `DashboardModule` to the `imports` array (after `ChatModule`).

- [ ] **Step 6: Verify build**

Run: `cd breeding-app && npm run build`
Expected: Build succeeds with no errors.

- [ ] **Step 7: Commit**

```bash
git add breeding-app/src/modules/dashboard/ breeding-app/src/app.module.ts
git commit -m "feat(dashboard): add dashboard module shell with stats and trends endpoints"
```

---

## Task 2: Backend — Implement Stats Endpoint

**Files:**
- Modify: `breeding-app/src/modules/dashboard/dashboard.service.ts`

- [ ] **Step 1: Implement getStats with all aggregation queries**

Replace the `getStats` method in `dashboard.service.ts` with real Prisma queries:

```typescript
async getStats(tenantId: string) {
  const now = new Date();

  // Run all counts in parallel
  const [
    activeProjects,
    chickInData,
    openPOs,
    pendingSOs,
    stockAlerts,
    invoiceAgg,
    overdueCount,
    projectPhaseData,
  ] = await Promise.all([
    // Active projects count
    this.prisma.project.count({
      where: { tenantId, isActive: true, deletedAt: null },
    }),

    // Bird population from active project chick-ins
    this.prisma.projectChickIn.findMany({
      where: {
        projectCoop: {
          project: { tenantId, isActive: true, deletedAt: null },
        },
      },
      select: { population: true },
    }),

    // Open purchase orders
    this.prisma.purchaseOrder.count({
      where: {
        tenantId,
        deletedAt: null,
        status: { in: ['ORDERED', 'PROCESSING'] },
      },
    }),

    // Pending sales orders
    this.prisma.salesOrder.count({
      where: {
        tenantId,
        deletedAt: null,
        status: { in: ['PENDING_APPROVAL', 'APPROVED'] },
      },
    }),

    // Critical stock alerts (join through warehouse for tenant scoping)
    this.prisma.inventoryStock.count({
      where: {
        warehouse: { tenantId, deletedAt: null },
        stockStatus: { in: ['LOW', 'CRITICAL'] },
      },
    }),

    // Pending invoice amount (UNPAID + PARTIAL)
    this.prisma.salesInvoice.aggregate({
      where: {
        tenantId,
        paymentStatus: { in: ['UNPAID', 'PARTIAL'] },
      },
      _sum: { remainingAmount: true },
    }),

    // Overdue invoice count
    this.prisma.salesInvoice.count({
      where: {
        tenantId,
        paymentStatus: 'OVERDUE',
      },
    }),

    // Project phase data: get all active chick-ins with dates
    this.prisma.projectChickIn.findMany({
      where: {
        projectCoop: {
          project: { tenantId, isActive: true, deletedAt: null },
        },
      },
      select: {
        rearingStartDate: true,
        rearingEndDate: true,
        harvestStartDate: true,
        harvestEndDate: true,
        cleaningStartDate: true,
        cleaningEndDate: true,
        prepStartDate: true,
        prepEndDate: true,
      },
    }),
  ]);

  // Sum bird population
  const birdPopulation = chickInData.reduce((sum, ci) => sum + ci.population, 0);

  // Determine project phases
  const projectsByPhase = { rearing: 0, harvest: 0, cleaning: 0, preparation: 0 };
  for (const ci of projectPhaseData) {
    if (ci.prepStartDate && now >= ci.prepStartDate && (!ci.prepEndDate || now <= ci.prepEndDate)) {
      projectsByPhase.preparation++;
    } else if (ci.cleaningStartDate && now >= ci.cleaningStartDate && (!ci.cleaningEndDate || now <= ci.cleaningEndDate)) {
      projectsByPhase.cleaning++;
    } else if (ci.harvestStartDate && now >= ci.harvestStartDate && (!ci.harvestEndDate || now <= ci.harvestEndDate)) {
      projectsByPhase.harvest++;
    } else if (ci.rearingStartDate && now >= ci.rearingStartDate && (!ci.rearingEndDate || now <= ci.rearingEndDate)) {
      projectsByPhase.rearing++;
    }
  }

  const pendingAmount = invoiceAgg._sum.remainingAmount;

  return {
    activeProjects,
    birdPopulation,
    avgFcr: null as number | null, // No recording module yet
    mortalityRate: null as number | null, // No recording module yet
    openPurchaseOrders: openPOs,
    pendingSalesOrders: pendingSOs,
    criticalStockAlerts: stockAlerts,
    pendingInvoiceAmount: pendingAmount ? pendingAmount.toString() : '0',
    overdueInvoiceCount: overdueCount,
    projectsByPhase,
  };
}
```

- [ ] **Step 2: Verify build**

Run: `cd breeding-app && npm run build`
Expected: Build succeeds.

- [ ] **Step 3: Commit**

```bash
git add breeding-app/src/modules/dashboard/dashboard.service.ts
git commit -m "feat(dashboard): implement stats endpoint with production, operations, and financial KPIs"
```

---

## Task 3: Backend — Implement Trends Endpoint

**Files:**
- Modify: `breeding-app/src/modules/dashboard/dashboard.service.ts`

- [ ] **Step 1: Implement getTrends method**

Replace the `getTrends` method:

```typescript
async getTrends(tenantId: string, days: number) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);

  // Sales trend: weekly aggregation of invoices
  const salesInvoices = await this.prisma.salesInvoice.findMany({
    where: {
      tenantId,
      invoiceDate: { gte: startDate },
    },
    select: {
      invoiceDate: true,
      totalAmount: true,
    },
    orderBy: { invoiceDate: 'asc' },
  });

  // Group sales by ISO week
  const salesByWeek = new Map<string, { revenue: number; orderCount: number }>();
  for (const inv of salesInvoices) {
    const weekStart = this.getWeekStart(inv.invoiceDate);
    const key = weekStart.toISOString().split('T')[0];
    const existing = salesByWeek.get(key) || { revenue: 0, orderCount: 0 };
    existing.revenue += Number(inv.totalAmount);
    existing.orderCount++;
    salesByWeek.set(key, existing);
  }

  const salesTrend = Array.from(salesByWeek.entries())
    .map(([week, data]) => ({
      week,
      revenue: Math.round(data.revenue),
      orderCount: data.orderCount,
    }))
    .sort((a, b) => a.week.localeCompare(b.week));

  // FCR trend: show standard curve for active projects (no actual recording data yet)
  const fcrTrend = await this.getFcrStandardTrend(tenantId, days);

  return {
    mortalityTrend: [] as Array<{ date: string; rate: number }>, // No recording module yet
    fcrTrend,
    salesTrend,
  };
}

private getWeekStart(date: Date): Date {
  const d = new Date(date);
  const day = d.getDay();
  const diff = d.getDate() - day + (day === 0 ? -6 : 1); // Monday start
  d.setDate(diff);
  d.setHours(0, 0, 0, 0);
  return d;
}

private async getFcrStandardTrend(tenantId: string, days: number) {
  // Get active projects with their FCR standards
  const projects = await this.prisma.project.findMany({
    where: { tenantId, isActive: true, deletedAt: null, fcrStandardId: { not: null } },
    select: {
      fcrStandard: {
        select: {
          details: {
            select: { bodyWeight: true, fcr: true },
            orderBy: { bodyWeight: 'asc' },
          },
        },
      },
      projectCoops: {
        select: {
          chickIns: {
            select: { rearingStartDate: true },
            take: 1,
          },
        },
        take: 1,
      },
    },
  });

  const now = new Date();
  const trend: Array<{ date: string; value: number; standard: number }> = [];

  // Generate daily FCR standard points for the last N days
  for (let i = days - 1; i >= 0; i--) {
    const date = new Date(now);
    date.setDate(date.getDate() - i);
    const dateStr = date.toISOString().split('T')[0];

    let totalFcr = 0;
    let count = 0;

    for (const project of projects) {
      const startDate = project.projectCoops[0]?.chickIns[0]?.rearingStartDate;
      if (!startDate || !project.fcrStandard) continue;

      const dayOfProject = Math.floor((date.getTime() - new Date(startDate).getTime()) / (1000 * 60 * 60 * 24));
      if (dayOfProject < 0) continue;

      // Estimate body weight from day (rough: ~60g/day growth for broilers)
      const estimatedWeight = dayOfProject * 0.06; // kg
      const details = project.fcrStandard.details;

      // Find closest FCR standard detail by body weight
      let closestDetail = details[0];
      for (const d of details) {
        if (Number(d.bodyWeight) <= estimatedWeight) {
          closestDetail = d;
        }
      }

      if (closestDetail) {
        totalFcr += Number(closestDetail.fcr);
        count++;
      }
    }

    if (count > 0) {
      const avgFcr = totalFcr / count;
      // value and standard are the same until a recording module provides actual FCR data
      trend.push({ date: dateStr, value: avgFcr, standard: avgFcr });
    }
  }

  return trend;
}
```

- [ ] **Step 2: Verify build**

Run: `cd breeding-app && npm run build`
Expected: Build succeeds.

- [ ] **Step 3: Commit**

```bash
git add breeding-app/src/modules/dashboard/dashboard.service.ts
git commit -m "feat(dashboard): implement trends endpoint with sales weekly aggregation and FCR standard curve"
```

---

## Task 4: Frontend — Add API Types and i18n Keys

**Files:**
- Modify: `breeding-dashboard/src/types/api.ts`
- Modify: `breeding-dashboard/messages/en.json`
- Modify: `breeding-dashboard/messages/id.json`

- [ ] **Step 1: Add DashboardStats and DashboardTrends types to api.ts**

Add at the end of `breeding-dashboard/src/types/api.ts`:

```typescript
// Dashboard Analytics
export interface DashboardStats {
  activeProjects: number;
  birdPopulation: number;
  avgFcr: number | null;
  mortalityRate: number | null;
  openPurchaseOrders: number;
  pendingSalesOrders: number;
  criticalStockAlerts: number;
  pendingInvoiceAmount: string;
  overdueInvoiceCount: number;
  projectsByPhase: {
    rearing: number;
    harvest: number;
    cleaning: number;
    preparation: number;
  };
}

export interface DashboardTrends {
  mortalityTrend: Array<{
    date: string;
    rate: number;
  }>;
  fcrTrend: Array<{
    date: string;
    value: number;
    standard: number;
  }>;
  salesTrend: Array<{
    week: string;
    revenue: number;
    orderCount: number;
  }>;
}
```

- [ ] **Step 2: Add dashboard i18n keys to en.json**

Add a `"dashboard"` key at the top level of `messages/en.json`:

```json
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
  "week": "Week",
  "birds": "birds",
  "critical": "Critical",
  "low": "Low"
}
```

- [ ] **Step 3: Add dashboard i18n keys to id.json**

Add a `"dashboard"` key at the top level of `messages/id.json`:

```json
"dashboard": {
  "title": "Dasbor",
  "description": "Ringkasan operasional peternakan Anda",
  "birdPopulation": "Populasi Ayam",
  "avgFcr": "Rata-rata FCR",
  "mortalityRate": "Tingkat Mortalitas",
  "activeProjects": "Proyek Aktif",
  "mortalityFcrTrend": "Tren Mortalitas & FCR",
  "projectsByPhase": "Proyek per Fase",
  "salesTrend": "Penjualan Mingguan",
  "stockAlerts": "Peringatan Stok",
  "openPOs": "PO Terbuka",
  "pendingSales": "Penjualan Tertunda",
  "invoiceAging": "Umur Faktur",
  "overdueInvoices": "Faktur Jatuh Tempo",
  "pendingAmount": "Jumlah Tertunda",
  "rearing": "Pemeliharaan",
  "harvest": "Panen",
  "cleaning": "Pembersihan",
  "preparation": "Persiapan",
  "revenue": "Pendapatan",
  "orders": "Pesanan",
  "viewAll": "Lihat Semua",
  "noData": "Tidak ada data",
  "last30Days": "30 Hari Terakhir",
  "vsStandard": "vs Standar",
  "fcrValue": "Nilai FCR",
  "mortalityPercent": "Mortalitas %",
  "standardFcr": "Standar FCR",
  "date": "Tanggal",
  "week": "Minggu",
  "birds": "ekor",
  "critical": "Kritis",
  "low": "Rendah"
}
```

- [ ] **Step 4: Commit**

```bash
git add breeding-dashboard/src/types/api.ts breeding-dashboard/messages/en.json breeding-dashboard/messages/id.json
git commit -m "feat(dashboard): add dashboard analytics types and i18n keys"
```

---

## Task 5: Frontend — KPI Banner Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/dashboard-kpi-banner.tsx`

- [ ] **Step 1: Create the KPI banner component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/dashboard-kpi-banner.tsx
"use client";

import { useTranslations } from "next-intl";
import { Bird, TrendingUp, Skull, FolderOpen } from "lucide-react";
import { DashboardStats } from "@/types/api";
import { formatQuantity } from "@/lib/utils";

interface DashboardKpiBannerProps {
  stats: DashboardStats;
}

export function DashboardKpiBanner({ stats }: DashboardKpiBannerProps) {
  const t = useTranslations("dashboard");

  const kpis = [
    {
      label: t("birdPopulation"),
      value: formatQuantity(stats.birdPopulation),
      icon: Bird,
      color: "text-blue-400",
    },
    {
      label: t("avgFcr"),
      value: stats.avgFcr !== null ? stats.avgFcr.toFixed(2) : "-",
      icon: TrendingUp,
      color: "text-amber-400",
    },
    {
      label: t("mortalityRate"),
      value: stats.mortalityRate !== null ? `${stats.mortalityRate.toFixed(1)}%` : "-",
      icon: Skull,
      color: "text-red-400",
    },
    {
      label: t("activeProjects"),
      value: String(stats.activeProjects),
      icon: FolderOpen,
      color: "text-green-400",
    },
  ];

  return (
    <div className="rounded-xl bg-gradient-to-r from-slate-900 to-slate-700 p-6 text-white">
      <div className="grid grid-cols-2 gap-6 md:grid-cols-4">
        {kpis.map((kpi) => (
          <div key={kpi.label} className="text-center">
            <div className="mb-2 flex items-center justify-center">
              <kpi.icon className={`h-5 w-5 ${kpi.color}`} />
            </div>
            <p className="text-sm text-slate-300">{kpi.label}</p>
            <p className={`text-3xl font-bold ${kpi.color}`}>{kpi.value}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/dashboard-kpi-banner.tsx
git commit -m "feat(dashboard): add KPI banner component"
```

---

## Task 6: Frontend — Mortality & FCR Chart Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/mortality-fcr-chart.tsx`

- [ ] **Step 1: Create the chart component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/mortality-fcr-chart.tsx
"use client";

import { useTranslations } from "next-intl";
import { LineChart, Line, XAxis, YAxis, CartesianGrid } from "recharts";
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  type ChartConfig,
} from "@/components/ui/chart";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { DashboardTrends } from "@/types/api";

interface MortalityFcrChartProps {
  trends: DashboardTrends | null;
}

export function MortalityFcrChart({ trends }: MortalityFcrChartProps) {
  const t = useTranslations("dashboard");

  const chartConfig: ChartConfig = {
    value: { label: t("fcrValue"), color: "var(--color-chart-1)" },
    standard: { label: t("standardFcr"), color: "var(--color-chart-3)" },
    rate: { label: t("mortalityPercent"), color: "var(--color-chart-4)" },
  };

  // Merge FCR and mortality data by date
  const mergedData = trends?.fcrTrend.map((fcr) => {
    const mortality = trends.mortalityTrend.find((m) => m.date === fcr.date);
    return {
      date: fcr.date.slice(5), // MM-DD format
      value: Number(fcr.value.toFixed(3)),
      standard: Number(fcr.standard.toFixed(3)),
      rate: mortality?.rate ?? null,
    };
  }) ?? [];

  const hasData = mergedData.length > 0;

  return (
    <Card>
      <CardHeader className="pb-2">
        <CardTitle className="text-base font-medium">
          {t("mortalityFcrTrend")}
        </CardTitle>
        <p className="text-xs text-muted-foreground">{t("last30Days")}</p>
      </CardHeader>
      <CardContent>
        {hasData ? (
          <ChartContainer config={chartConfig} className="h-[280px] w-full">
            <LineChart data={mergedData}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" fontSize={12} />
              <YAxis yAxisId="fcr" fontSize={12} />
              <YAxis yAxisId="mortality" orientation="right" fontSize={12} />
              <ChartTooltip content={<ChartTooltipContent />} />
              <Line
                yAxisId="fcr"
                type="monotone"
                dataKey="value"
                stroke="var(--color-value)"
                strokeWidth={2}
                dot={false}
              />
              <Line
                yAxisId="fcr"
                type="monotone"
                dataKey="standard"
                stroke="var(--color-standard)"
                strokeWidth={1}
                strokeDasharray="5 5"
                dot={false}
              />
              <Line
                yAxisId="mortality"
                type="monotone"
                dataKey="rate"
                stroke="var(--color-rate)"
                strokeWidth={2}
                dot={false}
                connectNulls={false}
              />
            </LineChart>
          </ChartContainer>
        ) : (
          <div className="flex h-[280px] items-center justify-center text-sm text-muted-foreground">
            {t("noData")}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/mortality-fcr-chart.tsx
git commit -m "feat(dashboard): add mortality & FCR trend chart component"
```

---

## Task 7: Frontend — Project Phase Chart Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/project-phase-chart.tsx`

- [ ] **Step 1: Create the donut chart component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/project-phase-chart.tsx
"use client";

import { useTranslations } from "next-intl";
import { PieChart, Pie, Cell } from "recharts";
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  type ChartConfig,
} from "@/components/ui/chart";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { DashboardStats } from "@/types/api";

interface ProjectPhaseChartProps {
  stats: DashboardStats | null;
}

const PHASE_COLORS = {
  rearing: "var(--color-chart-1)",
  harvest: "var(--color-chart-2)",
  cleaning: "var(--color-chart-3)",
  preparation: "var(--color-chart-4)",
};

export function ProjectPhaseChart({ stats }: ProjectPhaseChartProps) {
  const t = useTranslations("dashboard");

  const chartConfig: ChartConfig = {
    rearing: { label: t("rearing"), color: PHASE_COLORS.rearing },
    harvest: { label: t("harvest"), color: PHASE_COLORS.harvest },
    cleaning: { label: t("cleaning"), color: PHASE_COLORS.cleaning },
    preparation: { label: t("preparation"), color: PHASE_COLORS.preparation },
  };

  const phases = stats?.projectsByPhase;
  const data = phases
    ? [
        { name: "rearing", value: phases.rearing, fill: PHASE_COLORS.rearing },
        { name: "harvest", value: phases.harvest, fill: PHASE_COLORS.harvest },
        { name: "cleaning", value: phases.cleaning, fill: PHASE_COLORS.cleaning },
        { name: "preparation", value: phases.preparation, fill: PHASE_COLORS.preparation },
      ].filter((d) => d.value > 0)
    : [];

  const hasData = data.length > 0;

  return (
    <Card>
      <CardHeader className="pb-2">
        <CardTitle className="text-base font-medium">
          {t("projectsByPhase")}
        </CardTitle>
      </CardHeader>
      <CardContent>
        {hasData ? (
          <ChartContainer config={chartConfig} className="h-[280px] w-full">
            <PieChart>
              <ChartTooltip content={<ChartTooltipContent />} />
              <Pie
                data={data}
                dataKey="value"
                nameKey="name"
                cx="50%"
                cy="50%"
                innerRadius={60}
                outerRadius={100}
                paddingAngle={2}
              >
                {data.map((entry) => (
                  <Cell key={entry.name} fill={entry.fill} />
                ))}
              </Pie>
              <ChartLegend content={<ChartLegendContent nameKey="name" />} />
            </PieChart>
          </ChartContainer>
        ) : (
          <div className="flex h-[280px] items-center justify-center text-sm text-muted-foreground">
            {t("noData")}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/project-phase-chart.tsx
git commit -m "feat(dashboard): add project phase donut chart component"
```

---

## Task 8: Frontend — Sales Trend Chart Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/sales-trend-chart.tsx`

- [ ] **Step 1: Create the bar chart component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/sales-trend-chart.tsx
"use client";

import { useTranslations } from "next-intl";
import { ComposedChart, Bar, Line, XAxis, YAxis, CartesianGrid } from "recharts";
import {
  ChartContainer,
  ChartTooltip,
  ChartTooltipContent,
  ChartLegend,
  ChartLegendContent,
  type ChartConfig,
} from "@/components/ui/chart";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { DashboardTrends } from "@/types/api";

interface SalesTrendChartProps {
  trends: DashboardTrends | null;
}

export function SalesTrendChart({ trends }: SalesTrendChartProps) {
  const t = useTranslations("dashboard");

  const chartConfig: ChartConfig = {
    revenue: { label: t("revenue"), color: "var(--color-chart-1)" },
    orderCount: { label: t("orders"), color: "var(--color-chart-2)" },
  };

  const data = trends?.salesTrend.map((s) => ({
    ...s,
    week: s.week.slice(5), // MM-DD format
  })) ?? [];

  const hasData = data.length > 0;

  return (
    <Card>
      <CardHeader className="pb-2">
        <CardTitle className="text-base font-medium">
          {t("salesTrend")}
        </CardTitle>
        <p className="text-xs text-muted-foreground">{t("last30Days")}</p>
      </CardHeader>
      <CardContent>
        {hasData ? (
          <ChartContainer config={chartConfig} className="h-[280px] w-full">
            <ComposedChart data={data}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="week" fontSize={12} />
              <YAxis yAxisId="revenue" fontSize={12} />
              <YAxis yAxisId="orders" orientation="right" fontSize={12} />
              <ChartTooltip content={<ChartTooltipContent />} />
              <ChartLegend content={<ChartLegendContent />} />
              <Bar yAxisId="revenue" dataKey="revenue" fill="var(--color-revenue)" radius={[4, 4, 0, 0]} />
              <Line yAxisId="orders" type="monotone" dataKey="orderCount" stroke="var(--color-orderCount)" strokeWidth={2} dot={false} />
            </ComposedChart>
          </ChartContainer>
        ) : (
          <div className="flex h-[280px] items-center justify-center text-sm text-muted-foreground">
            {t("noData")}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/sales-trend-chart.tsx
git commit -m "feat(dashboard): add sales trend bar chart component"
```

---

## Task 9: Frontend — Stock Alerts List Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/stock-alerts-list.tsx`

- [ ] **Step 1: Create the stock alerts component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/stock-alerts-list.tsx
"use client";

import { useTranslations } from "next-intl";
import { AlertTriangle } from "lucide-react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { usePaginated } from "@/hooks/use-api";
import { InventoryStock } from "@/types/api";
import { Skeleton } from "@/components/ui/skeleton";

export function StockAlertsList() {
  const t = useTranslations("dashboard");

  const { data: lowStocks, isLoading: lowLoading } = usePaginated<InventoryStock>(
    "/inventory-stocks",
    { limit: 3, extra: { stockStatus: "LOW" } }
  );
  const { data: criticalStocks, isLoading: criticalLoading } = usePaginated<InventoryStock>(
    "/inventory-stocks",
    { limit: 3, extra: { stockStatus: "CRITICAL" } }
  );

  const isLoading = lowLoading || criticalLoading;
  const stocks = [...criticalStocks, ...lowStocks].slice(0, 5);

  return (
    <Card>
      <CardHeader className="pb-2">
        <CardTitle className="flex items-center gap-2 text-base font-medium">
          <AlertTriangle className="h-4 w-4 text-orange-500" />
          {t("stockAlerts")}
        </CardTitle>
      </CardHeader>
      <CardContent>
        {isLoading ? (
          <div className="space-y-3">
            {Array.from({ length: 3 }).map((_, i) => (
              <Skeleton key={i} className="h-10 w-full" />
            ))}
          </div>
        ) : stocks.length === 0 ? (
          <div className="flex h-[240px] items-center justify-center text-sm text-muted-foreground">
            {t("noData")}
          </div>
        ) : (
          <div className="space-y-3">
            {stocks.map((stock) => (
              <div
                key={stock.id}
                className="flex items-center justify-between rounded-lg border p-3"
              >
                <div className="min-w-0 flex-1">
                  <p className="truncate text-sm font-medium">
                    {stock.product?.name ?? stock.productId}
                  </p>
                  <p className="text-xs text-muted-foreground">
                    {stock.warehouse?.name ?? stock.warehouseId}
                  </p>
                </div>
                <div className="flex items-center gap-2">
                  <span className="text-sm font-mono">
                    {Number(stock.quantityOnHand).toLocaleString()}
                  </span>
                  <Badge
                    variant={stock.stockStatus === "CRITICAL" ? "destructive" : "secondary"}
                  >
                    {stock.stockStatus === "CRITICAL" ? t("critical") : t("low")}
                  </Badge>
                </div>
              </div>
            ))}
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/stock-alerts-list.tsx
git commit -m "feat(dashboard): add stock alerts list component"
```

---

## Task 10: Frontend — Operational Cards Component

**Files:**
- Create: `breeding-dashboard/src/app/(dashboard)/components/operational-cards.tsx`

- [ ] **Step 1: Create the operational cards component**

```typescript
// breeding-dashboard/src/app/(dashboard)/components/operational-cards.tsx
"use client";

import { useTranslations } from "next-intl";
import { ShoppingCart, Package, FileWarning } from "lucide-react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { DashboardStats } from "@/types/api";

interface OperationalCardsProps {
  stats: DashboardStats;
}

export function OperationalCards({ stats }: OperationalCardsProps) {
  const t = useTranslations("dashboard");

  const cards = [
    {
      label: t("openPOs"),
      value: stats.openPurchaseOrders,
      subtitle: null as string | null,
      icon: ShoppingCart,
      color: "text-blue-500",
    },
    {
      label: t("pendingSales"),
      value: stats.pendingSalesOrders,
      subtitle: null as string | null,
      icon: Package,
      color: "text-green-500",
    },
    {
      label: t("invoiceAging"),
      value: stats.overdueInvoiceCount,
      subtitle: `${t("pendingAmount")}: Rp ${Number(stats.pendingInvoiceAmount).toLocaleString()}`,
      icon: FileWarning,
      color: stats.overdueInvoiceCount > 0 ? "text-red-500" : "text-muted-foreground",
    },
  ];

  return (
    <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
      {cards.map((card) => (
        <Card key={card.label}>
          <CardHeader className="flex flex-row items-center justify-between pb-2">
            <CardTitle className="text-sm font-medium text-muted-foreground">
              {card.label}
            </CardTitle>
            <card.icon className={`h-4 w-4 ${card.color}`} />
          </CardHeader>
          <CardContent>
            <p className="text-2xl font-bold">{card.value}</p>
            {card.subtitle && (
              <p className="text-xs text-muted-foreground mt-1">{card.subtitle}</p>
            )}
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/components/operational-cards.tsx
git commit -m "feat(dashboard): add operational stat cards component"
```

---

## Task 11: Frontend — Rewrite Dashboard Page

**Files:**
- Modify: `breeding-dashboard/src/app/(dashboard)/page.tsx`

- [ ] **Step 1: Rewrite the dashboard page with all new components**

Replace the entire contents of `breeding-dashboard/src/app/(dashboard)/page.tsx`:

```typescript
"use client";

import { PageHeader } from "@/components/shared/page-header";
import { CardSkeleton } from "@/components/shared/loading-skeleton";
import { useApi } from "@/hooks/use-api";
import { DashboardStats, DashboardTrends } from "@/types/api";
import { DashboardKpiBanner } from "./components/dashboard-kpi-banner";
import { MortalityFcrChart } from "./components/mortality-fcr-chart";
import { ProjectPhaseChart } from "./components/project-phase-chart";
import { SalesTrendChart } from "./components/sales-trend-chart";
import { StockAlertsList } from "./components/stock-alerts-list";
import { OperationalCards } from "./components/operational-cards";
import { useTranslations } from "next-intl";

export default function DashboardPage() {
  const t = useTranslations("dashboard");
  const { data: stats, isLoading: statsLoading } = useApi<DashboardStats>("/dashboard/stats");
  const { data: trends, isLoading: trendsLoading } = useApi<DashboardTrends>("/dashboard/trends?days=30");

  const isLoading = statsLoading || trendsLoading;

  return (
    <div className="space-y-6">
      <PageHeader
        title={t("title")}
        description={t("description")}
      />

      {isLoading ? (
        <div className="space-y-6">
          <CardSkeleton />
          <div className="grid grid-cols-1 gap-4 md:grid-cols-2">
            <CardSkeleton />
            <CardSkeleton />
          </div>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-2">
            <CardSkeleton />
            <CardSkeleton />
          </div>
          <div className="grid grid-cols-1 gap-4 md:grid-cols-3">
            <CardSkeleton />
            <CardSkeleton />
            <CardSkeleton />
          </div>
        </div>
      ) : (
        <>
          {/* KPI Banner */}
          {stats && <DashboardKpiBanner stats={stats} />}

          {/* Charts Row */}
          <div className="grid grid-cols-1 gap-4 md:grid-cols-2">
            <MortalityFcrChart trends={trends} />
            <ProjectPhaseChart stats={stats} />
          </div>

          {/* Sales + Stock Alerts Row */}
          <div className="grid grid-cols-1 gap-4 md:grid-cols-2">
            <SalesTrendChart trends={trends} />
            <StockAlertsList />
          </div>

          {/* Operational Cards */}
          {stats && <OperationalCards stats={stats} />}
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
git add breeding-dashboard/src/app/(dashboard)/page.tsx
git commit -m "feat(dashboard): rewrite dashboard page with analytics layout"
```

---

## Task 12: Build Verification

- [ ] **Step 1: Build backend**

Run: `cd breeding-app && npm run build`
Expected: Build succeeds with no errors.

- [ ] **Step 2: Build frontend**

Run: `cd breeding-dashboard && npm run build`
Expected: Build succeeds with no errors.

- [ ] **Step 3: Fix any build errors**

If either build fails, fix the errors and re-run.

- [ ] **Step 4: Final commit (if fixes were needed)**

```bash
git add -A
git commit -m "fix(dashboard): resolve build issues"
```
