# AI Stock Trend Mobile Dev Rules

This file defines implementation rules for FE coding agents working on the mobile app.
Read together with:

- `AI_Stock_Trend_Prediction_Docs/FE/MOBILE_DESIGN.md`
- `AI_Stock_Trend_Prediction_Docs/01-project-overview.md`

## 1) Workspace & Repo Context

This project is opened as one parent workspace that contains two separate Git repos:

1. Docs repo: `AI_Stock_Trend_Prediction_Docs`
2. Code repo: `FE_AI_Stock_Trend_Prediction`

Rules:

- Mobile code changes go under: `FE_AI_Stock_Trend_Prediction/mobile`
- Mobile documentation updates go under: `AI_Stock_Trend_Prediction_Docs/FE`
- If implementation changes structure, routing, state management, or platform-specific setup, update docs in the same task when appropriate

## 2) Current Mobile Structure

Application source is rooted in `mobile/src`.

Use these ownership areas:

- `src/app`: Expo Router route entry files only
- `src/features`: screen modules and feature-specific UI
- `src/shared`: reusable presentational building blocks and design tokens
- `src/stores`: Zustand stores for shared client state
- `components/ui`: generated Gluestack primitives

Do not move product screens back into template-style mixed folders.

## 3) Styling & Component Rules

Current styling stack:

- NativeWind
- Gluestack UI primitives
- Shared tokens from `src/global.css`
- Shared mobile design tokens in `src/shared/design/tokens.ts`

Rules:

- Preserve the dark financial dashboard direction defined in `MOBILE_DESIGN.md`
- Reuse `components/ui/*` primitives before creating raw replacements
- Keep product-level composition in `src/features/*` or `src/shared/components/*`
- Prefer token-based colors and spacing over ad-hoc inline values

## 4) Zustand Rules

Current shared state implementation:

- Zustand is the shared client-state layer for mobile
- Current store: `src/stores/market.store.ts`

Use Zustand for:

- Cross-screen state
- Selected ticker / watchlist focus
- Session-like client state shared across routes
- App-wide UI preferences if they become shared

Do not use Zustand for:

- One-screen form inputs
- Temporary modal open/close booleans scoped to a single component
- Pure server cache when React Query is a better fit

## 5) Routing Rules

Current route shape:

- `src/app/index.tsx`
- `src/app/watchlist.tsx`
- `src/app/_layout.tsx`

Rules:

- Keep route files thin and delegate to feature screens
- Put actual screen implementation in `src/features/*`
- When renaming routes, regenerate Expo Router typed routes and clear Metro cache

## 6) Android Dev Rules

These rules are required because Metro, Expo Router, and NativeWind can hold stale state aggressively on Android.

When route aliases, route filenames, or CSS entry wiring changes:

- Restart with `npx expo start -c`

When Expo Router shows a false "missing default export" warning after structural changes:

- Assume module import failure first
- Verify the route file still has a default export
- Check imported modules in that route for runtime-native crashes

When changing aliases or route paths:

- Keep `babel.config.js` and `tsconfig.json` aligned
- Re-run an Android bundle verification with `npx expo export --platform android --dev`

Avoid introducing native-only animation/runtime dependencies casually:

- If a dependency loads at route-import time and fails on Android, Expo Router may report misleading route errors
- Prefer built-in React Native `Animated` unless a stronger native animation requirement is clearly justified

## 7) Delivery Checklist

1. Keep route files thin
2. Reuse Gluestack primitives first
3. Keep shared state in Zustand only where it is actually shared
4. Keep styling aligned with `MOBILE_DESIGN.md`
5. Run `npx tsc --noEmit`
6. Run `npx expo export --platform android --dev` after routing/alias/platform-sensitive changes
7. Update FE docs when structure, state, or Android development conventions change
