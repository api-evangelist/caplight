---
name: Price a private company on the Caplight secondary market
description: >-
  Resolve a company from a domain or name to a Caplight ID, then pull its MarketPrice, price history,
  live order book and closed trade history to answer "what is this private company worth on the
  secondary market right now, and how did it get there?"
api: openapi/caplight-rest-api-openapi-original.json
operations:
  - GET /v2/companies
  - GET /v1/companies
  - GET /v1/all-marketprices
  - GET /v1/market-price-history
  - GET /v1/market-price-fixed-eod
  - GET /v1/live-orderbook
  - GET /v1/trade-history
  - GET /v1/order-history
generated: '2026-08-09'
method: generated
---

# Price a private company on the Caplight secondary market

The Caplight OpenAPI declares **no `operationId` on any operation**, so every step below names the
operation the way the spec does — `METHOD path`. Proposed operationIds live in
`overlays/caplight-rest-api-overlay.yaml` and are API Evangelist's, not Caplight's.

## Before you start

- Base URL: `https://us-central1-caplight-prod.cloudfunctions.net/api/public`
- Send `api_key: <key>` as a **request header** on every call. There is no bearer token and no OAuth on
  the REST surface. An anonymous call returns `401 {"message":"Invalid/missing API key. ..."}`.
- Everything here is a `GET`. Nothing in this skill mutates state, and there is no idempotency key —
  retries are inherently safe.
- Staging host for rehearsal: `https://us-central1-caplight-staging.cloudfunctions.net/api/public`
  (same auth; see `sandbox/caplight-sandbox.yml`).

## Step 1 — resolve the company to the right ID

Caplight has **two** company ID formats and they are not interchangeable: a v1 ID for `/v1/` endpoints
and a v2 ID for `/v2/` endpoints. Do not guess.

1. Call `GET /v2/companies` with `domain`, `pitchbookId` or `v1Id`. It resolves up to **50 identifiers
   per request** — batch, do not loop one at a time.
2. Read both IDs off `caplightIds` in the response and carry them forward.
3. More than 50 identifiers, or none at all, returns `400`.

If you only need v1 pricing endpoints you can also call `GET /v1/companies` with `domain`,
`caplightId` or `pitchbookId`; its `Company` object embeds the current `MarketPrice`.

## Step 2 — get the current MarketPrice

- Single company: use the `MarketPrice` already embedded on the `GET /v1/companies` response.
- Whole coverage universe: `GET /v1/all-marketprices` — paginated, use it when you are screening rather
  than answering about one name.

## Step 3 — get the price history

- `GET /v1/market-price-history` for the continuous series. Bound it with `startDate` / `endDate`.
- `GET /v1/market-price-fixed-eod` when you need a fixed end-of-day series (use `fixed` and `utcHour`)
  — that is the one to use for anything that has to reconcile against a daily mark.

Both operations accept the `x-price-model` **header** if you need a specific price model.

## Step 4 — show the flow behind the price

The MarketPrice is implied from real flow, so quote the flow, not just the number:

- `GET /v1/live-orderbook` — current bids and offers. `Order.volume` is **deprecated**; read `minVolume`.
- `GET /v1/trade-history` — closed secondary transactions.
- `GET /v1/order-history` — historical orders, for depth and spread over time.

## Step 5 — page correctly

All four list operations use page-number pagination: send `pageNumber` (1-based) and `pageSize`, and read
`pagination.numPages` / `pagination.totalRecords` off the response before deciding to keep paging. There
are no cursors. See `conventions/caplight-conventions.yml`.

## Errors to handle

`errors/caplight-problem-types.yml` has the full catalog. For this flow:

- `401` — missing or invalid `api_key`. Body is `{"message": ...}`, not the `{status,error}` envelope.
- `400` on `GET /v2/companies` — no identifier supplied, or more than 50 in one request.
- `403` — the account lacks Companies V2 access, or the company is outside its whitelist / over its
  annual distinct-company cap. Treat this as an entitlement problem, not a bad request; retrying will not
  help.

Caplight publishes no rate limits and returns no rate-limit headers. Back off on any 5xx rather than
assuming a documented budget.
