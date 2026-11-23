# Template for .ai/vendors/<system>.md

One file per external system. The only API source for every agent:
if it is not in this file, it does not exist.

Sections:
- Identity: what the system is, what we use it for.
- Environments: base URLs (sandbox, production).
- Auth: scheme, where credentials live (env keys, never in this file).
- Endpoints: one block per endpoint — method, path, request fields,
  response fields, error codes. Field names verbatim from the API.
- Behavior: rate limits, idempotency semantics, timeouts, retry rules.
- Costs: pricing if usage-based.
- Evidence: official documentation URLs and the date they were verified.

Rules:
- Field names are copied from the API, never paraphrased.
- No examples beyond what the docs show; agents extrapolate enough already.
- Stale evidence (> 6 months) is a finding for the audit to flag.
