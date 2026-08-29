---
name: specright-query-specifications
description: >-
  Search and read Specright specifications, spec families (BOM / finished good) and
  suppliers, using sparse fieldsets, JSON filters, sorting and offset pagination, then walk
  the connections graph and retrieve attached files.
api: Specright API
base_url: https://api.specright.com/v1
operations:
  - GET /specs
  - GET /specs/{id}
  - GET /specfamilies
  - GET /specfamilies/{id}
  - GET /suppliers
  - GET /suppliers/{id}
  - GET /objects/{api-name}
  - GET /objects/{api-name}/{id}
  - GET /specs/{id}/files
  - GET /specs/{id}/files/{file-id}
  - GET /specfamilies/{id}/files
  - GET /suppliers/{id}/files
source: https://developer.specright.com/api-reference
grounding: >-
  Operations and parameters transcribed from the published Specright API v1.1.0 reference.
---

# Query Specright specifications

Read-only. Safe to run without confirmation.

## Prerequisite

Run `specright-discover-tenant-schema` first. You need real field API names before you can
filter, sort or narrow a response.

## Steps

1. **List records.**

   `GET /specs`, `GET /specfamilies`, `GET /suppliers`, or `GET /objects/{api-name}` for
   any other configured object.

   Query parameters, identical across all four:
   - `fields` — comma-separated API names to return. Always set this. The default response
     returns every configured field.
   - `filter` — search criteria as a JSON string.
   - `sort` — `field[:asc|:desc]`, comma-separated.
   - `skip` / `limit` — offset pagination.

2. **Paginate.** Increment `skip` by `limit`. There is no total, no `next` link and no
   `has_more` flag — **stop when a page returns fewer records than `limit`.** Neither the
   default nor the maximum `limit` is published, so choose a modest value and observe what
   comes back.

3. **Read one record.**

   `GET /specs/{id}` — where `{id}` is the Salesforce record ID, or the external ID when
   `externalid=<field API name>` is passed. `fields` narrows the response here too.

4. **Walk relationships.** `specification` and `specfamily` records carry `connections[]`.
   Each `connection` has `name` (`specs` or `specfamilies`), `api-name`, and `records[]` of
   `{Id, Name}` stubs. Re-fetch a stub by ID to get its fields.

   Note: `supplier` records have **no** `connections[]`. The spec-to-supplier link is not
   expressed in the published graph — look for a reference field on the spec via
   `connections-list` in the object definition instead.

5. **Files.** `GET /{resource}/{id}/files` returns `fileinfo` records (`id`, `title`,
   `extension`, `file-type`, `size-bytes`, `url`, dates, owner). Fetch content with
   `GET /{resource}/{id}/files/{file-id}`, which returns `file` — the same metadata plus
   `body`. Note `expiration-date`. File access is read-only; there is no upload operation.

## Cautions

- The `filter` grammar is **not published**. Specright documents it only as "search criteria
  specified in json format" with no operators or examples. Test any filter against a known
  record before relying on it, and fall back to `sort`+pagination if it does not behave.
- No rate limits are published and no rate-limit headers are returned. Pace yourself
  conservatively; there is no signal to back off from.

## Errors

`400` malformed parameter (a bad `filter` most likely) · `401` credential · `403` the
`x-user-id` principal lacks record access · `422` on the file endpoints, undocumented —
treat as terminal.
