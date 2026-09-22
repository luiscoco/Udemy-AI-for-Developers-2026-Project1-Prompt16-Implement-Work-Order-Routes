# Prompt 16 — Implement Work-Order Routes

This README documents, step by step, what was done to satisfy the latest prompt:

> Create `apps/backend/src/routes/workOrders.ts` and register these routes:
> - `GET /api/work-orders` with optional `state` and `priority` filters, validated against `STATES` and `PRIORITIES`.
> - `GET /api/work-orders/:id`.
> - `POST /api/work-orders` creating a new reported work order with `nextWorkOrderReference()`, null `technicianId`, and timestamps.
> - `POST /api/work-orders/:id/transitions` validating `ACTIONS` then calling `transition()`.
>
> Return 400 for invalid filters/payload/action, 404 for missing work orders/assets, and 409 for illegal lifecycle transitions. Use `ApiError`-shaped responses consistently.

The goal of this document is to walk through the *process*, not just the result, so you can reproduce the same approach on similar tasks.

## 1. Explore before writing any code

Before touching anything, the existing backend structure was read to understand established conventions:

- [`apps/backend/src/app.ts`](apps/backend/src/app.ts) — how the Fastify app is built and how existing routes (`/api/health`, `/api/assets`, `/api/technicians`, `/api/dashboard/summary`) are registered.
- [`apps/backend/src/data/repository.ts`](apps/backend/src/data/repository.ts) — the `Repository` interface, including `listWorkOrders(filters)`, `getWorkOrder(id)`, `saveWorkOrder(workOrder)`, `assetExists(id)`, and `references()`.
- [`apps/backend/src/domain/workOrderLifecycle.ts`](apps/backend/src/domain/workOrderLifecycle.ts) — the `transition(state, action)` state machine and its `{ ok: true, state } | { ok: false, reason }` result shape.
- [`apps/backend/src/domain/reference.ts`](apps/backend/src/domain/reference.ts) — `nextWorkOrderReference(existingReferences, year)` for generating `WO-YYYY-NNNN` references.
- [`packages/contract/src/index.ts`](packages/contract/src/index.ts) and [`types.gen.ts`](packages/contract/src/types.gen.ts) — the shared `@equipment-hub/contract` package exporting `STATES`, `PRIORITIES`, `ACTIONS`, and the `WorkOrder`, `NewWorkOrder`, `TransitionCommand`, and `ApiError` types generated from the OpenAPI spec.
- [`apps/backend/src/app.test.ts`](apps/backend/src/app.test.ts) — existing test patterns, including how a fake `Repository` is built for tests.

**Lesson:** always read the domain layer, the shared contract package, and existing tests first. The routes file should be a thin adapter over logic that already exists (`transition`, `nextWorkOrderReference`, `Repository`) rather than reimplementing anything.

## 2. Implement the routes file

Created [`apps/backend/src/routes/workOrders.ts`](apps/backend/src/routes/workOrders.ts) exporting `registerWorkOrderRoutes(app, repository)`, with one handler per endpoint:

- **`GET /api/work-orders`** — reads `state`/`priority` from the query string, validates each against `STATES`/`PRIORITIES` (returning 400 with an `ApiError` body on failure), then builds a `WorkOrderFilters` object and delegates to `repository.listWorkOrders(filters)`.
- **`GET /api/work-orders/:id`** — looks up the work order via `repository.getWorkOrder(id)`; 404 if not found.
- **`POST /api/work-orders`** — validates the required fields (`assetId`, `title`, `description`, `priority`) and that the referenced asset exists (`repository.assetExists`), returning 400 on any failure. On success it builds a new `WorkOrder` with `state: "reported"`, `technicianId: null`, a reference from `nextWorkOrderReference(repository.references(), currentYear)`, and matching `reportedAt`/`updatedAt` timestamps, persists it with `repository.saveWorkOrder`, and responds `201`.
- **`POST /api/work-orders/:id/transitions`** — validates the `action` against `ACTIONS` (400), looks up the work order (404), calls the existing `transition(state, action)` function, returns `409` with the function's `reason` when the transition is illegal, otherwise updates the work order's `state`/`updatedAt` and saves it.

A small `errorBody(message, details?)` helper was added to consistently produce the `ApiError` shape (`{ message, details? }`) used by every error response.

## 3. Wire the routes into the app

[`apps/backend/src/app.ts`](apps/backend/src/app.ts) was updated to import `registerWorkOrderRoutes` and call it with the app instance and repository, right after the existing route registrations — following the same pattern already used for `buildDashboardSummary`.

## 4. Verify the change

Because this workspace's `node_modules` hadn't been installed yet, `npm install` was run at the repo root first. Then, from `apps/backend`:

1. `npx vitest run` — confirmed all 57 existing tests (across `reference.test.ts`, `dashboard.test.ts`, `workOrderLifecycle.test.ts`, `app.test.ts`) still pass with the new routes registered.
2. `npx tsc --noEmit -p tsconfig.json` — confirmed the new file type-checks cleanly under the workspace's strict TS config (`strict`, `exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`, etc.). This caught two real type errors that had to be fixed:
   - Building the `WorkOrderFilters` object conditionally (only setting `state`/`priority` keys when defined) instead of assigning `undefined` directly, to satisfy `exactOptionalPropertyTypes`.
   - Narrowing `body.priority` to the `Priority` type only *after* validating it against `PRIORITIES`, instead of trusting the loosely-typed request body.
3. `npx eslint` on the touched files — no lint errors.

**Lesson:** running the type checker and test suite isn't optional polish — it caught genuine bugs (the `exactOptionalPropertyTypes` issue would have been a real runtime footgun if filters were passed with explicit `undefined` values).

## Files changed

- `apps/backend/src/routes/workOrders.ts` (new)
- `apps/backend/src/app.ts` (registers the new routes)

## Try it yourself

From `apps/backend`, after `npm install` at the repo root:

```bash
npx vitest run
npx tsc --noEmit -p tsconfig.json
```

Both should pass with no changes needed.

## Running the app (Windows terminal / PowerShell)

The backend has no bundler yet, so it's built with `tsc` and then run with plain Node from the compiled `dist/` output. From the **repo root**, in PowerShell:

```powershell
# 1. Install dependencies (only needed once, or after pulling new changes)
npm install

# 2. Move into the backend app
cd "apps\backend"

# 3. Compile TypeScript to dist/
npx tsc -p tsconfig.json

# 4. Start the server
node dist/server.js
```

The server listens on `http://127.0.0.1:3001` (override with `$env:PORT = "4000"` before step 4 if needed). Verify it's up in another terminal:

```powershell
curl.exe http://127.0.0.1:3001/api/health
curl.exe http://127.0.0.1:3001/api/work-orders
```

Press `Ctrl+C` in the server's terminal to stop it. `dist/` is git-ignored build output — safe to delete and regenerate with step 3 at any time.
