# Phase 07 — Discovery feed screen, filters, entry point

**Repo:** `auxi/` · **Owner:** mobile-dev · **Effort:** 5h
**Status:** pending · **Blocked by:** phase 06 + **decision D1**

> **No Figma this round.** Build against `auxi/src/theme/theme.ts` tokens and the rules in
> `auxi/docs/design-system/{design-system,color-rules,motion-rules,header-footer-rules}.md`.
> No qa-ui Compare, no designer 6.5 gate. `./scripts/auxi-lint-tokens.sh` must still be clean
> (it is mechanical, not a Figma gate).

## D1 — the "bottom tab" decision (resolve before coding)

**Verified fact:** there is no bottom tab-bar in this app. `AppNavigator.tsx` is a native stack
(`Stack.Screen` entries at `:173-380`, no `createBottomTabNavigator` anywhere in `src/navigation/`),
and navigation happens through the push-drawer (`src/components/layout/SidebarMenu.tsx:113-165`).
`auxi/docs/design-system/header-footer-rules.md:67-70` states this explicitly and `:101-102` calls a
new tab-bar "out-of-pattern → escalate to CEO". The stakeholder authorized a Discovery **bottom
tab** — but that authorizes the *feature entry point*, and building a real tab host is a chrome-wide
change (every screen's bottom inset, a root host, active-pill treatment, motion tokens).

**Plan (decoupled, so the decision can't block the feature):**

- **07a (default, do now):** Discovery is a normal stack route + a `SidebarMenu` row. Screen code is
  identical under either option — the host doesn't change the screen.
- **07b (only on CEO/stakeholder confirmation, separately scheduled):** introduce the root tab host.
  Requirements are already written down in `header-footer-rules.md:90-99` — z-index tier `sticky`
  (100), clears `useSafeAreaInsets().bottom`, active = filled pill (`figmaFooterActivePill`),
  selection animated with `motion` tokens (`scale.select` 1.03 / `fast` 120), defined ONCE at a root
  host like `RootDrawer` (never per screen). A ready-made DS component exists:
  `src/components/design-system/lib/MTabBar.tsx` (dark bar, spring-on-active, honors reduce-motion).
  Scope it as its own ticket: it touches every screen's bottom padding.

## Requirements (screen)

- Header: canonical `<Header>` (`src/components/layout/Header.tsx`) with the hamburger — do NOT
  hand-roll (MAJOR finding per `header-footer-rules.md:110`).
- Filter row: season chips (4 fixed + "All") + trend-tag chips from `useDiscoveryTrendTags()`.
  Single-select per axis, combinable across axes. Horizontal scroll, chips are token-styled pills.
- Grid/list of outfit cards: cover image (`composite_image_url`), title, season/tag pills,
  `item_count`. 2-column grid. `FlatList` with pagination via the phase-06 hook.
- States: loading skeleton (`Shimmer`, `src/components/features/Shimmer.tsx`), empty ("no outfits
  yet" / "no outfits match these filters" — distinct copy), error + retry, image-load failure →
  token placeholder.
- Bottom inset respected: `paddingBottom: insets.bottom + 24` (worked example
  `SidebarMenu.tsx:85`).
- `testID` on every interactive element (`auxi/CLAUDE.md` rule): `discovery-chip-season-<value>`,
  `discovery-chip-tag-<tag>`, `discovery-card-<index>`, `discovery-retry`.
- i18n: all copy through `i18next` (`src/translations/`), no inline strings.

## Analytics (required, ships in THIS phase)

Per `.claude/rules/analytics-tracking-required.md` — literal event names, snake_case, past tense,
routed through `src/services/analytics.ts`:

| Event | Fires | Props |
|---|---|---|
| `discovery_feed_viewed` | screen focus, once per focus | `filter_season?`, `filter_trend_tag?` |
| `discovery_filter_applied` | chip tap | `filter_type: 'season'\|'trend'`, `filter_value` |
| `discovery_outfit_opened` | card tap | `outfit_id`, `position`, `source: 'feed'` |

No PII. `filter_value` is a curated tag/enum, not free text.

## Related code files

- CREATE `auxi/src/screens/discovery/DiscoveryScreen.tsx` (<200 LOC — extract subcomponents)
- CREATE `auxi/src/screens/discovery/DiscoveryOutfitCard.tsx`
- CREATE `auxi/src/screens/discovery/DiscoveryFilterRow.tsx`
- MODIFY `auxi/src/types/navigation.ts` — add `Discovery: { season?: DiscoverySeason } | undefined`
- MODIFY `auxi/src/navigation/AppNavigator.tsx` — register `<Stack.Screen name="Discovery" …>`
  (both files are mandatory per `auxi/CLAUDE.md`; skipping either = silent cold-start breakage)
- MODIFY `auxi/src/components/layout/SidebarMenu.tsx` — add the Discovery row (07a)
- MODIFY `auxi/src/services/analytics.ts` — add the three tracked events if helpers are warranted
- MODIFY `auxi/src/translations/*` — new copy keys

## Implementation steps

1. Resolve D1; if 07b is chosen, file it as a separate ticket and still ship 07a first.
2. Screen + card + filter row against tokens.
3. Register route in `types/navigation.ts` AND `AppNavigator.tsx`; add the drawer row.
4. Wire the 3 analytics events.
5. `npx tsc --noEmit && yarn lint && ./scripts/auxi-lint-tokens.sh`.

## Todo

- [ ] D1 resolved and recorded
- [ ] screen + card + filter row, all 4 states covered
- [ ] route registered in BOTH files + drawer row
- [ ] testIDs + i18n keys
- [ ] 3 analytics events wired
- [ ] tsc / lint / token-lint clean

## Success criteria

Cold-start → drawer → Discovery renders published outfits from a live local backend (`:5001`, real
HTTP — not mocks). Season + tag chips filter the list. Empty/error states reachable and correct.
No hex literals (token-lint clean).

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Tab-host refactor creeps into this phase and destabilizes every screen | **M×H** | D1 split: 07a ships the feature, 07b is a separate ticket |
| Route added to `AppNavigator` but not `types/navigation.ts` (or vice versa) | M×H | Explicit checklist item; `auxi/CLAUDE.md` calls this out as silent runtime breakage |
| Filter chips derived from page 1 only → hidden tags | M×M | Dedicated `GET /api/discovery/trend-tags` (phase 03) |
| Skipping the Figma gate ships off-system visuals | H×M | Token-only build + token-lint; explicit follow-up Figma-fidelity ticket at merge |
| Cover image 403 (presigned expiry) → broken grid | M×M | `onError` → token placeholder, never a broken frame |

## Security

Authenticated routes only — the screen is inside the authed stack; no token handling here.

## Next steps

Phase 08 (outfit detail) — strictly serial: it shares `types/navigation.ts` and `AppNavigator.tsx`.
