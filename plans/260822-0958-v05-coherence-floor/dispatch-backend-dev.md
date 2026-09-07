# Dispatch prompt — F6 implementation (paste vào session mới trên `auxi-wardrobe/auxi-backend`)

> Prompt tự chứa (self-contained). Session nhận prompt này KHÔNG có umbrella repo,
> nên mọi thứ cần thiết đã được inline. Spec đầy đủ: `plans/260822-0958-v05-coherence-floor/plan.md`.
> Cách dùng: mở session Claude Code mới với source `auxi-wardrobe/auxi-backend`, dán phần dưới.

---

You are working in `auxi-wardrobe/auxi-backend` (FastAPI · SQLAlchemy · Postgres), the V05 outfit recommendation backend. Fresh session — no prior context. Read this whole brief before touching anything.

# Goal

Stop the V05 engine from serving incoherent outfits, per a CEO requirement. Real reported failure: the engine served **beige linen camp-collar shirt + black adidas track pants + tan leather driving loafers** — formality spread ~4, athleisure mixed with smart-casual. It surfaced EARLY from a wardrobe that still held better options.

CEO requirement, verbatim (Vietnamese):
1. "Nếu hết lựa chọn cũng không nên đưa ra gợi ý quá ngớ ngẩn" — even out of options, don't serve an absurd outfit.
2. "Trong tủ đồ rõ ràng còn nhiều lựa chọn phù hợp hơn, nhưng nó lại đưa ra lựa chọn không phù hợp này quá sớm" — better options existed; the bad one came too early.
3. "Chỉ đưa ra lựa chọn miễn cưỡng để đủ tạo thành 1 bộ đồ khi nó thực sự dùng hết các lựa chọn phù hợp về mặt thời trang" — only serve a compromise outfit after genuinely exhausting the fashion-coherent ones.

# Diagnosis to VERIFY FIRST (do not trust it blindly)

From analysis reports dated 2026-05/06 citing real `file:line`. It is now 2026-08. **Phase 0 is a hard gate — verify each against current code before writing anything.** If a claim is false, STOP and report; the design may need to change.

- **`engine_v05_layers.py:465-466,561-562`** — outfit score = mean of NON-ANCHOR slot scores; **the anchor is never scored**. An anchor that clashes with everything is never penalised. Pairs are scored against the anchor only, never all-pairs, and there is no outfit-level coherence term.
- **`engine_v05_layers.py:1007-1031`** — L5 novelty penalises only repeated color/silhouette/formality/tags, so an odd outfit reads as "fully novel" and gets a BONUS. Novelty cannot distinguish "fresh" from "wrong".
- `_score_pair_for_slot` reads only color / silhouette / formality / length-rise. No fabric/texture/archetype dimension exists anywhere.
- `engine_v05_signature.py:61-62` already reads `style_tags`.
- Existing multiplier precedent: `COMMON_INJECTED_PENALTY = 0.9`.
- Existing graduated-exhaustion precedent in try_another: `distance floor -> relaxed (0.5*floor) + flag relaxed_distance -> terminal`. **Reuse this exact shape — do not invent a new mechanism.**
- Existing graceful no-outfit path: `WardrobeGapError` -> HTTP 200 + `outfit=None` + `wardrobe_gap: true` + `wardrobe_gap_reason` + `fallback_flags`. Already wired to mobile.
- A prior eval flagged returning **422 as wrong UX**. Never return 422 on this path.

# Design (minimal — do NOT rewrite the engine)

## New leaf module `blueprints/recommendation/engine_v05_coherence.py`

Same leaf constraint as `engine_v05_distance.py`: pure dicts, no imports from `services/` or engine internals.

```
coherence(outfit) -> float in [0,1]
  = 1 - penalty(formality_spread) - penalty(archetype_conflict)
```

- `formality_spread = max(formality) - min(formality)` over **ALL items INCLUDING the anchor**. This is the key trick: it patches the unscored-anchor hole without rewriting the scoring architecture.
- `archetype_conflict`: small table over the **existing** `style_tags` field (e.g. athleisure x dressy). **No schema change, no migration, no backfill.** If `style_tags` coverage is too sparse (measure in Phase 0), ship formality-spread only and report that.

## Use it in TWO places

**(a) Ranking multiplier** — fixes "surfaced too early". Multiply into the outfit score before ranking, same pattern as `COMMON_INJECTED_PENALTY`. Better outfits naturally outrank worse ones.

**(b) Two-tier serving** — fixes "never absurd" + "compromise only as last resort". The floor **does NOT delete candidates, it demotes them**:

```
TIER 1  coherence >= V05_MIN_COHERENCE   -> serve first, ALWAYS exhaust fully
TIER 2  coherence <  V05_MIN_COHERENCE   -> only when TIER 1 is genuinely empty
```

Ladder:
```
1. TIER 1 candidate                          -> serve
2. TIER 1 empty -> recompose (existing)      -> serve
3. TIER 1 truly exhausted
   -> TIER 2: serve the HIGHEST-coherence remaining outfit (least bad — coherence is
      continuous, not binary)
   + fallback_flags += 'relaxed_coherence'
   + trace.coherence_score
4. No outfit composable at all -> existing wardrobe_gap 200 + reason
```

**Hard invariant (must have a test): TIER 2 is never served while any TIER 1 candidate exists.**

Apply at **both `/build` and `try_another`** — the reported case surfaced early, so it may be at build. Don't only patch try_another.

By construction this cannot raise the dead-end rate: anything servable today is still servable, only reordered and flagged.

## Constants (`engine_v05_constants.py`)

| Constant | Proposed | Note |
|---|---|---|
| `V05_MIN_COHERENCE` | `0.5` | TIER 1/2 boundary. Floor for "not absurd", not "perfect". Tune after eval. |
| `MAX_FORMALITY_SPREAD` | `3` | spread <=2 no penalty; 3 light; >=4 drops to TIER 2 |
| `V05_COHERENCE_ENABLED` | `false` | Feature flag via the ML config versioning / AlgorithmCockpit, same pattern as `NEW_ITEM_BOOST_ENABLED` |

## Observability (required)

- `relaxed_coherence` rate — % of serves landing in TIER 2. Most important new product metric: it measures "does this wardrobe actually contain a decent outfit".
- `coherence_score` + `formality_spread` into the build trace (trace object already carries `min_distance`, `seen_signatures_count` around `engine_v05.py:~306`).
- Log TIER 2 serves into the existing `v05_pool_insufficient_events` table with `failure_reason='coherence_relaxed'`. **Reuse the table — do not create a new one.**

# Phases

**Phase 0 — VERIFY (hard gate, no code until done). Report findings before proceeding.**
- [ ] Confirm the unscored-anchor claim against current `engine_v05_layers.py`. If the anchor IS scored now, STOP and report.
- [ ] Confirm the L5 novelty behaviour.
- [ ] Measure `style_tags` and `formality_level` coverage on real items — decides whether archetype conflict is viable at all.
- [ ] Locate the existing try_another graduated-exhaustion ladder and the `wardrobe_gap` path; you will mirror them.

**Phase 1** — `engine_v05_coherence.py` + unit tests. Test that the reported outfit (formality ~1 track pant / ~4 shirt / ~5 loafer) scores low, a coherent outfit scores high, spread 0 -> 1.0.

**Phase 2** — ranking multiplier behind `V05_COHERENCE_ENABLED`. Regression test for the exact reported case: given a pool holding both a coherent and an incoherent outfit, the coherent one must rank higher.

**Phase 3** — two-tier serving at build + try_another, ladder above, `relaxed_coherence` flag, observability, `API_DOCUMENTATION.md` updated (mandatory in this repo — the API doc IS the cross-repo contract). Test the hard invariant.

**Phase 4** — `pytest` green + `python test_server.py` green.

# Constraints

- **Do not rewrite the recommendation engine.** Minimal, additive changes reusing existing patterns (multiplier penalties, graduated exhaustion ladder, existing gap path, existing events table). YAGNI / KISS / DRY.
- Everything behind `V05_COHERENCE_ENABLED=false` by default. Flag off must reproduce today's behaviour exactly — test it.
- `relaxed_coherence` is an additive wire field. Old clients must degrade safely. No breaking changes.
- Never return 422 on this path.
- No mocks/fakes to make tests pass. Do not skip or weaken failing tests.
- `API_DOCUMENTATION.md` update is mandatory.

# Deliverable

Work on a new branch (e.g. `feat/v05-coherence-floor`), commit with conventional-commit messages, push the branch. **Do NOT open a pull request** — report back instead.

Report: Phase 0 findings (especially any claim that turned out false), what you implemented, test results verbatim, and anything you could not verify (e.g. if no DB or running backend is available, say so plainly rather than guessing — the eval gate and coverage measurement may need prod access you don't have).

Do not claim done until `pytest` and `python test_server.py` are actually green, or state clearly which ones you could not run and why.
