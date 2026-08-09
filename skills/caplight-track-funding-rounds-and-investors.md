---
name: Track funding rounds and investors, and keep a local copy fresh
description: >-
  Pull a company's funding history with amounts, valuations, participants and citations, walk the
  investors on its cap table, and then keep a local dataset current with the funding-round updates feed —
  handling Caplight's per-account entitlement caps and field-level restrictions correctly.
api: openapi/caplight-rest-api-openapi-original.json
operations:
  - GET /v2/companies
  - GET /v2/companies/{companyId}
  - GET /v2/companies/{companyId}/funding-rounds
  - GET /v2/funding-rounds/{roundId}
  - GET /v2/funding-rounds/updates
  - GET /v2/companies/{companyId}/investors
  - GET /v1/fund-marks
generated: '2026-08-09'
method: generated
---

# Track funding rounds and investors

The Caplight OpenAPI declares no `operationId`, so operations are named `METHOD path` throughout, exactly
as the spec identifies them.

## Before you start

- Base URL: `https://us-central1-caplight-prod.cloudfunctions.net/api/public`
- Header `api_key: <key>` on every request. All operations are `GET`.
- Every operation in this skill is a **v2** operation, and v2 is where Caplight enforces per-account
  access control. Read the entitlement section below before you write a loop.

## Step 1 — resolve to a v2 company ID

`GET /v2/companies` takes `domain`, `pitchbookId` or `v1Id` and resolves up to 50 identifiers per call.
v1 company IDs will not work on v2 paths. Read `caplightIds` off the response.

`GET /v2/companies/{companyId}` returns the firmographic profile — location, sector, vertical and
keyword tags — if you need context alongside the rounds.

## Step 2 — pull the funding history

`GET /v2/companies/{companyId}/funding-rounds` returns `FundingRoundSnapshot` objects carrying
`amounts` (a `MoneyAmount`), `valuation`, `pps`, `participants` (each resolving to an investor summary)
and `citations` (the sources Caplight used). Paginated.

`GET /v2/funding-rounds/{roundId}` fetches one round directly when you already hold a round ID.

> A `404` from this operation means **"not found or not accessible"** — Caplight deliberately does not
> distinguish a round that does not exist from a round outside your entitlement. Do not treat a 404 as
> proof the round is not in the dataset.

## Step 3 — handle restricted fields

Some accounts have **field-level restrictions**. A restricted field is *omitted from the response* and
its name appears in `restricted.fields` on the funding round object. Restricted fields may include
`amounts`, `valuation`, `pps`, `participants` and `citations`.

Always check for the `restricted` object before reading a field. A missing `valuation` is not a data gap
in the market — it may be a gap in your contract, and those need different handling downstream.

## Step 4 — walk the investors

`GET /v2/companies/{companyId}/investors` returns `CompanyInvestor` records, each with the `rounds` that
investor participated in. That is the join you want for co-investor networks — you do not need to
re-walk the funding-rounds endpoint to build it.

## Step 5 — keep it fresh

`GET /v2/funding-rounds/updates` with `updatedSince` is the incremental feed. Points to watch:

- It returns **only rounds for companies you have access to** — it is entitlement-filtered, not a
  complete firehose.
- `updatedSince` is required and validated; a missing or invalid value returns `400`.
- Page through with `pageNumber` / `pageSize` and check `pagination.numPages`.
- There are **no webhooks and no event stream** — polling this feed is the only push-like mechanism
  Caplight publishes.

## Entitlements — the thing that will actually break your integration

v2 company-scoped endpoints (funding rounds, investors, company details, comps) enforce:

- an optional **whitelist** of permitted companies, and/or
- an optional **annual cap** on the number of *distinct* companies accessed. Accessing the same company
  repeatedly counts **once**.

So a naive "enrich every company in the CRM" loop can burn an annual cap on companies nobody asked about.
Resolve and de-duplicate your target list first, then fetch. `403` means the account lacks v2 access, or
the company is outside the whitelist / over the cap — it is not retryable.

## Related

`GET /v1/fund-marks` (v1, different ID format) gives SEC-filing-derived valuation marks, which are a
useful cross-check on a stale round valuation.

See `conventions/caplight-conventions.yml`, `errors/caplight-problem-types.yml` and
`authentication/caplight-authentication.yml`.
