# AutoFi

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

AutoFi ("The Sales Momentum Company") is an AI-powered automotive commerce platform connecting
online and in-store car buying for dealerships, OEMs, lenders and marketplaces. It was surfaced as a
portfolio company of 500 Global.

## What this profile holds

AutoFi publishes a public REST API — its Lending-as-a-Service surface — documented at
[api.autofi.com/api.html](https://api.autofi.com/api.html). The OpenAPI 3.0.0 document behind that
Redoc page (11 operations, 174 component schemas) is harvested to `openapi/`, and everything in
`authentication/`, `scopes/`, `conventions/`, `errors/`, `data-model/`, `sandbox/`, `rate-limits/`,
`lifecycle/`, `asyncapi/` and `skills/` is derived from it or from the published reference.

Highlights:

- **JWT client-credential auth** — `POST /auth/token` exchanges a clientId/clientSecret for a bearer
  token; seven scope strings are attached to the per-operation security requirement.
- **Lender decisioning** — loan applications are routed to lenders through RouteOne or DealerTrack,
  with decisions returned as OpenAPI `callbacks` to a caller-supplied `callbackUrl`.
- **A real sandbox** — `api-uat.autofi.com`, with in-contract simulation of a named lender's decision
  (including a `delay` in milliseconds) so integrators can test without touching a lender.
- **A live OAuth-protected MCP server** at `www.autofi.com/wp-json/mcp/mcp-oauth-server`, advertised by
  RFC 8414 and RFC 9728 metadata on the marketing host. It is a WordPress content surface, not a
  wrapper around the lending API.

Recorded absences (checked, not assumed): no SDK in any public package registry, no idempotency key,
no pagination, no security.txt, no A2A agent card, no published pricing, no API changelog.

Backed by: 500-global — https://autofi.com
