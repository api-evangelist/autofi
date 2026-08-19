---
name: Estimate cash, finance and lease payments
description: Calculate an AutoFi payment estimate for a vehicle at a dealer — cash, finance or lease — including fees, taxes, rebates, trade-in and F&I products, without creating a loan application.
api: openapi/autofi-api-openapi.yml
operations:
  - POST /v1/estimate/cash
  - POST /v1/estimate/finance
  - POST /v1/estimate/lease
  - POST /v1/dealer/lookup
generated: '2026-08-14'
method: generated
source: openapi/autofi-api-openapi.yml + https://api.autofi.com/api.html
---

# Estimate cash, finance and lease payments

The three estimate operations are stateless calculators. They persist nothing,
create no loan application, and pull no credit — use them to price a deal on a
vehicle detail page or in a showroom tool before any application exists.

Authorize first — see `skills/autofi-authorize.md`. You need the
`create:estimate` scope (and `lookup:dealers` if you need to resolve a dealer
code).

## Step 0 — resolve the dealer code (optional)

`POST /v1/dealer/lookup` with `brand` (the OEM brand) and `oemSalesCodes[]` (the
external dealer codes the OEM assigns). The response is `{ brand, dealers[] }`
where each dealer carries `code`, `name` and `oemSalesCode`. The `code` is the
AutoFi dealer code every other operation expects.

## Step 1 — choose the operation

| Deal type | Operation | Required fields |
|---|---|---|
| Cash | `POST /v1/estimate/cash` | `dealer`, `vehicle` |
| Finance | `POST /v1/estimate/finance` | `dealer`, `vehicle` |
| Lease | `POST /v1/estimate/lease` | `dealer`, `vehicle`, `offerPreferences` |

All three additionally accept `applicant`, `fees`, `products`, `rebates` and
`tradeIn`. Finance also accepts `nonOemRebates`. Only lease requires
`offerPreferences`.

Send `applicant` even where it is optional — the reference states it is
recommended in order to calculate accurate taxes, since tax treatment depends on
the buyer's address.

## Step 2 — build the request

- `dealer` is a dealer-code-only object; supply the AutoFi `code`.
- `vehicle` carries `vin`, `year`, `make`, `model`, `trim`, `mileage`, `msrp`,
  `invoice`, `dealerRetailPrice`, `stockNumber` and `age`.
- `offerPreferences.apr` is a decimal (2% is `0.02`), `term` is months, and
  `downPayment` is a whole-currency integer. For lease, set `annualMileage`.
- `offerPreferences.pricePlan` marks an OEM discount program for eligible
  employees and family (documented example `Z`).
- `tradeIn` carries `amount`, `payoff` and `ownership` — a financed or leased
  trade with a payoff changes the amount financed materially.
- Do not set `fees[].isTaxable` — it is deprecated on both `FinanceFees` and
  `FinanceEstimateFeeInput`.

## Step 3 — read the estimate

Each operation returns its own pricing stack —
`EstimatePaymentCashPricingStack`, `EstimatePaymentFinancePricingStack` or
`EstimatePaymentLeasePricingStack` — carrying `amountFinanced`, `financeCharge`,
`monthlyPayment`, `numberOfPayments`, `termMonths`, `totalOfPayments`,
`totalTaxes`, `totalProducts` and `totalRebates`, alongside the resolved fee,
tax, rebate and product line items.

For lease, read taxes from `appliedTaxes` and ignore `totalTaxes` and
`taxItems` — both are deprecated for LEASE. On lease also avoid
`cashAppliedToCap`, which is deprecated.

## Step 4 — handle errors

A `400` returns the structured envelope with one `errors[].description` per
invalid field. The reference calls out the common causes: string fields (`dob`,
`email`, `phone`) have format and character restrictions, and numeric fields
(`apr`, `downPayment`, `term`, `timeInMonths`, `monthlyPayment`, `year`) have
range requirements — `apr` must be between 0 and 1.

Estimates are safe to retry: they create nothing, so a failed call can simply be
re-issued, subject to the account rate limit.
