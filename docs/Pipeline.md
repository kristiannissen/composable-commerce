# Agentic pipeline

Built for one engineer on a small budget. You give instructions; agents turn them into a delivered module. Orchestration is deterministic (Artisan commands), agents are stateless calls, and nothing carries over between runs except files in the repo.

## Flow

One engineer, one blocking gate per module. Everything else is a script, a skim or a batch.

```text
modules/<Module>/INSTRUCTIONS.md                         ← you
  → php artisan ai:requirements <Module> --chat    you + Requirements agent → requirements.md
  → php artisan ai:approve <Module> requirements   ── Gate A (you; the only blocking gate)
  → php artisan ai:gherkin <Module>                Gherkin agent → <module>.feature; scripts check it
  → php artisan ai:approve <Module> gherkin        skim the @E scenarios
  → php artisan ai:build <Module>                  Coder (Pi) until lint, PHPStan, Pest, arch, Behat, Infection pass;
                                                   every new event has an outbox write, every listener a dedup check.
                                                   A module touching the event bus runs the queue-driver Behat once.
                                                   attempts sharing a diagnosis count once; a third run needs
                                                   a new diagnosis, else Hardening escalation. Max per module:
                                                   three Coder runs + one escalated attempt.
  → php artisan ai:deliver <Module>                card facts checked, SYSTEM_MAP.md regenerated, spend logged, commit

Periodic (every 3–4 delivered modules, or weekly):
  → php artisan ai:harden --since=<last-audit>     Hardening audit of everything changed
  → php artisan ai:build <Module> --findings=<report>   fix rounds for blocking findings
```

Approval is a commit of the artefact. A command refuses to run if the previous artefact changed after approval. `ai:deliver` refuses to run while anything is red or the card doesn't match the code. Audits never block delivery.

Failures become pipeline improvements by one convention line in `.ai/conventions.md`: when the same failure shows up twice (audits and escalations are the record), add a check or a convention line. No learning agent, no lesson files.

## Your time

| Step | What you do | Time |
|---|---|---|
| `INSTRUCTIONS.md` | Write business lines and technical notes | 10 min |
| Requirements chat | Answer the agent's edge-case questions | 30–60 min |
| Gate A | Approve | 2 min |
| Gherkin | Skim the `@E` scenarios | 5 min |
| Escalation `fix-spec` / `fix-contract` | Decide | Rare |
| Periodic audit | Triage findings into fix rounds and `OPEN.md` | ~30 min per run |

## Agents

| Agent | Input (all it gets) | Output | Must NOT |
|---|---|---|---|
| Requirements | `conventions.md`, `INSTRUCTIONS.md` (Business + External systems), `SYSTEM_MAP.md`, cards of modules it names, current draft + your last answer | `requirements.md` | mention code or frameworks; invent rules or integrations; finish with open questions |
| Gherkin | `conventions.md`, approved `requirements.md`, `.ai/steps.md`, dependency cards | `<module>.feature` | add scope; skip an `[R-n]` or `[E-n]`; use steps outside the vocabulary |
| Coder (Pi) | `conventions.md`, requirements, feature, `INSTRUCTIONS.md` (Technical notes), dependency cards + schemas, `.ai/vendors/*` it needs, the module folder | code, fixture builders, migrations, `MODULE.md` notes | edit the `.feature`, `requirements.md`, step library, `composer.json` or other modules |
| Hardening (Pi) | `conventions.md`, `SYSTEM_MAP.md`, all cards, `OPEN.md`, the diff since the last audit; any file on demand | audit or escalation report | change code, tests, features or contracts |

Enforcement, in order of cheapness: the Coder runs on a DB role granted only its module's schema, so a cross-service query fails at the database. Pi's protected-paths extension enforces the Coder's "must not" list in the harness. Arch tests and card diffs catch the rest in `ai:deliver`.

## Requirements: discussion first

The expensive thinking happens here, with you. In `--chat` mode the agent challenges each rule against a fixed edge-case checklist (see `.ai/prompts/requirements.md`) and asks only what applies. Every answer becomes a rule `[R-n]`, an edge case `[E-n]` with an exact outcome, or an out-of-scope line.

Each turn sends `INSTRUCTIONS.md`, the current draft and your last answer, never the full transcript, so a long discussion costs the same per turn as a short one.

Gate A script: *Open questions* is empty, every `[R-n]` cites an `[I-n]`, every `[E-n]` cites an `[R-n]` and states its outcome with exact status and event names.

The zero-cost option is the default: hold the discussion in a Work chat you already pay for, using the same prompt, and paste the result into `requirements.md` (the discussion input is `INSTRUCTIONS.md` + draft + your last answer, so it fits one message). `ai:requirements <Module> --validate` runs the checks only.

## Gherkin: derived, not designed

One `Rule:` per `[R-n]`, at least one happy-path scenario per rule, and one scenario per `[E-n]`, tagged `@E-n`. Scripts check coverage both ways, `behat --dry-run`, and that every step comes from `.ai/steps.md`.

Step definitions live once in `tests/Behat/Steps/`, written by hand and protected from the Coder. They speak in events, statuses, HTTP requests, stored values and external responses, so no LLM ever writes test code and no step can be bent to pass. A step the vocabulary lacks goes under `# New steps` at the end of the feature and waits for you; expect that rarely after the first few modules. Modules supply only fixture builders in `Tests/Fixtures/`.

## Hardening

The only agent with a whole-system view. It runs in two situations, never per module.

**Audit** (`ai:harden --since=<commit>`), every 3–4 delivered modules or weekly: boundary leaks, event drift between publishers and consumers, outbox gaps (state changes that published without an outbox row), listeners missing dedup checks, missing error paths, security issues, cards that overstate the code, `OPEN.md` items made worse. Findings are `blocking` or `advisory`. Blocking findings become fix rounds (`ai:build <Module> --findings=<report>`) or questions for you. Advisory findings go to `OPEN.md`.

Between audits, scripts carry the per-module checks: arch tests, PHPStan level 8, Behat `--strict`, Infection on the module and the card diff.

**Escalation**, after three failed Coder attempts. Input: failing scenarios, trimmed output of all attempts, the module diff. One verdict:

| Verdict | Meaning | Next step |
|---|---|---|
| `fix-code` | Implementation is wrong; diagnosis names cause and files | One extra Coder round with the diagnosis |
| `fix-contract` | A dependency's schema or card doesn't match reality | Stops; you decide |
| `fix-spec` | Requirements or Gherkin contradict or are untestable | Stops; back to Gate A |

One escalation per build: a stuck module costs at most four Coder runs and one Hardening run.

`.ai/hardening/OPEN.md` is Hardening's only memory: unresolved advisory findings, capped at ~30 lines. No other agent reads it.

## Module cards

`MODULE.md` is how the next module learns about this one. `ai:deliver` generates the facts and the Coder writes only *Purpose* and *Integration notes*:

| Section | Source |
|---|---|
| Depends on | Coder, enforced by a generated arch test |
| Consumes / Publishes | Scan of listener type-hints and `event(...)` calls |
| HTTP routes | `php artisan route:list --name=<slug>. --json` |
| Tables | Module migrations |
| Config/env | `config()` / `env()` keys used in the module |

A mismatch between the card and the generated facts blocks delivery. `SYSTEM_MAP.md` concatenates all cards with the commit hash and is never edited by hand. Agents for the next module load `SYSTEM_MAP.md` plus only the cards and schemas of modules they depend on.

Every published contract (`schema.json`) carries a `version` field. The storefront pins it when generating types, so contract drift is a failed frontend build, not a runtime surprise.

## Hallucination controls

Every LLM output gets verified by a script or by you.

| Stage | Risk | Check |
|---|---|---|
| Requirements | Invented rules | Each rule cites `[I-n]` from `INSTRUCTIONS.md`; a script rejects missing or unknown citations. Each `[E-n]` cites an `[R-n]` and states its outcome. Gate A requires empty *Open questions*. |
| Gherkin | Missing or extra scenarios | Each `Rule:` carries `[R-n]`, each edge-case scenario `@E-n`; a script checks coverage both ways and the step vocabulary; `behat --dry-run`. You skim. |
| External APIs | Invented endpoints/fields | `.ai/vendors/<system>.md` is the only API source; `Http::preventStrayRequests()`; sandbox run before delivery. |
| Code | Non-existent classes or packages | PHPStan level 8, arch tests, Behat `--strict`; `composer.json` protected. |
| Step definitions | Tests that can't fail | No LLM writes steps: a hand-written, protected library; `Http::fake()` only for external systems; Infection on the module. |
| `MODULE.md` | Card lies | Facts generated and diffed. |
| Hardening | False findings | Each finding quotes file + line + code; a script checks the quote exists, else drops it. |
| Missed events | Silent event loss | Outbox row required in the same transaction; a script red-flags a state change that should have published but has no outbox row. |
| Double application | Duplicate delivery | Every listener asserts the dedup check exists; the queue-driver Behat suite kills a consumer mid-handler and asserts redelivery with no double effect. |

Low temperature (0–0.2) for Requirements, Gherkin and Hardening. "I don't know" is a valid answer; answer the questions instead of re-running the agent.

## Token budget

Ordered by how much they save.

- **You think, agents transcribe.** Edge cases get settled in the requirements discussion; Gherkin and code follow mechanically, with fewer failed attempts. The discussion runs in a chat you already pay for, so it costs no API tokens at all.
- **Draft, not transcript.** Each chat turn sends the current draft and your last answer, never the history.
- **Gherkin is the test, and no LLM writes steps.** Behat runs the approved file against the protected step library.
- **Cheapest check first.** The Coder loop runs `php -l`, Pint, PHPStan, Pest, Behat, Infection in that order and returns only the first failing check, trimmed to the failing item and the first stack trace.
- **Boilerplate by generators.** `make:module` and the `make:module:*` commands, TypeScript types from `schema.json`; agents only fill in logic.
- **Small Coder context.** The module folder, the dependency cards and schemas it needs, and nothing else up front; Pi reads further files on demand.
- **Stable prefix first.** Guardrail + conventions + template, then the task, so the provider cache stays warm. Keep one model per role so the cache stays warm.
- **Batched whole-system review.** One Hardening audit per 3–4 modules instead of one review each; escalation at most once per build.
- **Hard caps.** `max_tokens` per role, three attempts + one escalated attempt, a token budget per module (`ai.module_budget_tokens`) after which `ai:build` stops, and prepaid credit without auto top-up.
- **Measure.** `ai:deliver` sums the JSON usage of agent responses into `.ai/spend.csv` per module and role; the audit reports the trend.
- **Deterministic where possible.** Approval, card facts, `SYSTEM_MAP.md`, citations, coverage, step vocabulary, budgets and evidence checks are scripts.

## Models and harness

One provider: Mistral, via `api.mistral.ai/v1` (OpenAI-compatible). La Plateforme's catalog includes third-party families, so roles stay on different model families without a second vendor. Model IDs live only in `config/ai.php`: a retirement or price change is a one-line diff found by the audit, never an incident.

| Role | Model | Why |
|---|---|---|
| Requirements | `mistral-small-4` | Low volume, hybrid reasoning; escalate to `mistral-large-4` for tangled domains |
| Gherkin | `zai-glm-5.3-flash` | Mechanical, short; different family from the Coder |
| Coder | `deepseek-v4.1-flash-1017` | Highest volume, cheapest, built for agentic coding, 1M context |
| Hardening | `zai-glm-5.3` | Whole-system view and reasoning, 1M context |

`apps/api/config/ai.php`:

```php
return [
    'base_url' => env('AI_BASE_URL', 'https://api.mistral.ai/v1'),
    'api_key' => env('MISTRAL_API_KEY'),
    'module_budget_tokens' => 400_000,   // ai:build stops when a module has spent this
    'roles' => [
        'requirements' => ['model' => 'mistral-small-4', 'max_tokens' => 4000, 'temperature' => 0.1],
        'gherkin'      => ['model' => 'zai-glm-5.3-flash', 'max_tokens' => 3000, 'temperature' => 0.1],
        'hardening'    => ['model' => 'zai-glm-5.3'],
    ],
];
```

Text agents are one HTTP call:

```php
$role = config("ai.roles.{$name}");

$text = Http::withToken(config('ai.api_key'))
    ->baseUrl(config('ai.base_url'))
    ->timeout(120)
    ->post('/chat/completions', [
        'model' => $role['model'],
        'max_tokens' => $role['max_tokens'],
        'temperature' => $role['temperature'],
        'messages' => [
            ['role' => 'system', 'content' => $conventions.$prompt],
            ['role' => 'user', 'content' => $task],
        ],
    ])->throw()->json('choices.0.message.content');
```

Acting agents (Coder, Hardening) run through Pi, one git worktree per run:

```php
Process::path($worktree)->timeout(1800)->run([
    'pi', '-p', $prompt, '--mode', 'json',   // provider/model flags per Pi docs
])->throw();
```

Pi supplies the tool loop (read, edit, run, iterate), a minimal system prompt, scriptable JSON output that doubles as an audit log, and path protection that enforces the Coder's "must not" list at the filesystem. The contract is `process → JSON → worktree`, so any OpenAI-compatible harness can fill the slot. Manual, exploratory work runs in Vibe Code sessions, which share the same account, outside the automated chain.

## Getting started

1. Skeleton by hand (`docs/Setup.md`), `make:module`, Behat, `.githooks`, DB roles and schema grants for the first module.
2. Write the step library in `tests/Behat/Steps/` and its vocabulary in `.ai/steps.md`.
3. Review `.ai/conventions.md` and the four prompts.
4. Write the orchestrator commands: `ai:requirements`, `ai:approve`, `ai:gherkin`, `ai:build`, `ai:deliver`, then `ai:harden`, plus the budget check in CI.
5. `Catalog` by hand (smallest contract publisher), then `Merchant`, then `Checkout`, then `SalesOrder`, then `Accounting`.

## Design rationale

**Verification before code.** You own the oracle: approved requirements and edge cases, run as Gherkin against a hand-written step library. No agent writes or grades its own tests, so the Coder can't pass by bending them.

**Borrowed from Compound Engineering** (Every): the plan is the primary artefact, and every failure should make the next build easier. **Not borrowed:** self-fixing implementers and parallel post-hoc reviewers; here verification is locked before implementation, and improvement is one convention line: the same failure twice earns a check.

**Accepted risks:**
- **You are the bottleneck.** The pipeline can't outrun the requirements discussion. Keeping Gate A the only blocking gate and batching the rest is the mitigation, not a cure.
- **Machinery costs before it pays.** Start minimal (Getting started) and add only what a real failure justifies.
- **One provider.** Model availability is one vendor's catalog, and vendors retire models. Mitigated by model-per-role config lines and the audit's spend/model trend check.
- **One deployment unit.** The pipeline assumes the single Laravel app; extracting a service invalidates some per-module conventions and needs a pipeline decision, not just a deploy change.
- **Testable isn't everything.** UX quality, performance and search relevance sit outside Behat; you remain the only check there.
