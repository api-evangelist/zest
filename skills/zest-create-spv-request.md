---
name: Create a Zest SPV request
description: Authenticate as a partner and submit a contract-validated SPV creation request, then track it to materialisation via webhooks.
api: openapi/zest-openapi-original.json
operations: [exchangeJwtAssertion, list_contract_versions_v1_contracts_get, list_templates_v1_contracts_templates__version__get, createSpvRequest, getSpvRequest]
---

# Create a Zest SPV request

Use the Zest Public API to submit an SPV creation request as a partner.

## Auth
1. Mint a JWT assertion signed with your registered EdDSA private key (claims: `iss`, `sub`=client_id, `aud`=`https://sandbox-api.zestequity.com`, `client_id`, `iat`, `exp`=iat+60).
2. Call **`exchangeJwtAssertion`** — `POST /v1/oauth2/tokens` with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer` and `assertion=<JWT>`. Read `accessToken` from the response and send it as `Authorization: Bearer <token>` on every subsequent call.

## Pick a template
3. Call **`list_contract_versions_v1_contracts_get`** (`GET /v1/contracts`) to find the current contract version.
4. Call **`list_templates_v1_contracts_templates__version__get`** (`GET /v1/contracts/templates/{version}`) to find your assigned `templateId`.

## Submit
5. Call **`createSpvRequest`** — `POST /v1/spv-requests` with `templateId`, `templateVersion`, and `attributes`. **`Idempotency-Key` header is REQUIRED here** — generate a v4 UUID; a missing key returns `400 invalid_request`. On success you get `201` and Zest fires `spv_request.created`.
6. Poll **`getSpvRequest`** (`GET /v1/spv-requests/{slug}`) or wait for the `spv_request.completed` webhook (admin approval materialises the SPV). Terminal states: approved, rejected, cancelled.

## Rules
- Fields are camelCase JSON. On any non-2xx, parse the top-level `code`, show `detail`, and cite `errorId` to support. `validation_error` carries `validationErrors[]` with per-attribute codes.
- Same Idempotency-Key + different body within 24h returns `409 conflict`.
