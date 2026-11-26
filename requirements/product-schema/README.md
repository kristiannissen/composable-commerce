# Product Schema (draft)

Canonical product description format for B2C merchants on the platform.
Machine-readable contract: [product-schema.json](product-schema.json). The JSON
file is a complete worked example (JSON-LD, schema.org vocabulary), not a JSON
Schema; the shapes and property semantics below are the requirement.

Status: **draft** until approved at Gate A.

## Structure

- Root is `schema:ProductGroup`. Everything shared by all variants (name,
  brand, description, manufacturer, composition, guarantees, DPP link) lives on
  the group.
- Each sellable combination (color x size, ...) is a `schema:Product` in
  `hasVariant`, with `sku`, `color`, `size`, its own `Offer`, and an
  `isVariantOf` back-reference to the group.
- `variesBy` lists the full schema.org URLs of the properties that define the
  variant axes. Trim it for single-axis products (size-only or color-only).
- Stable `@id` on every node, so variants can also be embedded on separate
  pages and still resolve to the same entity.
- Per-variant `Offer` only (price/stock differ per combination), never a
  group-level offer.
- A variant that differs from the group (e.g. another fabric) repeats
  `additionalProperty` at variant level; variant values override group values.

## Required fields per variant

| Field | Notes |
|---|---|
| `sku` | mandatory |
| `gtin13` / `mpn` | at least one where available |
| `color`, `size` (or other axis values) | must match group `variesBy` |
| `offers` with `price`, `priceCurrency`, `availability` | mandatory, per variant |

## EU B2C disclosure (mandatory fields)

- **Material composition** (`materialComposition`): machine-readable
  percentage breakdown plus `materialCompositionFullText` matching the
  physical label wording. Textile fibre names follow Regulation (EU)
  1007/2011. Lives on the group unless a variant overrides it.
- **Guarantees** (`hasWarrantyPromise`): the statutory 2-year conformity
  guarantee (Directive (EU) 2019/771) is always published; any commercial
  guarantee is a second `WarrantyPromise` with its own scope and duration.
- **Digital Product Passport** (`subjectOf`): link to the ESPR
  (Regulation (EU) 2024/1781) passport document plus its unique identifier.
  Forward-compatible placeholder: the EU DPP Registry is live since July 2026
  and the harmonised standards (EN 18219:2026 unique identifiers, EN 18222:2026
  APIs) are published, but no product-category implementing act is adopted yet.
  Treat the DPP URL and identifier as per-product PIM fields so each variant
  SKU can resolve to its own passport once its category act applies.

## Notes

- Prices use ISO 4217 currency codes (`DKK`, `EUR`, ...); one Offer per
  market/currency.
- `availability` and `itemCondition` use schema.org URL values, not free text.
- Google Merchant/structured-data validation treats the group as one product;
  the variant pattern is required for variant-aware feeds.
