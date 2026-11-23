# Architecture

One system, one document: the services, their contracts, the data rules,
the storefront's role, and how the pieces connect. Implementation detail
for the storefront lives in [Frontend.md](Frontend.md), the build process
in [Pipeline.md](Pipeline.md).

## Shape

Standalone services, each one deployable and extractable without touching
the others. They share no code and no module named "Shared": anything
cross-cutting belongs to a concrete service or to infrastructure. A
service learns about another only through its published schema.json.
Today they run in a single Laravel deployment for cost, with per-service
database schemas and roles to keep the boundaries real.

## Services

| Service | Responsibility | Publishes | Consumes |
|---|---|---|---|
| Catalog | Products/variants, search docs, PDP data, signed prices | ProductUpdated | — |
| Merchant | B2B accounts, CVR + VIES validation, merchant discounts | MerchantApproved | — |
| Checkout | Cart validation (MOQ, pack sizes), VAT rule, order creation | SalesOrderPlaced | — |
| SalesOrder | Order ledger, state machine | SalesOrderInvoiced, SalesOrderRequiresManualAction | SalesOrderPlaced |
| Accounting | Invoice sync, idempotency, manual-action handling | — | SalesOrderPlaced |
| Notification | Alerts for manual-action orders | — | SalesOrderRequiresManualAction |

Dependencies are declared in each MODULE.md (Consumes:) and enforced
by arch tests generated from those lines. A consumer depends on the
publisher's schema.json; types come from generation, never by importing
publisher code.

## Data

One database instance, strictly partitioned:

- One schema per service, one DB role per service, granted only its own
  schema. The roles exist for the application runtime: each service's
  connection string points at its restricted role, so a cross-service
  query written by an agent fails at the database. Humans use a single
  operator account and never switch credentials.
- No cross-service tables, columns, FKs or queries, ever.
- Cross-service knowledge arrives by event and is projected into a local
  read model owned by the consumer (e.g. SalesOrder keeps its own record
  of what it needs from Merchant). Referencing another service's ID is a
  plain column, with validity implied by events.
- Migrations live in the service and touch only its schema.

Reliability, decided once as infrastructure:

- **Transactional outbox.** Every business event is written to the
  publisher's outbox table inside the same DB transaction as the state
  change; a dispatcher publishes to the broker and marks rows sent. No
  event is a hope about process lifetime.
- **Idempotent consumers.** Every listener records the event ID in a
  processed_events table and treats duplicates as no-ops. Bounded
  retries with backoff, then a dead-letter path; a business-level
  variant already exists (SalesOrderRequiresManualAction).
- Both mechanisms are exercised by one Behat suite running a real queue
  driver: kill a consumer mid-handler, assert redelivery and no double
  application.

## Layout

One autoload root in apps/api/composer.json, none per module:

    "autoload": { "psr-4": { "App\\": "app/", "Modules\\": "modules/" } }

Modules\SalesOrder\Listeners\X lives at
modules/SalesOrder/Listeners/X.php. Providers register by convention:
app/Providers/AppServiceProvider.php globs
modules/*/*ServiceProvider.php. Modules are created only via
php artisan make:module <Name>.

## HTTP

- A module with endpoints owns routes.php; its provider loads it under
  api/<slug> with middleware. Route names <slug>.<action>.
- No HTTP surface, no routes.php (Accounting, Notification are
  event-driven).
- routes/api.php holds only non-module endpoints (/api/health).
- No API versioning: a breaking change updates the endpoint and the
  storefront in the same commit. The storefront builds its TypeScript
  types from schema.json in CI, so contract drift is a failed build.

## Frontend role

Static-first with selective server rendering: CMS and category shells
prerendered; cart, checkout, merchant-specific and net-price pages
render per request. Net prices never appear in HTML served to anonymous
visitors. All traffic goes through one public gateway with HMAC-signed
requests; the storefront never addresses a service directly. URL
patterns, CMS, analytics and UX baseline: see
[Frontend.md](Frontend.md).

## Testing

- Approved Gherkin (.feature per module) runs as the acceptance test
  via Behat, against a hand-written, protected step library
  (tests/Behat/Steps/). No LLM and no module owns step definitions;
  modules supply fixture builders only. Http::fake() only for external
  systems.
- Pest arch tests enforce module boundaries (Depends on: lines and
  the schema rules above).
- Unit tests cover VIES failures, VAT rules, outbox writes, listener
  idempotency with fake vendor responses.
- Cheapest check first in every loop: php -l, Pint, PHPStan, Pest,
  arch, Behat --strict.

## Scaling path

Because services are standalone and communicate only through events and
the gateway, extraction is a deploy change: its own deployment unit, its
own database restored from its schema, its queue bindings unchanged,
route the HTTP prefix through the gateway. The first extraction
candidate is Accounting (event-driven, no HTTP surface, clear vendor
boundary).
