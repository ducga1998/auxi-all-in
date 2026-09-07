# Phase 10 — Analytics doc sync, verification gates, rollback

**Repos:** both · **Owner:** mobile-dev (doc) + qa-mobile (smoke) · **Effort:** 3h
**Status:** pending · **Blocked by:** phases 05, 09

## 1. Tracking-plan doc update (MANDATORY)

`.claude/rules/analytics-tracking-required.md`: a feature is not done until
`auxi/docs/analytics/mixpanel-tracking-plan.md` is updated. Add a new `### 5.27 Discovery (AU-457)`
subsection (the file's §5 runs to 5.26 — verify the next free number at write time; §5.26 is at
`mixpanel-tracking-plan.md:592`).

The 5 shipped events, each with `file:line` + props:

| Event | Phase | Props |
|---|---|---|
| `discovery_feed_viewed` | 07 | `filter_season?`, `filter_trend_tag?` |
| `discovery_filter_applied` | 07 | `filter_type`, `filter_value` |
| `discovery_outfit_opened` | 07 | `outfit_id`, `position`, `source: 'feed'` |
| `discovery_see_on_me_tapped` | 08 | `outfit_id`, `item_count` |
| `discovery_item_saved` | 08 | `outfit_id`, `item_id` |
| `discovery_deep_link_opened` | 09 | `outfit_id`, `resolved` |

Also add to §10 a suggested funnel:
`discovery_feed_viewed → discovery_outfit_opened → discovery_see_on_me_tapped → try-on completion`
and a second: `discovery_deep_link_opened → discovery_outfit_opened → discovery_item_saved`
(social-post attribution).

No PII: only ids and closed-vocabulary values; no titles, no free text, no URLs.

## 2. Verification gates (umbrella `CLAUDE.md` §Verification gates)

| Gate | Command / actor | Must be |
|---|---|---|
| Backend e2e | `cd wardrobe-backend && python test_server.py` | green |
| Backend suite | `pytest -m "not slow"` | green |
| Admin SPA | `cd wardrobe-backend/wardrobe-admin && yarn build` | green |
| Mobile types | `cd auxi && npx tsc --noEmit` | no NEW errors (legacy `_HomeScreen.tsx` baseline allowed) |
| Mobile lint | `cd auxi && yarn lint` | no NEW errors beyond the 4-error/3-warning baseline |
| Token lint | `./scripts/auxi-lint-tokens.sh` | clean |
| Smoke | `qa-mobile` on iOS sim against backend `:5001` | pass |

**Explicitly NOT gates this round** (stakeholder deviation): qa-ui Compare mode, designer 6.5
design-review, Figma extraction artifact. The PR template lines for those get "N/A — Figma pipeline
deferred, see plan.md §Workflow deviation" plus a link to the follow-up ticket.

### qa-mobile functional smoke script (no visual judgment)
1. Cold start, log in, open Discovery from the drawer — feed renders, no crash.
2. Apply a season chip, then a trend chip — list changes, no crash.
3. Open an outfit — detail renders with items.
4. Tap "See on me" — the capture/confirm flow starts (do not run a full paid render unless asked).
5. Tap an item → detail → Save to wardrobe → item appears in Wardrobe.
6. `xcrun simctl openurl booted "auxi://discovery-outfit?id=<published-id>"` (warm + cold) → lands
   on that outfit.
7. Same with an ARCHIVED id → Discovery feed + toast, no crash.
8. `mobile_get_crash` / `list_crashes` clean at the end.

## 3. Rollback plan (per phase)

| Phase | Rollback |
|---|---|
| 01 | `alembic downgrade -1` (drops both tables; nothing else references them) |
| 02–03 | Remove the `include_router` lines — routes vanish, tables stay harmless |
| 04 | Remove the SPA route + nav entry; page is dead code |
| 06–09 | Remove the drawer row / tab entry → feature unreachable while code stays; full revert = drop the discovery screens + revert the 4 shared files (`types/navigation.ts`, `AppNavigator.tsx`, `ItemDetailScreen.tsx`, `ItemDetailReadPanel.tsx`, `deepLinkHandler.ts`) |
| Runtime kill-switch | Set every outfit to DRAFT/disabled via the admin SPA → the feed is empty and every deep link 404s to the fallback. **This is the fastest production kill-switch and needs no deploy.** |

An Unleash flag (`src/services/featureFlags.tsx:42-44` pattern, e.g. `discovery_feed`) is optional;
the admin unpublish path above already gives content-level control. Add the flag only if the
stakeholder wants to hide the entry point itself without a release.

## 4. Backwards compatibility

- **Additive only.** New tables, new routes, new mobile screens. No existing endpoint, column, or
  response shape changes.
- Older mobile builds never call `/api/discovery/*` — no forced upgrade.
- `ItemDetail`'s new `origin` param is optional; every existing caller (grep: `ItemDetailScreen.tsx:771`
  is the single `ItemDetailReadPanel` consumer) keeps today's behavior byte-for-byte.
- `deepLinkHandler` change is additive; auth kinds regression-tested (phase 09).
- Deep links from social posts targeting a build without the feature → unknown slug → `null` → the
  link is silently ignored (existing behavior, `deepLinkHandler.ts:131`), no crash.

## Todo

- [ ] tracking-plan §5.27 + §10 funnels written with real `file:line`
- [ ] all 7 gates green
- [ ] qa-mobile smoke report filed in `plans/reports/`
- [ ] PR template filled, Figma/designer lines marked N/A + follow-up ticket linked
- [ ] follow-up ticket filed: "AU-457 follow-up — Figma fidelity + designer 6.5 pass on Discovery"

## Success criteria

Both repos green, smoke pass recorded, tracking doc merged in the same PR as the code.

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Tracking doc skipped → decision-debt | M×M | Checklist item in the PR template; rule is explicit |
| Mobile merged before backend deploys → empty feed in prod | M×M | Release order: backend (01–03) → admin (04) → mobile (06–09); content published last |
| Skipped design gates ship visual drift | H×M | Accepted, stakeholder-authorized; follow-up ticket is mandatory, not optional |
