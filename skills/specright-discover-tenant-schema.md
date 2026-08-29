---
name: specright-discover-tenant-schema
description: >-
  Read a Specright tenant's object and field vocabulary before composing any other call.
  Specright's API contract declares no business fields — every field name is per-tenant
  configuration served at runtime from the /definition endpoints. Run this first.
api: Specright API
base_url: https://api.specright.com/v1
operations:
  - GET objects/definition
  - GET /objects/{api-name}/definition
  - GET /specs/definition
  - GET /specfamilies/definition
  - GET /suppliers/definition
source: https://developer.specright.com/api-reference
grounding: >-
  Operations transcribed from the published Specright API v1.1.0 reference. Specright
  publishes no OpenAPI document, so these are method+path identifiers rather than
  operationIds — the reference itself has none.
---

# Discover a Specright tenant's schema

Specright is configured per customer. Two tenants running the same product expose
different objects and different fields, and the API contract describes none of them: every
record comes back as `{"fields": [{"field", "label", "value"}]}` where `field` is a
Salesforce API name and `value` is untyped. **Never hard-code a field name.** Resolve it
first.

## Steps

1. **Authenticate.** Send both headers on every call:
   - `x-api-key: <key issued by Specright>`
   - `x-user-id: <the Specright user to act as>`

   Both are documented as required. A bearer token from `POST /token` may be used in place
   of the API key, but `x-user-id` is still required.

2. **List the objects available in this tenant.**

   `GET objects/definition`

   Returns `objectinfo` records: `label`, `api-name`, `description`, `last-modified`,
   `custom`. The `custom` flag tells you whether an object is tenant-specific. `api-name` is
   what you pass to every `/objects/{api-name}` call.

3. **Read the definition of the object you need.**

   `GET /objects/{api-name}/definition` — or, for the three named families,
   `GET /specs/definition`, `GET /specfamilies/definition`, `GET /suppliers/definition`.

   You get an `objectdefinition`:
   - `object-info` — the `objectinfo` above.
   - `fields-list` — the field vocabulary. Match on `label` when a human named the field;
     use `field` (the Salesforce API name) in `fields`, `filter` and `sort` parameters.
   - `connections-list` — each relationship as `{name, api-name, field}`, telling you which
     field carries the reference.
   - `recordtypes-list` — Salesforce record types as `{name, id}`.

4. **Cache the definition for the session, keyed by tenant and object.** `last-modified`
   on `objectinfo` is your invalidation signal. Do not re-read it per record.

5. **Build every later call from the definition.** Pass resolved API names in `fields` to
   keep responses small, in `filter` to query, and in `sort` to order.

## Rules

- Values are typed `anyvalue`. The contract carries no type, format, enum or unit for any
  business field. Do not infer a type from a sample; if a caller needs one, read `label`
  and surface the raw value.
- Fields are positional inside an array, not keys on an object. Scan `fields[]` for a match
  — do not index. No field is guaranteed present.
- Managed-package fields carry the `specright__` prefix and `__c` suffix; standard
  Salesforce fields (`Id`, `Name`) do not.

## Errors

- `401` — bad or missing credential.
- `403` — the `x-user-id` principal lacks access to the object. This is a permissions
  problem, not a key problem. The documented remedy is to contact `api@specright.com`.
- `400` — malformed parameters.

See `../errors/specright-error-codes.yml` and `../conventions/specright-conventions.yml`.
