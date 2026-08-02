---
name: Book an Outdoorsy rental and take payment
description: >-
  Turn an accepted quote into a confirmed, paid booking — create the booking, attach add-ons, and
  drive it through its status transitions including the one that charges the renter. This flow
  reserves a physical vehicle and moves real money; it is NOT retry-safe.
api: openapi/outdoorsy-openapi-original.json
operations: [createQuote, createBooking, getBookingById, createBookingItem, copyRentalItemToBooking, updateBooking, bookingChangeStatus, listBookings]
generated: '2026-08-02'
method: generated
source: derived from the published Outdoorsy Swagger contract + https://developers.outdoorsy.com/api
---

# Book an Outdoorsy rental and take payment

> **Read this first.** The Outdoorsy API publishes **no idempotency key**. There is no
> `Idempotency-Key` header, parameter or extension anywhere in the contract. A retried
> `POST /bookings` creates a second booking; a retried payment transition can charge twice. Treat
> every write in this skill as at-most-once and confirm with a human before executing.

## Base URLs

- Production: `https://api.outdoorsy.com/v0`
- Staging (anonymous, no credentials): `https://api.staging.outdoorsy.com/v0`

Rehearse the entire flow on staging. Per the developer portal the demo environment can process
transactions, so the state machine can be exercised end to end without a partner token.

## Authentication

```
Partner-ID: <partner token>
```

Bookings made on behalf of a signed-in user additionally carry `Authorization: <token>`. For
anonymous checkout, Outdoorsy returns an `Anon-Token` response header — persist it and replay it so
the session can be continued and later attached to a real account.

## Steps

1. **Re-quote immediately before booking.**
   Call `createQuote` (`POST /quotes`) again with the exact rental id, dates and add-ons the
   traveller accepted. Prices move; never book against a quote you fetched minutes ago.

2. **Create the booking.**
   Call `createBooking` (`POST /bookings`) with the rental, dates, renter details and the quoted
   line items.

   Before you fire this call: check `listBookings` (`GET /bookings`) filtered by rental and date
   range for an existing booking that matches. This manual pre-check is the only duplicate
   protection available to you.

   Capture the returned booking id. If the call times out or the connection drops, **do not retry** —
   call `listBookings` to find out whether the booking landed.

3. **Attach add-ons.**
   Either `copyRentalItemToBooking` (`POST /bookings/{booking_id}/items/copy`) to pull an item
   straight from the rental's catalogue, or `createBookingItem`
   (`POST /bookings/{booking_id}/items`) to add a bespoke line item.

   Do **not** use `bookingItemsBatchUpdateLegacy` (`POST /bookings/{booking_id}/items/batch`) — it is
   marked `deprecated` in the contract.

4. **Verify before you charge.**
   Call `getBookingById` (`GET /bookings/{id}`) and confirm the total, the dates, the driver and the
   attached services match what the traveller agreed. Surface this to the human for confirmation.

5. **Drive the status transition that takes payment.**
   Call `bookingChangeStatus` (`PATCH /bookings/{id}/status`). Per
   `https://developers.outdoorsy.com/api` this is the transition that updates status *and pays*.

   The response carries `X-Ppp-Token`, a Stripe payment token. Treat it as a credential: never log
   it, never echo it into a transcript, never pass it to a model context.

   **This is the single highest-consequence call in the Outdoorsy API.** Require explicit human
   approval immediately before it. If it fails ambiguously, do not retry — reconcile with
   `getBookingById` first.

6. **Amend, don't recreate.**
   If the traveller wants changes after confirmation, use `updateBooking`
   (`PATCH /bookings/{id}`) for owner-side edits, or the proposal flow
   (`outdoorsy-manage-booking-changes.md`) when the counterparty must agree. Never cancel and
   rebook to make an edit — you lose the pricing and may lose the vehicle.

7. **Archive rather than delete.**
   `deleteBookingById` (`DELETE /bookings/{id}`) archives the booking and hides it from view. It is
   not a cancellation and it is not a refund.

## Conventions to respect

- **Pagination:** `limit` / `offset`, with the count in the `Total-Results` response header.
- **Rate limit:** `X-Rate-Limit-Limit: 2.00` over `X-Rate-Limit-Duration: 1` — about two requests
  per second, keyed on client IP, with no remaining/reset header and no `Retry-After`. Serialise
  writes.
- **Trace headers:** the API accepts `sentry-trace` and `X-DataDog-Trace-ID` on the way in but
  returns **no** correlation id, so log your own request identifier — there is nothing server-issued
  to quote in a support ticket.

## Errors

Core-API envelope: `{"error": "...", "code": "...", "original_error": "..."}`. Branch on `code`.

- `NotAuthorized` (401) — missing or invalid `Partner-ID` / `Authorization`.
- `InvalidFields` (400) — read `original_error` for the offending field.

401 is declared on 256 of the 283 operations and 400 on 180, none with a response schema, so expect
undocumented codes and fail safe on anything you do not recognise. See
`errors/outdoorsy-error-codes.yml`.

## Agent safety contract

| Operation | Consequence | Retry-safe | Human-in-the-loop |
|---|---|---|---|
| `createQuote` | read | yes | no |
| `createBooking` | physical | **no** | **required** |
| `createBookingItem` / `copyRentalItemToBooking` | write | no | recommended |
| `bookingChangeStatus` | physical (payment) | **no** | **required** |
| `updateBooking` | write | no | recommended |
| `deleteBookingById` | write (archive) | no | recommended |

See `agentic-access/outdoorsy-agentic-access.yml` for the full per-operation classification.
