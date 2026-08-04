# Outdoorsy

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

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
