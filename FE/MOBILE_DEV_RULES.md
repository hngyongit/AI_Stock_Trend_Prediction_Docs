# AI Stock Trend — Mobile Dev Rules (Revised June 2026)

This document defines the **strict architecture, component, and coding rules** for all AI agents and developers working on the mobile app.

**Read this file first** before any mobile implementation.

Companion docs:
- `AI_Stock_Trend_Prediction_Docs/FE/MOBILE_DESIGN.md` (design tokens, component specs)
- `AI_Stock_Trend_Prediction_Docs/01-project-overview.md` (project scope, MVP direction)

---

## 1. Project Direction

The mobile app is a **compact financial data visualization app** for HOSE/VN30 stock monitoring.

**Priority MVP features:**
1. Authentication (Login/Register)
2. Dashboard (market overview, data quality visibility)
3. Stock Detail (OHLCV chart, validated data, last updated timestamp)
4. Watchlist
5. Alerts (list + create)
6. Search (ticker/instrument lookup)
7. Profile (session controls, preferences)
8. Market status indicator

**Deliberately NOT prioritized (do not build):**
- AI prediction as a main feature
- AI chatbot / conversational UI
- Real-time heavy behavior (tick data)
- Trading game UI
- Portfolio or advanced analytics (future only)
- Large decorative screens
- Overengineered component hierarchies

---

## 2. Folder Structure

```
mobile/
├── App.tsx                          # Root: GestureHandlerRootView → ThemeProvider → NavigationContainer → RootNavigator
├── app.config.js                    # Expo config, env var bridge
├── app.json                         # Expo app manifest
├── babel.config.js                  # nativewind/babel + reanimated plugin
├── metro.config.js                  # nativewind/metro
├── tailwind.config.js               # NativeWind + custom colors
├── tsconfig.json                    # Path aliases: @/, @/app, @/shared, @/features, @/stores, @/assets
├── package.json
│
├── assets/
│   ├── images/
│   │   └── tabIcons/               # Tab bar icon assets
│   └── fonts/
│
└── src/
    ├── global.css                   # Tailwind directives + CSS custom properties
    │
    ├── app/
    │   ├── navigation/
    │   │   ├── RootNavigator.tsx     # NativeStack: Startup → Login/Register → MainTabs → detail screens
    │   │   ├── MainTabNavigator.tsx  # Bottom tabs (Dashboard, Watchlist, Alerts, Search, Profile) + session guard
    │   │   ├── AppTabBar.tsx         # Custom tab bar with icons, badges, responsive sizing
    │   │   └── navigation.types.ts   # RootStackParamList, MainTabParamList, screen prop types
    │   └── config/
    │       └── (env.ts, app.config.ts when present)
    │
    ├── features/                    # Feature modules
    │   ├── auth/
    │   │   ├── types.ts             # Auth response types
    │   │   ├── services/
    │   │   │   └── auth.service.ts  # API calls (login, register, logout, refresh, me)
    │   │   ├── hooks/
    │   │   │   └── useLoginForm.ts  # Form state + validation
    │   │   ├── components/
    │   │   │   ├── LoginHeader.tsx
    │   │   │   ├── LoginForm.tsx
    │   │   │   └── AuthFooterLinks.tsx
    │   │   └── screens/
    │   │       ├── LoginScreen.tsx      # ≤250 lines
    │   │       └── RegisterScreen.tsx   # ≤250 lines
    │   ├── dashboard/               # Market overview screen
    │   ├── stocks/                  # Stock detail: PriceHeader, ChartSection, StockInfoTable
    │   ├── watchlist/               # Watchlist screen + swipeable rows + filters
    │   ├── alerts/                  # Alert list + create alert screen
    │   ├── search/                  # Stock search screen
    │   ├── profile/                 # Profile, change password, edit profile
    │   └── startup/                 # Splash/startup screen + auth check
    │
    ├── shared/
    │   ├── design/
    │   │   └── tokens.ts            # THE SOURCE OF TRUTH: palette, spacing, radius, typography
    │   ├── services/
    │   │   ├── api.service.ts       # Single axios factory (getApiBaseUrl, createApiClient)
    │   │   └── tokenStorage.ts      # AsyncStorage helpers (persist, read, clear session)
    │   ├── ui/
    │   │   ├── index.ts             # Barrel exports: all shared UI components
    │   │   ├── primitives/          # Pure RN building blocks (Box, VStack, HStack, Text, Button, Input, etc.)
    │   │   ├── components/          # Composed shared UI (MetricCard, StatusBadge, StockListItem)
    │   │   ├── feedback/            # Toast, AlertBanner, LoadingSkeleton
    │   │   ├── forms/               # FormField, CheckboxRow, PasswordToggle
    │   │   ├── layout/              # AppScreen, SectionHeader
    │   │   └── utils/
    │   │       └── ThemeProvider.tsx # Theme context + toast system (replaces GluestackUIProvider)
    │   ├── hooks/                   # Shared hooks (useRefresh, useOffline)
    │   ├── utils/                   # Pure utility functions
    │   ├── constants/              # App-wide constants (future)
    │   └── types/                   # Shared TS types
    │
    └── stores/                      # Zustand global stores ONLY
        ├── auth.store.ts            # Session, token, user state
        ├── market.store.ts          # Market status, tickers (mock data currently)
        ├── app-shell.store.ts       # Offline, stale, notification badge count
        └── startup.store.ts         # Startup orchestration state
```

---

## 3. Import Rules (STRICT)

### Allowed import directions ONLY

```
features/  ──▶  shared/         (features import from shared)
features/  ──▶  app/navigation  (features import types only)
app/       ──▶  features/       (app loads screens)
shared/    ──▶  shared/         (shared imports from shared only)
app/       ──▶  shared/         (app imports providers/theme)
```

### Forbidden imports

| Forbidden Pattern | Why |
|---|---|
| `shared/` importing from `features/` | Creates circular dependency risk |
| Feature importing another feature directly | Must go through public index or `app/` |
| Screen importing `axios` or raw API client | Must use service layer |
| Screen importing `AsyncStorage` | Must use `tokenStorage.ts` |
| Screen importing RN primitives directly | Must use `@/shared/ui/primitives` wrappers |
| Deep imports from another feature's private folders | Breaks module boundaries |

### Path aliases (use these)

| Alias | Maps to |
|---|---|
| `@/app` | `mobile/src/app` |
| `@/shared` | `mobile/src/shared` |
| `@/features` | `mobile/src/features` |
| `@/stores` | `mobile/src/stores` |
| `@/assets` | `mobile/assets` |

### Barrel export rules

- `shared/ui/index.ts` exports only stable, reusable components
- Each feature's `index.ts` exports ONLY intentionally public types/components
- Do NOT use barrel exports for internal feature modules

---

## 4. File Size Limits (ENFORCED)

| File Type | Target | Hard Limit |
|---|---|---|
| Screen file | 80–180 lines | **250 lines** |
| Shared component | 40–120 lines | 180 lines |
| Hook | 30–100 lines | 150 lines |
| Service file | 40–150 lines | 200 lines |
| Store file | 40–120 lines | 180 lines |
| Type file | No limit | Keep organized |
| Utility file | 20–80 lines | 120 lines |
| Schema file | 20–60 lines | 80 lines |

### If a screen exceeds 250 lines, SPLIT into:

```
screens/
  ScreenName.tsx           # Container only — hooks + components
components/
  ScreenHeader.tsx         # Section component
  ScreenForm.tsx           # Form component
  ScreenListItem.tsx       # List item
hooks/
  useScreenLogic.ts        # State + side effects
services/
  screen.service.ts        # API calls
schemas/
  screen.schema.ts         # Validation
types.ts                   # Local types
```

---

## 5. UI Component Architecture (Post-Gluestack Removal)

**Gluestack was removed in June 2026.** All UI components are now pure React Native with `StyleSheet.create()` and `tva` (tailwind-variants wrapper) for variant styling. No gluestack dependencies remain.

### Primitive Components (`shared/ui/primitives/`)

These are pure React Native building blocks — never import RN primitives directly in feature screens; always use these wrappers:

| Component | File | Description |
|---|---|---|
| `Box` | `primitives/box/` | `<View>` wrapper with tva styling |
| `VStack` | `primitives/vstack/` | Vertical flex container |
| `HStack` | `primitives/hstack/` | Horizontal flex container |
| `Text` | `primitives/text/` | Styled RN Text with variants (size, bold, italic, truncate) |
| `Button` | `primitives/button/` | Multi-variant (primary, secondary, positive, negative, outline, link, sizes xs–xl) |
| `Input` | `primitives/input/` | RN TextInput with focus/error/disabled states |
| `Pressable` | `primitives/pressable/` | RN Pressable wrapper |
| `Card` | `primitives/card/` | Surface card container |
| `Divider` | `primitives/divider/` | Line separator |
| `Switch` | `primitives/switch/` | RN Switch wrapper |
| `Checkbox` | `primitives/checkbox/` | Pressable-based checkbox |
| `Spinner` | `primitives/spinner/` | ActivityIndicator wrapper |
| `Modal` | `primitives/modal/` | RN Modal re-export |
| `Icon` | `primitives/icon/` | Lucide-based icon component |
| `Badge` | `primitives/badge/` | Minimal badge stub |
| `Avatar` | `primitives/avatar/` | Minimal avatar stub |

### Shared Composed Components (`shared/ui/components/`)

- `MetricCard` — KPI display with label, value, detail, tone color
- `StatusBadge` — Pill badge with tone (up/down/warning/primary/neutral)
- `StockListItem` — Stock row with symbol, name, price
- `FeaturePlaceholderScreen` — Placeholder for incomplete features

### Import Rules for UI

- Feature screens import from `@/shared/ui` (the barrel export)
- NEVER import from `react-native` `View`, `Text`, etc. directly in feature code
- NEVER use inline `style` objects — use `StyleSheet.create()`
- NEVER hardcode colors/spacing/radii — use tokens from `shared/design/tokens.ts`

### Theme Provider

`shared/ui/utils/ThemeProvider.tsx` replaces the old `GluestackUIProvider`. It provides:
- Theme context (dark theme only)
- Toast system via `useToast()` hook

---

## 6. Design System (Fixed Tokens)

### Theme
**Dark theme only.** No light theme support.

### Colors — NEVER hardcode in screens

```
background:  #0F172A
surface:     #111827
elevated:    #1E293B
border:      #334155
textPrimary: #F8FAFC
textSecondary: #94A3B8
positive:    #22C55E
negative:    #EF4444
warning:     #F59E0B
info:        #38BDF8
offline:     #64748B
```

### Spacing (8pt system)

```
xs:  4
sm:  8
md:  16
lg:  24
xl:  32
```

### Typography

```
largePrice:   28–32
screenTitle:  24
sectionTitle: 20
cardTitle:    16
body:         14
secondary:    12
tiny:         10–11
```

### Touch targets
Minimum **44×44** for all interactive elements.

### Styling rules
- All colors, spacing, radii, and font sizes come from `shared/design/` tokens
- No ad-hoc color/radius/font values in screen files
- `StyleSheet.create()` only — no inline style objects in JSX
- No NativeWind/Tailwind classes in production screens (CSS is for global reset only)

---

## 7. Data Flow Architecture

### Strict 4-layer pattern

```
Screen
  → calls Hook(s)        (features/x/hooks/)
    → calls Service(s)    (features/x/services/)
      → calls apiClient   (shared/services/api.service.ts)
```

### Example

```
WatchlistScreen
  → useWatchlist()
    → watchlist.service.ts
      → createApiClient().get('/api/watchlists')
```

### Rules
- **Screens NEVER call Axios or fetch directly**
- **Screens NEVER access AsyncStorage directly**
- **Screens NEVER contain inline mock data** (move to service or a separate mock)
- **Screens NEVER contain validation schemas** (move to `schemas/`)
- **Hooks own all screen-level state** (loading, data, error)
- **Services own all API communication**
- **Services return typed responses** — never `any` unless documented temp exception

---

## 8. State Management Rules

### Global state → Zustand ONLY for:

- Auth session / token / user
- Selected market (if globally needed)
- Selected stock symbol (for deep links)
- App shell state (offline, stale, notification count)
- Theme/config (if needed)

### Local state → useState/useReducer for:

- Form field values
- Modal open/close toggles (screen-scoped)
- Single-screen loading/error states
- UI-only toggles

### Do NOT use Zustand for:

- Local form state
- Temporary modal visibility (scoped to one screen)
- Single-screen loading state
- Server cache (that's React Query's job)

---

## 9. Navigation Rules

## 9a. Header Architecture (Revised June 2026)

### Core principle: One contextual header per screen

No global app-name header is shown across all tab screens. Each screen manages its own top area. The tab navigator (`MainTabNavigator`) sets `headerShown: false` — no shared `AppHeader` is rendered.

### Main tab screen header behavior

Each tab screen renders its own contextual header at the top of its content:

| Tab | Header Content | Bottom Navbar |
|---|---|---|
| Dashboard | Small brand mark + market status badge (home screen) | Visible |
| Watchlist | "My Watchlist" title + market status subtitle + Sort/Add actions | Visible |
| Alerts | "Alerts" title + filter/create actions | Visible |
| Search | Search-focused header only | Visible |
| Profile | Account/profile header | Visible |

### Child screen header behavior

| Screen | Header | Bottom Navbar |
|---|---|---|
| Stock Detail | Back header | Hidden |
| Create Alert | Back header | Hidden |
| Edit Alert | Back header | Hidden |
| Notification Center | Back header | Hidden |
| Settings | Back header | Hidden |
| Change Password | Back header | Hidden |

### Header design rules

1. **Avoid duplicated headers.** No screen should have both a global app bar and its own title header.
2. **App branding** must not consume vertical space on every tab. It may appear on Splash, Login, or Dashboard (home screen).
3. **Header actions must be screen-specific.** Each tab's header owns its actions (e.g., Add, Sort, Filter).
4. **Profile/settings icon** should not float over every tab — profile access lives in the Profile tab.
5. **Safe area padding** is handled by each screen via `useSafeAreaInsets()`.
6. **Market status badge** is shown contextually — Dashboard (prominent) and Watchlist (compact subtitle badge). Other tabs may omit it.
7. **Child screens** must have a top-back header with no bottom navbar.
8. **Previous screen state** is preserved via `freezeOnBlur`.

### Implementation pattern

```tsx
// MainTabNavigator — no global header
<Tab.Navigator
  screenOptions={{
    headerShown: false, // each screen manages its own top area
    freezeOnBlur: true,
    lazy: true,
  }}
  tabBar={(props) => <AppTabBar {...props} />}>
  ...
</Tab.Navigator>
```

```tsx
// Each tab screen applies safe area at the top
import { useSafeAreaInsets } from 'react-native-safe-area-context';

export function WatchlistScreen() {
  const insets = useSafeAreaInsets();

  return (
    <View style={styles.shell}>
      <FlatList
        contentContainerStyle={{ paddingTop: insets.top }}
        ListHeaderComponent={renderContextualHeader}
        ...
      />
    </View>
  );
}
```

---

## 10. API Service Rules

### Layered structure

```
shared/services/
  api.service.ts     # Single Axios factory (getApiBaseUrl, createApiClient)
  tokenStorage.ts    # AsyncStorage read/write for tokens

features/x/services/
  x.service.ts       # Feature-specific API functions
```

### api.service.ts requirements
- `createApiClient()` returns a configured Axios instance
- `baseURL` from env (not hardcoded)
- Bearer token injected per-call via `useAuthStore.getState().session?.accessToken`
- 401 handling is per-service (no global interceptor yet)
- Timeout: 10s default
- Typed request/response functions

---

## 11. Mock Data Rules

- Mock data files live in `features/x/services/__mocks__/` or a top-level `mocks/` folder
- Mock data is NEVER inline in screen files
- Mock data is ONLY used when the API is not ready
- All mocks are removed or gated behind a config flag before production

---

## 12. TypeScript Rules

- `strict: true` in `tsconfig.json`
- No `any` types — exceptions require `// eslint-disable-next-line @typescript-eslint/no-explicit-any` with a reason comment
- API responses must have typed interfaces
- Navigation params must be typed in `navigation.types.ts`
- `as` casts are forbidden for API response data (use proper type guards)

---

## 13. Delivery Checklist

Before completing a mobile task:

1. **Structure check:** Files follow the folder structure from §2
2. **Size check:** No file exceeds its hard limit from §4
3. **Import check:** No forbidden imports from §3
4. **Component check:** No direct RN primitive imports in feature screens — all through `shared/ui/primitives/`
5. **Token check:** Colors/spacing/radii from `shared/design/` — no hardcoded values
6. **Data flow check:** Screen → Hook → Service → apiClient (no shortcuts)
7. **State check:** Zustand only for global state, useState for local
8. **Type check:** `npx tsc --noEmit` passes
9. **Navigation check:** Tab visibility rules from §9 followed
10. **Docs update:** Update MOBILE_DEV_RULES.md if structure, conventions, or architecture changed

---

## 14. AI Agent Instructions

When implementing a mobile feature:

1. Read this file fully first
2. Check the UI component architecture (§5) before creating any UI
3. Check the actual codebase folder structure in `mobile/README.md` §Project Structure for current state
4. Split files early — never let a file exceed 250 lines
5. Import ONLY from `@/shared/ui`, `@/shared/design`, `@/features/x`, `@/stores`, `@/app/navigation`
6. Always extract hooks, services, schemas, and types
7. Run `npx tsc --noEmit` before marking complete
8. Update docs if you change conventions or add new shared components
