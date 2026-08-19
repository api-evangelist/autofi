---
name: Authorize against the AutoFi API
description: Exchange AutoFi API client credentials for a JWT and call any AutoFi operation with it, honouring the documented scopes, error envelopes and rate-limit headers.
api: openapi/autofi-api-openapi.yml
operations:
  - POST /auth/token
generated: '2026-08-14'
method: generated
source: openapi/autofi-api-openapi.yml + https://api.autofi.com/api.html
---

# Authorize against the AutoFi API

Every AutoFi operation except the token exchange itself requires a bearer JWT.
The AutoFi OpenAPI declares no `operationId` values, so operations are addressed
by method and path throughout this skill.

## Pick the environment first

| Environment | Base URL |
|---|---|
| Test Sandbox (UAT) | `https://api-uat.autofi.com` |
| Production | `https://api.autofi.com` |

AutoFi does **not** use test-vs-live key prefixes. The credential shape is
identical in both environments — the host you call is what selects the
environment. Never point sandbox credentials at the production host.

## Step 1 — exchange credentials for a token

`POST /auth/token` with `Content-Type: application/json` and a body of:

```json
{ "clientId": "<your client id>", "clientSecret": "<your client secret>" }
```

This operation declares an empty `security` requirement, so send no
`Authorization` header on it.

The 200 response carries:

- `access_token` — a JSON Web Token
- `expires_in` — seconds until expiry (the documented example is `86400`)
- `token_type` — `Bearer`

Cache the token and reuse it until `expires_in` elapses. Do not request a new
token per call — AutoFi applies an account-level request quota.

## Step 2 — call an operation

Send `Authorization: Bearer {access_token}` on every other operation. The
operation you call must be covered by a scope your API client holds:

| Scope | Operations |
|---|---|
| `create:loanapplications` | `POST /v1/loan-application` |
| `read:loanapplications` | `GET /v1/loan-application/{loanApplicationId}`, `GET /v1/loan-application/{loanApplicationId}/externalResources` |
| `lookup:dealers` | `POST /v1/dealer/lookup` |
| `create:dealmaker` | `POST /v1/dealmaker` |
| `create:dealmakercredit` | `POST /v1/dealmaker/credit-application` |
| `create:estimate` | `POST /v1/estimate/cash`, `POST /v1/estimate/finance`, `POST /v1/estimate/lease` |
| `create:prequalification` | `POST /v1/prequalification` |

## Step 3 — handle the two error envelopes

AutoFi returns **two different shapes**. Parse both.

Authorization and not-found errors are flat:

```json
{ "error": "UnauthorizedError: invalid token" }
```

Documented 401 variants: `invalid token`, `No authorization token was found`,
`Format is Authorization: Bearer [token]`, `jwt malformed`. All four mean the
same thing operationally — request a new token and retry once.

Everything else uses the structured envelope:

```json
{ "code": 400, "message": "Validation Error", "errors": [ { "description": "..." } ] }
```

Surface every `errors[].description` to the caller; AutoFi returns one entry per
invalid field.

## Step 4 — respect the rate limit

Read these headers off every response:

- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (UNIX timestamp)
- `Retry-After` (seconds) on a 429
- `X-Response-Time` (milliseconds) for latency telemetry

On `429` the body is `{"code": 429, "message": "Account limit exceeded."}`. Back
off until `Retry-After` seconds have elapsed. The window length is not published,
so never compute your own — honour the header.

## What AutoFi does not give you

- **No idempotency key.** No creation endpoint accepts one. A retried POST
  creates a second record, except on Dealmaker where a reused `referenceId`
  returns `409`.
- **No refresh token.** Re-run `POST /auth/token` when the JWT expires.
- **No OAuth metadata for this API.** `/.well-known/openid-configuration` and
  `/.well-known/oauth-authorization-server` are 404 on `api.autofi.com`.
