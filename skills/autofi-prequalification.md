---
name: Run a prequalification for a shopper
description: Submit an AutoFi prequalification for a car shopper and read back the credit score range, handling the documented not-found and service-unavailable outcomes this operation uniquely returns.
api: openapi/autofi-api-openapi.yml
operations:
  - POST /v1/prequalification
generated: '2026-08-14'
method: generated
source: openapi/autofi-api-openapi.yml + https://api.autofi.com/api.html
---

# Run a prequalification for a shopper

Prequalification tells a shopper what they are likely to qualify for before any
credit application is submitted. It is the lightest-weight credit-adjacent
operation AutoFi publishes.

Authorize first — see `skills/autofi-authorize.md`. You need the
`create:prequalification` scope.

## Step 1 — submit the applicant

`POST /v1/prequalification` with a required `applicant` object. Within it these
are required:

- `name` — full name
- `address` — `street`, `city`, `state`, `zip` are required; `street2` optional
  (documented example: `1234 Main St`, `Madison`, `WI`, `53714`)
- `phone` — exactly 10 digits matching `^[2-9][0-9]{9}$` (no dashes, no country
  code, cannot start with 0 or 1)
- `email`
- `employmentIncome` — a non-negative number

Validate the phone pattern client-side. It is the single most common `400` on
this operation, and AutoFi returns it as a field description rather than a code.

## Step 2 — read the result

The 200 response nests everything under `prequalification`. Read
`creditScoreRange` — **not** `ficoRange`, which is deprecated in favour of it and
will be removed.

## Step 3 — handle all four documented outcomes

This is the only AutoFi operation that documents a `404` and a `503`, so handle
them explicitly rather than falling through to a generic error path.

| Status | Envelope | What it means | What to do |
|---|---|---|---|
| `200` | `{ prequalification: {...} }` | Result returned | Show `creditScoreRange` |
| `400` | `{ code, message, errors[] }` | Validation failure | Surface each `errors[].description` |
| `404` | `{ "error": "Error: Not Found" }` | No result located | Present as "no prequalification available", not as an error |
| `503` | `{ code: 503, message: "Service Unavailable", errors: [...] }` | An AutoFi service is down | Retry with backoff; check https://status.autofi.com |

Note that `404` uses the flat `{error}` envelope while `400` and `503` use the
structured one. Parse both shapes.

## Handling constraints

There is no idempotency key on this operation. A retry after a timeout submits a
second prequalification — treat a network timeout as unknown rather than failed,
and avoid automatic retries on anything other than the `503`.

Prequalification handles consumer PII (name, address, phone, email, income). Log
the `X-Request-Id` from the response for tracing rather than the request body,
and follow AutoFi's published privacy terms at
https://autofi.com/privacy-policy/.
