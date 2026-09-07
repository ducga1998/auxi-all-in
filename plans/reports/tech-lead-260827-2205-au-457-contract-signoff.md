# tech-lead contract sign-off — AU-457 `/api/discovery/*`

**SIGNED OFF**

Phase 06 (mobile service layer, `auxi/`) is now UNBLOCKED. mobile-dev may
build `auxi/src/services/discoveryService.ts` + `auxi/src/hooks/useDiscovery.ts`
against `wardrobe-backend/API_DOCUMENTATION.md`'s `## Discovery (AU-457)`
section (lines 2766-2966 as of this review) as the contract.

## What I reviewed

1. `plans/260827-2205-au-457-discovery/plan.md` (full)
2. `plans/260827-2205-au-457-discovery/phase-03-backend-public-api-contract.md`
3. `plans/reports/backend-dev-260827-2205-au-457-phases-01-03-05.md`
4. `wardrobe-backend/API_DOCUMENTATION.md` §Discovery (AU-457)
5. `wardrobe-backend/routers/discovery.py`, `services/discovery_service.py`,
   `repositories/discovery_repository.py`, `models/discovery.py`
6. `wardrobe-backend/tests/test_discovery_public_router.py` (22 tests across
   list/detail/trend-tags; model + repo + admin-router test files reviewed at
   a glance — not the object of this contract review)

## Contract verification

| Item | Phase-03 spec | Shipped doc | Shipped code | Verdict |
|---|---|---|---|---|
| 3 endpoints (`GET /outfits`, `GET /outfits/{id}`, `GET /trend-tags`) | yes | yes, all 3 documented | `routers/discovery.py:53,80,105` — all 3 present | match |
| Auth: `Depends(get_current_user)` on all 3 | yes | documented per-endpoint | `discovery.py:60,84,108` | match |
| Rate limit 60/min | yes | documented per-endpoint | one shared `SimpleRateLimiter(60)` (`discovery.py:35`) vs spec's "two instances" | **deviation, no contract impact** — same limit, same behavior; see below |
| List query params (`season`, `trend_tag`, `limit` default 20/max 50, `offset` default 0) | yes | yes, exact | `discovery.py:56-59` (`Literal[...]`, `Query(ge=1, le=50)`, `ge=0`) | match |
| List response shape (`outfits[]`, `count`, `total`, `limit`, `offset`) | yes | yes, exact field names/types | `discovery.py:71-77` | match |
| Card fields (`id`,`title`,`composite_image_url`,`season`,`trend_tags`,`item_count`) | yes | yes | `models/discovery.py:130-139 to_card_dict` | match |
| Unknown `season` → 422 (closed enum); unknown `trend_tag` → empty list, not error | yes | yes, both documented explicitly | `Literal["spring","summer","fall","winter"]` (422 via FastAPI validation) + Python-side tag filter returns `[]` on no match | match, tested (`test_list_invalid_season_returns_422`, `test_list_unknown_trend_tag_returns_empty_not_404`) |
| Detail response shape incl. `items[]` sub-fields | yes | yes, exact | `_item_dto` (`discovery_service.py:87-99`) matches doc 1:1 | match |
| **404 for BOTH missing id AND unpublished/disabled/all-items-soft-deleted, identical envelope** | yes — called out as the mobile deep-link fallback dependency | yes, explicitly documented with the exact envelope JSON and the "MUST NOT distinguish" instruction to mobile | `DiscoveryNotFoundError` raised uniformly in `get_servable` for both `outfit is None` and `not outfit.is_servable()` (`discovery_service.py:65-68`); router maps to one 404 shape (`discovery.py:94-102`) | match — this is the one behavior I checked hardest, since it's a security property (no draft-title enumeration) as well as a contract term. `test_detail_draft_id_returns_404` asserts the unknown-id and draft-id responses have identical envelope key sets and identical `error` value. Confirmed. |
| Soft-deleted last item → 404, no 500 | yes | yes | `live_items()` filter + re-check in `is_servable()` | match, tested (`test_detail_after_soft_deleting_last_item_returns_404_not_500`) |
| Pagination correctness (`total` = full filtered set, page sliced after filter) | yes | yes | `discovery_service.py:56-62` filters trend_tag in Python then slices; `total = len(servable)` before slicing | match, tested (`test_list_pagination_slices_and_total_counts_full_set`) |
| Ordering `sort_order` ASC, `created_at` DESC | yes | yes | `discovery_repository.py:63-64` (SQL order_by) | match, tested (`test_list_ordered_by_sort_order_then_created_at_desc`) |
| Unauthenticated → 401 | yes | yes | `get_current_user` dependency | match, tested both list+detail |
| Rate limit 429 | yes | yes | `_enforce_rate_limit` copied verbatim from `trending_drop.py` | **not covered by an automated test** — flagged by backend-dev, matches existing precedent (trending_drop's identical pattern is also untested at unit level, only exercised by `test_server.py` live-smoke). Not a contract defect; logic path is proven-reused code, not new logic. |
| Registration (`routers/__init__.py`, `app.py`) | yes | n/a | both present, `discovery_router` imported + included | match |

## Deviations from phase-03, assessed

1. **One shared `SimpleRateLimiter(60)` instead of two instances.** Phase-03
   said "two `SimpleRateLimiter(60)` instances" only because it was copying
   `trending_drop.py`'s pattern verbatim; that file needs two instances
   because its two routes are *different* rate tiers (60 vs 20/min). All
   three Discovery routes are the same 60/min read tier, so collapsing to
   one limiter is behaviorally identical and doesn't touch the contract
   surface (client sees the same 60/min ceiling either way). **No sign-off
   impact.**
2. **`DiscoveryRepository.list()` gained `is_enabled`/`season` filters**
   (nominally phase-02's file, extended by phase-03/05 work). Internal
   implementation detail, doesn't change the HTTP contract. **No sign-off
   impact.**
3. **`tests/conftest.py`'s `db_session` fixture got `discovery` added to its
   model-import list.** Test-infra only, not a contract concern.

None of these three deviations touch endpoint shape, field names, types,
auth, or error semantics — the actual contract mobile-dev builds against is
unaffected.

## Contract stability assessment

The shape is stable enough to build against without near-term breaking
risk:
- Field names/types (`composite_image_url`, `season`, `trend_tags`,
  `item_count`, `items[].category_code`/`layer_code`/`is_common_item`, etc.)
  are drawn straight from existing, already-stable model vocabulary
  (`models/wardrobe.py` category/layer codes, the `TrendingDrop` DRAFT/
  PUBLISHED/ARCHIVED lifecycle idiom) rather than invented fresh — low churn
  risk.
- Pagination shape (`count`/`total`/`limit`/`offset`) matches the existing
  `trending_drop` / common-items pagination convention already consumed
  elsewhere in `auxi/src/services/` — mobile-dev isn't learning a new shape.
  Reuse it directly, don't reinvent a client-side pager for this endpoint.
- The 404-collapse behavior is a **deliberate, documented, tested security +
  UX contract term** — mobile-dev must treat any Discovery-detail 404 as "go
  to the generic not-found/removed state," never attempt to special-case
  "was this ever published." This is the one behavior with real product
  consequence if mobile gets it wrong (a differently-worded empty state for
  404 vs some other status would be fine; branching logic trying to detect
  "existed once" would not — there's no signal for that, by design).
- `season` is a closed `Literal` server-side; if mobile hardcodes the same
  4-value enum for its filter chips (which it should, per `trend-tags`
  endpoint existing specifically to avoid a *different* kind of hidden-value
  bug), there's no drift surface.

## Pre-deploy follow-ups (not contract blockers — routing per protocol)

Per the task's explicit instruction, these are noted but do NOT block this
API contract sign-off; they block **deploy**, not mobile-dev starting phase
06. Both go on devops's plate before backend 01→03 actually ships to prod
(per the plan's "Release order": backend deploy precedes mobile ship):

1. **Migration verified on SQLite only, not against live Postgres.**
   backend-dev's isolated Alembic `upgrade()`/`downgrade()`/`upgrade()`
   round-trip is solid evidence the DDL is internally consistent, but the
   closest sibling migration (`trendingdrop1a2b`) is the only precedent for
   Postgres compatibility — untested here. **Action for devops:** run
   `alembic upgrade head` against a Postgres staging/dev DB (not the live
   Railway `.env` target) before this ships, per backend-dev's own
   recommendation in the report.
2. **`pytest-cov` + numpy/PIL sandbox incompatibility** blocked direct
   coverage measurement of `routers/discovery.py` and
   `services/discovery_*.py`. Reproduced on an unrelated pre-existing file,
   so it's environmental, not something this PR introduced. Correctness
   evidence instead rests on 35/35 green tests covering nearly every branch
   (auth 401, validation 422, publish-gate, pagination, filtering,
   soft-delete 404) per backend-dev's report. **Action:** re-run coverage in
   a normal (non-sandboxed) dev environment before treating the 80% target
   as formally verified — informational, not a re-review trigger.

Neither item changes the wire contract; both are execution/verification
concerns squarely in devops's and backend-dev's follow-up lane, not
mobile-dev's.

## Minor note (non-blocking)

No dedicated router-level test for admin `DELETE /{id}` beyond the
repository-level test — this is phase-02 (admin API) surface, not the
public contract phase-03 gates on. Not part of this sign-off's scope; if
backend-dev/tech-lead want it closed for completeness later, that's a
housekeeping item on the admin surface, unrelated to unblocking mobile.

## Sign-off statement

The shipped `/api/discovery/*` contract matches phase-03's spec field-for-
field, matches the mandatory `API_DOCUMENTATION.md` update requirement
(`wardrobe-backend/CLAUDE.md` §Rules, `.claude/rules/api-documentation.md`),
and the security-relevant 404-collapse behavior the mobile deep-link
fallback depends on is implemented correctly and tested. No drift between
doc and code found. **SIGNED OFF — phase 06 (mobile service layer) is
UNBLOCKED.**

**Status:** DONE
**Summary:** Reviewed AU-457 phase-03 `/api/discovery/*` public contract — doc, router, service, model, repository, and tests all align with the phase-03 spec; the load-bearing 404-not-403 (missing vs unpublished) behavior is correctly implemented and tested. Signed off; phase 06 unblocked. Flagged two pre-deploy (not pre-mobile-build) follow-ups for devops: Postgres migration dry-run, coverage re-measurement outside the sandbox.
**Concerns/Blockers:** None blocking. Two informational pre-deploy items routed to devops (see above) — do not block mobile-dev starting phase 06, but must clear before backend 01→03 goes to production per the plan's release order.
