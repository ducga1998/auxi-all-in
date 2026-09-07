# mobile-dev — AU-457 drawer-stops-opening fix (deep-link "Browse Discovery" recovery)

**Status: DONE_WITH_CONCERNS** (fix implemented + tested; visual confirmation on simulator not performed — no mobile-mcp access in this session, see below)

## TL;DR

QA's hypothesis ("deep link pushes Discovery outside a drawer navigator") is
**not correct as stated** — there is no `react-navigation` Drawer navigator in
this app at all. The sidebar/hamburger drawer is a custom root-level
"push-drawer" (`RootDrawer` + `SidebarContext`) that wraps the **entire** app
outside `NavigationContainer`, so every screen — regardless of how it was
reached — has the same drawer context. That part of the hypothesis is ruled
out by inspection.

The **actual root cause**: `DiscoveryOutfitDetailScreen.handleBrowseDiscovery`
recovers from the deep-link 404/unpublished fallback with a plain
`navigation.navigate('Discovery')`. This codebase has already hit — and
already fixed, twice — exactly this class of bug: a plain `navigate()` that
pops back across N screens (or past a modal) updates the JS navigation state
correctly but can leave `react-native-screens`' **native** view/gesture
teardown desynced, so a screen that's already gone from the JS state still has
a live native touch/gesture responder intercepting taps on the screen now on
top. The two existing precedents in this same repo:

- `src/screens/ItemDetailScreen.tsx` `handleBuildAround` — switched from
  `navigate('Home', …)` to `navigation.popTo('Home', …)` for exactly this
  reason (comment: "navigate() … can leave the native iOS modal still
  presented → desync: the sheet stays stuck on top and nothing responds").
- `src/screens/see-this-on-me/try-on-completion-notice.ts` — switched to
  `navigationRef.dispatch(StackActions.popTo('SeeThisOnMe', …))` for the same
  reason ("navigate() only updates JS nav state and leaves the native modal
  stuck on screen").

`handleBrowseDiscovery` was the one call site in the Discovery feature that
still used plain `navigate()` for a "leave a dead-end screen behind" recovery
action — the deep-link path can pop through an arbitrary number of screens
(whatever the deep link itself pushed) to reach `Discovery`, which is exactly
the shape that has broken native teardown here before. The reported symptom
(hamburger silently no-ops, 4/4, only on that specific screen instance; feed
scrolling/filtering/card-taps on the SAME screen instance work fine) is
consistent with a stale native gesture responder from the torn-down screen
still capturing the header tap area specifically, while RN's normal touch
delivery to the rest of the screen is unaffected.

Sidebar-reached Discovery (0/4+ repro) never hits this because `SidebarMenu`'s
`go()` navigates from a screen that (in the ordinary flow) doesn't already
have `Discovery` sitting further back in the stack behind extra pushed
screens — a single-level push/pop, not the multi-hop pop the deep-link
recovery path can produce.

## Root cause — confirmed vs QA's hypothesis

| QA hypothesis | Verdict |
|---|---|
| Deep link pushes Discovery outside the drawer's nesting | **Wrong** — there is no drawer navigator; drawer context (`SidebarContext`) is a single root-level React Context wrapping the whole app, unaffected by which screen is focused. |
| Header's `openDrawer()`/`toggleDrawer()` has no parent drawer to open | **Wrong premise** (same reason) — the header calls `useSidebar().open`, a stable `useCallback` from a Provider mounted once at `App.tsx` root; it's identical for every screen instance. |
| (Not raised by QA, found by tracing) plain `navigate()` recovery call desyncs native-stack teardown vs JS state, matching two prior fixes in this exact codebase | **Confirmed as the actual mechanism** — same class of bug, same codebase, same fix already proven twice. |

## Fix

`auxi/src/screens/discovery/DiscoveryOutfitDetailScreen.tsx` —
`handleBrowseDiscovery`: replaced `navigation.navigate('Discovery')` with
`navigation.popTo('Discovery')`.

`popTo` (React Navigation 7, already used elsewhere in this codebase and
already available on this screen's `NativeStackNavigationProp` typing)
resolves identically to `navigate` in terms of *destination* (pops to an
existing `Discovery` instance if one is in the stack, otherwise pushes a
fresh one — same resolution `SidebarMenu`'s `go('Discovery', close)` uses) but
issues real *pop* semantics so `react-native-screens` properly tears down the
screen(s) being left behind, instead of a JS-only state update.

```ts
const handleBrowseDiscovery = () => {
  toast.show({ type: 'info', text1: t('discovery.outfit_unavailable_toast'), position: 'bottom' });
  navigation.popTo('Discovery'); // was: navigation.navigate('Discovery')
};
```

Full rationale is captured as a code comment at the call site (cites the two
prior `popTo` fixes by name so a future reader doesn't have to re-derive it).

## Files changed

- `auxi/src/screens/discovery/DiscoveryOutfitDetailScreen.tsx:71-92` — the fix.
- `auxi/src/screens/discovery/__tests__/DiscoveryOutfitDetailScreen.test.tsx`
  — new regression test (see below).

## Regression test

Added `DiscoveryOutfitDetailScreen.test.tsx`, mirroring the existing
`ItemDetailScreen.test.tsx` `popTo`-regression pattern (mock
`@react-navigation/native`'s `useNavigation` with distinct `navigate`/`popTo`
spies, mock `discoveryService.getOutfit` to resolve `null` → the "not found"
branch, press `discovery-detail-browse-btn`):

```
DiscoveryOutfitDetailScreen — "Browse Discovery" recovery CTA
  ✓ pops to Discovery (not navigate) when the outfit is missing/unpublished
```

Asserts `popTo` was called with `'Discovery'` and `navigate` was **not**
called with `'Discovery'`. Ran 5x in isolation to rule out flakiness (react-
query's notify-manager batching needed a `setTimeout(0)`-based flush, matching
`HomeScreen.test.tsx`'s `flushTimersAndPromises` pattern, added to avoid an
initial flaky failure).

**Scope/limits of this test — stated explicitly, not glossed over**: this is
a *navigation-call* regression test. It proves the screen now calls the
pop-semantics API instead of the JS-only one, matching the two prior fixes'
verified remedy. It **cannot** exercise the actual native `react-native-
screens` teardown/gesture-responder behavior — that requires a real
simulator with real native views, which Jest's `react-test-renderer` doesn't
provide (this mirrors the same limitation the two prior `popTo` fixes' own
tests have — neither of those tests re-creates the native modal-stuck
repro either, they assert the same "correct call, not the wrong call" shape).
I did not force a fake assertion to paper over that gap.

I also checked `deepLinkHandler.ts` and its existing test suite
(`deepLinkHandler.test.ts`) — unchanged, no fix needed there. The deep-link
handler's own navigation (`navRef.navigate('DiscoveryOutfitDetail', …)`,
route-then-resolve) is a single-level push from whatever screen was current
when the link opened; it isn't part of this bug (the bug is specifically in
the *recovery* call one screen later). Left as-is; all 57 existing tests in
that file still pass.

## Verification

- `npx tsc --noEmit` — 3 pre-existing errors remain, all in
  `src/screens/see-this-on-me/{components.tsx,StomStepLayout.tsx}` (missing
  `poppinsTimeLg`/`poppinsBodySm` theme typography keys) — **unrelated to this
  change**, pre-existing in the working tree before I touched anything (confirmed:
  these files are not in my diff). Zero errors in `discovery/**` or the new
  test file.
- `yarn lint` — 2 pre-existing errors (`web/mocks/handlers.ts`,
  `web/stubs/toast.tsx`, both unused-var, unrelated to `auxi/src/**`) + 24
  warnings, none in my changed files. Ran `npx eslint` scoped to both changed
  files directly — clean, zero problems.
- `npx jest src/screens/discovery src/services/__tests__/deepLinkHandler.test.ts`
  — new test passes (5/5 repeat runs); all 57 existing `deepLinkHandler.test.ts`
  cases still pass. (Note: `ItemDetailScreen.test.tsx` has 13 pre-existing
  failures in this working tree, unrelated to my change — that file and its
  sibling `ItemDetailReadPanel.tsx` are already modified/uncommitted from
  other in-progress work in this repo, not touched by me this task.)
- Simulator verification: **NOT performed.** Per this repo's tool-grant
  convention (`auxi/CLAUDE.md` / umbrella `CLAUDE.md` mobile-mcp tiers),
  `mobile-dev` does not carry mobile-mcp tools — sim verification is qa-
  mobile's/qa-ui's job. I did not have mobile-mcp available in this session to
  run the exact repro (bogus deep link → Browse Discovery → tap hamburger →
  confirm drawer opens). **This is a pure JS/TS change (Fast Refresh
  applies), code complete, visual/behavioral verification pending** —
  recommend a qa-mobile pass re-running QA's exact retry #4 repro steps
  against the booted sim (device `34528D25-C08D-4E54-89B8-BDA0E3226B7F`) to
  close this out with a live confirmation.

## Unresolved / handoff

1. **Needs qa-mobile re-verification on the sim** to confirm the fix actually
   resolves the native-level symptom (the regression test proves the *call*
   changed; it can't prove the native teardown is now correct — that's the
   whole reason this class of bug is subtle).
2. Minor, unrelated observation (not fixed, out of scope for this task):
   `DiscoveryScreen.tsx`'s hamburger reuses
   `t('wardrobe.list.a11y_open_menu')` for its `accessibilityLabel` instead of
   a Discovery-scoped key. Harmless (same "Open menu" copy either way) — flagging
   only in case a future i18n pass wants a `discovery.a11y_open_menu` key for
   clarity, not fixing it here since it's not the bug and not a regression I introduced.

## Files (absolute paths)

- `/Users/hiep/Source/auxi-all-in/auxi/src/screens/discovery/DiscoveryOutfitDetailScreen.tsx`
- `/Users/hiep/Source/auxi-all-in/auxi/src/screens/discovery/__tests__/DiscoveryOutfitDetailScreen.test.tsx`
- Referenced precedent (unchanged, read-only): `/Users/hiep/Source/auxi-all-in/auxi/src/screens/ItemDetailScreen.tsx`,
  `/Users/hiep/Source/auxi-all-in/auxi/src/screens/see-this-on-me/try-on-completion-notice.ts`
- Read-only trace: `/Users/hiep/Source/auxi-all-in/auxi/src/services/deepLinkHandler.ts`,
  `/Users/hiep/Source/auxi-all-in/auxi/src/navigation/AppNavigator.tsx`,
  `/Users/hiep/Source/auxi-all-in/auxi/src/components/layout/RootDrawer.tsx`,
  `/Users/hiep/Source/auxi-all-in/auxi/src/components/layout/SidebarMenu.tsx`,
  `/Users/hiep/Source/auxi-all-in/auxi/src/context/SidebarContext.tsx`

**Status:** DONE_WITH_CONCERNS
**Summary:** Root cause found (not QA's literal hypothesis — no drawer
navigator exists, so "missing drawer context" was ruled out; the real
mechanism is a plain `navigate()` recovery call desyncing native-stack
teardown, the same class of bug already fixed twice elsewhere in this
codebase via `popTo`). Fixed by switching `handleBrowseDiscovery` to
`navigation.popTo('Discovery')`. Added a regression test proving the correct
navigation call is now made. `tsc`/`lint` clean on all touched files.
**Concerns/Blockers:** No simulator access in this session (mobile-dev has no
mobile-mcp grant) — the fix is code-complete but the live drawer-reopens
behavior has not been visually reconfirmed; needs a qa-mobile re-run of the
exact retry #4 repro on the already-booted sim.
