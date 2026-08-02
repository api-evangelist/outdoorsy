---
name: Search Outdoorsy rentals and price a trip
description: >-
  Find RVs, campervans and trailers that match a traveller's location, dates and requirements, then
  produce an accurate all-in price quote including add-ons, fees and taxes. Read-only end to end —
  this skill never creates a booking or moves money.
api: openapi/outdoorsy-search-openapi-original.json, openapi/outdoorsy-openapi-original.json
operations: [getAddressForIP, getLocality, searchRentals, getRentalById, listAvailability, listRentalItems, createQuote, services]
generated: '2026-08-02'
method: generated
source: derived from the published Outdoorsy Swagger contracts + https://developers.outdoorsy.com/api
---

# Search Outdoorsy rentals and price a trip

Outdoorsy is a peer-to-peer RV rental marketplace. Discovery runs on a separate host from the
transactional API, so this flow crosses two base URLs.

## Surfaces

| Purpose | Base URL |
|---|---|
| Search / discovery | `https://search.outdoorsy.com` |
| Core API (quotes, availability, add-ons) | `https://api.outdoorsy.com/v0` |
| Staging mirrors of both | `https://search.staging.outdoorsy.com`, `https://api.staging.outdoorsy.com/v0` |

## Authentication

Send your partner token on every production request:

```
Partner-ID: <partner token>
```

Staging requires no credentials at all — develop against staging first. The token is issued only
after Outdoorsy approves your integration (request access at
`https://wheelbase.typeform.com/to/XANb4C`, or email `partners@outdoorsy.co`).

## Steps

1. **Resolve where the traveller is searching.**
   If you were given a place name, resolve it with `getLocality`
   (`GET /localities/{id}`, or `getLocalities` for `GET /localities`). If you were given nothing,
   fall back to `getAddressForIP` (`GET /geodata/my-ip`) — but note this resolves *your* egress IP,
   not the end user's, if you are calling through a server.

2. **Search inventory.**
   Call `searchRentals` (`GET https://search.outdoorsy.com/rentals`) with the traveller's dates,
   location and filters. Page with `page[limit]` (max 100) and `page[offset]`; read the total from
   the `Total-Results` response header, not from the body.

   The response is a JSON:API compound document: `data[]` carries `{id, type: "rentals",
   attributes}`, and `included[]` side-loads related resources (photos arrive as `type: "images"`).
   Resolve `included[]` by `id` + `type` — do not assume ordering. Be aware the response is served
   with `Content-Type: text/plain` despite being JSON:API; parse it as JSON regardless.

   Use `searchRentalsMinimal` (`GET /rentals/minimal`) when you only need identifiers and price,
   and `searchRentalsSimilar` (`GET /rentals/similar`) to widen a thin result set.

3. **Pull the full listing.**
   Call `getRentalById` (`GET https://api.outdoorsy.com/v0/rentals/{id}`) on the core API for the
   authoritative record — amenities, sleeping capacity, cancellation policy, delivery radius,
   instant-book eligibility and the insurance plan attached to the listing.

4. **Confirm the dates are actually open.**
   Call `listAvailability` (`GET /availability`) for the date range. Search results reflect an index
   that can lag; availability is the live answer. Never quote dates you have not checked here.

5. **Enumerate add-ons.**
   Call `listRentalItems` (`GET /items`) to get the owner's purchasable extras — delivery, generator
   hours, mileage packages, cleaning, pet fees. Metered items (generator hours, miles) are tiered,
   so the price depends on projected usage, not just a flat count.

6. **Quote.**
   Call `createQuote` (`POST /quotes`) with the rental id, start and stop dates, and the selected
   add-ons. Despite being a POST this is non-mutating and safe to call repeatedly — it creates no
   reservation and takes no money.

   Then call `services` (`GET /quotes/{id}/services`) to retrieve the services attached to that
   quote (insurance, roadside assistance, weather protection). Present the quote total *including*
   services and taxes — quoting the nightly rate alone is the most common integration mistake.

7. **Present in the traveller's locale.**
   Both surfaces support `locale` (`en-us`, `en-gb`, `en-ca`, `en-au`, `en-nz`, `fr-fr`, `fr-ca`,
   `es-es`, `de-de`, `it-it`) and `currency` (`usd`, `gbp`, `cad`, `eur`, `aud`, `nzd`). Set both
   explicitly rather than relying on defaults.

## Conventions to respect

- **Pagination differs by host.** Search uses `page[limit]`/`page[offset]`; the core API uses
  `limit`/`offset`. Both return `Total-Results`.
- **Range filters** use a `_gt`/`_lt` suffix (`created_gt`, `from_lt`).
- **Sorting** uses `order_by` plus `order`.
- **Rate limits are tight and undocumented.** The API host emits `X-Rate-Limit-Limit: 2.00` with
  `X-Rate-Limit-Duration: 1` — roughly two requests per second, keyed on client IP, with no
  remaining or reset header and no `Retry-After`. Serialise your calls and back off exponentially
  on failure. See `rate-limits/outdoorsy-rate-limits.yml`.

## Errors

The two hosts use different envelopes.

- Core API: `{"error": "...", "code": "NotAuthorized"}`, with an `original_error` string carrying
  per-field detail on `InvalidFields` (400).
- Search API: `{"status": "404", "code": "E102", "detail": "No results"}`.

Neither is RFC 9457. Branch on `code`, not on the human-readable message. Full catalogue in
`errors/outdoorsy-error-codes.yml`.

## Do not

- Do not proceed to `createBooking` from this skill. Booking is a separate, money-moving flow —
  see `outdoorsy-book-a-rental.md`.
- Do not cache quotes. Pricing depends on live availability and seasonal rules; re-quote before
  showing a checkout.
