# Scheduled Charges API v1 — the versioned public contract

**Released:** 2026-08-22

Scheduled charges now live under the versioned public API at
`/api/v1/scheduled-charges`. It's the canonical surface for API-key
integrations, consistent with the rest of the Garu v1 API and SDK. The
previous un-versioned `/api/scheduled-charges/*` paths kept working but
were the dashboard's internal API and never meant as a public contract.

Unlike products, customers, and webhook events, `scheduled_charge.id` was
already a stable, non-enumerable string (`sch_...`) before this move — there
is no separate uuid and no identifier change here, only the path and the
list envelope.

If you integrated against the docs before this release, see **Migration**
below — nothing breaks, but the recommended path and list-response shape
changed.

## Added

### gateway — v0.22.0 equivalent (SDK/CLI/MCP), same-day gateway release

Full read/write surface under `/api/v1/scheduled-charges`, all authenticated
with your API key (`sk_test_` / `sk_live_`):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/scheduled-charges/available-methods` | Payment methods this seller can offer |
| `GET` | `/api/v1/scheduled-charges` | List (status, type, customerId, due-date range, search) |
| `GET` | `/api/v1/scheduled-charges/{id}` | Detail bundle: charge + event timeline + linked transactions |
| `GET` | `/api/v1/scheduled-charges/{id}/attempts` | Per-attempt billing log |
| `POST` | `/api/v1/scheduled-charges` | Create (one-time or recurring) |
| `POST` | `/api/v1/scheduled-charges/{id}/charge-now` | Force-bill the current cycle immediately |
| `POST` | `/api/v1/scheduled-charges/{id}/postpone` | Move to a new due date |
| `POST` | `/api/v1/scheduled-charges/{id}/pause` / `/resume` | Suspend / re-enable |
| `POST` | `/api/v1/scheduled-charges/{id}/cancel-recurrence` | Hard-stop future cycles (recurring only) |
| `POST` | `/api/v1/scheduled-charges/{id}/cancel-at-period-end` | Stripe-style soft cancel; reversible |
| `POST` | `/api/v1/scheduled-charges/{id}/mark-paid` | Manually mark paid (off-Garu reconciliation) |
| `POST` / `DELETE` | `/api/v1/scheduled-charges/{id}/payment-method` | Swap / clear the saved card (recurring only) |

## The contract

- **`id` is unchanged.** It was already a stable, opaque string (`sch_...`) — no migration, no new field.
- **camelCase** request and response bodies — the same vocabulary as the SDK.
- **List envelope**: `{ "data": [...], "count", "totalCount", "totalPages" }`.
- **Single resources return a bare object** (no `{ data }` wrapper), except `GET /{id}`, which returns the deliberate `{ charge, events, transactions }` bundle that already existed pre-v1 — that shape is unchanged.
- **All money fields are decimal BRL** (`297.50`), including `transactions[].value` in the detail bundle — never centavos. (A stale docstring/description in the SDK and MCP tool claimed `transactions[].value` was centavos; it never was, and both were corrected the same day as this release.)
- **`mark-paid` always returns the parent scheduled charge**, even for a recurring series with `cycleNumber` set (the underlying mutation there returns the cycle row internally; v1 re-fetches the parent for a consistent response shape).

## Migration

Nothing breaks. The old `/api/scheduled-charges/*` paths remain as the internal/dashboard API and keep responding. To move onto the versioned contract:

| Before (interim docs) | Now |
|---|---|
| `GET /api/scheduled-charges` | `GET /api/v1/scheduled-charges` |
| `GET /api/scheduled-charges/{id}` | `GET /api/v1/scheduled-charges/{id}` |
| `POST /api/scheduled-charges` | `POST /api/v1/scheduled-charges` |
| `POST /api/scheduled-charges/{id}/{action}` | `POST /api/v1/scheduled-charges/{id}/{action}` |

The only response shape that changed is the list envelope: `meta.page`/`meta.total`/`meta.totalPages` became `count`/`totalCount`/`totalPages`, matching every other `/api/v1` list endpoint.

### SDK, CLI, MCP

Breaking change (list envelope only), same precedent as prior v1 migrations:

- **`@garuhq/node` 4.0.0**: `scheduledCharges.list()`/`.listAttempts()` return `{ data, count, totalCount, totalPages }` instead of `{ data, meta }`. No method signatures changed.
- **`@garuhq/cli` 0.12.0**: `garu scheduled-charges list`/`attempts` pretty-print output updated to match; no flags changed.
- **`@garuhq/mcp` 0.22.0**: no tool schema changed.
