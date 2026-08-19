---
name: Create a lead and attach it to an event
description: Create a Mobly lead from an external capture source, associate it with an event, mark registration/check-in state, and apply the organization's qualifiers safely without an idempotency key.
api: openapi/mobly-rest-api-v0-openapi.yml
base_url: https://core-api.getmobly.com/api/v0
generated: '2026-08-13'
method: generated
source: https://help.getmobly.com/api-reference/readme/mobly-rest-api
operations:
  - GET /tagGroups
  - GET /tagGroups/{tagGroupId}
  - POST /leads
  - PUT /events/{eventId}/leads
  - PATCH /leads/{leadId}
  - GET /leads
---

# Create a lead and attach it to an event

Use this when leads are captured somewhere other than the Mobly app — a partner scanner, a
registration vendor, a form — and need to land in Mobly attached to the right event.

## Before you start

- `x-api-key: <key>` on every request. Keys come from your Mobly CSM.
- **There is no idempotency key on this API.** `POST /leads` is not idempotent: a retried create
  after a timeout produces a second lead. Read the write-safety rule below before you retry
  anything.

## Steps

1. **Read the organization's qualifiers first.** `GET /tagGroups`, then
   `GET /tagGroups/{tagGroupId}` for the ones you care about. Each group publishes
   `allowsMultipleSelection`, `required`, `allEvents`, `usageInstructions` and its `tagOptions[]`
   (`id`, `value`, `displayOrder`). Qualifier values are per-organization — never hardcode them.

2. **Create the lead.** `POST /leads` with `LeadCreateV0Input`. Required:
   `createdByEmail`, `firstName`, `lastName`, `companyName`. Optional: `email`, `phoneNumbers[]`
   (each a `PhoneNumber` with a `PhoneNumberTypeEnum` of MOBILE / LANDLINE / DIRECT / OFFICE /
   HQ / UNKNOWN). `createdByEmail` must be a real Mobly user in the organization — it is how the
   capture gets attributed on the leaderboard.
   Keep the returned `results.id` (a **UUID string**).

3. **Attach it to the event.** `PUT /events/{eventId}/leads` with
   `{"eventLeads":[{"leadId":"<uuid>","registered":true,"checkedIn":false}]}`.
   This operation is an **upsert** — replaying the same body does not create a duplicate
   association, so it is the one write in this flow you can safely retry. Send the batch in one
   call rather than one call per lead; it accepts an array.

4. **Enrich the record as data arrives.** `PATCH /leads/{leadId}` with `LeadPatchV0Input` to fill
   `jobTitle`, `email`, `linkedin`, `address`, `description`, `phoneNumbers`. PATCH is a partial
   update — omitted fields are left alone, and an explicitly `null` field clears it.

5. **Write your own system's id back.** In the same `PATCH /leads/{leadId}`, send
   `remoteId: {"id":"<your-record-id>","integrationType":"WEBHOOK"}` (or the matching CRM enum).
   Do this on the first successful create — it is what makes step 2 recoverable next time.

6. **Check in at the door.** Re-send `PUT /events/{eventId}/leads` with `checkedIn: true` for the
   same `leadId`. Same upsert, safe to repeat.

## Write-safety rule (read this before retrying)

Mobly publishes no `Idempotency-Key` header. If `POST /leads` times out or returns 5xx you do
**not** know whether the lead was created.

- Do **not** blind-retry the POST.
- Instead, page `GET /leads` and match on `firstName` + `lastName` + `companyName` + `createdAt`
  before deciding. If you find it, continue at step 3.
- Better: always write your own id into `remoteId` (step 5) so the next reconciliation can match
  on a key you control instead of on name similarity.

## Rules

- **Rate limit** — 20-request burst, 1/second sustained, per API key. Batch step 3 rather than
  looping. On 429, honour `Retry-After`.
- **Ids are not uniform** — `eventId` is an integer, `leadId` and `tagOptionId` are UUID strings.
- **Errors** — 400 Bad request on schema violations, 401 missing key, 403 invalid key, 404 when
  the event or lead is not visible to your organization. None of them carry a machine-readable
  error code; validate against the schema before sending.

## Related

- `conventions/mobly-conventions.yml` — the full idempotency posture, per operation
- `errors/mobly-problem-types.yml`
- `data-model/mobly-data-model.yml`
