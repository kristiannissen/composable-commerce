# PDP (Product Detail Page) — frontend requirements

Storefront product detail page for one product group and its variants.
Draft. Sources: Baymard checkout UX research (delivery-date, cutoff-time
and cost-transparency findings, applied at the PDP); canonical product
format ([product-schema/](../product-schema/README.md)). Not yet Gate A
approved.

Cross-references: [backend/pdp.md](../backend/pdp.md) owns the PDP
query contract, viewer scoping and delivery estimate data. This file owns
presentation and interaction only. The list-to-PDP flow (state restore
on back navigation) is owned by [frontend/plp.md](plp.md) FR-15 and is
not restated here.

Scope: the storefront PDP for one product group. Cart, checkout and
account flows are separate feature files; the PDP only hands over to
them.

## URL and variants

- FR-1: One PDP per product group. The selected variant is part of the
  URL, so a fully selected variant is shareable and reopens identically;
  the group URL is the canonical URL (URL shape: Open question 1).
- FR-2: Variant axes (`variesBy`) each render one selector (same
  pattern as PLP FR-6). Selecting values for every axis resolves one
  variant and updates the page in place: image gallery, SKU, price,
  availability, delivery estimate, pack-size/MOQ. No page reload.
- FR-3: Axis combinations with no existing variant, and variants that
  are OutOfStock, stay visible but render as unavailable (dimmed, not
  selectable), never hidden; a selection never dead-ends.

## Gallery

- FR-4: All images of the selected variant render as thumbnails (the
  group image set until a variant with its own image is selected). No
  imageless state at any time.
- FR-5: Images show actual contents; multipacks and kits show all
  included items (same rule as PLP FR-5; misleading pack imagery is a
  defect).
- FR-6: The main image zooms (hover or click); thumbnails and zoom are
  keyboard operable.

## Price and cost transparency

- FR-7: The price shown is the selected variant's Offer with its ISO
  4217 currency (canonical schema: per-variant Offer only). Before a
  full selection, the page shows a "from" price: the facing variant's
  price per the backend rule (backend/plp.md BR-11), labeled as "from".
- FR-8: Net prices render only for an authenticated merchant session.
  Anonymous visitors never see net prices in any served HTML
  (Architecture.md). Anonymous price presentation follows the PLP
  decision (frontend/plp.md Open question 1).
- FR-9: The VAT treatment of every shown price is stated next to it
  (net plus VAT rate, or gross, per the viewer's price context), so the
  cost breakdown is never first revealed at checkout. Baymard: extra
  costs are the top abandonment driver, so cost transparency belongs
  before the buy decision, not at payment.
- FR-10: The buy box links the total-cost picture early: price with VAT
  treatment, the delivery estimate (FR-11) and a shipping-cost basis
  or calculator entry point render above the fold, not only at
  checkout.

## Delivery (Baymard checkout UX findings applied at the PDP)

- FR-11: Delivery is presented as an estimated date or date range
  ("Delivery December 3" / "December 3-5"), never as a bare speed label
  ("2 business days") the user must convert. Estimates come from the
  PDP query (backend BR-8) computed server-side in the viewer's
  timezone; the page performs no timezone math.
- FR-12: When a same-day-dispatch cutoff applies, the cutoff renders as
  a live countdown ("Order within 2 h 15 min: ships today"), computed
  from the server-provided cutoff instant and the viewer's timezone
  (backend BR-9). A static cutoff in another timezone, which forces
  mental conversion, is a defect.
- FR-13: Estimates state their scope: advisory at the PDP, revalidated
  in the cart and at checkout (cart/checkout feature files own that
  revalidation; the PDP never presents itself as the final promise).

## Buy box

- FR-14: The quantity input steps in the selected variant's pack size
  and enforces its MOQ before add-to-cart. The constraint (pack size,
  MOQ) is visible next to the quantity input, sourced from backend
  BR-7; it is never discovered as a cart error afterwards.
- FR-15: Add-to-cart keeps the visitor on the PDP: inline confirmation
  and an updated cart indicator (item count, line total), never a
  navigational redirect to the cart page.
- FR-16: An out-of-stock selected variant renders an honest unbuyable
  state: no add-to-cart, no fabricated urgency. Low-stock or countdown
  messaging renders only from real data (backend BR-10); scarcity
  without data behind it is a defect.
- FR-17: Anonymous visitors get the authenticate CTA per FR-8; whether
  add-to-cart itself is reachable anonymously is Open question 2.

## EU B2C disclosures (full display is PDP scope)

- FR-18: The PDP displays the full disclosure set from the canonical
  format: material composition full text matching the physical label
  wording, the statutory 2-year guarantee plus any commercial
  guarantee with its scope and duration, and the Digital Product
  Passport link with its identifier. Shapes and semantics in
  [product-schema/](../product-schema/README.md); linked, not restated.
- FR-19: Variant-level overrides (`additionalProperty`) replace the
  group value wherever displayed; an override is surfaced, never hidden
  behind the group value.

## Crawlability and performance

- FR-20: Anonymous PDPs are static-first, server-rendered; merchant
  sessions with net prices render per request, per Architecture.md. Net
  prices never appear in statically cached HTML.
- FR-21: The PDP embeds schema.org structured data generated from the
  canonical format: the group as one product with per-variant offers and
  `variesBy`, consistent with the variant-aware feed requirement in
  [product-schema/README.md](../product-schema/README.md) and PLP FR-24.
- FR-22: A failed PDP query shows a retry affordance and the last
  rendered state, not a blank error page; a degraded (stale-label)
  response renders labeled as degraded.

## Accessibility

- FR-23: Variant selectors, gallery, quantity and add-to-cart are
  keyboard operable. On variant switch, the changed price/availability
  region is announced (focus or ARIA live), so the update is
  perceivable without sight.

## Out of scope

- Cart, checkout and their cost/delivery revalidation (separate feature
  files).
- Product reviews, ratings, Q&A (no data source in the platform yet).
- Recommendations, recently viewed, cross-sell bundles.
- Wishlist and price alerts.

## Open questions

1. Variant URL shape: path segments, query param, or hash (affects FR-1
   and BR-14 structured data links)?
2. Can anonymous visitors add to cart (price-less cart line), or does
   add-to-cart require an authenticated merchant session (FR-17)?
3. Is lead-time display per variant needed beyond in/out of stock, and
   with what granularity?
