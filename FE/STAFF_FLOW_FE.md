# Staff Flow — FE Implementation Guide

Created: 2026-06-25
Last updated: 2026-06-25

**Prerequisite docs**: `WEB_DESIGN.md`, `WEB_DEV_RULES.md`, `BE/06-staff-flow-implementation-plan.md`

---

## Overview

The Staff role monitors and manages the data pipeline. Staff does **not** trade stocks or trigger crawls — they ensure data is collected, stored, and displayed correctly. Crawl execution is handled by system cron, not the UI.

### What Staff sees

| FE Page | BE Endpoints | Data Source | Priority |
|---------|-------------|-------------|----------|
| **Staff Dashboard** (already built) | `GET /api/dashboard/staff` | Enhanced with ETL + quality stats | 🟢 Already done |
| **Crawl Logs** (placeholder) | Logs endpoints | `crawlLogs` + `crawlLogDetails` | 🟢 Build first |
| **Data Sources** (placeholder) | Data sources endpoints | `dimDataSources` | 🟢 Build second |
| **ETL Monitor** (placeholder) | Data quality endpoints | `factCrawlQualities` | 🟡 Build third |
| **Data Validation** (placeholder) | Data quality /missing | `factCrawlQualities` | 🟡 Build third |

**Pages to skip (for now)**: Crawl Jobs (decorative), Import History, Stock Data Monitor.

---

## Existing FE Infrastructure

### Shell & Navigation

The Staff shell is fully built in `src/components/staff-shell/index.tsx`. Nav items are defined in `src/layouts/StaffLayout.tsx`:

```typescript
const STAFF_NAV_ITEMS: StaffNavItem[] = [
  { label: "Staff Dashboard",  to: "/staff/dashboard",      icon: LayoutDashboard },
  { label: "Data Sources",     to: "/staff/data-sources",     icon: Database },
  { label: "Crawl Jobs",      to: "/staff/crawl-jobs",      icon: Activity },
  { label: "Crawl Logs",      to: "/staff/crawl-logs",      icon: Logs },
  { label: "ETL Monitor",     to: "/staff/etl-monitor",     icon: FileStack },
  { label: "Data Validation", to: "/staff/data-validation", icon: BadgeCheck },
  { label: "Import History",  to: "/staff/import-history",  icon: HardDriveDownload },
  { label: "Stock Data Monitor", to: "/staff/stock-data-monitor", icon: Archive },
  { label: "Subscriptions",   to: "/staff/subscriptions",   icon: CreditCard },
]
```

Status pills (in topbar) show: **Crawler**, **ETL**, **DB**, **API** — each with a `healthy` dot.

### Service Layer Pattern

All API calls go through `src/services/auth.service.ts` → `authenticatedRequest()`. Pattern:

```typescript
import { authenticatedRequest } from "@/services/auth.service"

export async function getMyData(params?: MyParams): Promise<MyData> {
  const response = await authenticatedRequest<ApiResponse<MyData>>({
    url: "/api/staff/my-endpoint",
    method: "GET",
    params: cleanParams,
  })
  const payload = response.data
  if (response.status < 200 || response.status >= 300 || payload?.success === false || !payload?.data) {
    throw new Error(payload?.message || "Unable to load data")
  }
  return payload.data
}
```

Each page uses `useState` + `useEffect` for data fetching (no React Query). States to handle:

- `isLoading` → `<TableLoading>` or skeleton
- `error` → `<TableError message={error} onRetry={loadData} />`
- Empty (no filters) → `<TableEmpty />`
- Empty (with filters) → `<TableNoMatch />`

### Existing empty service file

An empty `src/services/crawl.service.ts` already exists — this is where all crawl/log/quality service functions should be added.

---

## Page 1: Staff Dashboard (Already Implemented)

Located at `src/pages/StaffDashboardPage/index.tsx`. Already calls `getStaffDashboard()` from `dashboard.service.ts`.

**What changed**: The BE now returns extra fields in `GET /api/dashboard/staff`:

```typescript
// Response now includes:
data_quality: {
  avg_success_rate_percent: number
  total_records_failed: number
  total_failed_symbols: number
}
etl_status: Array<{
  data_type: string      // "DAILY_MARKET_PRICE" | "QUARTERLY_FINANCIAL_STATEMENT" | ...
  latest_status: string  // "SUCCESS" | "FAILED" | "PARTIAL_SUCCESS" | "PENDING"
  latest_run: string     // ISO date
}>
database_status: {
  collections: number
  total_documents: number
}
```

**What to update on the dashboard page** (minimal):
- Use `data_quality` to populate status pills in the topbar
- Show `etl_status` as a "Latest Crawl Status" mini-table or card
- Show `database_status` as a summary stat

---

## Page 2: Crawl Logs — Full Build

**File**: `src/pages/CrawlLogsPage/index.tsx` (currently `<div>--</div>`)
**Services**: `src/services/crawl.service.ts` (currently empty)

### API Endpoints

#### `GET /api/staff/crawl-logs`

List all crawl runs with filtering and pagination.

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | Page number (default: 1) |
| `limit` | int | Items per page (default: 20) |
| `crawl_job_id` | string | Filter by crawl job |
| `status` | string | `PENDING`, `SUCCESS`, `FAILED`, `PARTIAL_SUCCESS` |
| `date_from` | string (ISO date) | Filter start date |
| `date_to` | string (ISO date) | Filter end date |
| `sort_by` | string | `started_at` (default) |
| `sort_order` | string | `asc` or `desc` |

**Response**:

```typescript
{
  items: Array<{
    id: string
    crawl_job: { id: string; job_name: string; data_type: string } | null
    started_at: string
    ended_at: string | null
    status: "PENDING" | "SUCCESS" | "FAILED" | "PARTIAL_SUCCESS"
    records_fetched: number
    records_inserted: number
    records_updated: number
    records_failed: number
    error_message: string
    created_at: string
  }>
  pagination: { page: number; limit: number; total_items: number; total_pages: number }
}
```

#### `GET /api/staff/crawl-logs/:id`

Full detail of a single crawl run + all per-symbol detail records.

**Response**:

```typescript
{
  crawl_log: CrawlLogItem  // same shape as above
  details: Array<{
    id: string
    stock: { id: string; symbol: string; company_name: string } | null
    symbol: string
    data_type: string
    status: "SUCCESS" | "FAILED" | "SKIPPED"
    message: string
    created_at: string
  }>
}
```

#### `GET /api/staff/crawl-logs/:id/failed-symbols`

All FAILED records within a specific log. Same detail shape as above.

#### `GET /api/staff/crawl-logs/failed-symbols`

All FAILED records across all logs (optionally `?crawl_job_id=xxx` and `?limit=50`).

### Page Layout

```
┌─────────────────────────────────────────────────────────┐
│  Crawl Logs                                              │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ [Status filter ▼] [Date from] [Date to] [🔍 Search]│ │
│  │ [Refresh ↻]                                         │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │ Log Table:                                          │ │
│  │ Started ↓ │ Job │ Type │ Status │ Records │ Error   │ │
│  │──────────│─────│──────│────────│─────────│─────────│ │
│  │ 25/06    │Daily│PRICE │ ✅     │ 400/0/2 │--       │ │
│  │ 24/06    │Daily│PRICE │ ⚠️     │ 380/5/20│Timeout  │ │
│  │ 23/06    │--   │--    │ ❌     │ 0/0/0   │DB down  │ │
│  │ ...                                                 │ │
│  └─────────────────────────────────────────────────────┘ │
│  [Pagination: < 1 2 3 ... >]                            │
│                                                          │
│  (Click a row → navigate to detail view)                 │
└─────────────────────────────────────────────────────────┘
```

### Detail View (modal or separate page)

```
┌─ Crawl Log Detail ─────────────────────────────────────┐
│  Status: PARTIAL_SUCCESS                                │
│  Started: 2026-06-25 08:00  Ended: 2026-06-25 08:12    │
│  Fetched: 400  Inserted: 380  Updated: 5  Failed: 20   │
│  Error: Timeout on 3 symbols                            │
│                                                          │
│  ┌─ Per-Symbol Results ───────────────────────────────┐ │
│  │ Symbol │ Status  │ Message                          │ │
│  │────────│─────────│──────────────────────────────────│ │
│  │ FPT    │ ✅      │ OK                               │ │
│  │ HPG    │ ✅      │ OK                               │ │
│  │ VIC    │ ❌      │ Page load timeout                │ │
│  │ VNM    │ ❌      │ Data parse error                 │ │
│  │ ...                                                 │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  [Failed Symbols: 20]  [Export CSV]                     │
└──────────────────────────────────────────────────────────┘
```

### States

| State | What to show |
|-------|-------------|
| Loading | Table skeleton (rows shimmer) |
| Error | `<TableError>` with retry button |
| Empty (no filters, no data) | "No crawl logs yet. Crawler has not run." |
| Empty (with filters) | "No logs match your filters" |
| Success | Table with data |

### Color coding for `status`

| Status | Color | Icon |
|--------|-------|------|
| `SUCCESS` | `var(--status-positive)` green | ✅ |
| `PARTIAL_SUCCESS` | `var(--status-warning)` yellow | ⚠️ |
| `FAILED` | `var(--status-negative)` red | ❌ |
| `PENDING` | `var(--status-info)` blue | ⏳ |

---

## Page 3: Data Sources — Full Build

**File**: `src/pages/DataSourcesPage/index.tsx` (currently `<div>--</div>`)
**Services**: Add to `src/services/crawl.service.ts`

### API Endpoints

#### `GET /api/staff/data-sources`

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | Default 1 |
| `limit` | int | Default 20 |
| `status` | string | `active` or `inactive` |
| `provider_type` | string | `crawler`, `api`, `file_import` |
| `keyword` | string | Search name, description, base_url |
| `sort_by` | string | `name` (default), `provider_type`, `status`, `created_at` |

**Response**:

```typescript
{
  items: Array<{
    id: string
    name: string                    // "vietstock"
    provider_type: string            // "crawler"
    base_url: string                // "https://finance.vietstock.vn"
    description: string
    config: Record<string, any>
    status: "active" | "inactive"
    created_at: string
    updated_at: string
  }>
  pagination: { page, limit, total_items, total_pages }
}
```

#### `POST /api/staff/data-sources`

Create: `{ name: required, provider_type?, base_url?, description?, status? }`

#### `PUT /api/staff/data-sources/:id`

Update: `{ name?, provider_type?, base_url?, description?, status? }`

#### `PATCH /api/staff/data-sources/:id/toggle-status`

Toggle active/inactive.

### Page Layout

```
┌─ Data Sources ──────────────────────────────────────────┐
│  [+ Add New]  [🔍 Search]  [Status filter ▼]            │
│                                                          │
│  ┌─ Source Table ─────────────────────────────────────┐ │
│  │ Name       │ Type    │ Status │ URL                │ │
│  │───────────│─────────│────────│────────────────────│ │
│  │ vietstock  │ crawler │ 🟢     │ finance.vietstock… │ │
│  │ ...                                                  │ │
│  └──────────────────────────────────────────────────────┘ │
│  [< 1 2 >]                                               │
│                                                           │
│  ┌─ Add/Edit Modal ───────────────────────────────────┐ │
│  │ Name: [________________] (required)                 │ │
│  │ Type: [crawler ▼]                                   │ │
│  │ URL:  [________________]                            │ │
│  │ Desc: [________________]                            │ │
│  │ Status: [Active ✓]                                  │ │
│  │ [Cancel] [Save]                                     │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

### States

| State | What to show |
|-------|-------------|
| Loading | Table skeleton |
| Error | `<TableError>` with retry |
| Empty | "No data sources configured" with "Add First Source" CTA |
| Create mode | Modal form with validation |
| Edit mode | Modal form pre-filled |
| Toggle | Confirm dialog before toggle |

### Important note for FE

`dimDataSources` is a **registry**. Adding "cafef" here does NOT create a working crawler — it just documents the source. Each source type needs a separate Python crawler script. The value is: toggle broken sources inactive, store config, single source of truth.

---

## Page 4: ETL Monitor — Full Build

**File**: `src/pages/EtlMonitorPage/index.tsx` (currently `<div>--</div>`)
**Services**: Add to `src/services/crawl.service.ts`

### API Endpoints

#### `GET /api/staff/data-quality`

Quality dashboard with aggregates.

**Response**:

```typescript
{
  overall: {
    total_runs: number
    records_fetched: number
    records_inserted: number
    records_updated: number
    records_failed: number
    avg_success_rate_percent: number
  }
  worst_source: {
    source_id: string
    source_name: string
    avg_success_rate: number
    total_failed: number
    runs: number
  } | null
  total_failed_symbols: number
  recent_trends: Array<{
    date: string          // "2026-06-25"
    avg_success_rate: number
    total_failed: number
    runs: number
  }>
}
```

#### `GET /api/staff/data-quality/by-source`

Quality grouped by data source. 

**Response**: `{ items: Array<{ source_id, source_name, source_type, avg_success_rate_percent, records_fetched, records_inserted, records_updated, records_failed, runs }> }`

#### `GET /api/staff/data-quality/by-job`

Quality grouped by crawl job. Same shape but with `job_name`, `data_type` instead of source.

#### `GET /api/staff/data-quality/missing`

Missing records analysis for a date range.

| Param | Type | Description |
|-------|------|-------------|
| `date_from` | ISO date | Default: 7 days ago |
| `date_to` | ISO date | Default: today |

**Response**:

```typescript
{
  date_range: { from: string; to: string }
  total_active_stocks: number
  records_inserted: number
  records_updated: number
  records_failed: number
  failed_symbols_in_range: number
}
```

### Page Layout

```
┌─ ETL Monitor ───────────────────────────────────────────┐
│  ┌─ KPI Cards Row ────────────────────────────────────┐ │
│  │ Total Runs  │ Success Rate │ Failed Records │ ↻    │ │
│  │    42       │   96.15%     │     23         │      │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─ Success Rate Trend (7 days) ──────────────────────┐ │
│  │ [Bar chart: daily success rate]                     │ │
│  │                                                     │ │
│  │ 100% ┤  ██  ██  ██  ███ ██  ██  ██                │ │
│  │  95% ┤  ██  ██  ██  ██  ██  ██  ██                │ │
│  │  90% ┤  ██  ██  ██  ██  ██  ██  ██                │ │
│  │      └──19──20──21──22──23──24──25──               │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─ By Source ───────────────────────┬─ By Job ────────┐ │
│  │ Source │ Rate │ Failed │ Runs    │ Job │ Rate│Failed│ │
│  │────────│──────│────────│────────│─────│─────│──────│ │
│  │ vietst. │ 96%  │ 20    │ 42     │ Dly │ 96% │ 20   │ │
│  └──────────────────────────────────┴──────────────────┘ │
│                                                           │
│  ┌─ Missing Records Analysis ───────────────────────────┐ │
│  │ Date Range: [25/06] to [25/06]  [Analyze]           │ │
│  │                                                      │ │
│  │ Active Stocks: 400  │  Missing Data: --             │ │
│  │ Failed Symbols: 3   │  Records: 380/400             │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## Page 5: Data Validation — Light Build

**File**: `src/pages/DataValidationPage/index.tsx` (currently `<div>--</div>`)

This page is a simpler view into the same `data-quality/missing` endpoint. It shows:

- Date range picker
- Summary: "X of Y stocks have data"
- Table of stocks with missing/failed data (if any)
- Option to view failed symbols from `GET /api/staff/crawl-logs/failed-symbols`

Can reuse the same service function as ETL Monitor's missing records.

---

## Service File: `src/services/crawl.service.ts`

This file is **empty** and ready to fill. Add these functions:

| Function | Method | Endpoint |
|----------|--------|----------|
| `getCrawlLogs(params)` | `GET` | `/api/staff/crawl-logs` |
| `getCrawlLogDetail(id)` | `GET` | `/api/staff/crawl-logs/:id` |
| `getCrawlLogFailedSymbols(id)` | `GET` | `/api/staff/crawl-logs/:id/failed-symbols` |
| `getAllFailedSymbols(params)` | `GET` | `/api/staff/crawl-logs/failed-symbols` |
| `getDataSources(params)` | `GET` | `/api/staff/data-sources` |
| `createDataSource(data)` | `POST` | `/api/staff/data-sources` |
| `updateDataSource(id, data)` | `PUT` | `/api/staff/data-sources/:id` |
| `toggleDataSourceStatus(id)` | `PATCH` | `/api/staff/data-sources/:id/toggle-status` |
| `getQualityDashboard()` | `GET` | `/api/staff/data-quality` |
| `getQualityBySource()` | `GET` | `/api/staff/data-quality/by-source` |
| `getQualityByJob()` | `GET` | `/api/staff/data-quality/by-job` |
| `getMissingRecords(params)` | `GET` | `/api/staff/data-quality/missing` |

### Error handling pattern

All functions follow this template:

```typescript
import { authenticatedRequest } from "@/services/auth.service"

export async function getCrawlLogs(params?: CrawlLogParams): Promise<CrawlLogListData> {
  const response = await authenticatedRequest<ApiResponse<CrawlLogListData>>({
    url: "/api/staff/crawl-logs",
    method: "GET",
    params,
  })
  const payload = response.data
  if (response.status < 200 || response.status >= 300 || payload?.success === false || !payload?.data) {
    throw new Error(payload?.message || "Unable to load crawl logs")
  }
  return payload.data
}
```

---

## API Route Summary (BE already built)

| Method | Endpoint | FE Page | Priority |
|--------|----------|---------|----------|
| `GET` | `/api/dashboard/staff` | Dashboard | 🟢 Already integrated |
| `GET` | `/api/staff/crawl-logs` | Crawl Logs | 🟢 **Build** |
| `GET` | `/api/staff/crawl-logs/:id` | Crawl Log Detail | 🟢 **Build** |
| `GET` | `/api/staff/crawl-logs/:id/failed-symbols` | Crawl Log Detail | 🟢 **Build** |
| `GET` | `/api/staff/crawl-logs/failed-symbols` | Data Validation | 🟡 |
| `GET` | `/api/staff/data-sources` | Data Sources | 🟢 **Build** |
| `POST` | `/api/staff/data-sources` | Data Sources | 🟢 **Build** |
| `PUT` | `/api/staff/data-sources/:id` | Data Sources | 🟢 **Build** |
| `PATCH` | `/api/staff/data-sources/:id/toggle-status` | Data Sources | 🟢 **Build** |
| `GET` | `/api/staff/data-quality` | ETL Monitor | 🟡 |
| `GET` | `/api/staff/data-quality/by-source` | ETL Monitor | 🟡 |
| `GET` | `/api/staff/data-quality/by-job` | ETL Monitor | 🟡 |
| `GET` | `/api/staff/data-quality/missing` | Data Validation | 🟡 |

### Skipped (decorative, no BE impact)
- `/api/staff/crawl-jobs` (CRUD) — config table, no backend reads it
- `/api/staff/subscriptions` — already built

---

## Recommended Build Order

1. **Services** (`crawl.service.ts`) — all API functions, then build pages in order:
2. **Crawl Logs** page — most valuable, directly shows crawler output
3. **Crawl Log Detail** view — modal or sub-page
4. **Data Sources** page — CRUD with modal form
5. **ETL Monitor** page — quality dashboard with charts
6. **Data Validation** page — simple missing-data check
7. **Dashboard update** — add new ETL/quality fields to existing page

---

## Design Tokens to Use

From `WEB_DESIGN.md`:

```css
color: var(--status-positive)   /* #22C55E — SUCCESS */
color: var(--status-negative)   /* #EF4444 — FAILED */
color: var(--status-warning)    /* #FACC15 — PARTIAL_SUCCESS */
color: var(--status-info)       /* #38BDF8 — PENDING */
color: var(--text-secondary)    /* #94A3B8 — muted text */
background: var(--surface-card) /* #111827 — card bg */
border: var(--border-subtle)   /* #334155 — borders */
```

Component classes to reuse: `staff-shell__*` namespace, `StatusPill`, sidebar navigation pattern.

---

## Existing Files Reference

| File | Purpose |
|------|---------|
| `src/components/staff-shell/index.tsx` | StaffShell layout with sidebar, topbar, status pills |
| `src/layouts/StaffLayout.tsx` | Nav items definition, route layout |
| `src/routes/layoutRoutes.tsx` | All staff route → component mapping |
| `src/services/crawl.service.ts` | **Empty** — fill with crawl/log/quality functions |
| `src/services/dashboard.service.ts` | Already has `getStaffDashboard()` |
| `src/pages/StaffDashboardPage/index.tsx` | Already built, may need minor updates |
| `src/pages/CrawlLogsPage/index.tsx` | Placeholder — **build target** |
| `src/pages/DataSourcesPage/index.tsx` | Placeholder — **build target** |
| `src/pages/EtlMonitorPage/index.tsx` | Placeholder — **build target** |
| `src/pages/DataValidationPage/index.tsx` | Placeholder — **build target** |
| `src/pages/Staff/StaffSubscriptionManagement/StaffSubscriptionManagement.tsx` | Reference for full page pattern (search, table, pagination, modal) |