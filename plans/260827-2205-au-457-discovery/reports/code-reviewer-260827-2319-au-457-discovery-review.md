# AU-457 Discovery — independent review (backend + admin SPA + mobile)

**Verdict: SHIP-WITH-FIXES.** No security/auth-bypass/data-leak found — admin
gating, public-route auth, and the 404-collapse (missing-vs-unpublished) all
verified correct in code and covered by tests. The blocking-grade issues are
one real N+1 query pattern (will bite as the catalog grows, cheap to fix now)
and one confirmed-but-narrow deep-link data-loss bug during first-login
onboarding (self-flagged by mobile-dev, verified real by tracing
`AppNavigator.tsx`). Everything else is medium/low and can ship with a
follow-up ticket.

---

## Critical

None found. Admin routes are gated by `get_current_admin` (checks
`user.role == "admin"`, 403 otherwise) — `wardrobe-backend/deps/auth.py:108-135`.
Public routes gated by `get_current_user` — `routers/discovery.py:60,84,108`.
The 404-collapse for missing-vs-unpublished is implemented in
`services/discovery_service.py:65-68` and asserted at the test level
(`tests/test_discovery_public_router.py:266-281` compares envelope key sets
+ error value between an unknown id and a DRAFT id) — this is the one
security-relevant behavior explicitly called out by the task and it holds up
under a second independent read, not just trust of the tech-lead sign-off.

---

## High Priority

### 1. N+1 query on every Discovery read endpoint (list, detail-adjacent list, trend-tags)
**File:** `wardrobe-backend/repositories/discovery_repository.py:37-61`,
`models/discovery.py:107-117` (`live_items()`), `models/discovery.py:221`
(`DiscoveryOutfitItem.item` relationship).

`DiscoveryRepository.list()` eager-loads `DiscoveryOutfit.items` via
`selectinload` (good — avoids one query per outfit), but **not**
`DiscoveryOutfitItem.item`, i.e. the `WardrobeItem` FK relationship each
join-row points at. Every call site that determines servability —
`is_servable()` → `live_items()` — iterates `self.items` and touches
`oi.item is not None` for each row, which lazy-loads that `WardrobeItem`
individually (SQLAlchemy default `lazy='select'`).

**Concrete failure scenario:** `DiscoveryService.list_servable()`
(`services/discovery_service.py:51-54`) calls `repo.list(...)` then runs
`o.is_servable()` on **every** published+enabled outfit returned (not just
the paginated page — filtering happens in Python before slicing). With 50
published outfits × ~3 items each, a single `GET /api/discovery/outfits`
call issues ~150 extra individual `SELECT ... FROM wardrobe_items WHERE id
= ?` round-trips. `list_trend_tags()` (`discovery_service.py:76-84`) does
the identical full-scan-and-check for every request. This isn't
theoretical — it's on the hot path of the feed endpoint mobile calls on
every screen focus, and it scales with total catalog size, not page size.

**Fix:** add `.options(selectinload(DiscoveryOutfit.items).selectinload(DiscoveryOutfitItem.item))`
in `DiscoveryRepository.list()` and `get_with_items()`. Two extra lines,
collapses back to ~2-3 queries per request regardless of outfit/item count.

### 2. Deep-linked outfit is silently lost for a first-login user mid-onboarding
**File:** `auxi/src/services/deepLinkHandler.ts:215-216` (`isAuthedTreeMounted`),
`auxi/src/navigation/AppNavigator.tsx:97-103` (post-login replay effect),
`auxi/src/navigation/AppNavigator.tsx:184-220` (onboarding branch — no
`DiscoveryOutfitDetail` route registered).

`isAuthedTreeMounted` only checks `rootRoute.name !== 'Auth'`. Traced the
actual `AppNavigator` render tree: when `user` is truthy AND
`user.is_first_login` is true, the mounted root stack is
`Welcome → LocationPermission → OnboardingWardrobe → ... → OnboardingOutro`
— none of which is named `'Auth'`, so `isAuthedTreeMounted` returns `true`
even though `DiscoveryOutfitDetail` is not a registered screen in this
branch (it only exists in the post-onboarding `else` branch,
`AppNavigator.tsx:283-291`).

**Concrete failure scenario:** a brand-new user signs up, taps a Discovery
social link before finishing onboarding (or the app cold-starts into the
link during onboarding). `dispatchDeepLink` clears `pendingDeepLink` and
calls `navRef.navigate('DiscoveryOutfitDetail', ...)` against a tree that
doesn't have that route registered — React Navigation silently no-ops (dev
console warning only, no crash). The link is now gone forever: the
post-login `useEffect` (`AppNavigator.tsx:97-103`) already fired at the
`user` truthy transition and consumed `pendingDeepLink`, and nothing re-fires
when `is_first_login` later flips to `false` (that effect's dependency
array is `[user]`, not `[user, user.is_first_login]`). The social-link entry
point — the ticket's core viral-growth surface — is a dead end for exactly
the highest-value case: a first-time user coming in from a shared post.

Self-flagged by mobile-dev in their report (unresolved question #2) and
confirmed real by tracing the code, not just taking the flag at face value.

**Fix options (either is small):** (a) extend `isAuthedTreeMounted` to also
treat the onboarding branch as "not ready" (e.g. check `user.is_first_login`
directly instead of route name), keeping the link stashed; or (b) widen the
post-login effect's deps to `[user, user?.is_first_login]` so it re-attempts
replay when onboarding completes. Either requires `pendingDeepLink` to
survive the failed navigate attempt in scenario (a) — currently it's cleared
unconditionally at the top of `dispatchDeepLink` before the route-mounted
check for `discovery-outfit`, so (a) needs the check moved before the clear,
or the link needs to be restashed on the "not mounted" branch (it already
is stashed correctly for the logged-out case — the gap is specifically the
onboarding branch not being recognized as "not mounted").

---

## Medium Priority

### 3. Duplicate `item_ids` in create/replace payload → unhandled 500, not a clean 422
**File:** `wardrobe-backend/services/discovery_admin_service.py:30-42`
(`validate_item_ids`), `repositories/discovery_repository.py:64-79` (`create`),
`repositories/discovery_repository.py:99-130` (`replace_items`),
`models/discovery.py:215-219` (`uq_discovery_outfit_item` unique constraint).

`validate_item_ids` checks each id independently for "is a live SYSTEM
common item" but never checks for duplicates within the same array. Pydantic's
`Field(min_length=1, max_length=4)` on `item_ids` also doesn't dedupe.
Sending `item_ids: [A, A]` (2 entries, same valid common item) passes both
gates, then `repo.create()`/`replace_items()` inserts two
`DiscoveryOutfitItem` rows with identical `(discovery_outfit_id, item_id)`,
tripping the `uq_discovery_outfit_item` unique constraint on commit. The
`except Exception: rollback; raise` in `replace_items` (and the bare
`db.commit()` in `create`, inside `get_db_context`'s try/except) surfaces as
an **unhandled `IntegrityError` → generic 500** (`app.py:191-202`), not a
422 naming the problem. No stack trace or PII leaks (the general handler is
sanitized), so this isn't a security issue, but it's a correctness gap: a
trivially-detectable bad input (repeat an id) produces a confusing
"Internal Server Error" instead of the same clean validation UX every other
bad-input path gets. Not reachable through the admin SPA UI today (the item
picker filters out already-selected ids from its own options — confirmed in
`wardrobe-admin/src/components/discovery-outfits/DiscoveryItemPicker.tsx`),
but reachable via direct API call, and not covered by any test in
`tests/test_discovery_admin_router.py`.

**Fix:** add a `len(set(item_ids)) == len(item_ids)` check next to
`validate_item_ids`, return the same `_invalid_items_error`-shaped 422.

### 4. Client-side-only double-tap guard on a non-idempotent clone endpoint
**File:** `auxi/src/hooks/useSaveCommonItemToWardrobe.ts:65-70`,
`wardrobe-backend/services/wardrobe_service.py:196-236` (`clone_common_item`
— always inserts, no idempotency key, unique constraint, or existing-clone
check).

Both the mobile report and this review confirm: `save()` blocks re-entry via
`isSaved || mutation.isPending`, and `MButton`'s `loading` prop additionally
disables the underlying `Pressable`
(`auxi/src/components/design-system/lib/MButton.tsx:122`) — a real (if not
airtight) guard against a normal double-tap. But this is entirely
client-side; the backend endpoint this wraps has no dedup of its own. A
retried request (e.g. a flaky network causing an axios timeout retry, or a
future second entry point calling the same clone endpoint for the same
item) can still create duplicate wardrobe rows for the same SYSTEM item.
This is explicitly self-flagged in the mobile-dev report and is a
pre-existing, out-of-scope backend endpoint (reused, no new backend work per
plan) — flagging as a known gap to track, not a blocker for this ticket, but
worth a follow-up ticket on `clone_common_item` itself (e.g. a
`(user_id, source_item_id)`-scoped debounce or a "you already have this"
check) since Discovery is about to become a second caller of a
previously single-entry-point endpoint.

### 5. `GET /api/admin/discovery-outfits` (list) is fully unbounded
**File:** `wardrobe-backend/routers/admin/discovery_outfits.py:110-122`.

No `limit`/`offset` — `repo.list(status=status_filter)` returns every row
matching the optional status filter, each with a `to_admin_dict()` that also
lazy-loads `oi.item` per item (same N+1 shape as finding #1, admin side).
Low risk today (admin-curated catalog, likely dozens not thousands of rows),
but there's no ceiling if the catalog grows — combine the fix for #1 here
too since the admin list is a second hot path lacking eager-loading.

---

## Low Priority / Simplification (YAGNI check)

- **Backend deviations from the phase docs are all justified and
  documented** — shared `SimpleRateLimiter(60)` instead of two instances
  (same tier, no behavior change), `DiscoveryRepository.list()` gaining
  `is_enabled`/`season` filters instead of a second query path, `422`
  literal instead of the named `status` constant to match
  `trending_drops.py`'s existing convention. None of these are
  over-engineering or scope creep — if anything they're simplifications
  relative to the phase docs' literal instructions. No objection.
- **`DiscoveryOutfit.to_admin_dict()`'s `has_dead_items` doesn't also flag
  "over the 4-item cap"** (`models/discovery.py:171-174`) — the admin SPA
  compensates with a client-side `items.length > 4` check
  (per phase-04 report). Since item count can never exceed 4 through any
  server-enforced write path (pydantic caps both create and replace-items at
  4), this is genuinely unreachable in practice, not a real gap — the
  client check is defensive redundancy, not a missing feature. No action
  needed.
- **`ItemDetail` gaining `discoveryOutfitId` and `DiscoveryOutfitDetail`
  gaining `source?: 'deep_link'`** (both flagged by mobile-dev as deviations
  from the plan's literal param list) — both are minimal, single-purpose
  additions purely for analytics correlation, not speculative generality.
  No YAGNI violation.
- **No dedicated router-level test for admin `DELETE /{id}`** beyond a
  repository-level test — already flagged by both backend-dev and tech-lead
  as a non-blocking, thin-passthrough route. Agreed, low value to add.

---

## Contract Fidelity — second-pass spot check

Independently verified 3 fields tech-lead's sign-off already checked, plus
the one item that wasn't in tech-lead's table:

- **404 envelope** (`API_DOCUMENTATION.md:2882-2889` vs
  `routers/discovery.py:94-102`): doc says
  `{"error": "Outfit not found", "message": "No such outfit.", "request_id": ...}`
  — code matches exactly, field names and casing included.
- **List response shape** (`API_DOCUMENTATION.md:2812-2823` vs
  `routers/discovery.py:71-77`): `outfits`/`count`/`total`/`limit`/`offset`
  — matches, and `count` (page size) vs `total` (filtered-set size)
  semantics match the doc's clarifying note.
- **Item DTO fields** (`API_DOCUMENTATION.md` detail example vs
  `services/discovery_service.py:87-99` `_item_dto`): `id`, `position`,
  `name`, `image_url`, `image_png`, `category`, `category_code`,
  `layer_code`, `is_common_item` — all 8 present in both, no drift.
- **Admin `PUT /{id}` scope** — doc says content-fields-only, item
  membership goes through the dedicated route; confirmed in code
  (`DiscoveryOutfitUpdateRequest` in `routers/admin/discovery_outfits.py:69-85`
  has no `item_ids` field) and in the admin SPA's typed client
  (`DiscoveryOutfitUpdateInput` in `discoveryOutfitsService.ts:61-63`
  `Pick`s only content fields).

No drift found between doc, backend code, and the admin SPA's typed client.

---

## Test Coverage — verified against the specific failure modes asked for

- **404 parity (missing vs unpublished):**
  `tests/test_discovery_public_router.py:266-281` —
  `test_detail_draft_id_returns_404` explicitly diffs envelope key-sets and
  `error` value between a real DRAFT id and a random UUID. Confirmed
  present and correctly asserting non-distinguishability, not just status
  code parity.
- **Publish-gate rejection:**
  `tests/test_discovery_admin_router.py:226-249` —
  `test_patch_publish_with_zero_live_items_returns_422_status_unchanged`
  asserts both the 422 AND that the row stays `DRAFT` (not just "request
  failed"). Confirmed.
- **Item cap (1..4) at create AND replace:**
  `test_create_with_five_item_ids_returns_422` +
  `test_put_items_five_ids_returns_422` — both present, both assert 422.
  **Not separately tested:** the publish-time re-check
  (`assert_publishable`'s `live_count > MAX_ITEMS_PER_OUTFIT` branch) is
  unreachable via any tested write path (items can never exceed 4 once
  written) — this is dead defensive code more than an untested branch; not
  a real coverage gap.
- **Soft-deleted item exclusion:**
  `test_detail_after_soft_deleting_last_item_returns_404_not_500` (public) +
  `test_patch_publish_with_zero_live_items_returns_422_status_unchanged`
  (admin, via soft-deleting after creation) — both confirmed present and
  actually soft-deleting via `item.is_deleted = True` then re-querying,
  not just constructing a pre-deleted fixture.
- **Duplicate item_ids** (finding #3 above) — **not covered**, confirmed by
  grep across both admin test files; no test constructs a payload with a
  repeated id.
- **Mobile deep-link kind-specific test:** confirmed present and non-trivial
  — `deepLinkHandler.test.ts` has dedicated `describe` blocks for
  `discovery-outfit` parse (custom scheme, universal link, missing-id
  rejection), dispatch (navigate-when-authed, stash-when-logged-out,
  replay-after-login), and the `markAuthDeepLinkSeen` non-marking regression
  for both cold and warm start. This is not just "old kinds still pass" —
  it's real coverage of the new kind's specific branches. **Not covered:**
  the onboarding-tree edge case in finding #2 — no test exercises a root
  route name other than `'Auth'`/the main authed tree, so the gap traced
  above is untested as well as unfixed.

---

## Unresolved Questions

1. Should finding #2 (onboarding deep-link loss) block merge, or ship with a
   fast-follow given it requires signup-then-tap-link-during-onboarding
   timing (mobile-dev already called this "low likelihood")? Recommend
   fixing before merge since the fix is ~3 lines and Discovery's whole
   raison d'être for the deep-link phase is the social-share growth loop —
   losing it silently for exactly new users is the worst-case audience to
   lose it for.
2. Devops pre-launch items already flagged by backend-dev/tech-lead/phase-04
   report (S3 `discovery_outfits/` prefix public-read ACL, Postgres
   migration dry-run) are outside this review's scope but are real
   blockers-before-prod-traffic, not blockers-before-merge. Flagging again
   only to make sure they don't get lost between three separate reports.
3. No live-backend / simulator smoke was run by mobile-dev or backend-dev
   (both reports self-flag this). This review is a static code review, not
   an execution verification — recommend `qa-mobile` still runs the
   exploratory pass (cold/warm deep link, live feed render, See-on-me,
   save-to-wardrobe → Wardrobe grid) called for in the plan before this
   ships, since none of the three delivery reports actually exercised the
   full path end-to-end against a running stack.

---

## Files reviewed (full read, not just diff)

Backend: `models/discovery.py`, `repositories/discovery_repository.py`,
`services/discovery_service.py`, `services/discovery_admin_service.py`,
`routers/discovery.py`, `routers/admin/discovery_outfits.py`,
`tests/test_discovery_public_router.py`, `tests/test_discovery_admin_router.py`,
`API_DOCUMENTATION.md` §Discovery, plus diffs of `app.py`, `routers/__init__.py`,
`routers/admin/__init__.py`, `migrations/env.py`, `tests/conftest.py`.

Admin SPA: `wardrobe-admin/src/services/discoveryOutfitsService.ts`,
`wardrobe-admin/src/services/api.ts` (shared client, FormData handling).

Mobile: `auxi/src/services/deepLinkHandler.ts` (full),
`auxi/src/navigation/AppNavigator.tsx` (traced onboarding vs main branch),
`auxi/src/services/discoveryService.ts`, `auxi/src/hooks/useDiscoveryFeed.ts`,
`auxi/src/hooks/useSaveCommonItemToWardrobe.ts`,
`auxi/src/screens/discovery/DiscoveryOutfitDetailScreen.tsx`,
`auxi/src/screens/discovery/DiscoveryItemStrip.tsx`,
`auxi/src/screens/ItemDetailScreen.tsx` (hook call-order check),
`auxi/src/screens/item-detail/ItemDetailReadPanel.tsx`,
`auxi/src/components/design-system/lib/MButton.tsx`,
`auxi/src/services/__tests__/deepLinkHandler.test.ts` (grep-verified new-kind
coverage), translation diffs (en/fr/vi — all 3 in sync, 31 lines each).
