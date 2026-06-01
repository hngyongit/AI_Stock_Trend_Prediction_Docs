---
name: AI Stock Trend
colors:
  surface: '#10131a'
  surface-dim: '#10131a'
  surface-bright: '#363941'
  surface-container-lowest: '#0b0e15'
  surface-container-low: '#191b23'
  surface-container: '#1d2027'
  surface-container-high: '#272a31'
  surface-container-highest: '#32353c'
  on-surface: '#e1e2ec'
  on-surface-variant: '#c2c6d6'
  inverse-surface: '#e1e2ec'
  inverse-on-surface: '#2e3038'
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
  background: '#10131a'
  on-background: '#e1e2ec'
  surface-variant: '#32353c'
  background-primary: '#0F172A'
  surface-card: '#111827'
  surface-elevated: '#1E293B'
  border-subtle: '#334155'
  text-primary: '#F8FAFC'
  text-secondary: '#94A3B8'
  status-positive: '#22C55E'
  status-negative: '#EF4444'
  status-warning: '#FACC15'
  status-info: '#38BDF8'
  status-disabled: '#64748B'
typography:
  headline-xl:
    fontFamily: Inter
    fontSize: 28px
    fontWeight: '700'
    lineHeight: 36px
  headline-lg:
    fontFamily: Inter
    fontSize: 22px
    fontWeight: '600'
    lineHeight: 28px
  headline-md:
    fontFamily: Inter
    fontSize: 18px
    fontWeight: '600'
    lineHeight: 24px
  body-md:
    fontFamily: Inter
    fontSize: 14px
    fontWeight: '400'
    lineHeight: 20px
  label-md:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: '500'
    lineHeight: 16px
  kpi-display:
    fontFamily: Inter
    fontSize: 20px
    fontWeight: '700'
    lineHeight: 24px
    letterSpacing: -0.01em
  headline-xl-mobile:
    fontFamily: Inter
    fontSize: 24px
    fontWeight: '700'
    lineHeight: 32px
rounded:
  sm: 0.125rem
  DEFAULT: 0.25rem
  md: 0.375rem
  lg: 0.5rem
  xl: 0.75rem
  full: 9999px
spacing:
  xs: 4px
  sm: 8px
  md: 12px
  lg: 16px
  xl: 24px
  xxl: 32px
  sidebar-width: 20%
  gutter: 16px
---

# Stock Market Data Visualization Platform Design System

## Brand Identity
- **Name**: AI Stock Trend
- **Core Values**: Operational efficiency, fast scanning, readability, trust, data clarity.
- **Visual Style**: Dark-first, professional financial dashboard, data-centric, dense layout, compact.

## Colors (Dark Theme Only)
- **Background**: #0F172A (Primary background)
- **Surface**: #111827 (Card backgrounds)
- **Elevated**: #1E293B (Hover states, modals)
- **Border**: #334155 (Subtle separators)
- **Primary**: #3B82F6 (Main brand color, primary actions)
- **Secondary**: #60A5FA (Support color)
- **Text Primary**: #F8FAFC (High contrast text)
- **Text Secondary**: #94A3B8 (Lower contrast metadata)
- **Positive**: #22C55E (Gains, success, upward movement)
- **Negative**: #EF4444 (Losses, danger, errors, downward movement)
- **Neutral**: #FACC15 (Warnings, pending states)
- **Info**: #38BDF8 (Informational badges)
- **Offline/Disabled**: #64748B (Inactive states)

## Typography
- **Primary Font**: Inter, Roboto, or Arial
- **H1**: 28px (Bold, Page titles)
- **H2**: 22px (Section headers)
- **H3**: 18px (Card titles)
- **Body**: 14px (Standard content)
- **Small**: 12px (Minimum size, metadata, labels)
- **KPIs**: Bold, high visibility for prices and percentages.

## Layout & Spacing
- **Sidebar**: 18-20% width (Collapsible)
- **Main Content**: 80-82% width
- **Spacing Scale**: 4, 8, 12, 16, 24, 32, 48
- **Density**: High (compact data-dense layouts)

## Components
- **Navbar**: Top-aligned, logo, ticker selection, market status badges, notifications, profile.
- **Sidebar**: Left-aligned, navigation items with active states.
- **KPI Cards**: Metric label, value, and % change indicator.
- **Charts**: Candlestick with OHLCV tooltips, volume bars, technical indicators (SMA, EMA, RSI).
- **Tables**: Compact, sticky headers, right-aligned numeric values, hover states.
- **Alerts**: Toast, inline banners, notification drawer with severity coloring.
- **Forms**: Labels above inputs, timeframe selectors, search inputs, dropdowns.
- **Badges**: Small, rounded, used for Market Status (Open/Closed), Exchange (HOSE/VN30).

## Visual Treatment
- **Minimal**: No excessive gradients, glassmorphism, or large empty spaces.
- **Separation**: Clear visual hierarchy using borders and surface color changes.
- **Animation**: Limited to skeletons, toasts, chart transitions, and panel toggles.
