# PLP (Product Listing Page) — backend requirements

Query contract and data pipeline that power the storefront PLP.
Draft. Not yet Gate A approved. Service owner: Catalog.

Cross-references: [frontend/plp.md](../frontend/plp.md) owns presentation
and interaction. This file owns the query contract, faceting semantics,
indexing and access control.

Naming: **the search service** is the faceted search engine backing this
contract. No vendor name is part of any contract text; the engine is a
replaceable component behind the Catalog service.

Scope: category browsing and faceted filtering. Keyword search is a later
feature and reuses this contract shape with an added query parameter.

## Query contract (Catalog schema.json)

- BR-1: The Catalog service publishes a PLP query in its `schema.json`:
  input = category ref, filter selections, sort, batch size, batch cursor,
  viewer context; output = product page (cards payload), facet definitions
  with per-option counts, total hit count, next cursor.
- BR-2: The PLP query is reachable only through the gateway with signed
  requests (X-Signature HMAC, conventions.md). No public unsigned route.
- BR-3: Viewer context is resolved from the signed session. Responses are
  scoped to the merchant account: products published to that account's
  catalog and prices from its price context. Anonymous context receives
  no net prices at all, per Architecture.md.

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
  desc, name, newest. Price sorting uses the viewer's price context
  (BR-3); for anonymous sessions price sorting operates on the anonymous
  price presentation, see Frontend Open questions.
- BR-9: Batching uses cursors, default and maximum batch size defined in
  the contract. Cursor reads within a short window are consistent; a
  stale cursor degrades to the nearest valid position, never an error.
- BR-10: Hit count is returned once per query, exact, not "more than N".

## Variants

- BR-11: Variants are indexed under one parent product; the PLP returns
  one listing item per product. Variant attributes participate in
  filtering while the product stays collapsed. Default variant facing
  (which image/price represents the card), Catalog decides and documents
  in the contract.

## Indexing pipeline

- BR-12: The primary catalog database is the source of truth. The search
  service is a secondary store: the catalog syncs documents into it; the
  catalog never reads the search service as the only copy of data.
- BR-13: Search documents are built from the catalog schema and updated
  from `ProductUpdated` events (outbox, conventions.md). Redelivered
  events are no-ops (processed_events). Document updates are idempotent.
- BR-14: A category's facet definition change (new merchandised attribute)
  requires a reindex without downtime; the old contract stays valid until
  the new index is live.
- BR-15: Index-read path is read-only; a search outage degrades the PLP
  to an unfiltered, category-ordered fallback list or a honest empty
  result, never stale random data, and the storefront shows the degraded
  state (Frontend FR-18).

## Comparison endpoint

- BR-16: A comparison query takes the selected product refs and returns
  the attribute comparison table data for the same viewer context (BR-3).

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

## Open questions

1. Anonymous price presentation for the contract (none, list price, or sentinelled), decides BR-3 and Frontend FR-12 wording.
2. Default variant facing rule (cheapest in price context? selected by merchandiser?).
3. Cursor window length for BR-9 (seconds of consistency before degrade).
4. Reindex trigger on facet definition change: event-driven or scheduled?
