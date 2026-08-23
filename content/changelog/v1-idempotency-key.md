# Idempotency-Key now enforced on three more endpoints

**Released:** 2026-08-22

`X-Idempotency-Key` already protected `products`, `charges`, and
`installment-plans` creation. It now also protects:

| Endpoint | What a retry could do before this release |
|---|---|
| `POST /api/v1/scheduled-charges` | Double-book recurring billing for the same customer |
| `POST /api/v1/customers` | Register a duplicate customer record |
| `POST /api/v1/installment-plans/{uuid}/refund-requests` | Open a second refund request (already guarded by a business-level dedup on a pending request for the same carnê — this closes the request-in-flight window) |
| `POST /api/v1/charges/{uuid}/refund` | Same, for the Pix/boleto manual refund-request path. Ignored for card, which reverses automatically. |

## The contract

Same shape as the existing `products`/`charges`/`installment-plans` behavior:
pass `X-Idempotency-Key: <your-key>`, and a retry within 24h with the same
key (scoped to your account) returns the original result instead of
creating a duplicate. Omit it and the SDK auto-generates a UUIDv4 per call.

## Migration

Nothing breaks. The header was already accepted (and, for `scheduled-charges`,
already sent by the SDK) — it just wasn't enforced by the gateway on these
four call sites. No request/response shape changed.

### SDK, CLI, MCP

- **`@garuhq/node` 4.1.0**: `customers.create()`, `charges.refund()`, and
  `installmentPlans.requestRefund()` now attach the header automatically
  (new optional `idempotencyKey` param on each). `scheduledCharges.create()`'s
  docstring dropped the "gateway does not deduplicate" caveat.
- **`@garuhq/cli` 0.13.0**: `--idempotency-key <key>` added to
  `garu customers create`, `garu charges refund`, and
  `garu installment-plans request-refund`.
- **`@garuhq/mcp` 0.23.0**: no schema changed; `create_customer`,
  `refund_charge`, and `request_plan_refund` tool descriptions now say they're
  safe to retry.
