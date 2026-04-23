# Codebase Analysis (April 23, 2026)

## 1) High-level architecture

- **Framework/runtime:** Next.js App Router application using React 18 and TypeScript with strict mode enabled.
- **Primary responsibilities:** This repo is an **admin panel + API backend** for a tailoring/e-commerce system (stores, catalog, orders, measurements, design collections).
- **Data layer:** Prisma with a **MongoDB datasource**.
- **Auth stack:** Mixed approach using Clerk (session + middleware) and custom email/password JWT endpoints.
- **UI stack:** Tailwind CSS + Radix UI primitives + client/server component split.

## 2) Code organization

- `app/`: route groups for root, auth, and dashboard UI plus route handlers under `app/api/*`.
- `components/`: reusable UI and feature components (navbar, table clients, modals).
- `lib/`: shared integrations (Prisma, auth helpers, Redis, Razorpay utilities).
- `prisma/schema.prisma`: core domain model and relations.
- `actions/`: server-side data-fetch helpers for dashboard metrics.

Overall structure is conventional for App Router and mostly easy to navigate.

## 3) Data model assessment

The schema models are rich and cover:

- multi-tenant stores
- catalog attributes (sizes/colors/categories)
- products and images
- orders and order-items
- customer profile and cart
- design collections/items/variations
- wallet + wallet transactions
- measurements and recent-work showcase

Strengths:

- Good use of explicit relation names in many places.
- Cascading deletes defined where orphan risk is high (`DesignItem`, `DesignVariation`, `Image`, `Measurement`).
- Useful indexing on common FK paths.

Risks / concerns:

- The schema mixes admin and storefront concerns in one DB model without obvious bounded-context boundaries; still valid, but complexity is rising.
- `DesignCollection.slug` is globally unique, not store-scoped; if you expect identical slugs per store, this will block it.

## 4) Authentication and authorization findings

### What looks good

- Middleware protects non-public routes via Clerk.
- API and UI route groups are clearly separated.

### Critical findings

1. **Authorization gap in dashboard store layout:**
   - Dashboard store lookup checks `id` only, not ownership by current `userId`.
   - A signed-in user could potentially access another store context by URL if downstream routes also miss ownership checks.

2. **Mixed auth systems (Clerk + custom JWT):**
   - `app/api/auth/signin` and `app/api/auth/signup` create JWT sessions independent of Clerk.
   - This is likely intentional during migration, but it introduces two parallel trust models and potential policy drift.

3. **Per-request PrismaClient in auth routes:**
   - `new PrismaClient()` is instantiated in route files and disconnected manually.
   - Elsewhere you correctly use the singleton (`lib/prismadb.ts`).
   - This inconsistency can cause connection churn under load.

## 5) API surface review

- CRUD coverage is broad and consistent for store-scoped resources.
- Naming consistency is generally good.

Risks:

- Some endpoints use weak input validation; many handlers deserialize request JSON directly without schema validation.
- Error payload shapes vary route to route (plain text vs JSON), which can complicate client handling.

## 6) Tooling / operational findings

- `npm run lint` currently fails due incompatibility between `next lint` wrapper and supplied ESLint options.
- This blocks basic CI confidence and should be addressed quickly.

## 7) Prioritized recommendations

### P0 (security/reliability)

1. Enforce **store ownership checks** in dashboard layout and every mutating/read API route by `storeId + userId`.
2. Decide on one primary auth strategy (Clerk-only or explicit hybrid), then document and enforce token boundaries.
3. Replace per-route Prisma client creation with `lib/prismadb` singleton everywhere.

### P1 (maintainability)

4. Add Zod validation for request bodies in all write endpoints.
5. Standardize API error response contracts (`{ code, message, details? }`).
6. Add service-layer helpers for repeated store authorization logic.

### P2 (developer experience)

7. Migrate from `next lint` to direct ESLint CLI as recommended by Next.js 15/16 migration path.
8. Add architecture docs (auth flow, API conventions, data ownership model).

## 8) Suggested quick wins (1-2 days)

- Fix dashboard layout ownership check.
- Refactor `signin/signup` handlers to use `lib/prismadb`.
- Introduce a shared `assertStoreAccess(userId, storeId)` utility.
- Repair lint pipeline and add it to CI.

