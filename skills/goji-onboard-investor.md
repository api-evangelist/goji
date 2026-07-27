---
name: Onboard and KYC an investor with Goji
description: >-
  Create an investor on the Goji Platform, run KYC/KYB, and open an IF ISA
  account, using HMAC-signed requests and safe idempotent retries.
api: Goji Platform API (https://docs.api.goji.investments/)
operations:
  - "GET /platformApi/terms"
  - "POST /platformApi/investors"
  - "GET /platformApi/investors/{clientId}"
  - "GET /platformApi/investors/{clientId}/kyc"
  - "POST /platformApi/investors/{clientId}/kyc/documents"
  - "POST /platformApi/investors/{clientId}/accounts/ISA"
grounding: >-
  Goji publishes no OpenAPI; the operations above are the REST method+path pairs
  documented in the Goji API reference. Do not invent endpoints beyond these.
---

# Onboard and KYC an investor with Goji

Use this flow to bring a new investor onto the Goji Platform and get them ready
to invest in private-market funds.

## Auth and conventions (read first)
- **Production** requests must be signed with **HMAC-SHA256**: send `Authorization`
  (the Base64 + URL-encoded signature), plus `x-nonce` (unique per request) and
  `x-timestamp` (ms since epoch) to prevent replay. See
  `authentication/goji-authentication.yml`.
- **Sandbox** (`https://api-sandbox.goji.investments`) accepts a Basic HTTP API
  key and password for prototyping — use it first.
- Set a unique **`X-CLIENT-REQUEST-ID`** on every write so retries are idempotent
  and never create a duplicate investor or payment.
- Read **`X-GOJI-REQUEST-ID`** from each response and log it for support.
- All dates are ISO 8601. Select the API version with the `version` header.

## Steps
1. **Fetch the hosted Terms** — `GET /platformApi/terms`. Present them to the
   investor and record acceptance.
2. **Create the investor** — `POST /platformApi/investors` with a fresh
   `X-CLIENT-REQUEST-ID`. Save the returned **`clientId`**; every later call uses it.
3. **Check KYC/KYB** — `GET /platformApi/investors/{clientId}/kyc`. If the status
   is not yet verified, continue to step 4.
4. **Upload KYC documents** — `POST /platformApi/investors/{clientId}/kyc/documents`
   for any documents the check requests. Then subscribe to the
   `INVESTOR_KYC_STATUS_CHANGE` webhook rather than polling.
5. **Confirm the investor** — `GET /platformApi/investors/{clientId}` and proceed
   only once KYC is verified.
6. **Open an IF ISA (optional)** — `POST /platformApi/investors/{clientId}/accounts/ISA`
   for investors who want a tax-wrapped account. Watch for the
   `ISA_PROVISIONALLY_OPENED` webhook.

## Events to subscribe to
`INVESTOR_CREATED`, `INVESTOR_KYC_STATUS_CHANGE`, `ISA_PROVISIONALLY_OPENED`.
See `asyncapi/goji-webhooks.yml`.

## Errors and retries
Retry idempotently by re-sending the same `X-CLIENT-REQUEST-ID`; a repeated
create returns the existing investor rather than a duplicate. Always capture
`X-GOJI-REQUEST-ID` when raising a support request.
