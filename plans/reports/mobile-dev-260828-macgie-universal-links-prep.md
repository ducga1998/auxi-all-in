# Universal-link host switch: auxi.app → macgie.com (AU-457 prep)

File-only prep. No native rebuild triggered. `yarn ios:clean` / pod install /
Metro kill were NOT run.

## Changes

1. `auxi/src/services/deepLinkHandler.ts`
   - `SUPPORTED_HOSTS = new Set(['auxi.app'])` → `new Set(['macgie.com', 'www.macgie.com'])`
   - JSDoc header updated: Universal Link examples now show `https://macgie.com/...`
     for `verify-email` / `reset-password` / `discovery-outfit`, and the AASA
     hosting-path comment now points at `https://macgie.com/.well-known/apple-app-site-association`.
   - Left `cdn.auxi.app` alone everywhere (it's not referenced in this file —
     that host lives in test fixtures / CDN URLs, unrelated to deep-link routing).

2. `auxi/src/services/__tests__/deepLinkHandler.test.ts`
   - Line ~92: `parseDeepLink('https://auxi.app/discovery-outfit?id=outfit-2')` →
     `https://macgie.com/discovery-outfit?id=outfit-2`.
   - Line ~121: renamed test `'returns null for a non-auxi, non-auxi.app URL'` →
     `'returns null for a non-auxi, non-macgie.com URL'`. Body already used
     `https://example.com/...`, an unrelated host that correctly fails
     `SUPPORTED_HOSTS` regardless of what the supported host is — only the
     name needed updating, not the URL.
   - Left `cdn.auxi.app` composite-image-URL assertions (~160/165) untouched —
     confirmed unrelated: that's the push-notification try-on composite CDN
     host, never passed through `parseDeepLink`/`SUPPORTED_HOSTS`.
   - Left the `https://auxi.app/promo` "external kind" test (~257/260)
     untouched. Reasoning: `resolveNotificationData`'s `external` branch does
     a bare `isHttpUrl` regex check and calls `Linking.openURL(url)` directly
     — it never touches `SUPPORTED_HOSTS` or `parseDeepLink`. The URL here is
     an arbitrary "any https URL" fixture for that unrelated code path, not a
     host-support assertion, so changing the host wouldn't change what's
     being tested and isn't required. Left as-is rather than churning an
     unrelated fixture.

3. `auxi/ios/auxi/auxi.entitlements`
   - Added `com.apple.developer.associated-domains` array
     (`applinks:macgie.com`, `applinks:www.macgie.com`) as a sibling to the
     existing `aps-environment` and `com.apple.developer.applesignin` keys.
   - `plutil -lint` → `OK`.

4. `auxi/android/app/src/main/AndroidManifest.xml`
   - Added a second `<intent-filter android:autoVerify="true">` (HTTPS App
     Links, `macgie.com` + `www.macgie.com`) as a sibling to the existing
     `auxi-deeplink` custom-scheme filter inside `MainActivity`.
   - Added a one-line comment above it pointing at
     `auxi/docs/deep-linking/cloudflare-setup.md` Step 4 for the real
     assetlinks.json / signing-cert fingerprint requirement (didn't duplicate
     the doc's wording, just referenced it per the task).
   - Eyeballed the XML — well-formed, tags balanced, matches existing style.

## Context read (not modified)

`auxi/docs/deep-linking/cloudflare-setup.md`, `apple-app-site-association`,
`assetlinks.json` — confirms the promised paths/appID/hosts match what I
wired into the entitlements/manifest (`macgie.com` + `www.macgie.com`, both
`.well-known/apple-app-site-association` and `.well-known/assetlinks.json`).
That doc already documented (pre-written by whoever staged it) that the app
side "ships in the next build" with exactly this entitlements/manifest
shape — my changes match it.

## Verification

- `npx tsc --noEmit` — no new errors from my changes. 3 pre-existing errors
  remain in `src/screens/see-this-on-me/{components.tsx,StomStepLayout.tsx}`
  (`poppinsTimeLg`/`poppinsBodySm` missing from theme font tokens) —
  confirmed pre-existing via `git stash` + re-run (same 3 errors present on
  the unmodified tree), unrelated to this task's files.
- `yarn lint` — no new problems from my 4 changed files (`npx eslint
  src/services/deepLinkHandler.ts src/services/__tests__/deepLinkHandler.test.ts`
  → 0 errors, 1 pre-existing warning at line 267, `no-script-url` on the
  `javascript:alert(1)` fixture, unrelated/unchanged). Full-repo `yarn lint`
  currently reports 26 problems (2 errors, 24 warnings) vs. the CLAUDE.md
  baseline of "4 errors + 3 warnings" — all in `web/mocks/handlers.ts`,
  `web/stubs/blur.tsx`, `web/stubs/toast.tsx`, and other web-preview files
  untouched by this task. That's pre-existing drift on this branch, not
  something this change introduced; flagging it here since it's out of my
  scope to fix but worth someone's attention.
- `yarn jest deepLinkHandler` — 57/57 passed (2 suites — the workspace
  variant under `.claude/worktrees/au-428-pin-refine-crash/` also ran and
  passed unchanged).
- `plutil -lint auxi/ios/auxi/auxi.entitlements` → `OK`.
- Manifest XML eyeballed for correctness — no build attempted.

## Explicitly inert until

(a) The `.well-known/apple-app-site-association` and
`.well-known/assetlinks.json` files are actually live on `macgie.com` +
`www.macgie.com` (separate, pending — tracked in
`auxi/docs/deep-linking/cloudflare-setup.md`, someone with Cloudflare access
owns this, real Android signing fingerprint still a placeholder per that
doc's Step 4).
(b) A native rebuild (new TestFlight/Play build or fresh local
`ios:clean`/pod install) ships the updated entitlements/manifest — this
session did not trigger one, per the task's file-only constraint.

Until both land, the app behaves exactly as before: `auxi://` custom-scheme
links work, `https://macgie.com/...` links do not yet open the app (same
non-functional state `https://auxi.app/...` links were already in).

## Unresolved questions

- None blocking. One observation: repo-wide `yarn lint` baseline has drifted
  (26 problems vs. documented 4+3) from unrelated `web/` files — flagging
  for whoever owns lint-baseline hygiene, not fixing here (out of scope,
  would violate "minimal targeted edits").

**Status:** DONE
**Summary:** Switched the app's universal-link/App-Link host from the dead
`auxi.app` to `macgie.com`/`www.macgie.com` across the deep-link parser,
its tests, iOS entitlements, and the Android manifest. All file-only, no
native rebuild triggered; typecheck/lint/jest clean for the touched files.
