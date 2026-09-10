---
name: fis-read-a-customers-financials
description: Read a connected business's accounting data from FIS Accounting Data as a Service — accounts, invoices, bills and customers — with correct pagination, date scoping, and the accounting-method switch that changes what the numbers mean.
api: FIS Accounting Data as a Service
generated: '2026-09-10'
method: generated
source: openapi/_original/fis-accounting-data-as-a-service-openapi.json
base_url: https://api.railz.ai
operations:
  - accounts
  - invoices
  - getBills
  - customers
  - invoicesPayments
  - financialRatios
  - businessValuations
---

# Read a customer's financials

Every read is scoped by `connectionUuid` — the authorised link produced by the onboarding flow.
Without it you are asking the API to read nothing.

## The parameters that matter

| Parameter | On | Why it matters |
|---|---|---|
| `connectionUuid` | 84 operations | Which authorised upstream system to read from. Required in practice. |
| `startDate` / `endDate` | 94 / 99 operations | Scope the period. Omitting them pulls the whole ledger. |
| `offset` / `limit` | 103 operations | Offset pagination. `meta` returns `offset`, `limit` and `count`. |
| `orderBy` | 99 operations | Sort key. |
| `accountingMethod` | 23 operations | **cash or accrual.** This does not filter rows — it changes what the figures *mean*. Two calls differing only in this parameter return different, both-correct numbers. Never leave it to the default when reporting a figure to a person. |

## Steps

1. **Chart of accounts** — `GET /v2/accounting/accounts` (`accounts`). Start here; account IDs
   are the join key for most other reads.
2. **Receivables** — `GET /v2/accounting/invoices` (`invoices`) and
   `GET /v2/accounting/invoices/payments` (`invoicesPayments`).
3. **Payables** — `GET /v2/accounting/bills` (`getBills`).
4. **Counterparties** — `GET /v2/accounting/customers` (`customers`).
5. **Derived analytics** — `GET /v2/analytics/financialRatios` (`financialRatios`) and
   `GET /v2/analytics/businessValuations` (`businessValuations`) return figures the API computes
   for you rather than raw ledger rows.

## Pagination

Offset/limit only — there is no cursor. Read `meta.count` and page until `offset + limit >= count`.
On a large ledger deep pages get progressively slower; prefer narrowing with `startDate`/`endDate`
over paging to the end.

## Use the /v2/ paths

The contract carries both generations — `/accounts` and `/v2/accounting/accounts` are the same
entity reached two ways. **Nothing is flagged `deprecated` and no Sunset header exists**, so the
contract will not warn you when the older surface goes away. Write new integrations against
`/v2/` only.
