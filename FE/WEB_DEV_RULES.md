# AI Stock Trend Web Dev Rules

This file defines implementation rules for FE coding agents.
Read together with:

- `AI_Stock_Trend_Prediction_Docs/FE/WEB_DESIGN.md`
- `AI_Stock_Trend_Prediction_Docs/01-project-overview.md`

## 1) Workspace & Repo Context

This project is opened as one parent workspace that contains **two separate Git repos**:

1. Docs repo: `AI_Stock_Trend_Prediction_Docs`
2. Code repo: `FE_AI_Stock_Trend_Prediction`

Rules:

- FE code changes go under: `FE_AI_Stock_Trend_Prediction/web`
- FE documentation updates go under: `AI_Stock_Trend_Prediction_Docs/FE`
- Do not mix code commits into docs repo paths and vice versa.
- If implementation introduces a new shared convention, route pattern, or architecture decision, update docs in `AI_Stock_Trend_Prediction_Docs/FE` in the same task when appropriate.

## 2) Reuse-First Policy

Before creating new components or utilities:

1. Search existing `src/components`, `src/lib`, `src/services`
2. Extend existing shared modules where possible
3. Only add new component files when reuse/extension is not viable

High-priority shared areas:

- `src/components/topbar/*`
- `src/components/ui/*` (shadcn)
- `src/lib/role-routes.ts`
- `src/components/topbar/account-actions.ts`
- `src/stores/*`

If an element appears in all 3 layouts (user/staff/admin), it must be implemented as shared behavior/component.

## 3) shadcn-First Component Rule

If UI can be achieved with existing shadcn primitives, use them first.

Examples:

- Avatar: `src/components/ui/avatar.tsx`
- Button/Input/Badge: `src/components/ui/*`

Do not build custom raw HTML versions of controls that already exist in shadcn unless there is a clear functional gap.

Custom components may wrap shadcn primitives, but should not duplicate them.

## 4) Styling & Token Rule

Custom CSS must follow both:

1. Tokens/variables from: `web/src/index.css`
2. Visual principles from: `AI_Stock_Trend_Prediction_Docs/FE/WEB_DESIGN.md`

Specific rules:

- Prefer `var(--...)` tokens from `index.css` for color, borders, radii, foreground/background semantics.
- Keep shell namespaces:
  - `.terminal-shell__*`
  - `.staff-shell__*`
  - `.admin-shell__*`
- Shared topbar components rely on `classNamePrefix`; ensure corresponding CSS hooks exist in each shell CSS.
- Do not introduce ad-hoc color systems that conflict with `WEB_DESIGN.md`.

## 5) Architecture Rules (Current Codebase)

- Routing:
  - Top-level routes: `src/routes/AppRoutes.tsx`
  - User/Staff route maps: `src/routes/layoutRoutes.tsx`
  - Protected wrappers: `renderProtectedLayoutRoute.tsx`, `RequireAuth.tsx`
- Layouts:
  - `UserLayout.tsx`, `StaffLayout.tsx`, `AdminLayout.tsx`
- Shared topbar:
  - `TopbarBrand`, `TopbarSearch`, `TopbarNotifications`, `TopbarUserMenu`, `TopbarControls`
- Client state:
  - Shared auth/session state is implemented with Zustand in `src/stores/auth.store.ts`
  - `src/providers/AuthProvider.tsx` is now a compatibility wrapper around `useAuthStore()`
  - Do not reintroduce a separate React Context source of truth for auth
- Role-aware helpers:
  - `getDefaultHomeRouteByRole`
  - `getProfileRouteByRole`
  - `getSettingsRouteByRole`

Do not hardcode role paths in multiple places.

## 6) API/Auth Rules

Source of truth:

- `src/services/auth.service.ts`
- `src/stores/auth.store.ts`

Use:

- `authenticatedRequest(...)` for protected endpoints
- Built-in refresh flow on `401` (`/api/auth/refresh-token`) with retry-once behavior
- `useAuthStore().setSession(...)` after successful login
- `useAuthStore().signOut()` for logout behavior
- `AUTH_SESSION_CLEARED_EVENT` keeps Zustand auth state in sync when refresh flow fails

Profile endpoint convention:

- `GET /api/auth/me`

Do not reintroduce obsolete profile endpoint usage.

Zustand rule:

- Zustand is required for cross-route auth/session state
- Page-local UI state such as form fields, search inputs, dropdown visibility, and loading booleans should remain local component state unless there is a proven cross-screen need
- Server cache/state should prefer React Query over Zustand when data becomes backend-driven

## 7) Data Integrity in UI

- Use `--` fallback for missing fields.
- Do not fabricate business values/charts/tables.
- Keep placeholders explicit until backend integration is complete.

## 8) Required Update Triggers for Docs Repo

When any of the following changes, update docs under `AI_Stock_Trend_Prediction_Docs/FE`:

- Shared component architecture
- Route conventions (especially role-based)
- Auth/session behavior
- Zustand store responsibilities
- Styling/token conventions
- Cross-layout reusable patterns

## 9) Delivery Checklist

1. Reuse existing component/helper first
2. Prefer shadcn primitive where applicable
3. Keep CSS token-based and DESIGN.md-aligned
4. Keep role routing consistent
5. Run `npm run build` in `web`
6. Update FE docs when architecture/conventions changed
