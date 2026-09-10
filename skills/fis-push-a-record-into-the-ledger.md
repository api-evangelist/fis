---
name: fis-push-a-record-into-the-ledger
description: Write an invoice, bill, payment or account back into a connected customer's accounting system through FIS Accounting Data as a Service — and know exactly what can and cannot be taken back.
api: FIS Accounting Data as a Service
generated: '2026-09-10'
method: generated
source: openapi/_original/fis-accounting-data-as-a-service-openapi.json
base_url: https://api.railz.ai
operations:
  - pushOptions
  - push-invoices
  - push-bills
  - push-accounts
  - push-customers
  - push-invoices-payments
  - pushStatus
  - delete-invoices
  - delete-bills
---

# Push a record into the ledger

**Read this before writing anything.** These operations write into a real business's books
through a third-party accounting system. There is no dry-run, no preview, and no idempotency
key anywhere in this contract.

## Steps

1. **Ask what this connection accepts.** `GET /v2/data/pushOptions` (`pushOptions`). Push support
   varies by upstream system — not every connection accepts every record type, and finding out
   from a `400` is the expensive way.

2. **Push.** One of:
   - `POST /v2/accounting/invoices` (`push-invoices`)
   - `POST /v2/accounting/bills` (`push-bills`)
   - `POST /v2/accounting/accounts` (`push-accounts`)
   - `POST /v2/accounting/customers` (`push-customers`)
   - `POST /v2/accounting/invoices/payments` (`push-invoices-payments`)

3. **Confirm it landed.** Pushes are queued, not synchronous. `GET /v2/data/pushStatus`
   (`pushStatus`) is the only way to know the write reached the upstream system. A `202` means
   accepted for processing, not written.

## Retry rule — the one that costs money

**There is no `Idempotency-Key` on any of the 86 mutating operations in this API.**

So: if a push times out, or returns `500`/`502`/`503`/`504`, **do not blindly retry**. Call
`pushStatus` first and find out whether the original one landed. A blind retry writes a second
invoice into a customer's ledger, and nothing in the API will stop it or tell you it happened.

## What can be taken back

Reversal paths exist, but **the provider states no window for any of them** — no retention
period, no undo deadline, nothing. Treat every one as best-effort and verify the result.

| Wrote | Reverse with |
|---|---|
| `push-invoices` | `DELETE /v2/accounting/invoices/{id}` (`delete-invoices`) |
| `push-bills` | `DELETE /v2/accounting/bills/{id}` (`delete-bills`) |
| `push-invoices-payments` | `DELETE /v2/accounting/invoices/payments/{id}` |
| `push-bills` payments | `DELETE /v2/accounting/bills/payments/{id}` |
| `push-refund` | **nothing** — a refund cannot be reversed through this API |
| `create businesses` | `DELETE /v2/businesses/{uuid}` — tears down connections and synced data, no restore operation exists |
| a queued `dataSync` | **nothing** — there is no cancel |

Deletion is proxied to the customer's own accounting system, so whether it actually reverses
depends on that system's rules — which this contract does not surface. Confirm with a read.

## Errors

The envelope is `{ error: { statusCode, message[], error }, payload: {} }` — **`message` is an
array**. It is not RFC 9457 and there is no `type` URI. `404` is declared on exactly one
operation in the whole contract, so do not rely on the spec to tell you which reads can miss.
