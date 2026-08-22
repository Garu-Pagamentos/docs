# Webhook Events API v1 — the versioned public contract

**Released:** 2026-08-22

Webhook events now live under the versioned public API at `/api/v1/webhook-events`. It's the canonical surface for API-key integrations: uuid-keyed, camelCase, and consistent with the rest of the Garu v1 API and SDK. The previous un-versioned `/api/webhook-events/*` paths kept working but were the dashboard's internal API — numeric-id keyed and never meant as a public contract.

If you integrated against the docs before this release, see **Migration** below — nothing breaks, but the recommended paths and identifiers changed.

## Added

### gateway — v0.21.0

Full read + resend/retry surface under `/api/v1/webhook-events`, all authenticated with your API key (`sk_test_` / `sk_live_`):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/webhook-events` | List your webhook events (`status`, `eventType`, `endpointId`, `page`, `limit`) |
| `GET` | `/api/v1/webhook-events/{uuid}` | Retrieve one event (payload, attempts, last response) |
| `POST` | `/api/v1/webhook-events/{uuid}/resend` | Clone the event and deliver the clone (preferred) |
| `POST` | `/api/v1/webhook-events/{uuid}/retry` | Reset the original to `pending` and redeliver in place (legacy) |

## The contract

- **UUID-keyed.** Every path uses the event `uuid`, never the numeric id used internally. `webhookEndpoint.id` stays numeric — endpoint configuration (URL, subscribed events, secret) is still dashboard-only and didn't move to v1.
- **camelCase** request and response bodies (`eventType`, `responseStatus`, `responseBody`, `lastAttemptAt`, `nextRetryAt`, `manualResendOf`) — the same vocabulary as the SDK.
- **List envelope**: `{ "data": [...], "count", "totalCount", "totalPages" }`.
- **Single resources return a bare object** (no `{ data }` wrapper).
- **`/resend` and `/retry` return `201 Created`** with the resulting event (the clone for `/resend`, the reset original for `/retry`).
- **`manualResendOf` is a uuid** pointing back to the original event when the row is a resend clone; `null` on originals.

## Migration

Nothing breaks. The old `/api/webhook-events/*` paths remain as the internal/dashboard API and keep responding. To move onto the versioned contract:

| Before (interim docs) | Now |
|---|---|
| `GET /api/webhook-events` | `GET /api/v1/webhook-events` |
| `GET /api/webhook-events/{id}` | `GET /api/v1/webhook-events/{uuid}` |
| `POST /api/webhook-events/{id}/resend` | `POST /api/v1/webhook-events/{uuid}/resend` |
| `POST /api/webhook-events/{id}/retry` | `POST /api/v1/webhook-events/{uuid}/retry` |

Response bodies were already camelCase, so the main change is the path and using the `uuid` (not a numeric id) to address an event. `manualResendOf` now reports a uuid instead of a numeric id.

### SDK, CLI, MCP

Breaking change, same major/minor-bump precedent as the products and customers v1 migrations:

- **`@garuhq/node` 3.0.0**: `webhookEvents.get/resend/retry` now take a `uuid` string instead of a numeric `id`.
- **`@garuhq/cli` 0.11.0**: `garu webhooks events get/resend/retry <uuid>`.
- **`@garuhq/mcp` 0.21.0**: `get_webhook_event`, `resend_webhook_event`, `retry_webhook_event` tools now take `uuid`.
