---
generated: '2026-09-05'
method: generated
name: Collect invoices for a period
description: Find the transactions in a window that have an invoice and download each one through its short-lived URL, without wasting calls on transactions that have none.
api: openapi/cledara-api-openapi.json
operations: ['GET /v0/transactions', 'GET /v0/transactions/{transactionId}/invoice-url']
source: >-
  Grounded in openapi/cledara-api-openapi.json, captured 2026-09-05 verbatim from
  https://cledara-public.s3.eu-west-2.amazonaws.com/public-api/open-api.json. The 5-minute URL
  validity and the hasInvoice guard are stated in that spec. Cledara declares no operationIds;
  camelCase names come from overlays/cledara-api-overlay.yaml.
---

# Collect invoices for a period

Two operations, one guard, and a five-minute clock.

## Auth
- `Authorization: Bearer <api-key>`; base URL `https://api.cledara.com`.

## Steps

1. **List the window** — `GET /v0/transactions?from=<RFC3339>&to=<RFC3339>`, paging on `offset`/`nextOffset`/`hasMore` as in *Pull the transaction ledger for a period*.
2. **Filter on `hasInvoice`.** Keep only rows where `hasInvoice` is true. This is the whole point of the field — the contract says so directly: *"You can download the invoice from the v0/transactions/{id}/invoice-url endpoint."*
3. **Prefer the metadata you already have.** If all you need is the invoice number, date or VAT rate, read `invoiceDetails {number, date, vat{name,rate}}` off the transaction and stop — no second call needed. `invoiceDetails` is only populated for *confirmed* invoices, and each of its three fields is independently nullable.
4. **Request the URL only when you will use it immediately** — `GET /v0/transactions/{transactionId}/invoice-url` (overlay id `getTransactionInvoiceUrl`). The response is `{invoiceUrl}`.
5. **Download inside 5 minutes.** The contract states the URL is *valid for 5 minutes*. Fetch the bytes now; do not persist the URL, hand it to another system, or put it in a queue that might drain later.

## Rules an agent must follow

- **Never call step 4 on a transaction whose `hasInvoice` is false.** It returns `404 Record not found`, and against a 120/minute budget a blind sweep across a month of transactions will exhaust the limit on guaranteed failures.
- **Budget the fan-out.** One call per invoice plus one per page of transactions. At 120 requests/minute per key, an export of 500 invoices takes at least five minutes of wall clock — pace on `X-RateLimit-Remaining` rather than firing in parallel.
- **`vat.rate` is a fraction, not a percentage.** `0.21` means 21%; the human label is in `vat.name` (e.g. "Standard rate - 21%").
- **A 404 here is not always "missing".** It means either the transaction id does not exist or the transaction has no invoice attached. Re-read the transaction before reporting a gap to a human.
- **Read-only, so retries are free** — but a retry re-issues a *new* short-lived URL rather than resurrecting the old one. See `conventions/cledara-conventions.yml`.
