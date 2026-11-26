# PLP (Product Listing Page) — frontend requirements

Storefront product listing page: category browsing and filtered sets.
Draft. Sources: Baymard PLP UX research; PLP feature list (requirements
chat, 2026-11-26). Not yet Gate A approved.

Cross-references: [backend/plp.md](../backend/plp.md) owns facets, counts,
filter semantics, sorting, pagination and merchant-scoped pricing. This file
owns presentation and interaction only. Cards follow the canonical product
format ([product-schema/](../product-schema/README.md)): one card per
`ProductGroup`, offers per variant, not per group.

Scope: category PLPs and filtered results rendered by the storefront.
Keyword search results reuse the same components but are a separate
feature file (`search`, later).

## Layout

- FR-1: Grid and list view toggle. Grid suits visually driven products,
  list suits spec-heavy categories; the storefront default per category
  comes from the catalog. The active view is part of the URL.
- FR-2: Product cards are uniform across one list: same fields, same
  styling, same positions, so the list is scannable.
- FR-3: Card shows: product image, group name, brand, the selected
  variant's SKU (once one is selected), key category attributes, price
  with currency (FR-6, FR-12, FR-23). Attributes are styled distinctly
  from the title, separated by spacing or line breaks, not run into the
  title.

## Cards and variants

- FR-4: Every card shows an image: the group image at minimum; once a
  variant is selected, that variant's own image when the product data
  contains one. No imageless cards.
- FR-5: Images represent the actual contents: multipacks and kits show
  all included items; misleading pack imagery is a defect.
- FR-6: One card per product group, never per variant. The group's
  variant axes (`variesBy`, e.g. color x size) each render a
  selector on the card (swatches for color, size buttons, per axis).
  Selecting values for every axis resolves one variant and updates the
  card in place: image, SKU, price and availability switch to that
  variant's own Offer. Unselected axes keep their values selectable; a
  combination no variant exists for renders as unavailable instead of
  failing on selection.

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
  that category), not a fixed global set. Variant axes (`variesBy`
  values such as color and size) are always facetable.

## Prices

- FR-12: Net prices render only for an authenticated merchant session.
  Anonymous visitors never see net prices (Architecture.md: net prices
  never render for anonymous visitors). Default anonymous presentation is
  "log in for prices"; see Open questions.
- FR-23: Prices render with their ISO 4217 currency (per-variant Offer
  in the canonical schema). Before a variant is selected, the card shows
  a "from" price: the facing variant's price per the backend rule
  (BR-11). Once a variant is selected, the card shows that variant's
  exact Offer price and availability.

## Sorting

- FR-13: Sort by relevance, price low-high, price high-low, name, newest.
  Default: relevance. Price sorting uses the viewer's price context
  (FR-12); the sort key per product group is the same facing price as
  BR-11, so card price and sort order never disagree. Active sort is part
  of the URL.

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

- FR-19: Product groups can be selected for comparison from the list
  (spec-heavy categories). The comparison view shows a scannable
  attribute table with a "show differences only" toggle. Attribute
  values resolve variant-level overrides per the canonical schema: a
  variant's `additionalProperty` overrides the group value, and
  the row surfaces the override instead of hiding it. Max selectable
  groups: see Open questions.

## Accessibility and performance

- FR-20: Filters, sorting, pagination and chips are keyboard operable,
  with focus management in the mobile drawer.
- FR-21: First batch is server-rendered (static-first, selective SSR per
  Architecture.md) so category URLs are crawlable.
- FR-22: Facet counts after a selection arrive in one request; selecting
  a facet value does not trigger spinner-per-card re-rendering.
- FR-24: PLP pages embed schema.org structured data generated from the
  canonical product format: the group as one product with per-variant
  offers, consistent with the variant-aware feed requirement in
  [product-schema/README.md](../product-schema/README.md).

## Out of scope

- Keyword search, autocomplete and search-as-you-type (separate `search` feature).
- Personalization, recommendations, recently viewed.
- Merchandising overrides (boosting, pinning).
- EU B2C disclosure presentation (material composition full text,
  guarantee terms, Digital Product Passport link): indexed and compared
  as attributes, but their full display is PDP scope.
- Multi-currency/market switching UI: the currency comes from the viewer
  context, one currency per response.

## Open questions

1. Anonymous price presentation: "log in for prices" chip, list price, or hidden price column?
2. Comparison limit (3, 4 or 5 product groups)?
3. Does the storefront need per-category default view from the catalog contract, or one global default?
4. Are groups with all variants OutOfStock visible (unbuyable) or hidden by default?
