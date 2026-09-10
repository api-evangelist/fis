---
name: fis-onboard-a-business-connection
description: Onboard an end customer onto FIS Accounting Data as a Service — create the business, generate a Connect URL, wait for the connection to come online, and confirm the first sync landed.
api: FIS Accounting Data as a Service
generated: '2026-09-10'
method: generated
source: openapi/_original/fis-accounting-data-as-a-service-openapi.json + https://docs.railz.ai/
base_url: https://api.railz.ai
operations:
  - create businesses
  - generateUrl
  - connections
  - syncStatus
  - syncInfo
---

# Onboard a business connection

Nothing in this API can be read until a **connection** exists. A connection is an authorised
link between one business and one upstream accounting or banking system, and it is created by a
human clicking through the Connect widget — not by an API call. Every data operation takes the
resulting `connectionUuid`, so this flow is the gate on all 233 operations.

## Before you start

Mint an access token first. `POST` the token endpoint with HTTP Basic — `client_id` as the
username, `secret_key` as the password. The token is a JWT and is **valid for 60 minutes**; the
provider recommends minting a fresh one before each call rather than caching it. Send it as
`Authorization: Bearer <access_token>`.

Your key prefix decides the environment: `SB_` is sandbox, `ID_` is production. There is no
separate hostname — both hit `https://api.railz.ai`.

## Steps

1. **Create the business.** `POST /v2/businesses` (`create businesses`). This is the record the
   customer's books will hang off.

2. **Generate the Connect URL.** `POST /v2/businesses/{uuid}/generateUrl` (`generateUrl`). Hand
   the returned URL to the customer, or embed the widget with `@railzai/railz-connect`. They
   authenticate against their own accounting system inside that flow.

3. **Wait for the connection.** Poll `GET /v2/connections` (`connections`) filtered by the
   business, or subscribe to the `auth` and `connectionStatus` webhook events and skip the
   polling. Webhooks are configured in the Dashboard, not through this API, and a maximum of
   five event types can be registered per URL.

4. **Confirm the first sync.** A connection is not the same as data. `GET /v2/data/syncStatus`
   (`syncStatus`) tells you whether the initial pull completed; `GET /v2/data/syncInfo`
   (`syncInfo`) tells you when it last ran. Do not read business data before one of these says
   the sync landed — you will get an empty result and mistake it for an empty ledger.

## Rules that bite

- **No idempotency anywhere.** `POST /v2/businesses` accepts no `Idempotency-Key`. If the call
  times out you cannot safely retry it — check with `GET /v2/businesses` first, then decide.
- **Read the whole `message` array.** On a `400`, `error.message` is an **array** of per-field
  strings, not a string. Rendering it as a scalar loses every validation failure after the first.
- **`401` usually means the token aged out**, not that the key is wrong. Mint a new one and retry.
- **No rate-limit headers.** A `429` arrives with no `Retry-After` and no `RateLimit-*` header.
  Back off exponentially; the interval is your choice because the provider gives you nothing.
