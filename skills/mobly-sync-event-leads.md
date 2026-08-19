---
name: Sync event leads out of Mobly
description: Pull every lead captured at a Mobly event, page by page, and reconcile them against an external system of record using the lead's remoteIds.
api: openapi/mobly-rest-api-v0-openapi.yml
base_url: https://core-api.getmobly.com/api/v0
generated: '2026-08-13'
method: generated
source: https://help.getmobly.com/documentation/rest-api/rest-api-v0
operations:
  - GET /events
  - GET /events/{eventId}/leads
  - GET /leads
  - GET /leads/{leadId}
  - GET /leadActivityEvents
---

# Sync event leads out of Mobly

Use this when you need the leads captured at one trade show or activation moved into a CRM,
warehouse or spreadsheet. Mobly has no outbound webhook contract you can subscribe to, so this
is a **poll-and-reconcile** flow.

## Before you start

- You need an organization API key. Keys are **not self-serve** — Mobly's API reference says
  "contact your Mobly CSM". There is no sandbox and no test key; the one base URL is production.
- Send the key on every request as the header `x-api-key: <key>`. A missing key returns
  **401 Unauthorized**; an invalid one returns **403 Forbidden**. Both come back as
  `{"status": <code>, "error": "<reason>"}`.

## Steps

1. **Find the event.** `GET /events?limit=50&offset=0`. Read `results.events[]` and
   `results.total`. Match on `name` and `startDate`. Note `id` — for **events only** this is an
   integer, not a UUID.

2. **Page to the end.** `offset` is a zero-based **page index**, not a record offset —
   `?limit=25&offset=2` returns the *third* page of 25. `limit` defaults to 20 and is clamped at
   50. Keep requesting until `(offset + 1) * limit >= results.total`.

3. **List the event's leads.** `GET /events/{eventId}/leads`. This returns the association
   records — `leadId`, `registered`, `checkedIn` — not the full lead.

4. **Hydrate each lead.** `GET /leads/{leadId}` (leadId is a **UUID string** here, unlike the
   event's integer id). The response carries the enriched payload: `company` (name, industry,
   linkedin, address), `phoneNumbers`, `jobTitle`, `tags[]`, `activations[]`, `numberOfScans` and
   `isEnriched`.
   - Only trust enrichment when `isEnriched` is true. Enrichment is asynchronous; a lead read
     seconds after capture may still be bare.
   - `personalEmails` / `professionalEmails` are **semicolon-joined strings, not arrays**, and
     Mobly's own docs warn the separator is not guaranteed: split on `;`, trim each entry, drop
     empties, and handle the single-address case.

5. **Reconcile, do not duplicate.** Each lead carries `remoteIds[]` — `{id, integrationType}`
   pairs where `integrationType` is one of `HUB_SPOT`, `SALES_FORCE`, `MARKETO`, `PARDOT`,
   `PIPE_DRIVE`, `ZOHO`, `ELOQUA`, `WEBHOOK`. If a remoteId already exists for your system,
   update that record rather than creating a new one.

6. **Optional — pull the timeline.** `GET /leadActivityEvents` returns the activity stream
   (`eventType`, `eventNotes`, `associatedLead`, `associatedEvent`, `associatedActivation`,
   `associatedTagOption`). This is the closest thing Mobly publishes to an event feed.

## Rules

- **Rate limit.** One token bucket per API key: default `maxTokens` 20, `refillRate` 1/second.
  So burst up to 20, then hold at 1 request/second. Read `X-RateLimit-Remaining` and
  `X-RateLimit-Reset` on every response. On **429**, sleep for `Retry-After` seconds — do not
  retry immediately.
- **No incremental sync.** There is no `updated_since` filter on `/leads` or `/events`. A
  re-sync re-walks the whole collection; diff on your side.
- **No idempotency key.** This flow is read-only, so retries are safe. Write flows are not — see
  `skills/mobly-capture-and-qualify-lead.md`.
- **Errors carry no code.** A 400 is `{"status":400,"error":"Bad request"}` and nothing more.
  Log the request that produced it; the response will not tell you which field failed.

## Related

- `conventions/mobly-conventions.yml` — pagination, envelope, rate-limit semantics
- `errors/mobly-problem-types.yml` — every documented status
- `data-model/mobly-data-model.yml` — the entity graph
