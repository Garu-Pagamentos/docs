# Webhook event resend (clone-on-resend) + Idempotency-Key contract change

**Released:** 2026-05-19

A second webhook resend endpoint ships alongside an **important contract change** in the outbound `Idempotency-Key` header. If your webhook handler deduplicates by `Idempotency-Key` (the pattern Garu recommends), **you need to update it** — manual resends now carry a `resend_*` prefix that must be treated as a fresh delivery, not a duplicate of the original.

## Breaking-ish change — `Idempotency-Key` on manual resends

Every outbound webhook POST from Garu carries an `Idempotency-Key` header. The value changes depending on the origin of the delivery:

| Origin | `Idempotency-Key` value |
|---|---|
| First delivery of an event | `evt_<hash>` (= `payload.id`) |
| Auto-retry from the retry policy | `evt_<hash>` (same as first delivery) |
| **Manual resend via `/resend` or dashboard "Reenviar"** | **`resend_<numeric_event_id>`** |
| Manual resend via legacy `/retry` | `evt_<hash>` (unchanged) |

**Handler impact** depends on what your dedup keys on:

- **Dedup keyed on the raw `Idempotency-Key` header value** (the recommended pattern): no code change needed. `resend_<id>` is a distinct string from `evt_<hash>`, so your existing cache check naturally treats it as a fresh delivery. The catch: pre-v0.11.0 your dedup was silently *dropping* manual resends because the key was identical to the original — you'll now start seeing those deliveries arrive. **Business-level idempotency (keying on `orderId`, `transactionId`, etc.) must absorb the increased delivery count.**
- **Dedup keyed on `payload.id`** (which is identical between original and clone): **must be updated**. Either switch the dedup key to the raw header value, or allow-list keys starting with `resend_` to bypass dedup. See [Reenviar webhook → Mudança no Idempotency-Key](/guias/reenviar-webhook#mudanca-no-idempotency-key) for both fixes side-by-side.
- **Dedup with header normalization** (stripping prefixes, regex truncation, etc.): may hide the `evt_…` vs `resend_…` distinction. Audit and update.

The rationale for the change: pre-v0.11.0, a manual resend reused the original's `Idempotency-Key`, which meant correctly-implemented receivers silently dropped it. Sending `resend_<id>` is the contract that lets a deliberate manual redelivery actually reach the receiver. Business-level idempotency (don't credit the same order twice) stays the receiver's job and should key off a domain identifier (`orderId`, `transactionId`), not the HTTP header.

## Added

### gateway — v0.11.0

- `POST /api/webhook-events/:id/resend` — **clone-on-resend semantics**. Creates a new event row carrying `manualResendOf = <original_id>`, leaves the original at its terminal status forever with full response audit preserved, and immediately delivers the clone. Works on any source status (`success`, `failed`, `pending`). Rate-limited 20 req/min per IP, same as `/retry`.
- New field on the webhook event resource: `manualResendOf: number | null`.
- Dashboard: the "Reenviar" button is now visible on **every** event row (was hidden on `success` / `pending`). Resending a `success` event prompts for confirmation; clones are visually tagged with a 🔁 badge in the events table and a "Reenviado a partir de #X" row in the detail drawer.

### Changed

- Outbound `Idempotency-Key` header behavior — see breaking section above.
- Frontend attempts counter display fixed: previously read `attempts/5`, now correctly reads `attempts/6` (the backend's `MAX_ATTEMPTS` was 6 all along — display bug).

## Why two endpoints (`/resend` vs `/retry`)

`/retry` (v0.10.3) was the first cut: it reset the original event to `pending`, zeroed the attempt counter, and reused the same delivery row. That's fine for a one-off support fix but two things hurt at scale:

1. **No audit trail.** Each `/retry` overwrites the prior tentativa history. If a support agent retries an event three times, you have no record of the first two outcomes.
2. **Same `Idempotency-Key`.** The outbound retry carried `payload.id`, indistinguishable from a normal auto-retry — so receiver-side header dedupe couldn't tell a deliberate manual resend from a duplicate.

`/resend` fixes both by cloning instead of mutating. Use `/resend` for everything from now on; `/retry` stays alive for existing automation.

## Docs

- Guide rewritten: [Reenviar webhook](/guias/reenviar-webhook) — covers `/resend` vs `/retry`, the `Idempotency-Key` change with before/after handler code, dashboard flow, and rate limits.
- New API reference page: [`POST /webhook-events/{id}/resend`](/api-reference/webhooks/eventos/resend).
- Existing API reference page [`POST /webhook-events/{id}/retry`](/api-reference/webhooks/eventos/reenviar) updated with a deprecation-style note pointing to `/resend`.
- [Event details](/api-reference/webhooks/eventos/detalhes) response now documents the `manualResendOf` field.

## Not yet shipped

SDK, CLI, and MCP wrappers for `/resend` are not in this release — they ship in upcoming versions of `@garuhq/node`, `@garuhq/cli`, and `@garuhq/mcp`. For now, call `/resend` via direct HTTP (`curl` / `fetch`). The dashboard "Reenviar" button already uses `/resend` under the hood.

## Notes

- `/retry` is **not deprecated** in this release — it remains documented and supported. Migration to `/resend` is recommended, not forced.
- Webhook events retention is still 30 days; clones count as separate events for the retention window (each clone starts its own 30-day clock from `createdAt`).
- The `payload.id` (`evt_…`) of a clone is **identical** to the original — only the outbound `Idempotency-Key` HTTP header differs.
