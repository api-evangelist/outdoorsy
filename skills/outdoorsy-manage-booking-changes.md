---
name: Negotiate changes to an Outdoorsy booking
description: >-
  Propose, review and act on changes to a confirmed booking — date shifts, add-on changes, price
  adjustments — using Outdoorsy's two-sided proposal flow, and communicate with the counterparty
  through the booking conversation.
api: openapi/outdoorsy-openapi-original.json
operations: [getBookingById, getProposals, createProposal, getProposalById, updateProposal, actOnProposal, deleteProposal, batchUpdateProposals, listConversations, createConversation, listMessages, createMessage]
generated: '2026-08-02'
method: generated
source: derived from the published Outdoorsy Swagger contract
---

# Negotiate changes to an Outdoorsy booking

Outdoorsy is a two-sided marketplace: once a booking is confirmed, neither party can unilaterally
change it. Changes go through **proposals**, which the counterparty accepts or declines.

## Base URL

`https://api.outdoorsy.com/v0` (staging: `https://api.staging.outdoorsy.com/v0`, no credentials)

## Authentication

`Partner-ID: <partner token>`, plus `Authorization: <token>` for the acting user's context. The
proposal flow is inherently user-scoped — the API distinguishes the proposing owner from the
responding renter — so a bare partner token is not enough to act on a proposal.

## Steps

1. **Load current state.**
   `getBookingById` (`GET /bookings/{id}`) — read the current dates, items, services, drivers and
   total. Then `getProposals` (`GET /bookings/{booking_id}/proposals`) to see whether a proposal is
   already pending. Never stack a second proposal on top of an open one.

2. **Propose the change.**
   `createProposal` (`POST /bookings/{booking_id}/proposals`). The proposal carries the delta, and
   the response includes `daily_changes` — the per-day price effect of the change. Show the
   traveller the daily breakdown, not just the delta total; a two-day shift into a peak weekend can
   change the price far more than the day count suggests.

3. **Review a proposal.**
   `getProposalById` (`GET /bookings/{booking_id}/proposals/{id}`) for one, or `getProposals` for
   the thread. Read `user_notice_events` on the proposal to understand what the counterparty has
   already been notified about.

4. **Act on it.**
   `actOnProposal` (`GET /bookings/{booking_id}/proposals/{id}/act`) accepts or declines.

   > Note the contract defect: this is a **GET** that mutates state. Do not let a crawler, prefetch,
   > link-preview or speculative agent read touch this URL. Treat it as a write despite the verb.

   Accepting a proposal changes the money owed on the booking. Require explicit human approval.

5. **Amend or withdraw a pending proposal.**
   `updateProposal` (`PATCH /bookings/{booking_id}/proposals/{id}`) to revise,
   `deleteProposal` (`DELETE /bookings/{booking_id}/proposals/{id}`) to withdraw. For several at
   once: `batchUpdateProposals` (`PATCH /bookings/{booking_id}/proposals/batch-update`) and
   `batchDeleteProposals` (`DELETE /bookings/{booking_id}/proposals/batch-delete`).

6. **Talk to the counterparty.**
   Proposals succeed when they are explained. Find or open the thread with `listConversations`
   (`GET /conversations`) or `createConversation` (`POST /conversations`), read it with
   `listMessages` (`GET /messages`), and post with `createMessage` (`POST /messages`).

   A `Conversation` links renter, owner and the bookings in scope; a `Message` can reference a
   booking (`ref_booking_id`) or a rental (`ref_rental_id`). Set those references so the
   counterparty sees the message in context.

   Conversations may be SMS-proxied (`owner_sms_proxy_id`, `renter_sms_proxy_id`). Assume anything
   you write may arrive as a text message on a phone: keep it short, and never include tokens,
   links to internal tooling, or PII beyond what the counterparty already has.

## Conventions to respect

- **Pagination:** `limit` / `offset`; total in the `Total-Results` response header.
- **Unread count:** the `Total-Unread-Activity` response header carries unread conversation
  activity — use it to poll cheaply instead of re-listing messages.
- **No webhooks.** Outdoorsy publishes no event or webhook surface, so proposal acceptance and
  inbound messages can only be discovered by polling. Respect the ~2 req/s rate limit and back off.
- **No idempotency.** A retried `createProposal` creates a duplicate proposal.

## Errors

`{"error": "...", "code": "...", "original_error": "..."}` — branch on `code`
(`NotAuthorized`, `InvalidFields`). See `errors/outdoorsy-error-codes.yml`.

## Agent safety contract

| Operation | Consequence | Human-in-the-loop |
|---|---|---|
| `getProposals` / `getProposalById` | read | no |
| `createProposal` | write (changes money owed if accepted) | required |
| `actOnProposal` | write via GET — accepts/declines and re-prices | **required** |
| `updateProposal` / `deleteProposal` | write | recommended |
| `createMessage` | write (may send an SMS to a real person) | recommended |
