# qa-mobile — AU-457 drawer-fallback fix re-verification

**Status: DONE**

## What was verified

Fix: `mobile-dev-260828-au-457-drawer-fallback-fix.md` — `DiscoveryOutfitDetailScreen.handleBrowseDiscovery`
changed `navigation.navigate('Discovery')` → `navigation.popTo('Discovery')` to fix the
react-native-screens touch-responder desync that left the Discovery hamburger
("Open menu") non-functional after reaching the feed via the deep-link
recovery path. JS-only change; no rebuild, Fast Refresh / fresh dev bundle on
each launch picked it up. Scope: this repro only, not a full AU-457 re-smoke
(everything else already covered in `qa-mobile-260828-au-457-smoke-test-retry4.md`).

## Environment

- `./scripts/mcp-doctor.sh` → healthy (sim booted, WDA up on :8100).
- Simulator: iPhone 17 Pro, iOS 26.5, `34528D25-C08D-4E54-89B8-BDA0E3226B7F`.
- App: `com.auxi2026.app` build 24, already installed — not rebuilt.
- Did not restart backend/Metro/simulator. Did terminate/relaunch and
  reload the app process itself several times (JS reload only) while
  diagnosing a delivery quirk in the deep-link trigger (see note below) —
  not a native rebuild.

## Trigger note (environmental, not a fix regression)

`xcrun simctl openurl <device> "auxi://discovery-outfit?id=..."` (the only
mechanism available to me — `mobile_open_url` is not in this session's tool
grant, so I used the CLI equivalent) reliably delivered the deep link **only
when fired while the Discovery/DiscoveryOutfitDetail route was already
mounted in the stack** (i.e., after having visited Discovery at least once
this session). Fired from a cold launch or from Home before ever visiting
Discovery, the link silently no-oped (confirmed via `log stream` that iOS
delivered the `UIOpenURLAction` to the app process; confirmed via a control
test with `auxi://verify-email?token=...` that the JS `Linking` bridge and
`dispatchDeepLink` path are wired and firing generally — that link correctly
produced React Navigation's "action NAVIGATE not handled" dev warning). This
looks like a `isDiscoveryRouteMounted` / route-readiness quirk in
`deepLinkHandler.ts` unrelated to the `popTo` fix under test (which only
touches `handleBrowseDiscovery`, downstream of the screen already having
mounted). Once I visited Discovery once via the drawer, every subsequent
`auxi://discovery-outfit?id=<bogus>` deep link landed on the fallback screen
reliably and repeatably (used for all 4 repro attempts below) — this matches
the exact screen/state the original bug report describes (deep-link entry →
"no longer available" fallback → Browse Discovery → broken hamburger). Not
filing this as a new bug (out of scope for a narrow re-check with 4/4
successful triggers), but flagging for whoever next touches
`deepLinkHandler.ts`/`isDiscoveryRouteMounted` — worth a real look if a
future deep-link QA pass sees a "link does nothing from cold/Home" report.

## Repro — 4/4 attempts

Steps each time: `xcrun simctl openurl ... "auxi://discovery-outfit?id=does-not-exist-bogus-id"`
→ confirm "This outfit is no longer available" + `discovery-detail-browse-btn`
→ tap `discovery-detail-browse-btn` → lands on `DiscoveryScreen`
(`discovery-menu-button` present) → tap the hamburger → check background
element x-coordinates shift (the same signal used in retry #4 to confirm a
functioning drawer: `discovery-menu-button` and sibling elements shift from
`x=12` to `x=333` when the drawer slides open).

| Attempt | Fallback reached | Browse Discovery → Discovery feed | Hamburger opens drawer |
|---|---|---|---|
| 1 | PASS | PASS | **PASS** (x: 12→333, confirmed via screenshot — drawer visibly open, "Discovery" row highlighted) |
| 2 | PASS | PASS | **PASS** (x: 12→333) |
| 3 | PASS | PASS | **PASS** (x: 12→333) |
| 4 | PASS | PASS | **PASS** (x: 12→333) |

**4/4 — the drawer opens every time.** This is a clean reversal of the
previous 4/4 failure reported in retry #4.

## Sanity check — normal path not broken

Navigated Home → drawer → Discovery (the ordinary, non-deep-link entry
point) and tapped the hamburger there too: **PASS** (x: 12→333, drawer
visibly opened with "Discovery" row highlighted). The fix does not regress
the already-working path.

## Screenshot

`auxi/docs/qa-findings/screenshots/2026-08-28/qa-mobile-drawer-fallback-verify-attempt1.png`
— drawer open after attempt 1, "Discovery" row highlighted, confirming the
fix's target state.

## Crash check

`mobile_list_crashes` → one entry, unrelated: `Path of Exile` at `13:26:14`
(a different macOS app, timestamped before this verification session's
testing began). Zero `auxi` crashes across all 4 repro attempts + the sanity
check.

## What I did NOT do

- Did not modify `auxi/src/**`.
- Did not rebuild native or restart Metro/simulator/backend.
- Did not re-run the full AU-457 smoke suite (out of scope per dispatch —
  see retry #4 report for full coverage).
- Did not file a new finding for the deep-link cold/Home delivery quirk
  above — noted as an observation only, since it didn't block reaching the
  target repro state and is orthogonal to the fix being verified.

**Status:** DONE
**Summary:** The `popTo` fix resolves the drawer "Open menu" bug — 4/4 repro
attempts now show the drawer opening correctly after the deep-link recovery
path (previously 4/4 failures in retry #4), and the normal drawer→Discovery
path still works (no regression). Zero app crashes.
**Concerns/Blockers:** None blocking. One environmental observation (deep-link
delivery from cold-start/Home before Discovery has been visited once this
session appears to no-op) — informational only, not re-tested against main
line as a bug, and not something the `popTo` fix touches.
