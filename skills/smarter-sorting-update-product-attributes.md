---
name: smarter-sorting-update-product-attributes
description: >-
  Create, replace or partially update a single product's attributes by UPC in the Smarter Sorting
  Customer Classification API to trigger or refine its regulatory classification.
api: smarter-sorting-customer-classification-api
generated: '2026-08-28'
method: generated
source: >-
  Grounded in openapi/smarter-sorting-customer-classification-v1-openapi.yml (fetched verbatim from
  https://api.smartersorting.com/classification/v1/docs) and the developer guide at
  https://api.smartersorting.com/docs/index.md.
operations:
  - putProductAttributesByIdentifier
  - patchProductAttributesByIdentifier
  - getProductByFieldMatch
---

# Update one product's attributes

Base URL `https://api.smartersorting.com`. Auth: `Authorization: Bearer <API key>`.

Use this when you are correcting or enriching a single product rather than loading a catalog. Both
operations are keyed on the UPC in the path — there is no separate product id.

## Choose PUT or PATCH

`putProductAttributesByIdentifier` — `PUT /classification/v1/products/{upc}` — **fully replaces**
the submitted attribute set. Anything you omit is dropped.

`patchProductAttributesByIdentifier` — `PATCH /classification/v1/products/{upc}` — merges the
attributes you send and leaves the rest alone. This is what you want for a single-field correction.

Both take `application/json` and both return `404` if the UPC is not in your catalog. Upload it
first (via PUT, or the bulk CSV endpoint) before you try to PATCH it.

## What to send

`upc` and `name` are the minimum for a product to be classifiable at all. Beyond that, the
attributes that most change the answer are the ones that feed the regulatory determination:

- `sds` — a Safety Data Sheet URL. The single highest-value field: flash point, pH and hazard
  characteristics are read from it.
- `ingredients` — drives composition-based determinations.
- `battery_chemistry` and `un38` (a UN 38.3 test report URL) — required for anything containing a
  cell or battery, and what separates a shippable item from a restricted one.
- `brand`, `supplier`, `size`, `form` — improve matching and enrichment.
- `external_id` — your SKU or database id; echoed back on read and searchable/filterable.

The provider's guidance is explicit that unexpected classifications are usually an input problem:
if the result surprises you, the response payload explains what drove it, so add the missing
attribute and resubmit rather than disputing the output.

## Then re-poll

An update re-queues the product. Read it back with `getProductByFieldMatch` —
`GET /classification/v1/products/{upc}` — and wait for `status` to return to
`CLASSIFICATION_COMPLETE`. Allow at least 30 seconds.

## Before you write

**This API has no reversal path.** There is no DELETE, no undo, no restore and no version history —
you cannot read back what a product's attributes were before your call. Nothing in the docs states
a correction window.

The practical consequence: **before a PUT, GET the product and keep the response.** A PUT is a full
replace, and the only way to put the previous attribute set back is to send it again from a copy
you made yourself. A PATCH is safer for narrow corrections precisely because it does not discard
what you did not send.

No `Idempotency-Key` is supported, but both PUT and PATCH here are idempotent by HTTP semantics —
re-sending the same body is safe.

## Errors

RFC 9457 `application/problem+json`. `401 Unauthorized` (detail `No Authorization Header`);
`404` when the specified product was not found; `400 Bad request, must provide valid UPC.` The
`trace.requestId` in the body is the handle to quote to https://support.smartersorting.com/s/.
