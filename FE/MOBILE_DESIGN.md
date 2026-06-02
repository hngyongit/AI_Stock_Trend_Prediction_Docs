---
name: Precision Analytical Dashboard
colors:
  surface: '#101415'
  surface-dim: '#101415'
  surface-bright: '#363a3b'
  surface-container-lowest: '#0b0f10'
  surface-container-low: '#191c1e'
  surface-container: '#1d2022'
  surface-container-high: '#272a2c'
  surface-container-highest: '#323537'
  on-surface: '#e0e3e5'
  on-surface-variant: '#c2c6d6'
  inverse-surface: '#e0e3e5'
  inverse-on-surface: '#2d3133'
  outline: '#8c909f'
  outline-variant: '#424754'
  surface-tint: '#adc6ff'
  primary: '#adc6ff'
  on-primary: '#002e6a'
  primary-container: '#4d8eff'
  on-primary-container: '#00285d'
  inverse-primary: '#005ac2'
  secondary: '#a4c9ff'
  on-secondary: '#00315d'
  secondary-container: '#0267b8'
  on-secondary-container: '#d6e5ff'
  tertiary: '#ffb786'
  on-tertiary: '#502400'
  tertiary-container: '#df7412'
  on-tertiary-container: '#461f00'
  error: '#ffb4ab'
  on-error: '#690005'
  error-container: '#93000a'
  on-error-container: '#ffdad6'
  primary-fixed: '#d8e2ff'
  primary-fixed-dim: '#adc6ff'
  on-primary-fixed: '#001a42'
  on-primary-fixed-variant: '#004395'
  secondary-fixed: '#d4e3ff'
  secondary-fixed-dim: '#a4c9ff'
  on-secondary-fixed: '#001c39'
  on-secondary-fixed-variant: '#004883'
  tertiary-fixed: '#ffdcc6'
  tertiary-fixed-dim: '#ffb786'
  on-tertiary-fixed: '#311400'
  on-tertiary-fixed-variant: '#723600'
  background: '#101415'
  on-background: '#e0e3e5'
  surface-variant: '#323537'
typography:
  price-display:
    fontFamily: Inter
    fontSize: 32px
    fontWeight: '700'
    lineHeight: 40px
    letterSpacing: -0.02em
  screen-title:
    fontFamily: Inter
    fontSize: 24px
    fontWeight: '600'
    lineHeight: 32px
  section-title:
    fontFamily: Inter
    fontSize: 20px
    fontWeight: '600'
    lineHeight: 28px
  card-title:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: '600'
    lineHeight: 24px
  body:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '400'
    lineHeight: 20px
  secondary-info:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '400'
    lineHeight: 16px
  tiny-label:
    fontFamily: Inter
    fontSize: 11px
    fontWeight: '500'
    lineHeight: 14px
    letterSpacing: 0.05em
  price-display-mobile:
    fontFamily: Inter
    fontSize: 28px
    fontWeight: '700'
    lineHeight: 36px
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  unit: 8px
  screen-padding: 16px
  card-padding: 16px
  section-gap: 24px
  table-cell-padding-v: 8px
  table-cell-padding-h: 12px
  stack-gap-sm: 4px
  stack-gap-md: 8px
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
