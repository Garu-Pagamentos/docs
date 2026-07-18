# Products API v1 — the versioned public contract

**Released:** 2026-07-18

Products now have a proper **versioned public API** at `/api/v1/products`. It's the canonical surface for API-key integrations: conventional REST, keyed on the product **UUID**, camelCase throughout, and consistent with the rest of the Garu API and SDK. The previous un-versioned `/api/products/*` paths kept working but were the dashboard's internal API — inconsistent (`/products/seller` for listing, no bare list route) and never meant as a public contract.

If you integrated against the docs before this release, see **Migration** below — nothing breaks, but the recommended paths changed.

## Added

### gateway — v0.15.0

Full CRUD + payment-page config under `/api/v1/products`, all authenticated with your API key (`sk_test_` / `sk_live_`):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/products` | List your products (`page`, `limit`, `search`) |
| `GET` | `/api/v1/products/{uuid}` | Retrieve one product |
| `POST` | `/api/v1/products` | Create a product |
| `PATCH` | `/api/v1/products/{uuid}` | Update a product (partial) |
| `DELETE` | `/api/v1/products/{uuid}` | Deactivate a product |
| `GET/POST/PATCH/DELETE` | `/api/v1/products/{uuid}/portal-config` | Payment-page theming |

## The contract

- **UUID-keyed.** Every path uses the product `uuid` (the stable, non-enumerable identifier), never a sequential integer id. The numeric id is no longer returned in responses.
- **camelCase** request and response bodies (`creditCard`, `isSubscription`, `returnUrl`, …) — the same vocabulary as the SDK.
- **`installments` is the per-parcela breakdown**, not a count: `[{ "quantity": 1, "value": 297.00 }, { "quantity": 2, "value": 151.20 }]`. The per-installment `value` already includes the `fator` financing markup — the exact amount the customer is charged.
- **`value` is a JSON number** in BRL (e.g. `297.00`).
- **List envelope**: `{ "data": [...], "count", "totalCount", "totalPages" }`.
- **Single resources return a bare object** (no `{ data }` wrapper).
- **Deactivation** (`DELETE`) returns the product with `isActive: false` (HTTP 200). `isActive` is **read-only** — there is no reactivation endpoint; create a new product if you need to sell it again.

## Migration

Nothing breaks. The old `/api/products/*` paths remain as the internal/dashboard API and keep responding. To move onto the versioned contract:

| Before (interim docs) | Now |
|---|---|
| `GET /api/products/seller` | `GET /api/v1/products` |
| `GET /api/products/uuid/{uuid}` | `GET /api/v1/products/{uuid}` *(now requires your API key)* |
| `POST /api/products` | `POST /api/v1/products` |
| `PATCH /api/products/{id}` | `PATCH /api/v1/products/{uuid}` |
| `DELETE /api/products/{id}` | `DELETE /api/v1/products/{uuid}` |
| `…/api/products/{id}/portal-config` | `…/api/v1/products/{uuid}/portal-config` |

Response bodies were already camelCase, so the main change is the path and using the `uuid` (not a numeric id) to address a product. The list response now reports `totalCount`/`totalPages` and each item no longer includes a numeric `id`.
