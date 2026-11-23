# Conventions — hard rules for every agent
Prepended to every call. Breaking any of these is a failed run.

## Shape
- Standalone services speaking only via events. No shared code, no Shared module.
- A service knows a dependency only through its schema.json. Generated types, never imports.
- One DB schema per service; your connection reaches only that schema. Cross-service queries are impossible, don't try.
- Anything cross-cutting belongs to a concrete service or infrastructure.

## Events
- Publish only from your module's mapped class. Every published event = an outbox row in the same transaction.
- Every listener checks the processed_events table first. Duplicate delivery is a no-op.
- Event names: <Thing><PastTense>. Payloads match the published schema.json version.

## HTTP
- Routes under api/<slug>, named <slug>.<action>. No new public routes without approval.
- Request signing: Stripe-style X-Signature header (t=<unix-ts>, v1=HMAC-SHA256 over t + '.' + raw body). Verify with hash_equals, 5-minute window, replay-safe because t is inside the signed payload. Never invent a signing format.
- HMAC is verified by middleware before any controller runs. Nothing trusts the client.
- External calls only via .ai/vendors/<system>.md. If it's not documented there, stop.

## Verification
- You never write or edit step definitions or tests/Behat/Steps/.
- Never weaken, skip or delete a test to reach green.
- Cheapest check first: php -l -> Pint -> PHPStan -> Pest -> arch -> Behat --strict.
- Attempts sharing the same diagnosis of the failure count once. A third
  run only with a fundamentally different diagnosis; otherwise stop and
  report.

## Honesty
- "I don't know" is a valid answer. So is "this is not in my inputs".
- Every claim must be checkable by a script, or named as an opinion.
- Money, VAT, auth and events are where you are most likely wrong; re-check before claiming done.

## Improvement
- Nothing is added "for later". Only what this module needs.
- The same failure twice earns a check or a line here; propose, don't decide.
