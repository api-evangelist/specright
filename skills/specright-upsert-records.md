---
name: specright-upsert-records
description: >-
  Create and update Specright records safely, using external-ID upsert for retry-safe
  writes. Covers the destructive default on the operation parameter and the fact that no
  write in this API has a documented reversal.
api: Specright API
base_url: https://api.specright.com/v1
operations:
  - POST /specs
  - PATCH /specs/{id}
  - POST /specfamilies
  - PATCH /specfamilies/{id}
  - POST /suppliers
  - PATCH /suppliers/{id}
  - POST /objects/{api-name}
  - PATCH /objects/{api-name}/{id}
  - DELETE /specs/{id}
  - DELETE /specfamilies/{id}
  - DELETE /suppliers/{id}
  - DELETE /objects/{api-name}/{id}
source: https://developer.specright.com/api-reference
grounding: >-
  Operations, parameters and the documented upsert default transcribed from the published
  Specright API v1.1.0 reference.
---

# Write to Specright safely

**Every write in this API is irreversible.** There is no undo, restore, rollback or cancel
operation among the 46 published operations, and no retention window is stated for a
deleted record. There is also no dry-run or validate-only mode. Read
`../conventions/specright-conventions.yml` before acting.

## Prerequisite

Run `specright-discover-tenant-schema`. Body fields use the tenant's Salesforce API names.

## Prefer PATCH-with-external-ID over POST

`POST /specs` has no external-ID form. If you retry it after a timeout you create a
duplicate. `PATCH /specs/{id}?externalid=<field API name>` converges on the same record no
matter how many times it runs, which is the only retry-safe write available — Specright
publishes no `Idempotency-Key` header and no request deduplication.

## The destructive default

On any `PATCH` with `externalid`, the `operation` query parameter takes `update` or
`upsert`. The reference states plainly: *"If external ID is provided and operation is
absent then an upsert is assumed."*

**Omitting `operation` creates the record if it does not exist.** If you intend to modify an
existing record only, send `operation=update` explicitly. Never rely on the default.

## Steps

1. **Read before you write.** `GET` the record with `fields` covering everything you will
   change, and keep the response. The `PATCH` response does not echo prior values, so this
   local copy is the only way to reconstruct the old state.
2. **Compose the body** using resolved field API names.
3. **Write:**
   - Update only: `PATCH /specs/{id}?externalid=<field>&operation=update`
   - Create-or-update: `PATCH /specs/{id}?externalid=<field>&operation=upsert`
   - Create only, no external ID available: `POST /specs` — retry-unsafe, do not retry
     automatically on a timeout; re-query first to see whether the record landed.
4. **Verify.** Re-read the record. Do not treat `200` alone as proof of the intended change.

## Deletes

`DELETE /specs/{id}` returns `200` with no body schema. Specright does not state whether the
delete is soft or hard, and exposes no restore operation. Specright runs on Salesforce, whose
platform has a recycle bin, but that is **not** documented as an API guarantee and is not
reachable through any of the 46 operations — do not rely on it.

Require explicit human confirmation before any delete.

## Errors

`400` malformed body or parameter · `401` credential · `403` the `x-user-id` principal
lacks write access on the object or record · no `404` is documented, so behaviour when an
ID does not resolve is unspecified · no `409` and no concurrency control, so last write
wins and there is no ETag or version field to guard with.
