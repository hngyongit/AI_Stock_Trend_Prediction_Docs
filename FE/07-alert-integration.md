# Alert Feature — FE Integration Guide

## 1. Overview

This doc covers all FE work needed to integrate the Alert feature. It includes:

- **AlertsPage** (`/alerts`) — list, create, toggle, delete alerts
- **AlertCreateDialog** — modal form to create new alerts
- **StockDetailPage** — Alert Configuration card + Bell button + mock polling
- **SettingsPage** — plan limit display + email toggle

---

## 2. API Reference

Base URL: `http://localhost:5000` (Dev) / `https://api.aistockprediction.vn` (Prod)

### 2.1. GET /api/alerts — List alerts

| Detail | Value |
|--------|-------|
| Method | `GET` |
| URL | `/api/alerts` |
| Auth | Bearer Token |
| Purpose | Fetch all alerts for the logged-in user, with stock info + latest price |

**Request:**
```
Authorization: Bearer <access_token>
```

**Response 200:**
```json
{
  "success": true,
  "message": "Get alerts successfully",
  "data": [
    {
      "id": "665f1a2b9c1e2a0012a99991",
      "symbol": "FPT",
      "company_name": "Công ty Cổ phần FPT",
      "alert_type": "PRICE_ABOVE",
      "threshold": 140000,
      "status": "ACTIVE",
      "triggered_at": null,
      "triggered_value": null,
      "latest_price": {
        "close_price": 135000,
        "volume": 2500000
      },
      "created_at": "2026-07-01T10:00:00.000Z",
      "updated_at": "2026-07-01T10:00:00.000Z"
    }
  ]
}
```

### 2.2. POST /api/alerts — Create alert

| Detail | Value |
|--------|-------|
| Method | `POST` |
| URL | `/api/alerts` |
| Auth | Bearer Token |
| Purpose | Create a new price/volume alert for a watchlist stock |

**Request:**
```json
{
  "symbol": "FPT",
  "alert_type": "PRICE_ABOVE",
  "threshold": 140000
}
```

**Response 201:**
```json
{
  "success": true,
  "message": "Create alert successfully",
  "data": {
    "id": "665f1a2b9c1e2a0012a99991",
    "symbol": "FPT",
    "company_name": "Công ty Cổ phần FPT",
    "alert_type": "PRICE_ABOVE",
    "threshold": 140000,
    "status": "ACTIVE",
    "created_at": "2026-07-01T10:00:00.000Z"
  }
}
```

**Error 400 — plan limit:**
```json
{ "success": false, "message": "Alert stock limit exceeded (Maximum 2 stocks allowed)" }
```
```json
{ "success": false, "message": "Alert limit for this stock exceeded (Maximum 2 alerts per stock)" }
```

**Error 400 — not in watchlist:**
```json
{ "success": false, "message": "Stock must be in your watchlist to create an alert" }
```

**Error 404:**
```json
{ "success": false, "message": "Stock symbol not found" }
```

### 2.3. GET /api/alerts/:id — Alert detail

| Detail | Value |
|--------|-------|
| Method | `GET` |
| URL | `/api/alerts/:id` |
| Auth | Bearer Token |
| Purpose | Get single alert with price data |

**Response 200:** Same shape as a single item in the list response.

**Error 404:**
```json
{ "success": false, "message": "Alert not found" }
```

### 2.4. PUT /api/alerts/:id — Update alert

| Detail | Value |
|--------|-------|
| Method | `PUT` |
| URL | `/api/alerts/:id` |
| Auth | Bearer Token |
| Purpose | Change threshold or toggle status (ACTIVE↔DISABLED, TRIGGERED→ACTIVE) |

**Request:**
```json
{
  "threshold": 145000,
  "status": "DISABLED"
}
```
Both fields optional. Only provide what you want to change.

**Response 200:** Same shape as list item.

**Error 400 — invalid transition:**
```json
{ "success": false, "message": "Cannot transition from ACTIVE to TRIGGERED" }
```

Valid transitions:
| From | To |
|------|----|
| ACTIVE | DISABLED |
| DISABLED | ACTIVE |
| TRIGGERED | ACTIVE (resets triggered_at, triggered_value) |

### 2.5. DELETE /api/alerts/:id — Delete alert

| Detail | Value |
|--------|-------|
| Method | `DELETE` |
| URL | `/api/alerts/:id` |
| Auth | Bearer Token |
| Purpose | Permanently delete an alert |

**Response 200:**
```json
{ "success": true, "message": "Delete alert successfully" }
```

---

## 3. Frontend Service Layer

Create `src/services/alert.service.ts`:

```typescript
import { authenticatedRequest } from './auth.service';

export type AlertType = 'PRICE_ABOVE' | 'PRICE_BELOW' | 'VOLUME_SPIKE';
export type AlertStatus = 'ACTIVE' | 'TRIGGERED' | 'DISABLED';

export interface AlertItem {
  id: string;
  symbol: string;
  company_name: string;
  alert_type: AlertType;
  threshold: number;
  status: AlertStatus;
  triggered_at: string | null;
  triggered_value: number | null;
  latest_price: { close_price: number; volume: number } | null;
  created_at: string;
  updated_at: string;
}

export interface CreateAlertPayload {
  symbol: string;
  alert_type: AlertType;
  threshold: number;
}

/** Validate threshold is positive. For PRICE_ABOVE/BELOW, min 1000. For VOLUME_SPIKE, min 1.0. */
export function validateThreshold(alertType: AlertType, value: string): string | null {
  const num = parseFloat(value);
  if (isNaN(num) || num <= 0) return 'Threshold must be a positive number';
  if ((alertType === 'PRICE_ABOVE' || alertType === 'PRICE_BELOW') && num < 1000) return 'Price threshold min 1,000 VND';
  if (alertType === 'VOLUME_SPIKE' && num < 1) return 'Volume multiplier min 1.0x';
  return null;
}

export const getAlerts = (): Promise<AlertItem[]> =>
  authenticatedRequest({ method: 'GET', url: '/api/alerts' })
    .then(res => res.data?.data || []);

export const createAlert = (payload: CreateAlertPayload) =>
  authenticatedRequest({ method: 'POST', url: '/api/alerts', data: payload })
    .then(res => {
      if (res.status === 400) throw new Error(res.data?.message || 'Failed to create alert');
      return res.data?.data;
    });

export const getAlertById = (id: string) =>
  authenticatedRequest({ method: 'GET', url: `/api/alerts/${id}` })
    .then(res => res.data?.data);

export const updateAlert = (id: string, updates: { threshold?: number; status?: AlertStatus }) =>
  authenticatedRequest({ method: 'PUT', url: `/api/alerts/${id}`, data: updates })
    .then(res => res.data?.data);

export const deleteAlert = (id: string) =>
  authenticatedRequest({ method: 'DELETE', url: `/api/alerts/${id}` });

/** Toggle ACTIVE ↔ DISABLED. For TRIGGERED, pass 'ACTIVE' to reset. */
export const toggleAlert = (id: string, currentStatus: AlertStatus) => {
  const nextStatus = currentStatus === 'ACTIVE' ? 'DISABLED' : 'ACTIVE';
  return updateAlert(id, { status: nextStatus });
};
```

---

## 4. Screen Implementation Guide

### 4.1. AlertsPage (`/alerts`)

**Status:** Route exists in `layoutRoutes.tsx`. Page file exists at `src/pages/AlertsPage/index.tsx` — currently an empty div.

**Implementation steps:**

1. **Data fetching on mount:**
```typescript
const [alerts, setAlerts] = useState<AlertItem[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState<string | null>(null);

useEffect(() => {
  loadAlerts();
}, []);

const loadAlerts = async () => {
  setLoading(true); setError(null);
  try {
    const data = await getAlerts();
    setAlerts(data);
  } catch (e) {
    setError(e instanceof Error ? e.message : 'Failed to load alerts');
  } finally {
    setLoading(false);
  }
};
```

2. **Empty state** (when `alerts.length === 0 && !loading`):
```
┌─────────────────────────────────┐
│      🔔 No alerts yet           │
│  Create your first alert to     │
│  get notified about stock       │
│  price movements.               │
│        [+ Create Alert]         │
└─────────────────────────────────┘
```

3. **Table columns:**

| Column | Data | Format |
|--------|------|--------|
| Stock | symbol + company_name | `<a>` to `/stocks/:symbol` |
| Type | alert_type | Badge: PRICE_ABOVE=blue "↑", PRICE_BELOW=red "↓", VOLUME_SPIKE=orange "Volume" |
| Threshold | threshold | Price: `formatNumber(value)` VND. Volume: `${value}x` |
| Status | status + triggered badge | ACTIVE=green "Active", TRIGGERED=yellow "Triggered", DISABLED=gray "Disabled" |
| Last Trigger | triggered_at | `new Date(value).toLocaleDateString()` or "--" |
| Actions | toggle + delete | Button pair: power/bell-off toggle + trash delete |

4. **Delete flow:**
   - Show confirm dialog: "Are you sure you want to delete this alert?"
   - On confirm → `deleteAlert(id)` → `loadAlerts()`
   - On error → toast error

5. **Toggle flow:**
   - `toggleAlert(id, status)` → `loadAlerts()`
   - No optimistice update (let API response be source of truth)

6. **Loading state:** Show skeleton rows (5 placeholder rows with pulse animation)

7. **Error state:** Show error message + Retry button

### 4.2. AlertCreateDialog

**Implementation steps:**

1. **Dialog content:**
   - Stock symbol dropdown — fetch from `getWatchlistData()` in `watchlist.service.ts`
   - Alert type — 3 radio options: Price Above, Price Below, Volume Spike
   - Threshold input — number input, label changes per type:
     - Price: "Threshold (VND)" with step=1000
     - Volume: "Volume multiplier (x)" with step=0.5, default=3.0
   - Helper text below input: "Notify when price goes above X" / "Notify when volume exceeds Xx average"

2. **Dialog open triggers:**
   - "+ Create Alert" button on AlertsPage — no pre-fill
   - Bell button on StockDetailPage — pre-fill `symbol` (disabled, not editable)

3. **Submit flow:**
   - Client-side validate threshold via `validateThreshold()`
   - Submit button shows spinner, disabled while submitting
   - On success → close dialog, call parent's refresh function (pass via prop or callback)
   - On 400 limit error → show toast with "Upgrade to PRO" link
   - On other error → toast error message

4. **Props interface:**
```typescript
type AlertCreateDialogProps = {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  prefillSymbol?: string;   // optional, from StockDetailPage
  onSuccess: () => void;    // parent refresh callback
};
```

5. **Stock dropdown data source:**
```typescript
import { getWatchlistStocks, type WatchlistStock } from '@/services/watchlist.service';
// getWatchlistStocks() returns { symbol, companyName, exchange }[]
```

### 4.3. StockDetailPage — Alert Configuration Card

**Location:** `src/pages/StockDetailPage/StockDetailPage.tsx`

**Current state:** Has mock polling + sidebar with static "Alert Configuration" placeholder showing `--` values.

**Implementation steps:**

1. **Fetch alerts for this symbol** on mount:
```typescript
const [stockAlerts, setStockAlerts] = useState<AlertItem[]>([]);

useEffect(() => {
  if (!symbol || !isAuthenticated) return;
  let cancelled = false;
  getAlerts().then(list => {
    if (!cancelled) setStockAlerts(list.filter(a => a.symbol === symbol));
  }).catch(() => {});
  return () => { cancelled = true };
}, [symbol, isAuthenticated]);
```

2. **Replace the Alert Configuration sidebar block** (around line 557-564):
```
┌─────────────────────────────┐
│  Alert Configuration        │
├─────────────────────────────┤
│  Active: 2                   │
│  Latest trigger: 02/07/2026  │
│                             │
│  [FPT] Price > 140,000  ●  │  ← toggle: green = ACTIVE
│  [FPT] Price < 130,000  ○  │  ← toggle: gray = DISABLED
│                             │
│  [+ Add Alert]              │  ← opens AlertCreateDialog with prefillSymbol=symbol
└─────────────────────────────┘
```

3. **Each alert row:**
```typescript
{alerts.map(a => (
  <div key={a.id} className="flex items-center justify-between">
    <span>
      {a.alert_type === 'PRICE_ABOVE' ? '↑' : a.alert_type === 'PRICE_BELOW' ? '↓' : '📊'}
      {' '}{a.alert_type === 'VOLUME_SPIKE' ? `${a.threshold}x vol` : `${formatNumber(a.threshold)}`}
    </span>
    <button onClick={() => toggleAlert(a.id, a.status).then(refresh)}>
      {a.status === 'ACTIVE' ? '●' : '○'}
    </button>
  </div>
))}
```

4. **Bell button** — the existing `<Bell className="size-3.5" /> Alert` button on line 484:
   - Currently just a static button
   - Wire it to open AlertCreateDialog with `prefillSymbol={symbol}`

### 4.4. StockDetailPage — Mock Polling

**Already implemented.** For reference:
- Polls `GET /api/stocks/:symbol` every 1.5s when mock session active
- Appends candles to chart
- Shows toast on `alert_triggered: true`
- Stops on `_done: true`
- Shows "LIVE DEMO" badge during mock

Fields returned during mock:
```typescript
_mock: boolean         // true when mock session active
_cursor: number        // current tick (1-based)
_remaining: number     // ticks remaining
_done: boolean         // true = last tick
alert_triggered: boolean  // true when this tick just crossed threshold
alert_status: string | null // "TRIGGERED" or null
```

The mock is started by staff via Swagger (`POST /api/staff/mock-data/start`) — no user-facing trigger.

### 4.5. SettingsPage — Plan Info

**Status:** Route exists. Page exists at `src/pages/SettingsPage/index.tsx` — currently empty div.

**Implementation steps:**

1. **User plan from auth store:**
```typescript
const user = useAuthStore(s => s.user);
const plan = user?.plan || 'FREE';
const subscription = user?.subscription;
```

2. **Plan limits display:**
```
┌─────────────────────────────────┐
│  Plan Information               │
├─────────────────────────────────┤
│  Current plan: FREE             │
│  Alert stocks: 1 / 2            │  ← fetch from GET /api/alerts count
│  Alerts per stock: 1 / 2        │
│  [Upgrade to PRO]               │  ← link to /upgrade
│                                 │
│  PRO expires: N/A               │
└─────────────────────────────────┘
```

3. **Data source for usage counts:**
```typescript
const alerts = await getAlerts();
const alertedStocks = new Set(alerts.map(a => a.symbol)).size;
const alertsPerStock = Math.max(...['FPT','HPG'].map(s => alerts.filter(a => a.symbol === s).length).filter(Boolean), 0);
```

4. **Email toggle:**
```typescript
const [emailEnabled, setEmailEnabled] = useState(() => localStorage.getItem('alert_email') !== 'false');
const toggleEmail = (on: boolean) => {
  setEmailEnabled(on);
  localStorage.setItem('alert_email', String(on));
};
```
Note: email sending is server-side (based on SMTP config + user email). This toggle is just a FE preference for showing/hiding email channel indicator. Server always tries to send.

**Plan limit constants:**
```typescript
const PLAN_LIMITS = {
  FREE: { max_alert_stocks: 2, max_alerts_per_stock: 2 },
  PRO: { max_alert_stocks: 50, max_alerts_per_stock: 10 },
};
```

---

## 5. Error Handling

### Toast messages per scenario:

| Scenario | Toast |
|----------|-------|
| Create success | "Alert created for FPT" |
| Create limit | "Alert stock limit exceeded. Upgrade to PRO" + link |
| Create not in watchlist | "Stock must be in your watchlist" |
| Toggle success | "Alert updated" |
| Delete success | "Alert deleted" |
| Delete error | "Failed to delete alert" |
| Load error | "Unable to load alerts" + Retry button |
| Alert triggered (mock) | "Alert Triggered! — FPT crossed the threshold" |

### AlertCreateDialog inline validation errors:

| Field | Error |
|-------|-------|
| symbol (empty) | "Please select a stock" |
| threshold (empty) | "Please enter a threshold" |
| threshold (not positive) | "Threshold must be a positive number" |
| threshold (price < 1000) | "Price threshold min 1,000 VND" |
| threshold (volume < 1) | "Volume multiplier min 1.0x" |

---

## 6. File Checklist

| Action | File | Notes |
|--------|------|-------|
| **Create** | `src/services/alert.service.ts` | Copy from Section 3 |
| **Modify** | `src/pages/AlertsPage/index.tsx` | Full page: list, empty, loading, error states |
| **Modify** | `src/pages/StockDetailPage/StockDetailPage.tsx` | Alert Configuration card + Bell dialog + mock polling (already done) |
| **Modify** | `src/pages/SettingsPage/index.tsx` | Plan info + email toggle |

No route changes needed — all routes already registered in `layoutRoutes.tsx`.

---

## 7. Alert Lifecycle Reference

```
ACTIVE ──→ DISABLED  (user toggle off)
DISABLED ──→ ACTIVE  (user toggle on)
ACTIVE ──→ TRIGGERED (scheduler or mock triggers)
TRIGGERED ──→ ACTIVE (user re-enables, clears triggered_at)
```

**Per-status UI:**

| Status | Badge color | Toggle action |
|--------|------------|--------------|
| ACTIVE | green | → DISABLED |
| TRIGGERED | yellow/amber | → ACTIVE (reset) |
| DISABLED | gray | → ACTIVE |

---

## 8. Swagger API Docs

All alert endpoints documented in Swagger at `/api-docs` (tag: Alerts). Mock-data endpoints at tag: Staff Mock Data.

- `GET /api/alerts` — test with any authenticated user
- `POST /api/alerts` — requires stock in watchlist
- `PUT /api/alerts/:id` — test transitions
- `POST /api/staff/mock-data/start` — requires STAFF/ADMIN role
