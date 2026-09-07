# Backend fixes — AU-457 code-review findings (High #1/#5, Medium #3/#4)

Session was cut off by a usage-limit reset before writing this report; verified and completed by the orchestrator directly against the working tree left behind. All fixes were already correctly implemented — this documents what was found.

## Fixed

**High #1 + Medium #5 (N+1 query, public + admin list/detail):** `repositories/discovery_repository.py` — nested `selectinload(DiscoveryOutfit.items).selectinload(DiscoveryOutfitItem.item)` added at both query sites (lines 33, 55). One query loads outfits + items + their WardrobeItem rows instead of N+1 lazy loads per `is_servable()` check.

**Medium #3 (duplicate item ids → 500):** `services/discovery_admin_service.py::find_duplicate_item_ids` (new) returns any ids appearing more than once. Wired into `routers/admin/discovery_outfits.py` at both `POST /` (create, line 162) and `PUT /{id}/items` (replace, line 249) — raises a 422 with `duplicate_item_ids` before the write, via a new `_duplicate_items_error` helper. No behavior change for the non-duplicate path.

**Medium #4 (save-to-wardrobe not idempotent):** `services/wardrobe_service.py::clone_common_item` now checks `WardrobeRepository.get_active_clone(user_id, source_item_id)` (new method, `repositories/wardrobe_repository.py:72-86`) before inserting — returns the existing clone instead of creating a duplicate. Requires a new column `wardrobe_items.cloned_from_common_item_id` (migration `clonesrc1a2b`, parented on `discovery1a2b`), set on every clone going forward; existing rows backfill to NULL (never matched, which is correct — they predate lineage tracking). Deliberately a **new** column, not a repurposing of the existing boolean `is_cloned_from_common` (that column's own migration comment says "not a FK, YAGNI" — this fix supersedes that call because idempotency genuinely needs the specific source id) nor of the unrelated, unmapped `source_item_id` column from an abandoned migration.

All current callers of `clone_common_item` (single clone endpoint, batch clone, TrendingDrop's "add" response) were checked — all treat the return value as "the user's item now," none depend on getting a guaranteed-new id, so returning an existing clone is safe for all of them.

## Test results

New/changed test files: `tests/test_discovery_admin_router.py`, `tests/test_discovery_admin_service.py` (new), `tests/test_discovery_repository.py`, `tests/test_discovery_public_router.py`, `tests/test_discovery_model.py`, `tests/test_wardrobe_clone_common_item_idempotent.py` (new) — **48 passed**.

Full suite (re-run by orchestrator post-fix): `pytest -m unit` → 677 passed, 1 failed, 1 skipped. The 1 failure is `test_engine_v05_exploration.py::test_exploration_item_scores_higher` — byte-identical to the single pre-existing failure reported in the original phase-01/03/05 delivery (`backend-dev-260827-2205-au-457-phases-01-03-05.md`), confirmed unrelated (unrelated file, unrelated subsystem). Zero regressions.

## Not fixed / deferred

Nothing deferred — all four findings assigned to this task were resolved.

## Unresolved questions

None new. Pre-existing devops items from the original delivery (Postgres migration dry-run, pytest-cov sandbox gap) still stand and are unaffected by this fix pass.

**Status:** DONE
