# Webhook event resend across all surfaces

**Released:** 2026-05-19

The dashboard's "Reenviar" button is now exposed in the API, SDK, CLI, and MCP server.
Same backend behavior (`POST /api/webhook-events/:id/retry`) — now reachable with a
seller API key, so automation, scripts, and AI agents can resend a webhook without
JWT/dashboard access.

## Added

### gateway — v0.10.3

- `GET /api/webhook-events` — list webhook events with filters (`status`,
  `eventType`, `endpointId`, `page`, `limit`).
- `GET /api/webhook-events/:id` — fetch a single event (full payload, attempt
  count, last response status/body, next retry timestamp).
- `POST /api/webhook-events/:id/retry` — reset event to `pending` and trigger
  immediate redelivery. **Rate-limited at 20 req/min per IP.**
- All three endpoints accept seller API key (`sk_test_…` / `sk_live_…`) **or**
  dashboard JWT. Previously only the dashboard JWT path was authenticated.

### @garuhq/node — 0.11.0

- New resource `garu.webhookEvents` with `list(params)`, `get(id)`, `retry(id)`.
- New exported types: `WebhookEvent`, `WebhookEventStatus`,
  `ListWebhookEventsParams`, `WebhookEventList`.
- Status enum is `pending | success | failed` (not `delivered`).

### @garuhq/cli — 0.4.0

- New command group `garu webhooks events`:
  - `garu webhooks events list [--status <status>] [--event-type <type>] [--endpoint-id <id>] [--limit <n>] [--page <n>]`
  - `garu webhooks events get <id>`
  - `garu webhooks events retry <id>`
- TTY rendering uses status badges (`pending` / `success` / `failed`); `--json`
  emits structured output for scripting.

### @garuhq/mcp — 0.11.0

- Three new tools:
  - `list_webhook_events` — list with filters (status, event type, endpoint).
  - `get_webhook_event` — fetch a single event.
  - `retry_webhook_event` — resend an event by ID; works on any current status.
- Server instructions updated to clarify that webhook *endpoint* creation
  remains dashboard-only, but webhook *event* listing and retry are now
  available to agents.

## Docs

- New guide: [Reenviar webhook](/guias/reenviar-webhook) — listar → escolher →
  reenviar via cURL, SDK, CLI, MCP.
- New API reference pages under `/api-reference/webhooks/eventos/`:
  `listar`, `detalhes`, `reenviar`.

## Notes

- Webhook events are retained for 30 days. For longer-term auditing, persist
  webhooks on your side as you receive them.
- The event `id` (`evt_…`) is preserved on retry. Receivers with proper
  idempotency will treat the redelivery as a duplicate — which is the correct
  behavior.
- Existing dashboard "Reenviar" flow is unchanged (JWT path was not touched).
