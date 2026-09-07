# Phase 08 — Outfit detail: See-on-me + item detail + save to wardrobe

**Repo:** `auxi/` · **Owner:** mobile-dev · **Effort:** 6h
**Status:** pending · **Blocked by:** phases 06, 07

> No Figma this round (see phase 07 header). Tokens only.

## Context (traced, not assumed)

### "See on me" — the REAL generation flow
- `TryOnResultScreen.tsx` is a **read-only viewer** for an already-rendered URL from a push
  (`types/navigation.ts:274-279`). **Not** the generator. Do not route there.
- The real entry point is `navigation.navigate('SeeThisOnMeConfirm', { outfit })` — worked example
  `src/screens/FavouriteScreen.tsx:251-270` (also `MyCreationsScreen.tsx:149`, `ScheduleScreen.tsx:213`).
  The confirm gate owns the "reuse your saved body?" sheet and hands off to `SeeThisOnMe`, which
  owns AI-consent (`useAiConsentGate`), AI-limit (`useAiLimitGate`) and usage-limit
  (`useUsageLimitGate`) gating — see `SeeThisOnMeScreen.tsx:34-41`. Routing through
  `SeeThisOnMeConfirm` gets all of that for free; a direct `SeeThisOnMe` push does not.
- Payload shape `TryOnOutfitContext` (`types/navigation.ts:88-93`):
  `{ outfitHash, itemIds, itemImageUrls, stylingNote }`.
  For Discovery: `outfitHash = \`discovery_${outfit.id}\`` (client-side key only — it keys
  `tryOnGenerationStore`/`recordTryOnResult`, `try-on-generation-store.ts:317`; the backend
  `POST /api/tryon/highres` payload is `{ body_id, wardrobe_item_ids, gemini_opt_in }`
  (`services/tryOnService.ts:9-14`) and never receives the hash).
  `itemIds` = the outfit's item ids; `itemImageUrls` = `image_png ?? image_url` filtered non-null;
  `stylingNote` = the outfit description.
- **Backend already permits it — no backend work:** `routers/tryon.py:210` allows an id the user
  doesn't own when `item.is_common_item` is true. **But** `routers/tryon.py:168` enforces
  `1 <= len(wardrobe_item_ids) <= 4` → matches the phase-02 admin cap (D2). Client-side belt: if an
  outfit somehow arrives with >4 live items, disable the CTA with an explanatory toast instead of
  firing a doomed request.
- The admin's `composite_image_url` is **cover art only** — never feed it to try-on.

### Item detail + save to wardrobe
- `ItemDetailScreen` already handles "an item that is NOT in the user's wardrobe": pass
  `fallbackItem` and the screen renders from it when the wardrobe lookup misses
  (`ItemDetailScreen.tsx:121-140`, comment `:124-130` documents exactly this V05 common-item case).
  `isCatalogItem` (`:297-304`) then hides edit/delete and shows the common badge (`:703-709`).
- `ItemDetailReadPanel` currently exposes 5 wardrobe-ownership callbacks
  (`ItemDetailReadPanel.tsx:21-25`: onSwap/onBuildAround/onDelete/onToggleUsage/onEdit) and **no
  save affordance** — so "save to wardrobe" is a genuinely new, additive control.
- Save endpoint (no backend work): `POST /api/wardrobe/common-items/{item_id}/clone`
  (`wardrobe-backend/routers/wardrobe.py:460-497`) → `{ message, item }`; 404 when the id isn't a
  common item, 403 on permission, 201 on success. Batch variant exists at `routers/wardrobe.py:409`
  (`POST /common-items/clone` with `{item_ids}`) — use the single-item route.
  **Do NOT use `useAddWardrobeItem`** (that's the photo-upload creation flow).

## Requirements

1. **`DiscoveryOutfitDetailScreen`** — cover image, title, description, season/tag pills, ordered
   item strip/list, primary "See on me" CTA (sticky bottom, blur/tint treatment per
   `header-footer-rules.md:80-88`, clears bottom safe-area), loading/error/404 states.
   A 404 (service returns `null`) renders a "this outfit is no longer available" state with a
   "Browse Discovery" action — never a crash, never an infinite spinner.
2. **Item tap** → `navigation.navigate('ItemDetail', { itemId, fallbackItem, origin: 'discovery' })`.
   `fallbackItem` is built from the discovery item payload (id, name, image_url, image_png,
   category, category_code, layer_code, `is_common_item: true`).
3. **Save to wardrobe** — an additive `onSaveToWardrobe` control rendered by `ItemDetailReadPanel`
   only when `origin === 'discovery'`. Optimistic-free: disabled+spinner while in flight, success
   toast, `queryClient.invalidateQueries({ queryKey: wardrobeKeys.all })`, and the button flips to a
   disabled "Saved" state. Errors → friendly toast (reuse `getFriendlyError`).
   Duplicate taps: guard with the existing `saving` flag; a double-clone would create two items
   (the backend endpoint is NOT idempotent — verified: `clone_common_item` always inserts,
   `services/wardrobe_service.py:236-256`).

## Analytics (ships in THIS phase)

| Event | Fires | Props |
|---|---|---|
| `discovery_see_on_me_tapped` | See-on-me CTA | `outfit_id`, `item_count` |
| `discovery_item_saved` | clone success | `outfit_id`, `item_id` |

(`discovery_outfit_opened` already fires from phase 07; the deep-link variant is phase 09.)

## Related code files

- CREATE `auxi/src/screens/discovery/DiscoveryOutfitDetailScreen.tsx` (<200 LOC)
- CREATE `auxi/src/screens/discovery/DiscoveryItemStrip.tsx`
- CREATE `auxi/src/hooks/useSaveCommonItemToWardrobe.ts` (mutation wrapper over the clone endpoint)
- MODIFY `auxi/src/types/navigation.ts` — `DiscoveryOutfitDetail: { outfitId: string }`; extend
  `ItemDetail` params with `origin?: 'discovery'`
- MODIFY `auxi/src/navigation/AppNavigator.tsx` — register the detail screen
- MODIFY `auxi/src/screens/ItemDetailScreen.tsx` — pass `origin` + save handler down
- MODIFY `auxi/src/screens/item-detail/ItemDetailReadPanel.tsx` — optional save control
- MODIFY `auxi/src/services/wardrobeService.ts` — `cloneCommonItem(itemId)` if absent (grep first)
- MODIFY `auxi/src/translations/*`

**File ownership:** phases 07 and 08 both touch `types/navigation.ts` + `AppNavigator.tsx` → they
MUST run serially, never in parallel. No other phase touches `ItemDetailScreen.tsx` /
`ItemDetailReadPanel.tsx`.

## Implementation steps

1. Detail screen + item strip against tokens; sticky CTA with safe-area + blur treatment.
2. `useSaveCommonItemToWardrobe` mutation (`apiClient` via `wardrobeService`).
3. Additive optional props on `ItemDetailReadPanel` (default undefined → today's render is
   byte-identical for every existing caller — verify by grepping callers: only
   `ItemDetailScreen.tsx:771`).
4. Wire See-on-me through `SeeThisOnMeConfirm` with the constructed `TryOnOutfitContext`.
5. Analytics + i18n.
6. `npx tsc --noEmit && yarn lint && ./scripts/auxi-lint-tokens.sh`.

## Todo

- [ ] detail screen with 404/error/loading states
- [ ] item strip → ItemDetail with `fallbackItem` + `origin`
- [ ] save-to-wardrobe control + mutation + invalidation + saved state
- [ ] See-on-me → `SeeThisOnMeConfirm` with correct payload, CTA disabled when live items >4 or 0
- [ ] 2 analytics events
- [ ] tsc / lint / token-lint clean

## Success criteria

Against a live backend: open a published outfit → tap See-on-me → the real capture/render flow
starts (consent + limit gates behave as they do from Favourite) → returns a rendered image.
Tap an item → detail shows the common badge, no edit/delete → Save → item appears in Wardrobe.
Re-open the item → button shows "Saved".

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Wiring See-on-me to `TryOnResultScreen` (viewer) instead of the generator | **M×H** | Explicit: route to `SeeThisOnMeConfirm`; copy `FavouriteScreen.tsx:260` |
| Bypassing `SeeThisOnMeConfirm` → consent/AI-limit/usage gates skipped (compliance + cost) | M×H | Never `navigate('SeeThisOnMe')` directly from Discovery |
| >4-item outfit → 400 from try-on | M×H | Server cap (D2) + client CTA guard |
| Double-tap Save → duplicate wardrobe items (endpoint not idempotent) | **M×M** | In-flight guard + disabled button + "Saved" terminal state |
| Adding props to the shared `ItemDetailReadPanel` changes the wardrobe path | M×H | All new props optional with undefined defaults; single existing caller verified |
| `fallbackItem` missing fields → detail renders half-empty | M×L | Documented degradation (`ItemDetailScreen.tsx:128-130`): no created_at → date row hidden |

## Security

No new endpoints. Clone route is user-scoped server-side. No PII in analytics props (ids only).

## Next steps

Phase 09 (deep link) — depends on this screen existing.
