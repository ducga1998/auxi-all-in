# "Less use" must actually demote — V05 engine follow-up

**Date:** 2026-08-21
**Repo touched:** `wardrobe-backend` → branch `claude/less-use-item-suggestions-42g8un`
**Predecessor:** `plans/260626-0005-pr148-usage-frequency-backend/plan.md` §8 (deferred follow-up)
**Status:** **Merged 2026-09-07** — `/v05-eval` still owed, now post-deploy (§7)
**Backend PR:** `auxi-wardrobe/auxi-backend#172` (merged as `a46e395` on `main`)

---

## 1. The report

> "When user check the Less use function in the item detail, that item should not
> appear in new suggestions or decrease the chance of appearance in an outfit.
> The current version I still see a lot of less use item in the suggestions."

## 2. Root cause — the feature was never wired to the engine

PR #148 shipped `usage_frequency` deliberately as **display-only**; the down-ranking
was written down as a deferred follow-up and never picked up.

| Layer | State before this change |
|---|---|
| Mobile | ✅ correct — `updateUsageFrequency()` PATCHes, read-path falls back to the `less-used` style tag |
| `PATCH /wardrobe/items/{id}/usage-frequency` | ✅ persists field + dual-writes the tag |
| `models/wardrobe.py` / `to_dict()` | ✅ serves `usage_frequency` |
| **V05 engine** | ❌ **never read either spelling** |

`services/wardrobe_service.py:set_usage_frequency` said so in as many words:
*"Display-only — no recommendation-engine effect."* Nothing in
`blueprints/recommendation/**` referenced `usage_frequency` outside DTO
serialization. So a demoted item competed for every slot exactly like any other.

Measured on the repro wardrobe (2 normal + 1 demoted per family, 30 seeds):
**30/30 builds served at least one demoted item.** After the fix: **0/30.**

A second, quieter bug rode along: the dual-written `less-used` tag lands in
`style_tags`, which feeds `style_jaccard` (outfit cohesion), `dominant_style_tag`
(diversity), `compute_vibe_signature` (novelty/distance) and the L4 mood bonus.
Two demoted items therefore read as *stylistically coherent* purely because both
were demoted.

## 3. Design

Demotion is enforced at **Layer 1 (pool feasibility)**, not as a score penalty —
a penalty only reorders, and the report is that demoted items keep *appearing*.
The drop mirrors the existing rain-resistance gate: a soft preference with an
explicit relax path, never a hard gate that can strand a user.

```
L1 gates (gender / warmth / rain / formality)
   └── _drop_less_used(pool, keep_ids)          ← per category family
         ├─ family has non-demoted survivors  → drop the demoted ones
         └─ family is ENTIRELY demoted        → keep them (relax)
L2 compose
   └── zero candidates AND something was dropped
         → rebuild L1 with include_less_used=True, compose once more
   └── still zero → existing best-effort floor / wardrobe_gap paths
post-L5 distance / exclude filters  (try_another only)
   └── zero candidates → service ladder rung 3 (below)
```

Four escape hatches, so a demotion can never cost the user an outfit:

1. **Fully-demoted family relaxes.** Demoting your only pair of shoes still
   gets you dressed.
2. **Compose-level retry.** Surviving L1 isn't enough — the survivors must also
   *compose*. If the demoted-free pool can't (e.g. the one remaining BOTTOM is
   `HIGH_RISK` against every anchor), the engine restores the demoted items and
   retries once, before the least-bad best-effort floor.
3. **`try_another` last resort** (added 260821 after CEO clarification — see §3.1).
4. **Pinned item exempt.** `pinned_item_id` is passed as `keep_ids` — an explicit
   "build around THIS item" outranks the standing demotion.

### 3.1 The `try_another` exhaustion rung

> CEO, 260821: *"Những items có gắn nhãn Less use chỉ được sử dụng khi user đã
> thử hết tất cả các lựa chọn có khả thi. Hoặc trong trường hợp user Less use
> tất cả đôi giày có trong tủ đồ, thì những items gắn Less use mới được sử dụng."*

The second clause was already satisfied by hatch 1. The first was **not**, and
hatches 2–4 did not cover it:

Demoted items are dropped at L1, so a `try_another` recompose still composed a
healthy candidate set — the set only went empty **later**, at the post-L5
distance / exclude filters. The compose-level retry (hatch 2) sits at L2 and
never fired there. The ladder ran:

```
strict floor → 0.5× floor → cycle (re-serve a SEEN outfit) → pool_exhausted
```

so the demoted items were never reached. A user who swiped through every
variation got *"No more variations available"* — or a repeat — while perfectly
wearable demoted items sat unused. Reproduced: without the rung, the exhausted
session raises `no_outfits_after_distance_or_exclude_filter`.

The fix inserts a third rung, **before** the cycle — a fresh outfit built from a
demoted item beats re-serving one the user already swiped past:

```
strict → 0.5× → STRICT + include_less_used → cycle → pool_exhausted
                 ^ new (flag `less_used_last_resort`)
```

Design notes:

- **Back at the strict floor.** Demoted items open genuinely new item
  combinations, so there is no reason to also sacrifice diversity.
- **Gated on real exhaustion only** — `recompose_pool_insufficient` or
  `recompose_distance_unsatisfied`. A timeout / wardrobe gap / engine error is a
  *failure*, not exhaustion; a demoted item cannot fix it and the rung would
  only burn latency.
- **Layering respected**: pool membership stays the engine's concern
  (`BuildInput.include_less_used` → `layer1_feasibility`); the distance floor
  stays the service's, per the engine's existing "the SERVICE owns the
  graduated exhaustion ladder" comment.
- **Latency**: the rung only runs on an already-exhausted wardrobe, where the
  engine raises `PoolInsufficient` in <300ms — so the common case adds
  milliseconds. Worst case adds one `RECOMPOSE_TIMEOUT_SECONDS` + one LLM-3
  timeout to the window `V05_SESSION_LOCK_TTL_SECONDS` must cover. See §7.

Both spellings demote (`usage_frequency == 'LESS_USED'` **or** the legacy
`less-used` style tag), so items written before the column existed, or by a
client that hit the style-tag fallback path, are covered.

## 4. Files changed (`wardrobe-backend/`)

| File | Change |
|---|---|
| `blueprints/recommendation/engine_v05_constants.py` | `is_less_used()`, `style_signal_tags()`, `LESS_USED_MARKER_TAG`, `NON_STYLE_MARKER_TAGS` |
| `blueprints/recommendation/engine_v05_layers.py` | `layer1_feasibility(include_less_used=, keep_ids=)` + `_drop_less_used()`; style-signal reads routed through `style_signal_tags` |
| `blueprints/recommendation/engine_v05_signature.py` | `style_jaccard` / `dominant_style_tag` / `compute_vibe_signature` stop counting the marker tag |
| `blueprints/recommendation/engine_v05.py` | pass `keep_ids` (pinned item) to both L1 calls; compose-level safety retry; `BuildInput.include_less_used`; fallback flags |
| `services/v05_try_another_service.py` | exhaustion-ladder rung 3 (`less_used_last_resort`); threads `include_less_used` into the recompose `BuildInput` |
| `services/wardrobe_service.py` | docstring — the endpoint is no longer display-only |
| `API_DOCUMENTATION.md` | documented recommendation effect + the three relax paths |
| `tests/test_v05_less_used_demotion.py` | **new** — 26 tests |
| `tests/test_v05_try_another_recompose.py` | +3 ladder tests; `test_both_floors_fail_then_cycle_terminal` → `test_all_rungs_fail_then_cycle_terminal` (see §6) |

No migration, no schema change, no API contract change. **Mobile needs no change.**

## 5. Observability

| Signal | Where | Meaning |
|---|---|---|
| `less_used_demoted` | skipped log (per item) + `trace.fallback_flags` | item dropped from the pool |
| `less_used_relaxed_family_empty` | skipped log | family was entirely demoted, kept |
| `less_used_relaxed_no_compose` | `trace.fallback_flags` | survivors couldn't compose, demoted restored |
| `timings.L1_less_used_retry` | trace | cost of the safety retry |
| `less_used_last_resort` | try_another response `fallback_flags` | ladder rung 3 served a demoted item after exhaustion |
| `less_used_included_by_caller` | `trace.fallback_flags` | the build ran with the last-resort pool open (caller-driven, vs the default drop) |

`less_used_relaxed_no_compose` firing often would mean the drop is too
aggressive for real wardrobes — that's the metric to watch after deploy.

## 6. Verification

- 26 tests pass (`tests/test_v05_less_used_demotion.py`) + 30
  (`tests/test_v05_try_another_recompose.py`, 3 new), incl. end-to-end through
  `V05RecommendationEngine.build()` and through the service ladder.
- Pre-fix simulation (L1 forced to `include_less_used=True`): 30/30 seeds served
  a demoted item. Post-fix: 0/30.
- Rung 3 proven load-bearing: with the rung removed, an exhausted session raises
  `PoolInsufficient(no_outfits_after_distance_or_exclude_filter)` → the user
  sees *"no more variations"*. With it, a fresh demoted outfit is served.
- Full backend suite diffed against `origin/main` on the same machine —
  **218 failures/errors, identical set on both sides, zero delta**, no
  regressions. (All pre-existing environment gaps: missing `boto3`/google
  deps.) Re-measured after merging main at `d5933e9`; the earlier figure of
  264 was taken before PR #173 landed and fixed the `FakeItem` fixture drift
  that accounted for the difference.
- `flake8` on the changed files diffed against baseline — no new findings.

**One existing test was changed, deliberately.**
`test_both_floors_fail_then_cycle_terminal` asserted `call_count == 2` and
*"never a third attempt"* — precisely the contract the CEO clarification
alters. Renamed to `test_all_rungs_fail_then_cycle_terminal`, expects 3 rungs,
and now also asserts the `include_less_used` pattern is `[False, False, True]`.
Its cycle/terminal assertions are unchanged. Flagged here because "test updated
to match new behaviour" deserves a reviewer's eye, not a silent edit.

## 7. Not done / follow-ups

- **Ladder latency vs session-lock TTL.** The ladder's documented worst case was
  `2 × RECOMPOSE_TIMEOUT_SECONDS + 2 × LLM-3 timeout` (~22s with defaults)
  against `V05_SESSION_LOCK_TTL_SECONDS` (default 30). Rung 3 raises the
  ceiling by one recompose + one LLM-3 timeout. The realistic path is
  unaffected (the rung only runs on an exhausted wardrobe, where the engine
  raises in <300ms), but if the ceiling is ever hit in practice the TTL needs
  raising. Not changed here — it is a deployment tunable, and changing it
  blindly would be a guess.
- **`/v05-eval` still not run — now a POST-deploy check.** It needs a live DB
  + LLM keys, unavailable in the environment this was built in. It was flagged
  as a pre-merge step; PR #172 was merged (260907) without it, so the
  confirmation is now owed *after* deploy rather than before merge. What to
  watch on real wardrobes: outfit quality unchanged, `PoolInsufficient` rate
  not up, and the two new relax flags (`less_used_relaxed_no_compose`,
  `less_used_last_resort`) firing rarely — frequent firing would mean the drop
  is too aggressive. `scripts/eval_v05_outfits.py` already stores the full
  response verbatim, so `fallback_flags` and `trace` land in `outfits.json`
  with **no harness change needed**.
- **`python test_server.py` not run** (same reason — needs a live stack). The
  repo's PR checklist asks for it.
- **Live Redis `try_another` pools seeded before deploy** still hold demoted
  items until their TTL (`V05_TRY_ANOTHER_TTL_SECONDS`, default 1h) expires.
  Self-healing; flush the V05 session keys if the CEO wants it immediate.
- **Engine V2/V3 untouched.** Confirmed the mobile HomeScreen only calls
  `recommendV05` / `resetV05Session`; the legacy `/recommendation/next` path is
  not reachable from the suggestions surface.
- **Submodule pointer not bumped in this PR.** The blocker is gone — PR #172
  merged as `a46e395` on backend `main` — but pointer bumps are the
  maintainers' own routine `chore:` commit on this repo (the most recent being
  `e097358`, AU-457), so this PR leaves that to the next sweep rather than
  sweeping unrelated backend commits into a docs change.
- **`.gitmodules` URLs are stale.** All four submodules point at
  `ducga1998/*`, but the live repos are `auxi-wardrobe/*` (`wardrobe-backend`
  → `auxi-wardrobe/auxi-backend`, which does contain the currently-pinned
  commit `a950a7e` — same repo, transferred owner). `git submodule update
  --init` fails auth against the old URLs. Worth a `chore:` pass to repoint
  them.
- **Item-level favourite (`is_favorited`)** is the sibling gap flagged in the
  PR #148 plan §8 — still unaddressed, still a candidate for the inverse
  (promotion) treatment.

## Unresolved questions

- ~~Should a demoted item be excluded from `try_another` variations *harder*
  than from the initial build?~~ **Answered by the CEO 260821** — the opposite:
  `try_another` must eventually reach them, once every feasible non-demoted
  variation is exhausted. Implemented as ladder rung 3 (§3.1).
- Is the per-family relax the right granularity, or should TOP relax only when
  FULL_BODY is also starved (they're alternative anchors)? Current behaviour is
  the more permissive of the two.
- Deferred from PR #148 and still open: the Linear/GH issue id for traceability.
