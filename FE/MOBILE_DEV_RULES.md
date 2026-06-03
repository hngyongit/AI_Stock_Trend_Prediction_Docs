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
- Check `components/ui/*` for an existing Gluestack primitive before creating a manual replacement
- Prefer project-local resources and existing components before pulling in new libraries, patterns, or external examples
- Keep product-level composition in `src/features/*` or `src/shared/components/*`
- Prefer token-based colors and spacing over ad-hoc inline values
- Reuse predefined values from `src/global.css` and `src/shared/design/tokens.ts` before introducing new color, spacing, radius, or typography constants
- Create manual UI only when the existing Gluestack layer or current shared components do not reasonably support the use case

## 4) Zustand Rules

Current shared state implementation:

- Zustand is the shared client-state layer for mobile
- Current store: `src/stores/market.store.ts`

Use Zustand for:

- Cross-screen state
- Selected ticker / watchlist focus
- Session-like client state shared across routes
- App-wide UI preferences if they become shared
- Complex screen state with multiple transitions, retries, loading/error branches, or orchestration logic

Do not use Zustand for:

- One-screen form inputs
- Temporary modal open/close booleans scoped to a single component
- Pure server cache when React Query is a better fit

Escalation rule:

- If local component state starts becoming hard to reason about because several async states interact, move that logic into a focused Zustand store under `src/stores/*`

## 5) Forms, Validation, and API Rules

Use these defaults unless the task gives a strong reason otherwise:

- For complex forms, use `formik` for form state and `yup` for validation
- Keep API base URLs and API-facing configuration in environment variables, not hardcoded in screens or components
- Keep API access inside service modules under feature or shared service ownership; screens should call services, not construct request URLs directly
- Use `axios` for API requests instead of ad hoc `fetch` usage when building or expanding app services
- Store API logic in service files so later endpoint, auth, retry, and error-handling changes stay localized

Recommended ownership:

- Feature-specific API logic: `src/features/<feature>/...service.ts`
- Shared or cross-feature API helpers: `src/shared/services/*` if introduced later
- Shared client state: `src/stores/*`

## 6) Modularity Rules

Rules:

- Prefer modular screen construction so UI, service logic, and state logic can change independently later
- Keep route files thin, feature screens focused, and reusable visual pieces extracted when reused or when a screen becomes difficult to maintain
- Avoid burying endpoint calls, state orchestration, and formatting logic inside large screen files
- When adding new behavior, check whether an existing feature module, store, token file, or UI primitive already owns that concern before creating a new layer

## 7) Routing Rules

Current route shape:

- `src/app/index.tsx`
- `src/app/watchlist.tsx`
- `src/app/_layout.tsx`

Rules:

- Keep route files thin and delegate to feature screens
- Put actual screen implementation in `src/features/*`
- When renaming routes, regenerate Expo Router typed routes and clear Metro cache

## 8) Android Dev Rules

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

## 9) Code Review Expectations

Before considering a mobile task complete, review the change for:

- Whether an existing Gluestack primitive or project resource should have been reused
- Whether shared or complex state belongs in Zustand
- Whether API calls stayed inside service modules
- Whether design values came from `global.css` / shared tokens instead of new one-off constants
- Whether the implementation is modular enough to modify later without rewriting the screen
- Whether a new dependency was added without first exhausting existing project resources

## 10) Delivery Checklist

1. Keep route files thin
2. Check for reusable Gluestack primitives and existing project resources before building manually
3. Keep shared or complex orchestration state in Zustand only where it improves maintainability
4. Reuse values from `src/global.css` and `src/shared/design/tokens.ts`
5. Keep API calls in service modules and use `axios`
6. Use `formik` + `yup` for complex forms
7. Keep styling aligned with `MOBILE_DESIGN.md`
8. Run `npx tsc --noEmit`
9. Run `npx expo export --platform android --dev` after routing/alias/platform-sensitive changes
10. Update FE docs when structure, state, service conventions, or Android development conventions change
