# Phase 05 — Backend tests

**Repo:** `wardrobe-backend/` · **Owner:** backend-dev · **Effort:** 3h
**Status:** pending · **Blocked by:** phases 02, 03

## Context

**Verified gap:** `trending_drop` has **NO tests** (`grep -rl "TrendingDrop\|trending_drop" tests/`
returns nothing). So there is no trending-drop test file to mirror. Use instead the clean
model/repo/router triad already in the suite:
`tests/test_app_feedback_model.py`, `tests/test_app_feedback_repository.py`,
`tests/test_app_feedback_router.py` (the `threaded_db` fixture at `test_app_feedback_router.py:19-45`
— temp SQLite + `flask_db.metadata.create_all` + `dependency_overrides` — is the pattern to copy).
Conventions: `wardrobe-backend/.claude/rules/testing.md` (markers, naming
`test_<method>_<scenario>_<expected>`, 80% coverage floor).

## Test matrix

### `tests/test_discovery_model.py` (unit)
- `is_servable` False for DRAFT / ARCHIVED / `is_enabled=False` / all-items-soft-deleted.
- `is_servable` True for PUBLISHED + enabled + ≥1 live item.
- `live_items()` drops soft-deleted items, preserves `position` order.
- `to_card_dict` / `to_public_dict` key sets exactly as documented in phase 03.

### `tests/test_discovery_repository.py` (unit)
- `replace_items` assigns `position` 0..n-1 in the given order.
- `replace_items` called twice leaves no orphan children (count check).
- `delete` removes children then parent; the referenced `WardrobeItem` still exists.
- `update` with explicit `None` clears a nullable column.

### `tests/test_discovery_admin_router.py` (integration)
- create with a non-common item id → 422, no row written.
- create with 5 item ids → 422 (D2 cap).
- `PUT /items` reorder → positions reflect the new array.
- `PATCH status=PUBLISHED` with 0 live items → 422 and status unchanged.
- non-admin token → 403; no token → 401.

### `tests/test_discovery_public_router.py` (integration)
- list returns only servable outfits, ordered by `sort_order` then `created_at` DESC.
- `?season=fall` filters; `?season=autumn` → 422.
- `?trend_tag=y2k` filters; unknown tag → empty list + `total: 0`, **not** 404.
- pagination: `limit`/`offset` slice correctly and `total` counts the FULL filtered set.
- detail 200 embeds ordered items; DRAFT id → 404; unknown id → 404 (**identical envelope** —
  regression guard against draft enumeration).
- detail after soft-deleting the last item → 404, not 500.
- unauthenticated → 401.

## Todo

- [ ] 4 test files above
- [ ] `pytest tests/test_discovery_*.py -v` green
- [ ] `pytest -m "not slow"` full suite still green (no fixture/table leakage)
- [ ] `python test_server.py` green (pre-push gate per `wardrobe-backend/CLAUDE.md`)

## Success criteria

All listed cases pass; coverage of `models/discovery.py`, `repositories/discovery_repository.py`,
`services/discovery_*.py`, `routers/discovery.py`, `routers/admin/discovery_outfits.py` ≥ 80%.

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| New model not in the SQLite `create_all` set of a copied fixture | **H×M** | The `threaded_db` fixture imports models explicitly (`test_app_feedback_router.py:20-23`) — add `discovery` to that import list |
| Test seeds a common item incorrectly (needs `is_common_item=True`, `owner_id="SYSTEM"`) | M×M | Reuse the `seeded_test_wardrobe` fixture shape (`tests/conftest.py:394`) |
| Tag filter tested only on Postgres semantics | L×M | Python-side filter is engine-independent by design (phase 03 step 2) |

## Next steps

Green tests + phase 09 done → phase 10 verification gates.
