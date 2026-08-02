---
name: Onboard investors and record a Zest subscription
description: Bulk-create investors, create a subscription against a materialised SPV, then upload the signed form and funding receipt in strict order.
api: openapi/zest-openapi-original.json
operations: [bulkCreateInvestors, createSubscriptions, uploadSubscriptionForm, uploadFundingReceipt]
---

# Onboard investors and record a Zest subscription

Run this after an SPV has been materialised (see the create-SPV-request skill). Authenticate first (bearer token from `exchangeJwtAssertion`).

## Onboard investors
1. Call **`bulkCreateInvestors`** — `POST /v1/investors` with rows keyed by your `partnerInvestorId` (fields: email, firstName, lastName, dateOfBirth, countryOfResidence). Always returns `200` with a per-row `{ status, zestPersonId | error }`. Partial success: bad rows do not fail the batch. Idempotent — a repeated `partnerInvestorId` returns the existing `zestPersonId`. `Idempotency-Key` is optional here.

## Create the subscription
2. Call **`createSubscriptions`** — `POST /v1/spvs/{spv_slug}/subscriptions` with rows carrying `partnerSubscriptionId`, `zestPersonId`, `lumpSum`, and `shareClassSlug`. Fires `subscription.created`.

## Upload documents — STRICT ORDER
3. Call **`uploadSubscriptionForm`** — `POST /v1/spvs/{spv_slug}/subscription/{person_id}/forms` first. `multipart/form-data`, single `file` field, max 10 MB, content type PDF/JPEG/PNG/WEBP. Fires `signed_subscription_form.uploaded`.
4. Then call **`uploadFundingReceipt`** — `POST /v1/spvs/{spv_slug}/subscription/{person_id}/fundings` with the wire receipt + `amount`/`currency`/`wire_reference`. **The signed form must already be on record**, otherwise you get `409 conflict`. Fires `funding_receipt.uploaded`.

## Rules
- Uploads are content-addressed, so they do not use `Idempotency-Key`.
- `413 payload_too_large` (>10 MB) and `415 unsupported_media_type` (outside the allow list) are the upload-specific errors.
- Wait for `subscription.completed` (admin transitions the bid to Completed; a 60s reconciler backstops delivery).
