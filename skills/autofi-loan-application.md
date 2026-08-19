---
name: Submit a loan application and receive lender decisions
description: Create an AutoFi loan application for an applicant, dealer and vehicle, receive lender decisions on a callback URL, and read the resolved pricing stack — including how to simulate specific lender outcomes in the UAT sandbox.
api: openapi/autofi-api-openapi.yml
operations:
  - POST /v1/loan-application
  - GET /v1/loan-application/{loanApplicationId}
  - GET /v1/loan-application/{loanApplicationId}/externalResources
generated: '2026-08-14'
method: generated
source: openapi/autofi-api-openapi.yml + https://api.autofi.com/api.html
---

# Submit a loan application and receive lender decisions

This is AutoFi's marquee flow: a credit application is routed through RouteOne
or DealerTrack to lenders, and their decisions come back to you.

Authorize first — see `skills/autofi-authorize.md`. You need the
`create:loanapplications` and `read:loanapplications` scopes.

## Step 1 — create the application

`POST /v1/loan-application`

Required top-level fields: `applicant`, `callbackUrl`, `dealer`,
`offerPreferences`, `vehicle`. Optional: `cosigner`, `fees`, `products`,
`rebates`, `tradeIn`, `simulate`.

Notes that will save you a `400`:

- `dealer.code` is a 4-character uppercase alphanumeric code assigned by AutoFi
  (documented example `76KR`), and `dealer.middleman` is `routeOne` or
  `dealertrack`.
- `offerPreferences.apr` is a decimal, not a percentage — 2% is `0.02`.
- `offerPreferences.requestedOfferType` is `FINANCE` or `LEASE`.
- Phone numbers are exactly 10 digits, no punctuation.
- `callbackUrl` must be a valid absolute URL. It is required.
- Do **not** send `applicant.incomeReported` or `applicant.creditScore` — both
  are deprecated. Use `employmentIncome` and `otherIncome`.

The 200 response returns `loanApplicationId`, `referenceId`, `url` (the consumer
experience, which expires 30 days after creation), `agentUrl`, and
`loanApplicationExpires`.

## Step 2 — receive callbacks

AutoFi POSTs to your `callbackUrl` on each state change. The body is the same
document `GET /v1/loan-application/{loanApplicationId}` returns, plus a
`timestamp`. Trigger states:

`BEGAN_APPLICATION`, `SUBMITTED`, `PENDING`, `APPROVED`, `DECLINED`, `ACCEPTED`,
`FI_COMPLETED`, `ERROR`.

**There is no retry.** The reference states plainly that callbacks do not have a
re-try function, and there is no signature to verify the sender. Build for this:

1. Accept and `200` the callback fast, then process asynchronously.
2. Treat callbacks as a latency optimisation, not as the source of truth.
3. Reconcile by polling `GET /v1/loan-application/{loanApplicationId}` on a
   schedule until the application reaches a terminal state.

## Step 3 — read the decisions and pricing

`GET /v1/loan-application/{loanApplicationId}` — the path parameter is a
24-character string (documented example `6148d994e93d8a0018be1234`).

The response carries `state`, `decisions[]`, `creditScore`, `offerAvailable`,
`offerPreferences`, `selectedProducts`, `tradeIn`, `appliedFees`, `appliedTaxes`
and `pricingStack`.

Each `decisions[]` entry has `consumerState` (`APPROVED` / `DECLINED` /
`PENDING`), `lenderReferenceId`, `middleman`, `isAccepted`, an `error.message`
when the decision status is `ERROR`, and `comments.raw[]` — free-text lender
comments such as `APPROVED WITH COSIGNER`. Those comments are unstructured
strings; do not parse them for control flow.

`pricingStack` is only populated once `state` is `ACCEPTED`. It is a `oneOf`
between `FinancePricingStack` and `LeasePricingStack` — branch on
`offerPreferences.requestedOfferType`. Read taxes from `appliedTaxes`, not from
the deprecated `taxItems` / `totalTaxes` fields.

## Step 4 (experimental) — DMS resources

`GET /v1/loan-application/{loanApplicationId}/externalResources` returns
`dms.cdk` links into the dealer's DMS. It carries an "Experimental" stability
badge — do not build a production dependency on its shape.

## Testing without touching a lender

In UAT, add a `simulate` object to the create request:

```json
{
  "simulate": {
    "environment": "AUTOFI_TEST",
    "decisions": [
      { "lender": "CHASE", "decision": "APPROVED", "delay": 75000 }
    ]
  }
}
```

`environment` is `AUTOFI_TEST` (AutoFi's own test environment, requires the
`decisions` array) or `LENDER_TEST` (end to end against the RouteOne/DealerTrack
lender test environment). `decision` can be `APPROVED`, `DECLINED`, `PENDING`,
`APPROVED_WITH_COUNTEROFFER`, `CONDITIONAL` or `CONDITIONAL_WITH_COUNTEROFFER`,
and `delay` is milliseconds — use it to test your callback timeout handling.
Published lender codes are listed in `sandbox/autofi-sandbox.yml`.

When `simulate` is omitted the application follows the environment default: UAT
approves for all lenders, production submits to RouteOne/DealerTrack for live
decisioning.
