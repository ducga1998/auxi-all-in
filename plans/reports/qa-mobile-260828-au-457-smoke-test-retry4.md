# qa-mobile — AU-457 Discovery live E2E smoke test (retry #4)

**Status: DONE_WITH_CONCERNS**

## TL;DR

The `clonesrc1a2b` migration fix from the orchestrator unblocked everything the
prior attempt (retry #3, BLOCKED) couldn't reach. Confirmed via API and live
mobile walk-through: **feed card render, filters, outfit detail with items,
See-on-me navigation, and — the highest-priority re-validation — the
idempotent-clone fix (double-tap save → exactly ONE cloned item, verified via
direct DB query + backend log)** all work correctly. Deep link to a real
published outfit now opens with full content (previously 500'd). Zero app
crashes across the whole session.

One **new finding** surfaced during this retry (not present in prior reports):
after using the discovery-outfit deep link to a bogus/removed id and tapping
the "Browse Discovery" recovery button, the resulting Discovery feed screen's
hamburger "Open menu" button stops opening the drawer (reproduced 4x on that
specific screen instance; confirmed working normally everywhere else,
including a fresh-launched Discovery feed reached via normal drawer nav). See
finding below — routed to `mobile-dev`.

Two items remain **not practically testable** this session (best-effort, both
explicitly called out as such rather than faked): onboarding-stash replay
(would need a full fresh signup + email-verification flow) and logged-out
deep link (no obvious "Log out" affordance found in Settings without deeper
exploration than the time budget allowed).

## Environment (confirmed pre-flight, per dispatch — not re-touched)

- `./scripts/mcp-doctor.sh` → clean, MCP stack healthy.
- Simulator: iPhone 17 Pro, iOS 26.5, device id `34528D25-C08D-4E54-89B8-BDA0E3226B7F`, booted.
- App: `com.auxi2026.app` (build 24) already installed.
- Backend `:5001` — `/health` → healthy. **Alembic confirmed at head:**
  `clonesrc1a2b (head)` (was `discovery1a2b` in retry #3).
- Did **not** restart backend/Metro, did not rebuild native, did not touch
  migrations. Did relaunch the **app process** twice via mobile-mcp
  (`terminate_app` + `launch_app`) to clear a stale "Fast Refresh
  disconnected" banner left over from the prior session and once more to
  reset navigation state after the drawer finding — this is an app-process
  relaunch (JS reload), not a native rebuild, and is within qa-mobile's normal
  exploratory-verify toolset.

## Step 0 — API-level re-verification (before touching mobile)

| Check | Result |
|---|---|
| `GET /api/discovery/outfits/67e1fa9c-05e4-413c-a198-dc19c9224546` (authed, normal user `qa-test@auxi.app`) | **200** — full outfit with 3 real items (Knit Sweater · Camel, Jogger Pants · Black, Black LOA (CHK)). Previously 500'd in retry #3. |
| `GET /api/discovery/outfits` (feed) | **200** — `{"outfits":[...1 outfit...],"count":1,"total":1}`. |
| `GET /api/wardrobe/items` (baseline) | **200**, 52 items, 0 with `cloned_from_common_item_id` set — clean baseline for the idempotency check. |

Leftover seed outfit `67e1fa9c-05e4-413c-a198-dc19c9224546` ("QA Smoke —
Weekend Layered Look") reused as instructed — no reseed needed.

Note: `composite_image_url` is `null` on this seed outfit (no cover image was
ever uploaded for it). This is a pre-existing property of the seed data, not
a bug — the mobile card correctly renders a blank/placeholder cover image
area and shows title + chips + item count regardless (see screenshot).

## Step 1 — mobile smoke test (booted sim, mobile-mcp)

Logged-in account on the sim turned out to be `qa-signin@auxi.app` (not
`qa-test@auxi.app` — a different, already-onboarded, non-admin account left
over from a prior session). This satisfies the dispatch's requirement ("a
normal non-admin test user who has already completed onboarding") — Home
showed real AI-generated outfit recommendations, not an onboarding gate.

| # | Check | Result |
|---|---|---|
| a | Launch app, already logged in as onboarded normal user | **PASS** — Home shows "Generating"→real outfit tiles, no login screen, no error state. |
| b | Drawer → Discovery entry, navigate | **PASS** (carries over from retry #3, re-confirmed) — globe icon "Discovery" row present, navigates correctly. |
| c | **Feed card renders (cover, title, season/trend chips)** — NEW this retry | **PASS** — `discovery-card-0` renders "QA Smoke — Weekend Layered Look", "Fall" + "Quiet-Luxury" chips, "3 items" caption. Cover area is blank (expected — no `composite_image_url` set on this seed row, see note above). Screenshot: `qa-mobile-discovery-feed-real-card.png`. |
| d | Season/trend filter — match + no-match | **PASS** — tapped "Fall" chip: card still shows (correct, outfit's season is fall). Tapped "Summer" chip: clean **"No outfits match these filters / Try a different season or tag"** empty state, no crash, no blank screen. |
| e | Tap card → outfit detail loads with items | **PASS** — `DiscoveryOutfitDetailScreen` renders title, description ("Seeded by qa-mobile for AU-457 live smoke test."), season/trend/qa-smoke chips, and all 3 items in `discovery-item-strip-tile-*` with correct names/images. Screenshot: `qa-mobile-discovery-detail-with-items.png`. |
| f | "See on me" → real try-on flow, items pre-populated | **PASS** — `discovery-detail-see-on-me-cta` navigates into `SeeThisOnMe` generation screen (testIDs `stom-*`), showing live progress steps ("Matching your selected body shape" → "Mapping clothing fit and proportions" → "Adjusting layering and garment details" → "Rendering your personalized outfit"), confirming the 3 outfit items were passed through and real generation started. Left via "Leave — notify me when ready" per instructions (no need to wait for completion). Screenshot: `qa-mobile-see-on-me-generating.png`. |
| g | Item tap → item detail, "Save to wardrobe" affordance (not owner edit/delete) | **PASS** — tapped "Knit Sweater · Camel" tile → `ItemDetailScreen` shows `item-detail-save-to-wardrobe-btn` ("Save to wardrobe"). No owner-only edit/delete controls exposed in the accessibility tree for this common item. Screenshot: `qa-mobile-item-detail-save-button.png`. |
| g2 | **Double-tap save-to-wardrobe idempotency — HIGH PRIORITY** | **PASS** — tapped the button twice in rapid succession. UI transitioned to a green "Saved" state. **Verified via backend log**: only **one** `POST /api/wardrobe/common-items/d74302e7.../clone` request reached the server (`201 Created`, single log line). **Verified via direct DB query**: exactly **1** row in `wardrobe_items` with `cloned_from_common_item_id = 'd74302e7-6d78-4e9c-ae02-e0dcf2dba04a'` for this user (`3bd10f56-6590-4ce4-9712-abb093f7c685`, "Knit Sweater · Camel"). No duplicate clone. Also visually confirmed the item now appears in the Wardrobe grid. Screenshot: `qa-mobile-double-tap-save-result.png`. |
| h1 | Deep link, real outfit id (`auxi://discovery-outfit?id=67e1fa9c-...`) | **PASS** — opens directly to `DiscoveryOutfitDetailScreen` with full real content (title, description, chips, See-on-me CTA) — this is the case that 500'd in retry #3 and could not be verified then. Screenshot: `qa-mobile-deep-link-real-outfit-success.png`. |
| h2 | Deep link, bogus id (`auxi://discovery-outfit?id=00000000-...`) | **PASS** — graceful in-place fallback: "This outfit is no longer available / It may have been removed or is no longer published." + `discovery-detail-browse-btn` ("Browse Discovery"). Matches the graceful-404 behavior already confirmed as correct in retry #3. Screenshot: `qa-mobile-deep-link-bogus-id-fallback.png`. |
| i | Onboarding-stash replay | **NOT PRACTICAL / SKIPPED** — would require a fresh signup + email-verification flow within this session; explicitly skipping per dispatch instruction rather than faking a result. |
| j | Logged-out deep link | **NOT PRACTICAL / SKIPPED** — searched Settings screen (`settings-*` rows: daily reminder, personalization, privacy, about, reset data, delete account) for a sign-out control and found none exposed at that screen; didn't want to force a logout via account deletion or token-wipe workaround within the time budget. Explicitly flagging as unverified rather than assuming pass. |
| k | Regression — Home, Wardrobe | **PASS** — Home shows real generated outfit (not an error state). Wardrobe loads normally (previously 500'd in retry #3), shows the just-cloned sweater. `mobile_list_crashes` → `[]` throughout the whole session. |

## New finding — drawer "Open menu" stops working after deep-link recovery flow

**Severity**: minor (workaround exists — force-quit/relaunch, or back-navigate
before the drawer is needed; does not block the core AU-457 flows above)
**Repro rate**: 4/4 taps on the affected screen instance; 0/4+ on every other
screen instance tested this session (Home, Favourite, Wardrobe, Feedback, and
even a normally-reached Discovery feed all opened the drawer correctly on
first tap).

**Repro steps:**
1. From any screen, open `auxi://discovery-outfit?id=<bogus-uuid>`.
2. App shows the graceful "This outfit is no longer available" fallback.
3. Tap "Browse Discovery" (`discovery-detail-browse-btn`).
4. Lands on the Discovery feed (`DiscoveryScreen`) — looks and behaves
   identically to a normally-reached feed (filter chips work, card taps
   work).
5. Tap the hamburger "Open menu" button (`discovery-menu-button`, top-left).
6. **Nothing happens** — no drawer slide-in, `discovery-menu-button`
   coordinates stay unshifted in the accessibility tree (a working drawer
   open always produces an `x`-coordinate shift on background elements,
   confirmed present on every other screen in this session).

**Suspected area**: `auxi/src/services/deepLinkHandler.ts` (deep-link push
target) and/or `auxi/src/navigation/AppNavigator.tsx` — the deep-link handler
likely pushes `DiscoveryOutfitDetail`/`DiscoveryFeed` onto a stack that isn't
nested inside the drawer navigator (or the "Browse Discovery" navigate call
doesn't restore drawer context), so the menu button's `openDrawer()` call has
no parent drawer to open. Not confirmed at the code level — this is a
behavioral finding, not a code read.

**Routing**: `mobile-dev` — please confirm whether the deep-link/`Browse
Discovery` navigation path re-enters the drawer navigator correctly, or file
a fix if it's pushing outside that context. Workaround for now: force-quit
and relaunch, or tap a stack "Back" button if present instead of using the
drawer.

## What carries over as already-PASS from retry #3 (not re-verified in depth this time)

- Duplicate-item-id validation on admin create (Medium #3 fix) — pure input
  validation, unaffected by the migration gap either way, previously PASS.
- Deep-link 404-vs-500 differentiation concept — now both paths (real outfit,
  bogus outfit) return clean success/graceful-fallback states with real data
  behind them (this retry closes out the "with real content" half that
  retry #3 couldn't reach).
- Drawer → Discovery entry point exists and is correctly labeled.
- Zero-crash error-boundary pattern across screens.

## Screenshots

All under `auxi/docs/qa-findings/screenshots/2026-08-28/`:
- `qa-mobile-discovery-feed-real-card.png`
- `qa-mobile-discovery-detail-with-items.png`
- `qa-mobile-see-on-me-generating.png`
- `qa-mobile-item-detail-save-button.png`
- `qa-mobile-double-tap-save-result.png`
- `qa-mobile-deep-link-real-outfit-success.png`
- `qa-mobile-deep-link-bogus-id-fallback.png`

## Crash check

`mobile_list_crashes` → `[]` at end of session. Zero crashes across the
entire retry, including through two app relaunches and the drawer-not-opening
finding.

## What I did NOT do

- Did not modify `auxi/src/**` or any backend code.
- Did not run any Alembic migration (orchestrator confirmed already at head
  before this session started; I only ran read-only `alembic current` to
  re-verify).
- Did not author/edit Maestro YAML.
- Did not fabricate a result for onboarding-stash replay or logged-out deep
  link — both explicitly marked NOT PRACTICAL / SKIPPED.
- Did not force a logout via account deletion or manual token-wipe to chase
  the logged-out deep-link check — judged out of the "quick verify" time
  budget for a best-effort item.

## Recommended next steps (routing)

1. **mobile-dev** — investigate the drawer "Open menu" button becoming
   non-functional after the discovery-outfit deep-link → "Browse Discovery"
   recovery path (new finding, minor severity, repro steps above).
2. **qa-mobile (future, low priority)** — logged-out deep link and
   onboarding-stash replay remain unverified; revisit if/when a lighter-weight
   way to reach those states exists (e.g., a documented logout affordance, or
   a disposable test account seeded in "mid-onboarding" state).
3. No blockers remain for AU-457's core Discovery flow — feed, filters,
   detail, See-on-me, save-to-wardrobe (with verified idempotency), and deep
   links (both success and graceful-failure paths) are all confirmed working
   against live data.

**Status:** DONE_WITH_CONCERNS
**Summary:** All previously-blocked checks now pass against live data:
feed/card rendering, filters (match + no-match empty state), outfit detail
with items, See-on-me navigation into real generation, and — highest
priority — save-to-wardrobe double-tap idempotency (exactly 1 clone,
verified via backend log + DB query). Deep link to a real outfit now opens
with full content; bogus-id fallback remains graceful. Zero crashes.
**Concerns/Blockers:** (1) NEW minor finding — drawer hamburger button
stops working on the Discovery feed reached via the deep-link-recovery
("Browse Discovery") path, routed to mobile-dev; (2) onboarding-stash replay
and logged-out deep link remain genuinely untested (best-effort, explicitly
not faked).
