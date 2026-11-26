# PLP (Product Listing Page) — frontend requirements

Storefront product listing page: category browsing and filtered sets.
Draft. Sources: Baymard PLP UX research; PLP feature list (requirements
chat, 2026-11-26). Not yet Gate A approved.

Cross-references: [backend/plp.md](../backend/plp.md) owns facets, counts,
filter semantics, sorting, pagination and merchant-scoped pricing. This file
owns presentation and interaction only.

Scope: category PLPs and filtered results rendered by the storefront.
Keyword search results reuse the same components but are a separate
feature file (`search`, later).

## Layout

- FR-1: Grid and list view toggle. Grid suits visually driven products,
  list suits spec-heavy categories; the storefront default per category
  comes from the catalog. The active view is part of the URL.
- FR-2: Product cards are uniform across one list: same fields, same
  styling, same positions, so the list is scannable.
- FR-3: Card shows: product image, title, brand, model/SKU, key category
  attributes, price (per FR-12). Attributes are styled distinctly from the
  title, separated by spacing or line breaks, not run into the title.

## Cards and variants

- FR-4: Every product shows an image. No imageless cards.
- FR-5: Images represent the actual contents: multipacks and kits show
  all included items; misleading pack imagery is a defect.
- FR-6: One card per product, not per variant. Variant options appear as
  interactive swatches (e.g. color); selecting one updates the card image
  in place, without leaving the PLP.

## Filtering UI

- FR-7: Desktop: persistent filter panel alongside the grid. Mobile: a
  fixed "Filter" button opens a sticky filter drawer.
- FR-8: Facet options show a result count next to each value (from the
  backend facet data). Zero-count options are visible but dimmed and not
  selectable.
- FR-9: Multiple values within one facet are OR-combined and can be
  selected together. Multiple facets are AND-combined. Counts update for
  the current filter context after each selection.
- FR-10: Applied filters render as chips above the grid. Each chip is
  individually removable; a "clear all" control resets the filter context.
- FR-11: Facet set is category-specific (the attributes merchandised for
  that category), not a fixed global set.

## Prices

- FR-12: Net prices render only for an authenticated merchant session.
  Anonymous visitors never see net prices (Architecture.md: net prices
  never render for anonymous visitors). Default anonymous presentation is
  "log in for prices"; see Open questions.

## Sorting

- FR-13: Sort by relevance, price low-high, price high-low, name, newest.
  Default: relevance. Price sorting follows the viewer's price context
  (FR-12). Active sort is part of the URL.

## Batching and flow

- FR-14: Batching uses a "Load more" control, not infinite scroll. Batch
  sizes are set per device (mobile smaller than desktop) from the backend
  contract.
- FR-15: Returning to the PLP from a PDP (browser back, header back)
  restores the scroll position, loaded batches, filters and sort exactly.
- FR-16: All PLP state (filters, sort, view, batch count) is encoded in
  URL query params. A URL fully reproduces the same view; the back button
  never resets the list.

## Empty and error states

- FR-17: A filter combination with zero results shows an empty state:
    which filters produced no results (as chips), a clear-all control,
    and category suggestions. Never a blank page.
- FR-18: A failed PLP query shows a retry affordance and the last
    successfully rendered list state, not an error page discard.

## Comparison

- FR-19: Products can be selected for comparison from the list
  (spec-heavy categories). The comparison view shows a scannable
  attribute table with a "show differences only" toggle. Max selectable
  products: see Open questions.

## Accessibility and performance

- FR-20: Filters, sorting, pagination and chips are keyboard operable,
  with focus management in the mobile drawer.
- FR-21: First batch is server-rendered (static-first, selective SSR per
  Architecture.md) so category URLs are crawlable.
- FR-22: Facet counts after a selection arrive in one request; selecting
  a facet value does not trigger spinner-per-card re-rendering.

## Out of scope

- Keyword search, autocomplete and search-as-you-type (separate `search` feature).
- Personalization, recommendations, recently viewed.
- Merchandising overrides (boosting, pinning).

## Open questions

1. Anonymous price presentation: "log in for prices" chip, list price, or hidden price column?
2. Comparison limit (3, 4 or 5 products)?
3. Does the storefront need per-category default view from the catalog contract, or one global default?
4. Are out-of-stock products visible (unbuyable) or hidden by default?
