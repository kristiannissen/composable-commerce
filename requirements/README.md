# Requirements

One folder per surface:

- `frontend/` — the headless storefront
- `backend/` — standalone services behind the gateway

Each feature gets one file per surface it touches
(`<surface>/<feature>.md`). The second surface cross-references the
other; requirements are never duplicated across the two files. Shared
contracts (`product-schema/`) apply to every feature and are linked,
never restated, in feature files.

Status: every file is a draft until approved at Gate A. Nothing here is
implemented before approval.

Naming: the search capability is called **the search service** in all
requirement files. No vendor names; a replacement engine must satisfy the
same text unchanged.

| Feature | Frontend | Backend | Status |
|---|---|---|---|
| PLP | [frontend/plp.md](frontend/plp.md) | [backend/plp.md](backend/plp.md) | draft |
| Product schema | shared: [product-schema/](product-schema/README.md) | | draft |
