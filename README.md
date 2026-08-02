# Outdoorsy

Outdoorsy is an RV and campervan rental marketplace founded in 2014 and headquartered in Austin,
Texas, operating across North America, Europe and Australia/New Zealand. The Outdoorsy Group also
runs Roamly (embedded RV insurance) and Wheelbase (RV rental fleet-management software).

Developers integrate through the **Trailblazer Partner API** at
[developers.outdoorsy.com](https://developers.outdoorsy.com/), which offers three tiers: a full REST
API, INSTASearch embeddable JavaScript widgets, and zero-code partner deep links.

## APIs

| API | Base URL | Spec |
|---|---|---|
| Outdoorsy API | `https://api.outdoorsy.com/v0` | [Swagger 2.0](https://api.outdoorsy.com/swagger.json) — 195 paths, 283 operations, 399 definitions |
| Outdoorsy Search API | `https://search.outdoorsy.com` | [Swagger 2.0](https://search.outdoorsy.com/swagger.json) — 29 operations, JSON:API responses |

Both contracts are served anonymously from the API hosts themselves, and both are mirrored on the
open staging hosts (`api.staging.outdoorsy.com`, `search.staging.outdoorsy.com`).

## Links

- [Developer portal](https://developers.outdoorsy.com/)
- [REST API reference](https://developers.outdoorsy.com/api)
- [INSTASearch widgets](https://developers.outdoorsy.com/widgets)
- [Deep links](https://developers.outdoorsy.com/help/deep-links)
- [Request API access](https://wheelbase.typeform.com/to/XANb4C)
- [Website](https://www.outdoorsy.com/) · [Blog](https://www.outdoorsy.com/blog) · [Help](https://www.outdoorsy.com/help) · [GitHub](https://github.com/outdoorsy)

## What is in this repo

`openapi/` verbatim Swagger contracts · `authentication/` · `conventions/` · `errors/` ·
`rate-limits/` · `data-model/` · `lifecycle/` · `conformance/` · `sandbox/` · `components/` ·
`packages/` · `well-known/` · `mcp/` (candidate tools + crosswalk) · `skills/` (four agent skills) ·
`llms/` · `overlays/` · `agentic-access/` · `security/`

See `apis.yml` for the full APIs.json index.

## Notable findings (2026-08-02)

- **Rate limits are signalled but undocumented** — `X-Rate-Limit-Limit: 2.00` over
  `X-Rate-Limit-Duration: 1`, roughly 2 req/s keyed on client IP, with no remaining/reset header
  and no `Retry-After`.
- **No idempotency key anywhere in the contract**, on an API whose `POST /bookings` and
  `PATCH /bookings/{id}/status` reserve a physical vehicle and take payment.
- **No webhooks or event surface.** iCalendar sync is the only push-adjacent mechanism.
- **No `/.well-known/` document of any kind**, so the two public Swagger contracts are
  undiscoverable except by guessing.
- **Two incompatible error envelopes** across the two hosts, neither RFC 9457, and no 4xx/5xx
  response in either contract carries a schema or a description.
- **`actOnProposal` is a `GET` that mutates state** — accepting or declining a booking change
  proposal.
