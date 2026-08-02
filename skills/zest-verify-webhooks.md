---
name: Verify and handle Zest webhooks
description: Authenticate inbound Zest webhook deliveries with HMAC-SHA256 and process the nine lifecycle events idempotently.
api: openapi/zest-openapi-original.json
operations: [getSpvRequest]
---

# Verify and handle Zest webhooks

Zest delivers nine lifecycle events to your single registered HTTPS endpoint. Every delivery uses one envelope: `{ eventId, eventType, occurredAt, data }`.

## Verify every delivery
1. Read the `Zest-Signature` header, format `t=<unix>,v1=<hex>`.
2. Reject if `|now - t| > 300` (5-minute replay window).
3. Compute `expected = hmac_sha256(secret, "<t>.<raw_body>")` using the plaintext `whsec_<hex>` signing secret Zest issued you.
4. Compare `v1` to `expected` in **constant time** (`hmac.compare_digest` / `crypto.timingSafeEqual`). Reject on mismatch. Ignore unknown `vN` tokens.

## Handle idempotently
5. Respond `200` **before** heavy processing — handlers that exceed the 10s timeout trigger spurious retries.
6. Dedup on `eventId` (persist with ≥24h TTL); delivery is at-least-once and the retry chain spans ~15h. Order by `occurredAt`, not receipt order.
7. Dispatch on `eventType`: `spv_request.created|completed|rejected|cancelled`, `investor.created`, `subscription.created|completed`, `signed_subscription_form.uploaded`, `funding_receipt.uploaded`. For state you need to confirm, call **`getSpvRequest`** (`GET /v1/spv-requests/{slug}`).

## Rules
- Retry schedule: 30s, 5m, 30m, 2h, 12h, then dead-letter (retained for replay).
- Failure = any status ≥ 300, TCP/TLS failure, or no response within 10s.
