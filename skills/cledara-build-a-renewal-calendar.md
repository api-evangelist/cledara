---
generated: '2026-09-05'
method: generated
name: Build a renewal and upcoming-payment calendar
description: Turn the Cledara application list into a forward-looking calendar of renewals and expected payments, with the caveats that make the numbers honest.
api: openapi/cledara-api-openapi.json
operations: ['GET /v0/applications', 'GET /v0/transactions']
source: >-
  Grounded in openapi/cledara-api-openapi.json, captured 2026-09-05 verbatim from
  https://cledara-public.s3.eu-west-2.amazonaws.com/public-api/open-api.json. The derivation
  caveat on nextPayment is quoted from that spec. Entity graph per
  data-model/cledara-data-model.yml.
---

# Build a renewal and upcoming-payment calendar

The most useful thing this three-operation API can do, and the one where an agent is most likely to state a number more confidently than the data supports.

## Auth
- `Authorization: Bearer <api-key>`; base URL `https://api.cledara.com`.

## Steps

1. **Fetch every application** — `GET /v0/applications`. One call, no pagination.
2. **Drop what is not live.** Keep `status: active`; optionally keep `requested`/`pending` as a separate "incoming" bucket, and exclude `disabled`/`inactive`.
3. **Read the two forward fields.**
   - `nextRenewalDate` — an ISO 8601 calendar date, or `null`. Null means Cledara does not know a renewal date, not that there is no renewal.
   - `nextPayment` — `{date, amount}`, or `null`.
4. **Carry the derivation caveat with the number.** The contract describes `nextPayment` as *"Expected next payment, based on last payment and budget periodicity"*. It is an **estimate Cledara computed**, not a scheduled charge the vendor has confirmed. Label it as expected wherever you present it.
5. **Ground the estimate in history when it matters** — `GET /v0/transactions?applicationIds=<id>&from=<90 days ago>` and look at the actual `amount`/`authorizedAt` cadence for that application. A `nextPayment.amount` that disagrees with the last three real charges is the interesting finding, not a bug to smooth over.
6. **Group by currency, never sum across.** `budget.currency` and `Transaction.currency` are independent and Cledara applies no FX. A single "total upcoming spend" figure across GBP, EUR and USD is not derivable from this API.

## Rules an agent must follow

- **Do not present `nextPayment` as a commitment.** It is derived from last payment plus `budget.periodicity`. An agent that tells a CFO "you will be charged £X on date Y" has over-claimed.
- **Normalise budgets by `periodicity` before comparing.** `limit` is per period (`week`/`month`/`quarter`/`year`), and `noBudget` applications have `limit: null`.
- **`balance` is not a forecast.** It is the application's current balance, and it is nullable.
- **You cannot act on any of this.** The API is read-only: there is no cancel, no pause, no budget change. Produce the calendar and hand the decision to a human — `conventions/cledara-conventions.yml` records `reversibility: na` precisely because there is no write to reverse.
- **Nothing pushes.** No webhooks exist, so a calendar is only as fresh as your last poll. Re-run against the 120/minute per-key limit and state the as-of time on the output.
