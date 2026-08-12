---
name: Run a Skyvia integration and monitor it to completion
description: >-
  Start a Skyvia data integration package on demand through the Public REST API, poll its execution to
  completion, and cancel or kill it if it runs away. Covers the token scopes required, the offset paging
  idiom, and the two failure modes that are not in the spec — an undocumented error envelope and no
  idempotency on the start call.
api: openapi/skyvia-integrations-api-openapi.yml
base_url: https://api.skyvia.com
operations:
  - POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions
  - GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/active
  - GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions
  - GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/{executionId}
  - POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/cancel
  - POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/kill
  - GET /v1/workspaces/{workspaceId}/integrations
scopes:
  - Integration / Read
  - Integration / Execute
  - Workspace / Read
---

# Run a Skyvia integration and monitor it

Skyvia's Public API has no operationIds, so every step below names the operation by method and path exactly
as it appears in `openapi/skyvia-integrations-api-openapi.yml`.

## Before you start

- Create an API token in **Account Settings > API Settings** with the **Integration / Read**,
  **Integration / Execute** and **Workspace / Read** permissions. Tokens cannot be issued for longer than
  one year.
- Send it on every request in the `Authorization` header. Skyvia does not document whether a `Bearer `
  prefix is required — confirm against a known-good call before building on it.
- A missing or bad token returns **HTTP 403**, not 401, with no `WWW-Authenticate` header and this body:
  `{"errorCode":403,"errors":{},"message":"Authorization header is missing or invalid.","refresh":false}`.
  None of that is in the spec.

## 1. Find the workspace and the integration

`GET /v1/workspaces` returns the workspaces the token can see. Then
`GET /v1/workspaces/{workspaceId}/integrations` lists the integration packages in one.

Both are paged with `skip` and `take`, and return `{ "data": [...], "hasMore": true|false }`. There is no
total count and no cursor — increment `skip` by `take` until `hasMore` is `false`. Do not assume a default
page size; Skyvia does not publish one.

## 2. Check nothing is already running

`GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/active`

Do this first. The start call is **not idempotent** and there is no `Idempotency-Key` header anywhere in
the API, so a blind retry after a timeout can start a second run that consumes the account's monthly
records quota twice.

## 3. Start the run

`POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions`

Requires the **Integration / Execute** permission.

**If this call times out, do not retry it.** Call the `/executions/active` operation instead and inspect
what is running. Only start again if nothing is.

## 4. Poll to completion

Poll `GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/active` until it reports
nothing active, then read the finished run:

`GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions`

This one accepts `startDate`, `endDate`, `failed`, `sortBy`, `sortOrder`, `skip` and `take`. Pass
`failed=true` to filter to failures. The allowed `sortBy` values are not enumerated in the spec — both
`sortBy` and `sortOrder` are declared as free strings, so discover valid values before depending on them.

For the detail of one run: `GET /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/{executionId}`.

Skyvia emits **no outbound webhooks**, so polling is the only way to learn a run finished. Choose an
interval that respects the plan's scheduling tier rather than hammering the endpoint — there is no published
rate limit and no `Retry-After` to guide backoff.

## 5. Stop a runaway run

- `POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/cancel` — request a graceful stop.
- `POST /v1/workspaces/{workspaceId}/integrations/{integrationId}/executions/kill` — force termination.

Try cancel first. Both are state-changing but naturally idempotent — calling them against an already-stopped
run is safe.

## What can go wrong

| Symptom | Cause | What to do |
|---|---|---|
| 403 on every call | Token missing, expired, or lacking the permission | Re-issue the token with the right permissions; tokens expire within a year |
| Duplicate run started | The start call was retried after a timeout | Check `/executions/active` before starting; there is no idempotency key |
| Run never appears finished | You are polling the list, not `/active` | Poll `/active`, then read the list |
| Quota exhausted mid-run | Free plan caps 10,000 records/month | See `plans/skyvia-plans-pricing.yml` |

## Related artifacts

`conventions/skyvia-conventions.yml` · `errors/skyvia-problem-types.yml` · `scopes/skyvia-scopes.yml` ·
`rate-limits/skyvia-rate-limits.yml`
