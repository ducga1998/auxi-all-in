# Phase 09 — Deep link `discovery-outfit`

**Repo:** `auxi/` · **Owner:** mobile-dev · **Effort:** 2h
**Status:** pending · **Blocked by:** phase 08

## Context (verified)

- `auxi/src/services/deepLinkHandler.ts` today supports exactly two kinds —
  `DeepLinkKind = 'verify-email' | 'reset-password'` (`:43`), gated by `SUPPORTED_SLUGS` (`:52-55`).
- Parser: `parseDeepLink` (`:109-142`) accepts `auxi://<slug>?…` (slug may be host or first path
  segment, `:125-129`) and `https://auxi.app/<slug>?…` (`:41`, `:118-119`).
  **It currently requires a `token` query param and returns `null` without one (`:134-135`)** — the
  new kind carries an `id`, so this check must become kind-aware, not unconditional.
- Dispatch: `dispatchDeepLink` (`:186-257`), cold-start stash + `replayPendingDeepLink` (`:265-272`),
  listeners `registerDeepLinkListeners` (`:281-309`).
- `markAuthDeepLinkSeen()` (`:155`) is an **auth-recovery** guard — it currently fires for EVERY
  parsed link (`:290`). A Discovery link is not auth recovery; scope that call to the two auth kinds
  or it will suppress legitimate session-expired toasts.
- Custom scheme already registered: iOS `ios/auxi/Info.plist:37-40` (`auxi`), Android
  `android/app/src/main/AndroidManifest.xml:27-33`. **No native manifest change needed** for
  `auxi://discovery-outfit?id=…`. Universal links (`https://auxi.app/…`) still need AASA hosting —
  deferred, exactly as the file header says (`:12-17`). Ship the custom scheme; the universal-link
  path parses already and will Just Work when AASA lands.

## Requirements

1. `DeepLinkKind` gains `'discovery-outfit'`; `SUPPORTED_SLUGS` gains the slug.
2. `ParsedDeepLink` becomes a discriminated union (or `token?` + `id?`): auth kinds require `token`;
   `discovery-outfit` requires `id`. Missing required param → `null` (link ignored, as today).
3. Dispatch for `discovery-outfit`:
   - nav not ready → existing stash/replay path (no new mechanism).
   - **Not logged in** → the deep link must survive login. Reuse the pending-link slot: keep it
     stashed and replay after auth (`replayPendingDeepLink` is already called from
     `NavigationContainer.onReady`). If the authed tree isn't mounted, do NOT navigate into it.
   - Logged in → `navigate('DiscoveryOutfitDetail', { outfitId: id })`. The screen itself resolves
     the fetch; a 404 (service `null`, phase 06) renders the unavailable state.
   - **Graceful fallback:** unresolvable outfit → `navigate('Discovery')` + an info toast
     ("That outfit isn't available anymore"). Never an error screen, never a crash.
     Decision: resolve-then-route (call `getOutfit(id)` in the handler) vs route-then-resolve
     (let the detail screen 404). **Choose route-then-resolve + in-screen fallback CTA** — one
     network path, no double fetch, and it works identically for a cold start.
4. Analytics: `discovery_deep_link_opened` with `{ outfit_id, resolved: true|false }`.

## Related code files

- MODIFY `auxi/src/services/deepLinkHandler.ts` (types, `SUPPORTED_SLUGS`, param validation,
  dispatch branch, scope `markAuthDeepLinkSeen` to auth kinds)
- MODIFY `auxi/src/services/analytics.ts` (event)
- MODIFY `auxi/src/translations/*` (toast copy)
- CREATE/MODIFY `auxi/src/services/__tests__/deepLinkHandler.test.ts` (jest — parser cases)
- No native files change.

## Implementation steps

1. Widen the types; make the `token` requirement kind-conditional (this is the one real regression
   risk in the file — cover it with parser tests for both auth kinds).
2. Add the dispatch branch + toast fallback.
3. Scope `markAuthDeepLinkSeen()` to `verify-email` / `reset-password`.
4. Jest tests: `auxi://discovery-outfit?id=X` → parsed; missing `id` → null;
   `auxi://verify-email?token=T` → still parsed (regression); unknown slug → null;
   `https://auxi.app/discovery-outfit?id=X` → parsed.
5. Manual: `xcrun simctl openurl booted "auxi://discovery-outfit?id=<real-id>"` (warm and cold).

## Todo

- [ ] parser + dispatch + fallback toast
- [ ] `markAuthDeepLinkSeen` scoped
- [ ] jest parser tests incl. auth regressions
- [ ] warm-start + cold-start manual verification on sim
- [ ] analytics event

## Success criteria

- Warm start on a published outfit id → lands on that outfit's detail.
- Cold start (app killed) → same, after the nav tree mounts.
- Unpublished/deleted id → Discovery feed + info toast, no crash.
- Logged-out → link survives to post-login (or is dropped cleanly, never crashing the auth stack).
- Existing verify-email / reset-password links still work (jest + one manual check).

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Making `token` optional breaks auth-link parsing | **M×H** | Kind-conditional validation + regression tests for both auth kinds |
| `markAuthDeepLinkSeen` firing for discovery links suppresses real session-expired toasts | M×M | Scope it to the two auth kinds |
| Deep link fires while logged out → navigate into an unmounted authed tree | M×H | Stash/replay path; never navigate when `navRef` isn't ready |
| Universal link shared on social doesn't open the app (no AASA) | H×M | Ship custom scheme now; document the AASA follow-up (pre-existing, `deepLinkHandler.ts:12-17`) |

## Security

The link carries only an opaque outfit id. The detail fetch is authenticated and server-side gated
on servability — an id in a public post cannot expose a DRAFT outfit (phase 03 404).

## Next steps

Phase 10 — analytics doc sync + verification gates.
