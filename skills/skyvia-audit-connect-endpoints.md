---
name: Audit and control Skyvia Connect endpoints
description: >-
  Inventory every Connect endpoint in a workspace, read its request log to see who has been calling it, and
  disable one that should not be live. This is the governance surface over Skyvia's data plane — and it has
  one blind spot worth knowing about before you rely on it.
api: openapi/skyvia-endpoints-api-openapi.yml
base_url: https://api.skyvia.com
operations:
  - GET /v1/endpoints/types
  - GET /v1/workspaces/{workspaceId}/endpoints
  - GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}
  - GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}/executions
  - GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}/executions/{recordId}
  - POST /v1/workspaces/{workspaceId}/endpoints/{endpointId}/disable
  - POST /v1/workspaces/{workspaceId}/endpoints/{endpointId}/enable
scopes:
  - Endpoint / Read
  - Endpoint / Enable-Disable
  - Workspace / Read
---

# Audit and control Skyvia Connect endpoints

A Connect endpoint publishes a data source to the open web. This skill is how you find out what is
published, who is calling it, and how to switch it off.

Issue a token with **Endpoint / Read** and, for step 4, **Endpoint / Enable-Disable**.

## Know the blind spot first

`EndpointDto.type` in the published spec enumerates only **`OData`** and **`Sql`**. Skyvia Connect has
shipped **MCP endpoints** since 2025, and they are not representable in this v1 resource. So:

> **An inventory built from this API is not necessarily complete.** MCP endpoints — the ones an AI agent
> talks to — may not be enumerable, readable, or disable-able here. Cross-check the workspace in the Skyvia
> UI before concluding you have seen everything.

Confirm the current enum yourself with `GET /v1/endpoints/types`, which returns the endpoint types the
service recognizes. If MCP appears there, the API has moved on since this profile was written and the
warning above no longer applies.

## 1. Inventory

`GET /v1/workspaces/{workspaceId}/endpoints`

Paged with `skip` and `take`, returning `{ "data": [...], "hasMore": ... }`. Increment `skip` until
`hasMore` is false.

Each `EndpointDto` carries `id`, `name`, `token`, `active`, `type`, `created` and `modified`. Note what is
**not** there: no connection id. You cannot tell from this response which data source an endpoint publishes
— you have to correlate by name or look in the UI.

Flag anything where `active` is `true` and you cannot account for the name.

## 2. Read one endpoint

`GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}`

## 3. Read the request log — the actual audit trail

`GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}/executions`

This is the per-request log for the endpoint, and the only observability Skyvia gives you over the data
plane. It accepts `startDate`, `endDate`, `failed`, `sortBy`, `sortOrder`, `skip` and `take`.

For one record's full detail: `GET /v1/workspaces/{workspaceId}/endpoints/{endpointId}/executions/{recordId}`,
which returns the detailed log DTO.

Run this with a date window over the last 30 days when reviewing an endpoint you did not create. Pass
`failed=true` to surface rejected calls — repeated failures against a secured endpoint are what a
credential-guessing attempt looks like.

## 4. Disable what should not be live

`POST /v1/workspaces/{workspaceId}/endpoints/{endpointId}/disable`

Requires **Endpoint / Enable-Disable**. Naturally idempotent — disabling an already-disabled endpoint is
safe. Re-enable with the `/enable` operation.

Disabling is the fastest available containment action. It does not delete the endpoint or its
configuration, and the request log survives.

## Hardening you cannot do from this API

Endpoint security — user accounts, IP allow-lists, and per-object/per-operation permissions — is configured
in the Skyvia UI, not through the Public API. There is no operation here that reads or sets an endpoint's
security posture, so an audit through this API tells you an endpoint is **live**, never whether it is
**protected**. Note that endpoint security requires a Connect **Standard** plan or above; Free and Basic
plans cannot secure an endpoint at all.

## Related artifacts

`data-model/skyvia-data-model.yml` · `mcp/skyvia-mcp.yml` · `authentication/skyvia-authentication.yml` ·
`plans/skyvia-plans-pricing.yml` · `conformance/skyvia-conformance.yml`
