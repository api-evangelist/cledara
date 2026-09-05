---
generated: '2026-09-05'
method: generated
name: Export the software subscription inventory
description: Pull every application Cledara tracks for a workspace, with owner, team, budget, balance and next renewal, and turn it into an inventory an agent can reason over.
api: openapi/cledara-api-openapi.json
operations: ['GET /v0/applications']
source: >-
  Grounded in openapi/cledara-api-openapi.json, captured 2026-09-05 verbatim from
  https://cledara-public.s3.eu-west-2.amazonaws.com/public-api/open-api.json (the spec
  api-docs.cledara.com renders). NOTE: Cledara's spec declares NO operationIds, so the paths
  above are the real identifiers; the camelCase names used in prose come from
  overlays/cledara-api-overlay.yaml and are ours, not the provider's. Auth per
  authentication/cledara-authentication.yml, errors per errors/cledara-problem-types.yml,
  limits per rate-limits/cledara-rate-limits.yml, entity graph per
  data-model/cledara-data-model.yml.
---

# Export the software subscription inventory

One call gets the whole picture of what a company pays for.

## Auth
- `Authorization: Bearer <api-key>`. Mint the key in the Cledara app under **Settings → Profile Details → API Keys**.
- Base URL: `https://api.cledara.com`.
- The key carries **exactly the permissions of the user who created it**. There are no scopes and no service accounts — if the key is missing data, the answer is a key from a user with a broader role, not a retry.

## Steps

1. **Fetch the inventory** — `GET /v0/applications` (overlay id `listApplications`).
2. **Read the response as a bare array.** This operation returns a plain JSON array of `Application`, not an envelope. There is **no pagination and no filtering** on it, so one call is the whole workspace and there is no page to follow.
3. **Project the fields you need.** Every `Application` carries `id`, `name`, `status`, `budget`, `createdAt`, `owner {id,name}`, `nextRenewalDate`, `balance`, `nextPayment {date,amount}` and `teams[]` — all of them required, so none will be missing, though `nextRenewalDate`, `balance` and `nextPayment` are explicitly nullable.
4. **Segment on `status`** — `pending`, `requested`, `active`, `disabled`, `inactive`. Only `active` is spend that is actually running; `requested` and `pending` are the approval pipeline.
5. **Read `budget` as three cases, not a number.** `budget.type` is `soft`, `fixed` or `noBudget`. A `fixed` budget is a hard cap enforced on the card; a `soft` budget is a tracking figure only. `budget.limit` is null when `type` is `noBudget`, and `budget.periodicity` (`week`/`month`/`quarter`/`year`) is what makes `limit` comparable across applications — never sum raw limits across different periodicities.

## Rules an agent must follow

- **This is a read-only API.** All three published operations are `GET`. Nothing here can change a subscription, cancel an application or move money, so there is no idempotency key to send and nothing to undo — see `conventions/cledara-conventions.yml`, where `idempotency` and `reversibility` are both `na` by construction. If you are asked to cancel something, the API cannot do it; say so rather than reaching for another endpoint.
- **Rate limit: 120 requests per minute per API key.** Read `X-RateLimit-Remaining` and `X-RateLimit-Reset` from every 200. On `429`, wait exactly the seconds in `Retry-After`. See `rate-limits/cledara-rate-limits.yml`.
- **Errors are not RFC 9457.** A failure is `application/json` shaped `{message, error:{method,url,status,cledaraType,errorId}}`. `error.errorId` is a per-occurrence support id, **not** a stable error code — never branch on it. `401` means the key is missing or invalid; `403` means the key's user lacks account privileges. Neither is retryable. See `errors/cledara-problem-types.yml`.
- **Currencies do not convert.** `budget.currency` is an ISO 4217 code and Cledara performs no FX. Do not add amounts across currencies.
- **Cards, members and teams have no endpoints.** You only ever see them as embedded summaries (`owner`, `teams[]`). There is no way to enumerate them, so do not promise a complete list of either.
