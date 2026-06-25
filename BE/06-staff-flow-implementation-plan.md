# Staff Flow Implementation Plan — Backend

Created: 2026-06-25
Last updated: 2026-06-25

## TL;DR

Implement 4 new API modules for Staff (data sources, crawl jobs, crawl logs, data quality) and 1 new Mongoose model (`DimDataSource`). Staff monitors and configures the data pipeline — but **does NOT trigger crawls**. Crawl execution stays with cron, same as today.

---

## Architecture: Database-as-Bridge (read-only for monitoring)

```
┌─────────────────┐     writes CrawlLog,      ┌──────────────┐
│  Python Crawler  │     CrawlLogDetail,       │              │
│  (run.py→app.py) │     FactCrawlQuality      │   MongoDB    │
│  (cron / manual) │ ─────────────────────────▶│  (shared)    │
└─────────────────┘                            └──────┬───────┘
                                                       ▲ reads
┌─────────────────┐         reads from                │
│  Node.js API    │ ◀─────────────────────────────────┘
│  (Express)      │   reads crawl logs, quality stats
└─────────────────┘
```

**Key principle**: The API does NOT call the crawler. The crawler does NOT call the API. They share MongoDB. The API **reads** crawl results that the crawler already wrote. Staff configures but cron executes.

**What we're cutting**: `run-now`, `retry`, and all crawler bridge work. These don't make sense unless the Python crawler is modified to pick up PENDING jobs — a separate project.

---

## What Staff Can Actually Do

| Capability | How it works | Data source |
|---|---|---|
| **Manage data sources** (CRUD) | Staff creates/edits sources in dimDataSources | MongoDB collection — written by API |
| **Configure crawl jobs** (CRUD) | Staff creates/edits CrawlJob configs | MongoDB collection — written by API |
| **View crawl logs** (read-only) | Crawler writes → API reads | `crawlLogs` + `crawlLogDetails` — written by Python |
| **View data quality** (read-only) | Crawler writes → API reads | `factCrawlQualities` — written by Python |
| **Monitor dashboard** (read-only) | Aggregates from all above | Mixed |

---

## Phase 1: Foundation — `DimDataSource` Model & Data Sources CRUD

**Goal**: Create the Mongoose model for `dimDataSources` (currently only created by Python crawler via pymongo) and implement full CRUD for Staff/Admin.

**Important**: `dimDataSources` is a **registry** — not a driver. Adding "cafef" doesn't create a crawler; it just documents that the system acknowledges CafeF as a source. Each source needs a Python script built separately.

### Steps

1. **Create `api/src/database/models/dim-data-source.model.js`**
   - Schema fields: `name` (unique, required, e.g. "vietstock"), `provider_type` (e.g. "crawler"), `base_url`, `description`, `status` (active/inactive), `config` (Mixed), timestamps
   - Collection: `dimDataSources`
   - Follow conventions from `dim-stock-data-source.model.js`

2. **Implement `modules/data-sources/`** — all 5 files:
   - `data-sources.validation.js` — validation rules for create/update/list
   - `data-sources.service.js` — CRUD: list (paginated, filterable), detail, create, update, toggle
   - `data-sources.controller.js` — standard `formatDataSource()` + async handlers
   - `data-sources.routes.js` — `/api/staff/data-sources`, gate `['ADMIN', 'STAFF']`
   - 5 routes: GET /, GET /:id, POST /, PUT /:id, PATCH /:id/toggle-status

3. **Register in `app.js`**: `app.use('/api/staff/data-sources', dataSourcesRouter);`

4. **Add Swagger `@openapi` annotations**

5. **Seed `seed-data-sources.js`** for default (vietstock)

---

## Phase 2: Crawl Jobs CRUD (Config Only — No Execution)

**Goal**: Staff configures crawl jobs (what to crawl, from where, how often). Cron runs them.

**No `run-now` endpoint**. The `crawl-jobs.scheduler.js` file stays as an empty placeholder for a future scheduler service.

### Steps

1. **Update `CrawlJob` model** (`crawl-job.model.js`):
   - Change `data_source_id` ref from `'DimStockDataSource'` → `'DimDataSource'`
   - Add `'MARKET_OVERVIEW'` to `data_type` enum

2. **Implement `modules/crawl-jobs/`** — 6 files:
   - `crawl-jobs.validation.js` — create/update rules
   - `crawl-jobs.service.js`:
     - `list()` — paginated, filter by status/data_type/data_source_id/market_id, populate refs
     - `create()` — validate source exists, set `next_run_at` from cron_expression
     - `update()` — same as create
     - `toggleStatus()` — flip active/inactive, update `next_run_at`
     - `getDetail()` — single job with populated refs + last 5 logs
   - `crawl-jobs.controller.js` — standard pattern
   - `crawl-jobs.routes.js` — `/api/staff/crawl-jobs`, gate `['ADMIN', 'STAFF']`
   - `crawl-jobs.scheduler.js` — empty placeholder

3. **Register in `app.js`**: `app.use('/api/staff/crawl-jobs', crawlJobsRouter);`

4. **Add Swagger `@openapi` annotations**

---

## Phase 3: Crawl Logs (Read-only — Data Already in DB)

**Goal**: Surface the `crawlLogs` + `crawlLogDetails` data that the Python crawler already writes. **No API writes to crawl logs** in this phase.

**Why this works**: The Python crawler's `mongodb_service.py` already does:
```python
log_id = db_service.create_crawl_log()           # → crawlLogs { status: "PENDING", ... }
db_service.write_crawl_log_detail(log_id, ...)    # → crawlLogDetails { symbol, status, message }
db_service.update_crawl_log(log_id, ended_at, ...)  # → crawlLogs update
```

All the data is there. Staff just needs REST access.

### Steps

1. **Implement `modules/crawl-logs/`** — 4 files:
   - `crawl-logs.service.js`:
     - `list()` — paginated, filter by crawl_job_id, status, date range, keyword; populate crawl_job_id
     - `getDetail()` — single CrawlLog + ALL its CrawlLogDetails (populated with stock_id)
     - `getDetailBySymbol()` — filter details by symbol within a log
     - `getFailedSymbols()` — aggregate all FAILED details across logs
   - `crawl-logs.controller.js`
   - `crawl-logs.routes.js` — `/api/staff/crawl-logs`, gate `['ADMIN', 'STAFF']`

2. **Register in `app.js`**: `app.use('/api/staff/crawl-logs', crawlLogsRouter);`

3. **Add Swagger `@openapi` annotations**

---

## Phase 4: Data Quality (Read-only — Data Already in DB)

**Goal**: Surface `factCrawlQualities` data that the crawler writes. Staff can see success rates, worst sources, missing records.

### Steps

1. **Create `modules/data-quality/`** — 4 files:
   - `data-quality.service.js`:
     - `getQualityDashboard()` — aggregate from FactCrawlQuality: success_rate, records stats, worst source, worst job, trends
     - `getQualityBySource()` — grouped by data_source_id
     - `getQualityByJob()` — grouped by crawl_job_id
     - `getMissingRecords()` — stocks with no data in a date range
   - `data-quality.controller.js`
   - `data-quality.routes.js` — `/api/staff/data-quality`
   - `data-quality.validation.js`

2. **Enhance `GET /api/dashboard/staff`** (existing `dashboard.repository.js` + `dashboard.service.js`):
   - Add `etl_status` — latest run status per data_type
   - Add `database_status` — collection counts
   - Add `data_quality` summary

3. **Register in `app.js`**: `app.use('/api/staff/data-quality', dataQualityRouter);`

4. **Add Swagger `@openapi` annotations**

---

## Route Map Summary

| Phase | Method | Endpoint | Auth | Description |
|-------|--------|----------|------|-------------|
| 1 | GET | `/api/staff/data-sources` | ADMIN + STAFF | List (paginated, filterable) |
| 1 | GET | `/api/staff/data-sources/:id` | ADMIN + STAFF | Detail |
| 1 | POST | `/api/staff/data-sources` | ADMIN + STAFF | Create |
| 1 | PUT | `/api/staff/data-sources/:id` | ADMIN + STAFF | Update |
| 1 | PATCH | `/api/staff/data-sources/:id/toggle-status` | ADMIN + STAFF | Enable/disable |
| 2 | GET | `/api/staff/crawl-jobs` | ADMIN + STAFF | List (paginated, filterable) |
| 2 | GET | `/api/staff/crawl-jobs/:id` | ADMIN + STAFF | Detail + recent logs |
| 2 | POST | `/api/staff/crawl-jobs` | ADMIN + STAFF | Create |
| 2 | PUT | `/api/staff/crawl-jobs/:id` | ADMIN + STAFF | Update |
| 2 | PATCH | `/api/staff/crawl-jobs/:id/toggle-status` | ADMIN + STAFF | Enable/disable |
| 3 | GET | `/api/staff/crawl-logs` | ADMIN + STAFF | List (filterable) |
| 3 | GET | `/api/staff/crawl-logs/:id` | ADMIN + STAFF | Detail + all child records |
| 3 | GET | `/api/staff/crawl-logs/:id/failed-symbols` | ADMIN + STAFF | Failed symbols in a log |
| 4 | GET | `/api/staff/data-quality` | ADMIN + STAFF | Quality dashboard |
| 4 | GET | `/api/staff/data-quality/by-source` | ADMIN + STAFF | Quality by source |
| 4 | GET | `/api/staff/data-quality/by-job` | ADMIN + STAFF | Quality by job |
| 4 | GET | `/api/staff/data-quality/missing` | ADMIN + STAFF | Missing records analysis |
| 4 | GET | `/api/dashboard/staff` | ADMIN + STAFF | Enhanced (existing) |

---

## Relevant Files

### New files to create
- `api/src/database/models/dim-data-source.model.js`
- `api/src/modules/data-sources/` — 5 files (populate existing empties)
- `api/src/modules/crawl-jobs/` — 6 files (populate existing empties)
- `api/src/modules/crawl-logs/` — 4 files (populate existing empties)
- `api/src/modules/data-quality/` — 4 files (new module)
- `api/src/database/seeds/seed-data-sources.js`

### Existing files to modify
- `api/src/database/models/crawl-job.model.js` — update `data_source_id` ref + add `MARKET_OVERVIEW`
- `api/src/app.js` — register 4 new route modules
- `api/src/modules/dashboard/dashboard.repository.js` — add health/quality queries
- `api/src/modules/dashboard/dashboard.service.js` — include quality in staff dashboard

### Reference patterns
- `modules/staff-subscriptions/*` — staff routes, validation, controller, service
- `modules/admin-subscriptions/*` — full CRUD + Swagger annotations
- `modules/users/*` — pagination, error handling, `formatEntity()`

---

## Decisions & Scope

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Run-now / Retry** | ❌ Cut | Would require Python crawler changes to pick up PENDING logs. Not useful as just a database row. Cron handles execution. |
| **Crawler bridge** | ❌ Cut | Shared MongoDB is enough for monitoring. No HTTP bridge needed. |
| **Data Sources CRUD** | ✅ Include | Useful registry. Doesn't affect crawler operation. |
| **Crawl Jobs CRUD** | ✅ Include | Config management is useful. No execution involved. |
| **Crawl Logs** | ✅ Include | Data already exists in `crawlLogs` + `crawlLogDetails`. Pure read. |
| **Data Quality** | ✅ Include | Data already exists in `factCrawlQualities`. Pure read. |
| **Scheduler** | ❌ Cut | `crawl-jobs.scheduler.js` stays as empty placeholder. Cron runs crawler. |
| **CrawlJob ref** | Change to `DimDataSource` | Correct ref target. `DimStockDataSource` stays for per-stock URLs. |
| **Repository layer** | Skip for simple CRUD | Follow `staff-subscriptions` pattern. Use `.lean()` in service. |

---

## Verification

1. **Phase 1**: `POST /api/staff/data-sources` creates a doc; `GET /api/staff/data-sources` paginated list works; verify `dimDataSources` collection has it
2. **Phase 2**: Create a crawl job → toggle on → verify in `crawlJobs` collection; GET detail shows populated refs
3. **Phase 3**: If crawler has run before, GET logs shows data; filter by status/date works; detail returns CrawlLogDetails; failed symbols endpoint aggregates
4. **Phase 4**: Quality dashboard returns aggregates from `factCrawlQualities`; staff dashboard includes new fields
5. **Global**: All endpoints return 401 without token, 403 for wrong role, 200 for STAFF/ADMIN
