---
name: specright-bulk-jobs
description: >-
  Submit and monitor Specright asynchronous CSV bulk jobs across specs, spec families,
  suppliers and generic objects. High-consequence: bulk jobs cannot be cancelled and bulk
  delete cannot be undone.
api: Specright API
base_url: https://api.specright.com/v1
operations:
  - POST /specs/bulkjob
  - GET /specs/bulkjob/{job-id}/status
  - GET /specs/bulkjob/{job-id}/details
  - POST /specfamilies/bulkjob
  - GET /specfamilies/bulkjob/{job-id}/status
  - GET /specfamilies/bulkjob/{job-id}/details
  - POST /suppliers/bulkjob
  - GET /suppliers/bulkjob/{job-id}/status
  - GET /suppliers/bulkjob/{job-id}/details
  - POST /objects/{api-name}/bulkjob
  - GET /objects/{api-name}/bulkjob/{job-id}/status
  - GET /objects/{api-name}/bulkjob/{job-id}/details
source: https://developer.specright.com/api-reference
grounding: >-
  Operations, parameters, the 202 job record and the documented operation enum transcribed
  from the published Specright API v1.1.0 reference.
---

# Run a Specright bulk job

The bulk family is how Specright handles volume: upload a CSV, get `202 Accepted` with a
`job-id`, poll for completion. It is also the highest-consequence surface in the API.

> **Read this first.** A bulk job **cannot be cancelled.** Once it returns `202` it runs to
> completion — there is no stop, abort or cancel operation. `operation=delete` will delete
> every row in the file, with no undo and no stated retention. Require explicit human
> confirmation for any bulk job, and treat `operation=delete` as requiring a second one.

## Prerequisite

Run `specright-discover-tenant-schema`. CSV column headers must be the tenant's field API
names, and the `externalid` you key on must be a real field on the object.

## Steps

1. **Submit.**

   `POST /specs/bulkjob` (or `/specfamilies/bulkjob`, `/suppliers/bulkjob`,
   `/objects/{api-name}/bulkjob`)

   Query parameters:
   - `operation` — **required.** One of `insert`, `update`, `upsert`, `delete`. An
     unsupported value returns `406`.
   - `contenttype` — the bulk file type, e.g. `CSV`.
   - `externalid` — the Salesforce API Name of the field to key an `upsert` on.

   Body: a multipart file upload (`file`, binary).

   No file-size ceiling, row ceiling or concurrent-job limit is published. Start small.

2. **Capture the 202.** The response is the job record:

   `job-id`, `created-by`, `submission-timestamp`, `external-id`, `content-type`, `object`,
   `operation`, `status`.

   **Persist `job-id` before doing anything else.** It is the only handle on the job, and
   there is no operation that lists jobs — lose it and you cannot check the outcome.

3. **Poll status.** `GET /specs/bulkjob/{job-id}/status`.

   `InProgress` is the only status value Specright publishes; the full set is undocumented,
   so treat any non-`InProgress` value as terminal and read the details. No polling interval
   or backoff is recommended anywhere — poll at a modest fixed interval and cap total wait.

4. **Read the outcome.** `GET /specs/bulkjob/{job-id}/details` for per-row results.

5. **Verify.** Query the affected records back. Do not treat `202` as success — it means
   accepted, not applied.

## Retry safety

Retrying a `POST /{resource}/bulkjob` **creates a second job**. There is no idempotency key
and no deduplication. If a submission times out, poll or query before resubmitting — never
resubmit blindly.

## Completion signalling

Polling is the only option. Specright publishes no webhook, callback or event surface, so
nothing will tell you a job finished.

## Errors

`400` malformed request (inline body, schema unspecified) · `401` credential · `403` the
`x-user-id` principal lacks permission · `406` unsupported `operation` value.
