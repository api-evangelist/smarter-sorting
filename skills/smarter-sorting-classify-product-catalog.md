---
name: smarter-sorting-classify-product-catalog
description: >-
  Submit a retail product catalog to the Smarter Sorting Customer Classification API and retrieve
  regulatory classifications (RCRA waste codes, DOT/IATA/IMDG dangerous-goods descriptors, NFPA 704
  ratings, lithium-battery attributes) for each product, using bulk CSV upload and status polling.
api: smarter-sorting-customer-classification-api
generated: '2026-08-28'
method: generated
source: >-
  Grounded in openapi/smarter-sorting-customer-classification-v1-openapi.yml (fetched verbatim from
  https://api.smartersorting.com/classification/v1/docs) and the developer guide at
  https://api.smartersorting.com/docs/index.md. Every operationId below appears in that spec.
operations:
  - bulkUploadProducts
  - getListOfProducts
  - getCountOfProducts
  - getProductByFieldMatch
---

# Classify a product catalog

Base URL `https://api.smartersorting.com`. Staging is `https://api.staging.smarterx.com` — use it
first; the contract is identical.

## Before you start

Authenticate with a bearer token: `Authorization: Bearer <API key>`. The key is generated for you
when you sign in to the developer portal with Auth0 credentials — production keys come from
`https://api.smartersorting.com/docs`, staging keys from `https://api.staging.smarterx.com/docs`.
**There is no key prefix distinguishing the two**, so track which host a key belongs to yourself.

## 1. Upload the catalog

`bulkUploadProducts` — `POST /classification/v1/products/bulk`, `Content-Type: text/csv`.

Every row needs at minimum `upc` and `name`. Optional columns that materially improve the result:
`brand`, `supplier`, `ingredients`, `sds` (a Safety Data Sheet URL), `manual`, `un38` (a UN 38.3
report URL), `battery_chemistry`, `size`, `form`, `external_id`.

Set `external_id` to your own SKU or database id. It is echoed back on every read and is the only
way to join results to your own catalog — the API has no synthetic product id of its own.

A success is **HTTP 202** with `{"batchId": "<uuid>"}`.

> Two things to know before you press send. First, **no operation in the published contract accepts
> a batchId** — you cannot poll it, and you cannot cancel it. Second, there is **no DELETE, cancel
> or undo anywhere in this API**, and no dry-run or validate-only mode. A CSV submitted in error
> cannot be withdrawn through the API; correction is a support matter via
> https://support.smartersorting.com/s/, on no published timetable. Validate your CSV locally and
> run it against staging first.

A malformed body returns `400 Invalid file format`.

## 2. Poll for completion

Classification is asynchronous and there is **no webhook or callback** — polling is the only
completion signal.

Wait at least **30 seconds per product** in the request before the first poll; the provider states
that decisions relying on an LLM vary in processing time. Note also that the maximum number of
products per job is **5**.

`getCountOfProducts` — `GET /classification/v1/products/count?statuses=PENDING,IN_REVIEW` — is the
cheap way to ask "is there anything left?" without paging the collection. Poll that until it
reaches zero.

Statuses are `PENDING` (received and enqueued), `IN_REVIEW` (under evaluation) and
`CLASSIFICATION_COMPLETE` (results available).

## 3. Retrieve results

`getListOfProducts` — `GET /classification/v1/products`.

Parameters: `statuses`, `start_date`, `end_date`, `page_size` (1–100, default 50), `page_token`.

Page with the opaque token: read `pagination.next_page_token` from each response and pass it as
`page_token` on the next call. Stop when it is absent.

**Poll incrementally.** On every run after the first, set `start_date` to the timestamp of your
previous poll. That returns only products updated since, instead of re-paging the entire catalog —
this is the idiom the provider's own guide recommends.

For a single product, `getProductByFieldMatch` — `GET /classification/v1/products/{upc}`. Note the
spec writes this path as `:upc` (Express style, not OpenAPI `{upc}`), so generated clients may emit
a literal `:upc` segment; send the bare UPC value.

## 4. Read the payload

Each product carries two parallel arrays of `{name, value}` pairs: `attributes` (what you submitted
or what enrichment found) and `classifications` (what the model determined).

`value` is untyped in the schema — "can be of any type" — and the attribute vocabulary is
documented only through the spec's example, not as a schema. Do not expect a generated client to
validate it. The names you will see fall into stable families:

- `dot_*` (11 fields) — US DOT 49 CFR: hazard class, packing group, UN number, proper shipping name.
- `iata_*` (10 fields) — IATA Dangerous Goods Regulations, for air.
- `imdg_*` (11 fields) — IMDG Code, for sea, including `imdg_marine_pollutant`.
- `nfpa_health`, `nfpa_flammability`, `nfpa_reactivity`, `nfpa_special` — NFPA 704 fire diamond.
- `waste_rcra_codes` (e.g. `D001`), `waste_state_codes` (e.g. `CA 331, WA WT02`).
- Battery/UN 38.3: `battery_chemistry`, `lithium_watt_hrs`, `un38.3_document_link`, and others.
- Physical: `flash_point`, `ph_min`, `ph_max`, `sds_link`, `ifc_codes`.

## Errors

RFC 9457, `application/problem+json`: `{type, title, status, detail, instance, trace}`. The `trace`
object carries `requestId` — quote it to support. Common cases: `401 Unauthorized` with detail
`No Authorization Header`; `404` when the UPC is not yet in your catalog (upload it first);
`400 Bad request, must provide valid UPC.`

## Retries and limits

No rate limits are published — no documented ceiling, no `429`, no `Retry-After`, and no
`X-RateLimit-*` or `RateLimit-*` response headers. You have no runtime backpressure signal, so be
conservative and back off on your own schedule.

No `Idempotency-Key` is supported. `PUT /classification/v1/products/{upc}` is idempotent by HTTP
semantics and safe to retry. **`POST /classification/v1/products/bulk` is not** — it mints a new
`batchId` on every call, so a retry after a timeout may double-submit with no way to de-duplicate.
Prefer confirming with `getCountOfProducts` over blindly retrying an upload.
