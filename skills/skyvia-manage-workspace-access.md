---
name: Manage Skyvia account and workspace access
description: >-
  Review who can reach a Skyvia account, invite and remove users, and grant or revoke workspace membership
  through the Public REST API. The destructive operations here use request bodies on DELETE and have no
  idempotency key, so the order of the steps matters.
api: openapi/skyvia-account-api-openapi.yml
base_url: https://api.skyvia.com
operations:
  - GET /v1/account/users
  - DELETE /v1/account/users
  - GET /v1/account/invitations
  - POST /v1/account/invitations
  - POST /v1/account/invitations/{invitationId}/resend
  - DELETE /v1/account/invitations/{invitationId}
  - GET /v1/workspaces
  - GET /v1/workspaces/{workspaceId}/users
  - POST /v1/workspaces/{workspaceId}/users
  - DELETE /v1/workspaces/{workspaceId}/users/{userId}
scopes:
  - Account / Read
  - Account / Modify
  - Workspace / Read
  - Workspace / Read Users
  - Workspace / Modify Users
---

# Manage Skyvia account and workspace access

Skyvia has two levels of membership: the **account** (billing and identity boundary) and the **workspace**
(where assets live and where access is actually enforced). A user must be on the account before they can be
added to a workspace.

Token permissions needed: **Account / Read** and **Account / Modify** for the account operations,
**Workspace / Read**, **Workspace / Read Users** and **Workspace / Modify Users** for the workspace ones.

## 1. Review the account

`GET /v1/account/users` — accepts `searchMask` for free-text filtering plus `skip` and `take`. Returns
`{ "data": [...], "hasMore": ... }`.

`GET /v1/account/invitations` — same shape, same `searchMask` filter. These are people who have been invited
but have not yet accepted. Stale invitations are an access-review finding in their own right; check the
status field on each.

## 2. Invite someone

`POST /v1/account/invitations` with an invite request body. The DTO can carry workspace assignments, so an
invitation can grant workspace membership at the same time as account membership — check the request schema
in `openapi/skyvia-account-api-openapi.yml` before assuming you need a second call.

If the invitation was not received: `POST /v1/account/invitations/{invitationId}/resend`.

**This operation is not idempotent and has no idempotency key.** Calling it twice sends two invitations.
List the invitations first and confirm one does not already exist.

To withdraw: `DELETE /v1/account/invitations/{invitationId}`.

## 3. Grant workspace access

`GET /v1/workspaces` lists the workspaces the token can see, then
`GET /v1/workspaces/{workspaceId}/users` lists current membership.

`POST /v1/workspaces/{workspaceId}/users` adds a user to the workspace. The request body carries the user
and their role — Skyvia's workspace roles are documented at
`https://docs.skyvia.com/account-management/workspace-roles.html`.

## 4. Revoke access — order matters

Remove in this order:

1. `DELETE /v1/workspaces/{workspaceId}/users/{userId}` — remove them from each workspace. Do this for every
   workspace they appear in; there is no operation that lists a user's workspaces, so iterate the workspace
   list and check membership on each.
2. `DELETE /v1/account/users` — remove them from the account. **Note this DELETE takes a request body**
   (`RemoveAccountUserRequestDto`) rather than a path parameter, which some HTTP clients will not send by
   default. Verify your client actually transmits a DELETE body before relying on this call.

Removing account membership does not, on its own, invalidate anything that user created. API tokens are
issued per account under Account Settings, and there is no API operation to list or revoke tokens — token
cleanup is a UI task. If the departing user issued API tokens, revoke them in the UI, or their token keeps
working until its expiry date, up to a year out.

## Access-review checklist

- [ ] Every account user is accounted for (`GET /v1/account/users`)
- [ ] No stale pending invitations (`GET /v1/account/invitations`)
- [ ] Workspace membership reviewed per workspace (`GET /v1/workspaces/{workspaceId}/users`)
- [ ] API tokens reviewed in Account Settings > API Settings — **not** available through this API
- [ ] Account-level IP filtering and enforced 2FA considered (both shipped June 2026, UI-only)

## Failure modes

A missing or under-scoped token returns **HTTP 403** with
`{"errorCode":403,"errors":{},"message":"Authorization header is missing or invalid.","refresh":false}`.
Because Skyvia uses 403 for both "no credential" and "wrong permission", read the `message` string rather
than branching on the status code. The `errors` object carries field-level validation failures on the
invite and add-user bodies.

## Related artifacts

`scopes/skyvia-scopes.yml` · `authentication/skyvia-authentication.yml` · `errors/skyvia-problem-types.yml` ·
`data-model/skyvia-data-model.yml`
