# Mobile fix — AU-457 code-review finding High #2 (deep-link lost during onboarding)

Session was cut off by a usage-limit reset before writing this report; verified and completed by the orchestrator directly against the working tree left behind. The fix was already correctly implemented — this documents what was found.

## Root cause (confirmed by re-trace)

`deepLinkHandler.ts`'s `isAuthedTreeMounted` originally only checked `rootRoute.name !== 'Auth'`. The app has a third root state the original check didn't account for: the first-login onboarding branch (`Welcome` → `OnboardingWardrobe` → `OnboardingFit` → `OnboardingStyles` → `OnboardingLoading` → `OnboardingCompleted` → `OnboardingOutro`, mounted whenever `user.is_first_login` is true) — neither `'Auth'` nor the post-onboarding authed tree where `DiscoveryOutfitDetail` is registered. A Discovery link opened mid-onboarding would previously be treated as "authed tree mounted" and attempt `navigate()` into a route that doesn't exist yet in that branch, silently failing and losing the link permanently (the replay effect only re-fires on `[user]` changing, which already happened at login — onboarding completing afterward doesn't retrigger it).

## Fix

`deepLinkHandler.ts`: `isAuthedTreeMounted` now also returns `false` while the root route is in the onboarding branch (checked via an explicit onboarding-route-name set), so the link stays stashed rather than being dropped. The stash-replay mechanism (already built for the logged-out case) now also fires once the onboarding branch exits and the authed tree — with `DiscoveryOutfitDetail` registered — actually mounts, however long onboarding takes. `AppNavigator.tsx` gained the plumbing needed to signal onboarding-complete to the handler (28 lines).

Net diff: `deepLinkHandler.ts` +110/-17 (mostly the broadened root-state check + comments explaining the three-state model: Auth / onboarding / authed), `AppNavigator.tsx` +28.

## Test results

`auxi/src/services/__tests__/deepLinkHandler.test.ts` — 2 new tests added:
- `stashes instead of navigating when the root stack is the first-login onboarding branch`
- `replays the stashed link once onboarding completes and the authed tree mounts (not lost)`

Full run (re-run by orchestrator post-fix): `yarn jest deepLinkHandler` → **57 passed** (55 original incl. verify-email/reset-password/logged-out-discovery-link kinds + 2 new onboarding tests), zero failures.

`npx tsc --noEmit` and `yarn lint` (re-run by orchestrator, full project): pre-existing errors found in `src/screens/see-this-on-me/components.tsx`, `StomStepLayout.tsx` (tsc) and `web/mocks/handlers.ts`, `web/stubs/toast.tsx` (lint) — confirmed unrelated to this change (none of these files were touched by any AU-457 work, none appear in `git status` as modified by this feature). No errors or warnings in any Discovery or deep-link file.

## Not fixed / deferred

Nothing deferred — the assigned finding was resolved.

**Status:** DONE
