# PLP (Product Listing Page) — backend requirements

Query contract and data pipeline that power the storefront PLP.
Draft. Not yet Gate A approved. Service owner: Catalog.

Cross-references: [frontend/plp.md](../frontend/plp.md) owns presentation
and interaction. This file owns the query contract, faceting semantics,
indexing and access control. The product data model is the canonical
product format ([product-schema/](../product-schema/README.md)):
`ProductGroup` with `hasVariant` products, per-variant
`Offer`, variant-level attribute overrides.

Naming: **the search service** is the faceted search engine backing this
contract. No vendor name is part of any contract text; the engine is a
replaceable component behind the Catalog service.

Scope: category browsing and faceted filtering. Keyword search is a later
feature and reuses this contract shape with an added query parameter.

## Query contract (Catalog schema.json)

- BR-1: The Catalog service publishes a PLP query in its `schema.json`:
  input = category ref, filter selections, sort, batch size, batch cursor,
  viewer context; output = listing page (one item per product group, per
  the canonical format), facet definitions with per-option counts, total
  hit count, next cursor.
- BR-2: The PLP query is reachable only through the gateway with signed
  requests (X-Signature HMAC, conventions.md). No public unsigned route.
- BR-3: Viewer context is resolved from the signed session. Responses are
  scoped to the merchant account: products published to that account's
  catalog and prices from its price context. Anonymous context receives
  no net prices at all, per Architecture.md.
- BR-19: Viewer context includes market/currency. Prices and the price
  facet are computed for exactly one currency per response, from the
  canonical format's per-variant offers for that market; a response never
  mixes currencies.

## Faceting semantics

- BR-4: Facet definitions are category-specific, derived from the product
  attributes merchandised for the category.
- BR-5: Every facet option carries a result count computed within the
  current filter context (all other applied filters respected), so a
  selection can never silently yield zero results.
- BR-6: Multi-select within one facet is OR-combined; across facets is
  AND-combined. The contract accepts multi-valued filter parameters.
- BR-7: Zero-count options are returned as zero (visible, dimmed), not
  omitted, so the frontend can render them consistently.

## Sorting and batching

- BR-8: Sort options: relevance (category-defined), price asc, price
  desc, name, newest. The price sort key per listing item is the facing
  price defined in BR-11, in the viewer's price context (BR-3), so card
  price and sort order can never disagree. For anonymous sessions the
  sort key is the anonymous price presentation, see Frontend Open
  questions.
- BR-9: Batching uses cursors, default and maximum batch size defined in
  the contract. Cursor reads within a short window are consistent; a
  stale cursor degrades to the nearest valid position, never an error.
- BR-10: Hit count is returned once per query, exact, not "more than N".

## Variants

- BR-11: One search document per product group, per the canonical format:
  group fields (name, brand, composition, guarantees, DPP reference)
  plus its variants, each carrying its variant axes (`variesBy`
  values), SKU, per-variant Offer (price, currency, availability) and
  its variant-level attribute overrides (`additionalProperty`).
  Variant axes and variant attributes participate in filtering while the
  listing item stays collapsed to one group. The facing rule (which
  variant's image, price and availability represents the group when
  nothing is selected) is deterministic and documented in the contract:
  default = the lowest-priced in-stock variant in the viewer's currency
  (BR-3); final rule is Open question 2.
- BR-20: Availability semantics: a listing item is in stock iff at least
  one of its variants has an InStock offer (canonical schema per-variant
  availability). The availability facet and any stock-based filter
  operate on this rule.

## Indexing pipeline

- BR-12: The primary catalog database is the source of truth. The search
  service is a secondary store: the catalog syncs documents into it; the
  catalog never reads the search service as the only copy of data.
- BR-13: Search documents are built from the canonical product format
  (group with variants and per-variant offers) and updated from
  `ProductUpdated` events (outbox, conventions.md). Redelivered
  events are no-ops (processed_events). Document updates are idempotent.
- BR-14: A category's facet definition change (new merchandised attribute)
  requires a reindex without downtime; the old contract stays valid until
  the new index is live.
- BR-15: Index-read path is read-only; a search outage degrades the PLP
  to an unfiltered, category-ordered fallback list or a honest empty
  result, never stale random data, and the storefront shows the degraded
  state (Frontend FR-18).

## Comparison endpoint

- BR-16: A comparison query takes the selected product group refs and
  returns the attribute comparison table data for the same viewer context
  (BR-3). Values resolve variant-level overrides per the canonical
  format: an overriding variant's value is returned instead of the group
  value, and flagged as an override so the frontend can surface it.

## Non-functional

- BR-17: Filtered PLP query p95 under 200 ms at normal load; facet count
  updates are computed server-side in the same query round-trip.
- BR-18: At 5x normal load the PLP degrades by larger latency, never by
  wrong results: no fallback to unsafely cached per-merchant data.
  Per-merchant results are never served cross-merchant from cache.

## Out of scope

- Keyword search, typo tolerance, autocomplete (later `search` feature
  on the same contract base).
- Semantic/vector search, recommendations.
- Merchandising boosts and pinning.
- Full EU B2C disclosure delivery (composition full text, guarantee
  terms, DPP document): indexed as attributes, displayed on the PDP.

## Open questions

1. Anonymous price presentation for the contract (none, list price, or sentinelled), decides BR-3/BR-8 and Frontend FR-12 wording.
2. Facing rule (BR-11): lowest-priced in-stock variant, or a merchandiser-selected facing variant per group?
3. Cursor window length for BR-9 (seconds of consistency before degrade).
4. Reindex trigger on facet definition change: event-driven or scheduled?
5. Visibility default for all-variants-OutOfStock groups (with Frontend Open question 4).
