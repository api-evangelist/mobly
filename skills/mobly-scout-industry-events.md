---
name: Search and shortlist industry events with Scout
description: Full-text search Mobly's Scout industry-event database, read booth cost and attendee counts, bookmark the shortlist, and promote a chosen event into a real Mobly event.
api: openapi/mobly-rest-api-v0-openapi.yml
base_url: https://core-api.getmobly.com/api/v0
generated: '2026-08-13'
method: generated
source: https://help.getmobly.com/api-reference/readme/mobly-rest-api
operations:
  - GET /industryEvents
  - GET /industryEvents/{industryEventId}
  - POST /industryEvents
  - POST /industryEvents/{industryEventId}/bookmark
  - DELETE /industryEvents/{industryEventId}/bookmark
  - GET /industryEvents/bookmarks
  - POST /events
---

# Search and shortlist industry events with Scout

Use this to answer "which shows should we exhibit at next year?" against Mobly's market database
of industry events, then turn the winners into events your team can actually staff.

## Before you start

`x-api-key: <key>` on every request. `GET /industryEvents` is the only endpoint in the whole v0
API with real query filters — use them rather than paging blindly.

## Steps

1. **Search.** `GET /industryEvents?q=<full-text search>&limit=50&offset=0`. `q` is the only
   full-text search parameter published anywhere in this API. `eventUrl` does an exact-match
   lookup when you already have the show's URL and just want its record.

2. **Read the economics.** Each `IndustryEvent` carries the fields that decide the shortlist:
   `numberOfAttendees`, `numberOfExhibitors`, `attendeeCost`, `boothCost`,
   `exhibitorIndustries[]`, `attendeePersonas[]`, `keywords[]`, `venueCity` / `venueState` /
   `venueCountry` / `venueName` / `venueFullAddress` / `venueCoordinates`, plus `startDate`,
   `endDate`, `website`, `eventUrl` and `description`.
   - `attendeeCost` and `boothCost` are **strings, not numbers** — they carry currency and range
     text as published. Parse defensively; do not assume a numeric.

3. **Page correctly.** `limit` defaults to 20, max 50. `offset` is a zero-based **page** index.
   Stop when `(offset + 1) * limit >= results.total`.

4. **Bookmark the shortlist.** `POST /industryEvents/{industryEventId}/bookmark`. This is a
   set-membership toggle — sending it twice is harmless. `DELETE` the same path to remove.
   `isBookmarked` on the event record reflects current state.

5. **Review the shortlist.** `GET /industryEvents/bookmarks` returns only the bookmarked set,
   same envelope and pagination as the search.

6. **Add a show Mobly does not know about.** `POST /industryEvents` with
   `IndustryEventCreateV0Input` — only `name` is meaningfully required; everything else is
   nullable. Set `sourceType` / `sourceId` to record where you got it. Correct an existing record
   with `PUT /industryEvents/{industryEventId}` (a full replacement, safe to replay).

7. **Promote to a real event.** An industry event is market data, not something you staff. When
   the team commits, create the working event with `POST /events` and
   `EventCreateV0Input` — required `eventName`, `eventType` (an `EventType` enum value),
   `startDate`, `endDate`, `createdByEmail`; optional `registrationEnabled`.
   Copy `name` and the dates across yourself: **Mobly publishes no promote/convert operation**,
   and no field links the resulting `Event` back to the `IndustryEvent` it came from. Keep that
   mapping on your side.

## Rules

- **`POST /industryEvents` is not idempotent** — a retry after a timeout creates a second record.
  Search `?eventUrl=` or `?q=` first and confirm before re-posting.
- **`POST /events` is not idempotent either.** Same discipline: list and match before retrying.
- **Rate limit** — 20-request burst, 1/second sustained, per API key; honour `Retry-After` on 429.
  A wide `q` sweep will hit this fast.
- **404 means "not visible to your organization"** as often as it means "does not exist".

## Related

- `conventions/mobly-conventions.yml`
- `data-model/mobly-data-model.yml` — IndustryEvent vs Event
- `errors/mobly-problem-types.yml`
