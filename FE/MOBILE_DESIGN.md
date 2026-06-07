---
name: AI Stock Trend Mobile
colors:
  background: '#0F172A'
  surface: '#111827'
  elevated: '#1E293B'
  border: '#334155'
  textPrimary: '#F8FAFC'
  textSecondary: '#94A3B8'
  positive: '#22C55E'
  negative: '#EF4444'
  warning: '#F59E0B'
  info: '#38BDF8'
  offline: '#64748B'
  primaryAction: '#3B82F6'
typography:
  largePrice:
    fontSize: 28
    fontWeight: '700'
    lineHeight: 36
  screenTitle:
    fontSize: 24
    fontWeight: '700'
    lineHeight: 32
  sectionTitle:
    fontSize: 20
    fontWeight: '600'
    lineHeight: 28
  cardTitle:
    fontSize: 16
    fontWeight: '600'
    lineHeight: 24
  body:
    fontSize: 14
    fontWeight: '400'
    lineHeight: 20
  secondary:
    fontSize: 12
    fontWeight: '400'
    lineHeight: 16
  tiny:
    fontSize: 11
    fontWeight: '500'
    lineHeight: 14
spacing:
  xs: 4
  sm: 8
  md: 16
  lg: 24
  xl: 32
rounded:
  sm: 4
  md: 8
  lg: 14
  full: 9999
---

## Brand & Style
The design system is engineered for high-velocity financial environments where data density and clarity are paramount. The brand personality is clinical, authoritative, and operational, designed to minimize cognitive load during volatile market conditions. 

The aesthetic follows a **Corporate/Modern** style with a focus on functional minimalism. By removing decorative elements, the system ensures that every pixel serves a purpose in information delivery. The interface prioritizes a "lights-out" dashboard experience, catering to professional traders and analysts who require long-session comfort and immediate pattern recognition within the HOSE and VN30 markets.

## Colors
The palette is optimized for professional dark-mode environments. The base background uses a deep navy-slate to reduce eye strain, while surface layers utilize tonal shifts rather than shadows to define boundaries.

Semantic colors are strictly reserved for financial signals:
- **Uptrend:** Used for positive price delta, buy signals, and bullish indicators.
- **Downtrend:** Used for negative price delta, sell signals, and bearish indicators.
- **Warning:** Used for high volatility alerts or data latency issues.
- **Primary/Secondary:** Used for UI interactions and active states, providing a clear distinction from market movement data.

## Typography
The system uses **Inter** exclusively to take advantage of its excellent legibility in high-density data tables and small sizes. 

**Hierarchy Rules:**
1. **Price:** Always the largest element, utilizing semi-bold or bold weights.
2. **Movement (%):** Placed adjacent to Price, using semantic colors.
3. **Ticker Symbols:** Use `card-title` or `body` with high-contrast text.
4. **Data Labels:** Use `tiny-label` in uppercase with wider letter spacing for clear categorization.

Numerical data should use tabular figures (monospaced numbers) where possible to ensure alignment in lists and tables.

## Layout & Spacing
This design system utilizes a rigid **8pt spacing system** to ensure mathematical consistency across all operational views. 

**Grid Model:**
- **Desktop:** 12-column fluid grid for the main dashboard, allowing modular widget rearrangement.
- **Sidebars:** Fixed-width at 280px for watchlists and navigation.
- **Gaps:** A standard 24px gap between major functional sections.
- **Padding:** 16px internal padding for cards and containers to maximize content area without feeling cramped.

**Density:**
The layout is "Compact." Vertical rhythm is tight, particularly in financial tables, to allow as many ticker rows as possible to be visible above the fold.

## Elevation & Depth
Depth is communicated through **Tonal Layers** and **Low-Contrast Outlines**. In a data-heavy dark UI, traditional shadows can create visual "muddiness." Instead, this system uses:

1. **Base Level:** `#0F172A` (Background).
2. **Mid Level:** `#111827` (Main surface for cards and widgets).
3. **Top Level:** `#1E293B` (Popovers, tooltips, and active states).

**Borders:** All containers must have a 1px solid border of `#334155`. This creates sharp definition between data modules. Shadows are reserved for floating elements only (e.g., dropdowns) and must be subtle, using a black tint with 40% opacity and a 10px blur.

## Shapes
The shape language balances professional rigidity with modern software aesthetics.
- **Cards/Containers:** Use a consistent `rounded-lg` (12px to 16px) radius to soften the high-density grid.
- **Buttons/Inputs:** Use a 4px (Soft) radius to maintain a precise, tool-like feel.
- **Status Badges:** Use fully pill-shaped (rounded-full) corners for rapid identification of status icons or small labels.

Interactive elements should have a clear 1px border that brightens on hover to provide tactile feedback without shifting layout.

## Screen Header Architecture

The mobile app uses **one contextual header per screen** — no global app-name bar across all tabs.

- Main tab screens render their own header at the top of scrollable content (e.g., "My Watchlist" + subtitle + actions).
- Child screens (Stock Detail, Settings, etc.) use a back-header with no bottom navbar.
- Safe area padding is handled per screen via `useSafeAreaInsets()`.
- Market status and branding appear only where contextually relevant (Dashboard, compact subtitle on Watchlist).
- See `FE/MOBILE_DEV_RULES.md §9a` for the full header architecture spec.

## Components

### KPI Cards
- Background: Surface (`#111827`).
- Border: 1px (`#334155`).
- Structure: Ticker (Top Left), Sparkline (Top Right), Price (Large, Center-Left), % Change (Below Price).

### Financial Tables
- Row Height: 40px (Compact) or 48px (Standard).
- Header: `tiny-label` style with a subtle bottom border.
- Interaction: Row highlight on hover using `#1E293B`.
- Content: Text-align right for all numerical columns.

### Input Fields & Controls
- Background: `#0F172A` (Inset look).
- Typography: `body` (14px).
- State: Active/Focus uses `Primary #3B82F6` for the border.

### Candlestick Charts
- Bullish: Solid `#22C55E`.
- Bearish: Solid `#EF4444`.
- Gridlines: `#334155` at 10% opacity.
- Crosshair: Dashed line in `Text Secondary`.

### Status Badges
- Used for market state (Open/Closed) or trend signals.
- Transparent background with a 1px colored border and matching text color.
- Compact padding: 2px vertical, 8px horizontal.

## Implementation Notes

- Current mobile implementation uses Expo Router with thin route files under `mobile/src/app`.
- Shared client state is implemented with Zustand. The current store lives in `mobile/src/stores/market.store.ts`.
- Product screens should live under `mobile/src/features/*`, with reusable visual building blocks in `mobile/src/shared/*`.
- Android development must follow cache-clear and bundle verification rules from `FE/MOBILE_DEV_RULES.md` after alias, route, or CSS entry changes.
