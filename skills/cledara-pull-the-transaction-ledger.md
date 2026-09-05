---
generated: '2026-09-05'
method: generated
name: Pull the transaction ledger for a period
description: Page a dated window of Cledara card payments, transfers and account movements out of a workspace and into a ledger or accounting feed.
api: openapi/cledara-api-openapi.json
operations: ['GET /v0/transactions']
source: >-
  Grounded in openapi/cledara-api-openapi.json, captured 2026-09-05 verbatim from
  https://cledara-public.s3.eu-west-2.amazonaws.com/public-api/open-api.json. Cledara's spec
  declares no operationIds; camelCase names come from overlays/cledara-api-overlay.yaml.
  Pagination and money semantics per conventions/cledara-conventions.yml.
---

# Pull the transaction ledger for a period

The bulk-export path. This is the operation behind Cledara's own examples of accounting feeds and finance dashboards.

## Auth
- `Authorization: Bearer <api-key>`; base URL `https://api.cledara.com`. See `authentication/cledara-authentication.yml`.

## Steps

1. **Bound the window first** — `GET /v0/transactions?from=<RFC3339>&to=<RFC3339>`. `from` and `to` are RFC 3339 timestamps (`2026-01-01T00:00:00Z`). Both are optional; omitting them asks for everything, which is the slowest and least predictable call you can make here.
2. **Narrow by application if you can** — pass `applicationIds` (an array of UUIDs from `GET /v0/applications`) to scope the pull to specific subscriptions.
3. **Page with `offset`.** The response is an object, not an array: `{transactions[], nextOffset, hasMore, totalCount}`. Loop while `hasMore` is true, passing the previous `nextOffset` as the next `offset`. Stop when `hasMore` is false or `nextOffset` is null.
4. **Use `totalCount` as the completeness check.** Compare the number of records you collected against `totalCount` before declaring the export finished.
5. **Classify each row** on `type` (`cardSend`, `cardReceive`, `transferSend`, `transferReceive`, `applicationTopUp`, `applicationFlush`, `other`) and `accountType` (`application`, `saasMain`, `spend`, `others`). `applicationTopUp` and `applicationFlush` are internal funding movements, not vendor spend — including them in a spend total double-counts.

## Rules an agent must follow

- **There is no page-size parameter.** No `limit`, no `per_page`. The server chooses the page size, so never assume one and never compute an offset arithmetically from an assumed page size — always follow `nextOffset`.
- **Amounts are signed decimals, not minor units.** Negative is a debit, positive a credit. `amount`/`currency` are in the *account* currency (constrained to `GBP`, `EUR`, `USD`); `localAmount`/`localCurrency` are the original payment currency when it differs. **No FX rate is exposed**, so you cannot reconstruct the conversion — report per currency, as Cledara's own market data does.
- **`settledAt` is optional; `authorizedAt` is required.** Reconcile on `authorizedAt` unless you specifically need settled figures, and expect authorised-but-unsettled rows near the end of a window.
- **`accounting[]` is customer-defined.** Each row is `{field:{id,name}, value, name?}`. Key your mapping on `field.id` — the contract explicitly says the id is stable across renames while `field.name` is not. Empty values are omitted entirely, so absent ≠ empty string.
- **Do not re-page on a `429`.** Wait `Retry-After` seconds and resume from the same `offset`. The limit is 120/minute per key; a wide export will hit it. See `rate-limits/cledara-rate-limits.yml`.
- **Read-only.** Nothing in this flow writes, so retries are always safe and no reversal is ever needed — `conventions/cledara-conventions.yml` records `idempotency: na` and `reversibility: na`.
- **There are no webhooks.** Cledara publishes no event or streaming surface at all, so a ledger stays current only by re-polling this operation on a schedule. Choose the schedule deliberately against the 120/minute budget.
