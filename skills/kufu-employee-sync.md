---
name: kufu-employee-sync
description: >-
  Keep an external system in sync with SmartHR employee and family data —
  register webhooks, consume crew/dependent events safely, and reconcile
  through paginated reads or async bulk export. Use when building a directory,
  IdP, or downstream HRIS mirror on top of SmartHR.
api: SmartHR API v1
generated: '2026-07-19'
method: generated
source: >-
  openapi/kufu-smarthr-openapi.json and
  https://developer.smarthr.jp/api/about_webhook
operations:
  - postV1Webhooks
  - getV1Webhooks
  - patchV1WebhooksId
  - deleteV1WebhooksId
  - getV1Crews
  - getV1CrewsId
  - getV1CrewsCrewIdDependents
  - postV1BatchJobsJobTypesCrewExport
  - getV1BatchJobsId
  - getV1BatchJobAttachmentsIdFile
---

# Mirror SmartHR employee data into another system

## 1. Take the initial snapshot

Do not page through `getV1Crews` for a full backfill on a large tenant — you will spend your rate budget. Use the async export instead:

1. `postV1BatchJobsJobTypesCrewExport` to submit the job
2. Poll `getV1BatchJobsId` until it completes
3. `getV1BatchJobAttachmentsIdFile` to download the output

For small tenants or targeted reads, `getV1Crews` with `page`/`per_page` (max 100) is fine. Follow the `Link` header's `rel="next"` and use `x-total-count` to size the job.

## 2. Register a webhook

Call `postV1Webhooks` with:

- `url` — your HTTPS endpoint
- the event flags you want: `crew_created`, `crew_updated`, `crew_deleted`, `crew_imported`, `dependent_created`, `dependent_updated`, `dependent_deleted`, `dependent_imported`
- `secret_token` — always set this
- `expose_only_id: true` if you would rather receive just the changed id and re-read the record yourself. For sensitive HR data this is the safer default: it keeps statutory personal data out of your webhook logs.

Inspect existing registrations with `getV1Webhooks`, amend with `patchV1WebhooksId`, remove with `deleteV1WebhooksId`.

## 3. Consume events safely

Each delivery is a POST with `{event, triggered_at, sender, <payload_key>}`, where the payload key is one of `crew`, `dependent`, `crew_import_result`, `dependent_import_result`, or `workflow`.

- Verify the `X-SmartHR-Token` header against your secret, using a constant-time comparison.
- **There is no HMAC signature.** The token proves the sender knows the secret; it does not prove payload integrity. For anything consequential, re-read the record with `getV1CrewsId` rather than trusting the delivered body.
- Return a 2xx within 60 seconds. Do the work asynchronously — acknowledge first, process after.
- If you exceed the retry budget (~17 attempts over ~3 days in production, ~5 over ~8 minutes in sandbox), SmartHR **disables the registration**. Monitor for this; a silent integration is the failure mode.
- Ordering is not guaranteed and deliveries are not deduplicated. Key your handler on the record id and treat `updated_at` as the tiebreaker.

## 4. Avoid feedback loops

If your integration writes back into SmartHR, append `skip_sending_webhook=true` to those writes so your own consumer does not re-trigger on its own changes.

## 5. Understand effective dating

SmartHR records are bitemporal. A change booked today with a future effective date will fire `crew_updated` **on the effective date**, not at booking time. Payloads always carry state as of send time. If your mirror needs to know about pending future changes, webhooks alone will not tell you — you must reconcile by reading.

## 6. Reconcile periodically

Webhooks are best-effort. Schedule a periodic reconciliation — a full export, or a `getV1Crews` sweep sorted by `sort=-updated_at` — to catch anything dropped while your endpoint was down.

## Handling notes

Employee payloads include My Number, basic pension numbers, bank accounts, salary and residency documents. Minimize what you mirror, encrypt at rest, and prefer `expose_only_id` plus targeted reads over storing full payloads in a queue or log.
