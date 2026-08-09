---
name: Benchmark a private company with comps and composite indices
description: >-
  Build a defensible benchmark for a late-stage private mark: pull the comparable-company set for a
  target, read comparable performance over time, and place both against Caplight's sector and vertical
  composite indices.
api: openapi/caplight-rest-api-openapi-original.json
operations:
  - GET /v2/companies
  - GET /v2/companies/{companyId}/comps
  - GET /v1/comps-performance
  - GET /v2/composite-index/sectors
  - GET /v2/composite-index/verticals
  - GET /v2/composite-index
  - GET /v1/companies/quarterly-market-summary
generated: '2026-08-09'
method: generated
---

# Benchmark a private company with comps and composite indices

The Caplight OpenAPI declares no `operationId`; operations are named `METHOD path` as the spec names them.

## Before you start

- Base URL: `https://us-central1-caplight-prod.cloudfunctions.net/api/public`
- Header `api_key: <key>`. All `GET`.
- Comps shipped in July 2026 (`changelog/caplight-changelog.yml`) and split across both API versions —
  `/v2/` for the comparable *set*, `/v1/` for comparable *performance*. You will need both ID formats.

## Step 1 — resolve both IDs

`GET /v2/companies` (up to 50 identifiers per call) returns `caplightIds` with the v1 and v2 IDs. Keep
both: step 2 is a v2 path param, step 3 is a v1 query param.

## Step 2 — get the comparable set

`GET /v2/companies/{companyId}/comps` returns `Comparable` records with `ComparableDimension` entries —
the axes along which Caplight judged the company similar. Paginated; the response also carries the
target company as a `PublicCompanySummary`.

Report the dimensions, not just the names. A comp set without its dimensions is an assertion; with them
it is an argument.

## Step 3 — get comparable performance over time

`GET /v1/comps-performance` returns `ComparablePerformance` with a
`PublicComparablePerformanceOverTime` series. This is the "how has this peer group moved" leg of the
benchmark.

## Step 4 — place it against an index

1. `GET /v2/composite-index/sectors` and `GET /v2/composite-index/verticals` — list what you are allowed
   to slice by. **Do not hardcode a sector string**; ask for the list first.
2. `GET /v2/composite-index` with `sector` or `vertical` returns the timeseries
   (`CompositeIndexTimeseriesPoint`) plus its `constituents` and any `CompositeIndexWarning`. Surface the
   warnings — they are the index telling you the slice is thin.

> `GET /v2/composite-index` is **strict about parameters**: it accepts only the documented ones and
> *rejects* unknown parameters with a `400` rather than ignoring them. Do not pass your own tracking
> params through to it. A `404` means no companies matched the filter — that is an empty slice, not an
> outage.

## Step 5 — add the market context

`GET /v1/companies/quarterly-market-summary` (v1 ID, filter with `quarter` / `quarterFrom` / `quarterTo`)
gives the quarterly secondary-market picture to frame the benchmark.

## Pagination and errors

Page-number pagination (`pageNumber`, `pageSize`, `pagination.numPages`) on the list operations. `403`
on any v2 company-scoped call is an entitlement failure — whitelist or annual distinct-company cap — and
is not retryable. Full catalog in `errors/caplight-problem-types.yml`.
