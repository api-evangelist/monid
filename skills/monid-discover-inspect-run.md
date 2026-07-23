---
name: Discover, inspect, and run a data endpoint
description: >-
  Use the Monid HTTP API to find the right external data/tool endpoint in natural
  language, check its input schema and price, execute it, and retrieve results —
  paying per run from a workspace wallet balance.
api: openapi/monid-openapi.json
operations:
  - POST /v1/discover
  - POST /v1/inspect
  - POST /v1/run
  - GET /v1/runs/{runId}
  - GET /v1/wallet/balance
---

# Discover → inspect → run a Monid data endpoint

Monid is an agent-native router: instead of managing per-provider API keys, you
discover the right endpoint at runtime, pay per successful call from a single
workspace balance, and let Monid broker the upstream call.

## Auth

Send `Authorization: Bearer <token>` on every request. The token is either a
Monid API key (`monid_live_...`) or a Clerk-issued OAuth/JWT access token. Base
URL is `https://api.monid.ai`. If you use an OAuth token without org context,
add `x-workspace-id: <org_...>` (list workspaces with `GET /v1/auth/workspaces`).

## Steps

1. **Discover** — `POST /v1/discover` with `{ "query": "<natural language>", "limit": 5 }`.
   Returns `results[]` with `provider`, `endpoint`, `description`, `price`, and a
   `score`. Pick the best match by relevance and price.

2. **Inspect** — `POST /v1/inspect` with `{ "provider": "<slug>", "endpoint": "<path>" }`.
   Returns the `input` JSON Schema (pathParams/queryParams/body), `method`,
   `price`, and operator `notes`. Build your input strictly from this schema.

3. **Check balance (optional)** — `GET /v1/wallet/balance` returns `balance` and
   `held`. Ensure funds cover the inspected price before running.

4. **Run** — `POST /v1/run` with `{ "provider": "<slug>", "endpoint": "<path>", "input": { "body": {}, "queryParams": {}, "pathParams": {} } }`.
   A terminal response includes `status` (COMPLETED / FAILED / BLOCKED / TIMED_OUT),
   `output`, `providerResponse`, and `billing`. `COMPLETED` means the provider
   responded (any HTTP status); `FAILED` is an infrastructure failure.

5. **Poll if needed** — for long runs, `GET /v1/runs/{runId}` until a terminal
   status; `POST /v1/runs/{runId}/stop` to cancel a stoppable run.

## Rules an agent must respect

- **Billing:** you are charged per run that delivers results, from the prepaid
  wallet. Prices are shown in discover/inspect/run responses — read them first.
- **402 = your wallet is short.** Top up the workspace balance; it is never about
  the upstream provider. An upstream provider's own 402 surfaces as **502** and is
  not charged.
- **408 TIMED_OUT** is terminal and zero-billed.
- **Errors** are `{ "code": <http status>, "message": <string> }` (not RFC 9457).
- **No idempotency key** exists — do not blind-retry `POST /v1/run`; reconcile via
  `GET /v1/runs`, and cap spend with workspace budgets/run-caps.
- **Spend controls:** a run may return **BLOCKED** with `controls[]` when a budget
  or run-cap gate trips; surface the gate to the user rather than forcing it.
