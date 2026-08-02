---
name: Manage Outdoorsy listings and availability as an owner
description: >-
  Owner/dealer-side fleet operations — create and maintain rental listings, manage photos, define
  purchasable add-ons and locations, sync availability via iCalendar, and respond to reviews.
api: openapi/outdoorsy-openapi-original.json
operations: [listRentals, createRentalRequest, getRentalById, updateRentalRequest, deleteRentalById, listRentalImages, listRentalItems, createRentalItem, listLocations, createLocation, fetchCalendars, linkCalendar, syncCalendars, unlinkCalendar, exportURL, downloadICSFile, listReviews, createReview, getUserById, updateUser]
generated: '2026-08-02'
method: generated
source: derived from the published Outdoorsy Swagger contract
---

# Manage Outdoorsy listings and availability as an owner

This is the supply side of the marketplace — what an RV owner, dealer or fleet operator does. It is
the surface behind Wheelbase, Outdoorsy's fleet-management product.

## Base URL

`https://api.outdoorsy.com/v0` (staging: `https://api.staging.outdoorsy.com/v0`, no credentials)

## Authentication

`Partner-ID: <partner token>` plus `Authorization: <token>` for the owner's user context. Every
operation here is scoped to the owning account — `owner_id` filters and ownership checks apply.

## Steps

### Listings

1. **Inventory the fleet.** `listRentals` (`GET /rentals`), filtering by `owner_id`,
   `location_ids`, `rental_category`, `make`, `model`, `status` and `archived`.
2. **Create a listing.** `createRentalRequest` (`POST /rentals`). The `Rental` schema carries 125
   properties; `listing_steps` tracks which sections are complete, so read it back and drive the
   owner through the remaining steps rather than assuming a create call finishes the job.
3. **Read one.** `getRentalById` (`GET /rentals/{id}`).
4. **Update.** `updateRentalRequest` (`PATCH /rentals/{id}`) — PATCH semantics, send only changed
   fields.
5. **Retire.** `deleteRentalById` (`DELETE /rentals/{id}`).

### Photos

`listRentalImages` (`GET /rentals/{rental_id}/images`) returns the listing's photography. Images carry
`ai_description` and `ai_position` — Outdoorsy runs AI enrichment over uploads — plus `category`,
`primary`, `interior_primary` and a moderation `status`. Never overwrite an approved image's
`position` without checking `primary`/`interior_primary`, which drive the search card.

### Add-ons

1. `listRentalItems` (`GET /items`) — the owner's catalogue of purchasable extras.
2. `createRentalItem` (`POST /items`) — define a new one. Items attach to a `location_id` and a
   `rental_items_category_id`, and carry a `tax_rate_id`.
3. Metered extras (generator hours, mileage) are modelled as usage-based items with tiers — set the
   tiers, not a flat price, or the renter is billed incorrectly.

### Locations

`listLocations` (`GET /locations`) and `createLocation` (`POST /locations`). A `Location` carries
availability windows, special hours and a tax rate, and is what `current_location_id` on a rental
points at. Dealers with several yards model each as a location.

### Availability

Outdoorsy exposes **iCalendar (RFC 5545) sync** under the `ics-calendars` tag — six operations that
import and export a rental's availability against an external calendar feed:

- `fetchCalendars` (`GET /ics-calendars`) — list the feeds linked to the account.
- `linkCalendar` (`POST /ics-calendars`) — link an external feed to a rental.
- `syncCalendars` (`POST /ics-calendars/sync`) — pull the linked feeds now.
- `unlinkCalendar` (`DELETE /ics-calendars/{calendar_id}`) — remove a feed.
- `exportURL` (`GET /ics/export-url`) — get the outbound feed URL for another channel to subscribe to.
- `downloadICSFile` (`GET /ics-export.ics`) — download the outbound feed directly.

This is how you keep Outdoorsy in step with another booking channel. There is no webhook, so sync is
pull-based on both sides; schedule `syncCalendars` rather than polling tightly.

### Reviews

`listReviews` (`GET /reviews`) filtered by `rental_id` or `user_id`; `createReview`
(`POST /reviews`) to post an owner review of a renter. The `Review` schema carries
`owner_response`, so an owner reply to a renter review is part of the review record, not a separate
message.

### Account

`getUserById` (`GET /users/{id}`) and `updateUser` (`PATCH /users/{id}`). The `User` schema exposes
`missing_fields` and `permissions` — read `missing_fields` to tell an owner exactly what is blocking
payout or listing approval instead of guessing.

## Conventions to respect

- **Pagination:** `limit` / `offset`, total in `Total-Results`.
- **Range filters:** `created_gt`, `created_lt`, `from_gt`, `from_lt`.
- **CSV export:** several list and report operations accept `csv` and `csv-token` to render CSV
  instead of JSON — use this for bulk fleet exports rather than paging thousands of records.
- **Deprecated:** do not use `connectStripeBank` (`POST /users/bank`) — it is marked `deprecated`
  in the contract. Use the current banking operations under the `banks` tag.
- **Rate limit:** ~2 requests/second, IP-keyed, undocumented, no `Retry-After`. Bulk fleet updates
  must be serialised and throttled.
- **No idempotency.** Retried creates duplicate listings, items and locations.

## Errors

`{"error": "...", "code": "...", "original_error": "..."}`. On `InvalidFields` (400) the
`original_error` string names the offending field — surface it to the owner verbatim, it is the most
actionable message the API produces. See `errors/outdoorsy-error-codes.yml`.

## Agent safety contract

Listing management is `write`, not `physical` — no money moves. But a bad `updateRentalRequest` can
de-list a vehicle mid-season, and `deleteRentalById` removes a live listing. Require human approval
for deletes and for any bulk update touching more than one rental.
