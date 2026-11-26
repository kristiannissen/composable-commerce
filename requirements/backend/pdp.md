# PDP (Product Detail Page) — backend requirements

Query contract and read path that power the storefront PDP.
Draft. Not yet Gate A approved. Service owner: Catalog.

Cross-references: [frontend/pdp.md](../frontend/pdp.md) owns presentation
and interaction. The data model is the canonical product format
([product-schema/](../product-schema/README.md)): `ProductGroup` with
`hasVariant` products, per-variant `Offer`, variant-level attribute
overrides. Viewer-context and access-control semantics are the same as
[backend/plp.md](plp.md) BR-2, BR-3 and BR-19; stated here only where
the PDP contract adds to them. The storefront sells B2B: no order-cutoff
or same-day-dispatch semantics exist in this contract.

## Query contract (Catalog schema.json)

- BR-1: The Catalog service publishes a PDP query in its `schema.json`:
  input = product group ref or variant ref, viewer context; output =
  the full canonical product document (group fields, all variants with
  `variesBy` axis values, SKU, per-variant Offer, availability and
  `additionalProperty` overrides), the delivery estimate per variant,
  and pack-size/MOQ display values per variant.
- BR-2: Reachable only through the gateway with signed requests
  (X-Signature HMAC, conventions.md); no public unsigned route. Same
  viewer-context scoping as the PLP: merchant-account price context,
  one currency per response, no net prices for anonymous contexts.
- BR-3: A variant-ref query returns the same group document with that
  variant marked as the selected facing; the group is always returned
  whole, never an isolated variant payload.
- BR-4: The PDP query is served from the catalog primary database. The
  search service is not on the PDP read path: a search outage cannot
  break the PDP (backend/plp.md BR-15 degrades the PLP only).
- BR-5: The response is the canonical format document as published, so
  frontend structured-data generation (frontend FR-20) needs no
  server-side transformation. Availability and price are always
  per-variant (canonical schema: per-variant Offer only); no
  group-level offer is ever synthesized.

## Commerce display data

- BR-6: Delivery estimates are returned as concrete dates or date
  ranges in the viewer's timezone, computed server-side; raw speed
  labels ("2 business days") are never part of the contract. Each
  estimate carries its as-of timestamp. The estimate model owner is
  Open question 1.
- BR-7: Pack size and MOQ display values per variant SKU come from the
  Checkout service's published contract: Checkout owns cart validation
  of MOQ and pack sizes (Architecture.md), so their source of truth
  stays behind one service boundary. The storefront composes the PDP
  from gateway calls to both contracts; Catalog never reads Checkout.
  Contract shape is Open question 2.
- BR-8: PDP delivery estimates are advisory: the cart and checkout
  contracts own the binding revalidation. The PDP contract says so
  (scope field on the estimate).
- BR-9: The contract cannot express urgency the data does not
  support: stock statuses and any scarcity messaging derive only from
  real offer and state data. Fabricated low-stock counters are
  unrepresentable, not merely discouraged.

## Consistency and updates

- BR-10: PDP data reflects the catalog primary store, kept current
  through `ProductUpdated` via the transactional outbox. Redelivered
  events are no-ops (processed_events); updates are idempotent. The
  visibility window of an update on the PDP read path is Open
  question 3.
- BR-11: Anonymous PDP responses contain no net prices and may be
  cached and statically rendered; authenticated merchant responses are
  per-request and never served cross-merchant from cache (same rule
  as backend/plp.md BR-18).

## Non-functional

- BR-12: PDP query p95 under 200 ms at normal load. At 5x load the PDP
  degrades by larger latency, never by wrong results.

## Out of scope

- Cart validation, VAT computation and binding order-time delivery
  promises (Checkout service, separate feature files).
- Order-cutoff and same-day-dispatch semantics (consumer-retail
  pattern; the B2B contract models lead time, not cutoffs).
- Reviews, ratings and Q&A data (no source in the platform).
- Recommendations and merchandising (no engine yet).

## Open questions

1. Which service owns the delivery estimate model (candidate:
   Checkout, alongside its VAT and validation role), and does it
   publish those as part of its schema.json or as event projections
   other services keep locally? Decides BR-6 shape.
2. Pack-size/MOQ read contract: Checkout-published query endpoint, or
   `ProductUpdated`-style event projected into a local read model?
3. PDP staleness window for BR-10 (seconds vs. next-request freshness).
4. Per-variant lead time data (frontend Open question 3): does the
   estimate model need per-SKU lead times, or per-warehouse only?
