---
name: Trigger a Skyvia backup snapshot and verify it completed
description: >-
  Take an on-demand cloud-to-cloud backup snapshot through the Public REST API, confirm it finished, and
  manage the backup schedule. Includes the storage-quota arithmetic and the duplicate-snapshot hazard on
  retry.
api: openapi/skyvia-backups-api-openapi.yml
base_url: https://api.skyvia.com
operations:
  - GET /v1/workspaces/{workspaceId}/backups
  - GET /v1/workspaces/{workspaceId}/backups/{backupId}
  - GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots/active
  - POST /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots
  - GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots
  - GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots/{snapshotId}
  - GET /v1/workspaces/{workspaceId}/backups/{backupId}/schedule
  - POST /v1/workspaces/{workspaceId}/backups/{backupId}/schedule/enable
  - POST /v1/workspaces/{workspaceId}/backups/{backupId}/schedule/disable
scopes:
  - Workspace / Read
---

# Trigger a Skyvia backup snapshot and verify it

Use this before a risky migration, a bulk import, or any change you might need to roll back.

## 1. Find the backup

`GET /v1/workspaces/{workspaceId}/backups` — paged with `skip`/`take`, returning
`{ "data": [...], "hasMore": ... }`. Then `GET /v1/workspaces/{workspaceId}/backups/{backupId}` for detail.

## 2. Check nothing is already running

`GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots/active`

Always do this first. Snapshot creation has **no idempotency key**, so a retry after a timeout produces a
second snapshot and consumes storage quota twice.

## 3. Take the snapshot

`POST /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots`

If this call times out, **do not retry it.** Poll the `/snapshots/active` operation and see what is actually
running.

## 4. Verify it finished

Poll `/snapshots/active` until it reports nothing running, then:

`GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots`

This returns finished snapshots and accepts `startDate`, `endDate`, `failed`, `sortBy`, `sortOrder`, `skip`
and `take`. Pass `failed=true` to check for failures. Read one snapshot's full log with
`GET /v1/workspaces/{workspaceId}/backups/{backupId}/snapshots/{snapshotId}`, which returns the detailed
snapshot log DTO.

**Verify before you proceed.** Skyvia sends no webhook when a backup finishes or fails — if you do not poll,
you will not know. A "backup taken" assumption that turns out to be a failed run is the worst outcome this
skill exists to prevent.

## 5. Manage the schedule

- `GET /v1/workspaces/{workspaceId}/backups/{backupId}/schedule` — read the current schedule.
- `POST .../schedule/disable` — pause scheduled backups (during a maintenance window, say).
- `POST .../schedule/enable` — resume.

Both are naturally idempotent. **Re-enable the schedule when you are done** — a disabled schedule is silent,
and nothing will remind you.

## Quota arithmetic

Backup storage is capped by plan: **1 GB** Free (3-month snapshot retention), **20 GB** Standard, **200 GB**
Professional, **1 TB** Enterprise — all paid tiers with unlimited retention. On the Free plan, unlimited
on-demand snapshots against a 1 GB ceiling with only three months of retention will run out. Check the
existing snapshot list before adding another.

## Related artifacts

`plans/skyvia-plans-pricing.yml` · `conventions/skyvia-conventions.yml` · `scopes/skyvia-scopes.yml` ·
`asyncapi/skyvia-automation-webhooks.yml`
