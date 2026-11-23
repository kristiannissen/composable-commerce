# Audit and escalation checklist

Reference for the manual audit and the stuck-module escalation. The audit
runs every 3-4 delivered modules; the escalation when a module burns its
three Coder runs.

## Evidence rule
A finding only counts when you can quote file, line numbers and the
snippet. If you can't quote it, you don't have it.

## Audit categories, in order
1. Boundary leaks — a module reaching another's code, schema, tables or
   queries.
2. Event drift — payload vs published schema.json; consumer assumptions
   vs publisher guarantees; events published without an outbox row in the
   same transaction.
3. Idempotency — every listener checks processed_events first.
4. Missing error paths — external down, queue redelivery, DB failure:
   dead-letter or a manual-action path exists?
5. Security — unvalidated input, HMAC bypass, secrets in the diff.
6. Cards overstate the code — MODULE.md facts vs reality.
7. OPEN.md items made worse by the diff.

## Severity
- blocking: lost or inconsistent orders, security hole, service-wide
  regression -> fix round or a decision.
- advisory: everything else, including architectural erosion worth
  watching -> OPEN.md (~30 lines, newest first, shrink it when you can).

## Escalation verdict (exactly one)
Input: failing scenario, trimmed output of all attempts, the module diff.
- fix-code — implementation wrong: name the cause and the flaw in the
  approach, not the symptom -> one more Coder round with the diagnosis.
- fix-contract — a dependency's schema or card doesn't match reality ->
  stops, you decide.
- fix-spec — requirements and Gherkin contradict or are untestable ->
  stops, back to Gate A.
Evidence thin -> fix-code with stated uncertainty, never an invented
contract or spec problem.

## Automation trigger
The first audit that repeats the previous one, or ~6+ delivered modules:
turn this checklist into the Hardening agent prompt.
