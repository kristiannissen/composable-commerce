# AGENTS.md

House rules for any coding agent working in this repository. These apply
regardless of the tool used (CLI, IDE, orchestration). Read this file and
docs/Architecture.md before changing anything.

## Project map

Headless B2B commerce platform. Standalone services in apps/api/modules/,
communicating only via events. Each module publishes its contracts in its
own schema.json. Agent scaffolding lives in .ai/. Current rules for
pipeline agents live in .ai/conventions.md.

## Hard boundaries

- Work only inside the module you were given. Never edit another module,
  its composer.json, schema.json, or tests.
- Never edit: tests/Behat/Steps/, .ai/steps.md, .ai/conventions.md,
  .ai/prompts/, .ai/hardening/, .github/PULL_REQUEST_TEMPLATE.md,
  MODULE.md fact sections (generated). Human-owned files.
- No cross-service coupling: no imports from another module, no
  cross-schema tables, columns, FKs or queries. One DB schema per module,
  granted to that module only. Migrations touch your module's schema
  alone.
- No new module by hand: use `php artisan make:module <Name>`.
- Never invent APIs, packages, config keys or event fields. The only
  source for external APIs is .ai/vendors/*. If it is not there, stop.

## Verification first

- The failing test is the work order. Make the approved spec green
  (`.feature`, Pest). Never weaken, skip, mark flaky, or delete a test
  to achieve green.
- Reproduce a reported bug before fixing it, and attach proof (the
  failing output, before/after).
- Every change ends with the full chain and must pass:
  php -l -> Pint -> PHPStan -> Pest -> arch -> Behat --strict.
- Red is not delivered. Nothing is done until the chain is green.

## Stops

Stop and report instead of acting when:

- An event, schema.json contract or public route needs to change
- Anything touches pricing, VAT, order totals, auth/HMAC, or migrations
- A task is marked urgent: urgency never skips verification
- Attempts sharing the same diagnosis count once; a third run needs a
  new diagnosis, else stop
"I don't know" is a valid answer. Say it instead of guessing.

## Merge gate

Every delivered change states: what was built, which spec it implements
(test IDs), proof of the green chain, and anything left open. You don't
push what you haven't seen work. A change without proof stays a draft.
