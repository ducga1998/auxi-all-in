---
title: "AU-457 Discovery — admin-curated outfit browsing + See-on-me + save-to-wardrobe"
description: "Cross-repo: new /api/discovery/* contract, admin CRUD + SPA page, mobile Discovery feed/detail, deep link to one outfit."
status: pending
priority: P2
effort: 35h
branch: main
tags: [au-457, discovery, cross-repo, backend, mobile, admin-spa]
created: 2026-08-27
---

# AU-457 — Tính năng Discovery

Users browse admin-curated outfits; from an outfit they can (1) "See on me" (real AI try-on
against their own body photo), (2) filter by season/trend, (3) tap an item for detail, (4) save
that item to their wardrobe. A social post deep-links straight into ONE outfit.

**Out of scope (stakeholder-cut):** cross-referencing "other outfits using this item"; in-app AI
composite generation (admin uploads the cover image); marketing copy for the social post.

## Workflow deviation — READ FIRST

Root `CLAUDE.md` mandates the Figma→RN gate chain (extraction → qa-ui review-extraction → build →
qa-ui Compare → designer 6.5 → qa-mobile). **This plan deliberately skips it** — the stakeholder
(Hiệp) withheld the Figma pipeline for this round (Figma MCP not authorized in-session). Mobile UI
is built directly on design-system tokens (`auxi/src/theme/theme.ts`, `auxi/docs/design-system/*`).
Only a **functional** `qa-mobile` smoke pass gates the PR. No qa-ui Compare, no designer 6.5, no
Figma extraction artifacts. **Known follow-up (out of scope here):** a Figma-fidelity + designer
6.5 pass once Figma access is authorized — file it as a separate ticket at merge.

## Cross-repo contract gate

Per root `CLAUDE.md` "Two-Repo Contract" + `.claude/rules/orchestration-protocol.md`:
`tech-lead` MUST sign off on `/api/discovery/*` (phase 03 artifact in `API_DOCUMENTATION.md`)
**before** mobile phase 06 starts building against it. Backend phases 01–02 may run in parallel
with the sign-off; mobile is blocked on it.

## Phases

| # | Phase | Repo | Owner | Effort | Blocked by | Status |
|---|---|---|---|---|---|---|
| 01 | [Models + migration](phase-01-backend-models-migration.md) | backend | backend-dev | 2h | — | pending |
| 02 | [Admin API (repo/service/router)](phase-02-backend-admin-api.md) | backend | backend-dev | 4h | 01 | pending |
| 03 | [Public API + contract doc](phase-03-backend-public-api-contract.md) | backend | backend-dev → tech-lead | 3h | 01 | pending |
| 04 | [Admin SPA page](phase-04-admin-spa-ui.md) | admin SPA | backend-dev | 5h | 02 | pending |
| 05 | [Backend tests](phase-05-backend-tests.md) | backend | backend-dev | 3h | 02, 03 | pending |
| 06 | [Mobile service + types](phase-06-mobile-service-layer.md) | auxi | mobile-dev | 2h | 03 signed off | pending |
| 07 | [Discovery feed screen + filters + entry point](phase-07-mobile-discovery-feed.md) | auxi | mobile-dev | 5h | 06, D1 | pending |
| 08 | [Outfit detail + See-on-me + save item](phase-08-mobile-outfit-detail.md) | auxi | mobile-dev | 6h | 06, 07 | pending |
| 09 | [Deep link `discovery-outfit`](phase-09-mobile-deep-link.md) | auxi | mobile-dev | 2h | 08 | pending |
| 10 | [Analytics doc sync + verification gates](phase-10-analytics-and-verification.md) | both | mobile-dev + qa-mobile | 3h | 05, 09 | pending |

Parallelizable: 02 ∥ 03 (after 01); 04 ∥ 05 ∥ 06 (after their blockers). 07→08→09 strictly serial
(same nav/feature files).

## Open decisions (must resolve before the blocked phase starts)

- **D1 (blocks 07) — bottom tab host.** The app has **no bottom tab-bar today**: it is a native
  stack + push-drawer (`auxi/src/navigation/AppNavigator.tsx:173-380`, `SidebarMenu.tsx:113-165`;
  `auxi/docs/design-system/header-footer-rules.md:67-102` explicitly says none exists). Introducing
  a real tab host touches every screen (bottom inset/padding). Phase 07 defaults to **drawer row +
  stack route** (zero-risk, screen code identical either way) and carries the tab-host option as a
  separate, CEO-confirmable sub-step. See phase-07 §D1.
- **D2 (blocks 02) — max 4 items per outfit.** `wardrobe-backend/routers/tryon.py:168` hard-rejects
  a try-on with `len(wardrobe_item_ids)` outside 1..4. A 5-item Discovery outfit would 400 on "See
  on me". Plan enforces 1..4 at admin-create validation. Confirm with stakeholder that 4 garments
  is an acceptable curation ceiling.

## Key dependencies / reuse (verified)

- Lifecycle idiom: `models/trending_drop.py` (DropStatus, `_as_utc`, `to_admin_dict`).
- Router/repo/service templates: `routers/admin/trending_drops.py`, `routers/trending_drop.py`,
  `repositories/trending_drop_repository.py`, `services/trending_drop_admin_service.py`.
- Save-to-wardrobe needs **no backend work**: `POST /api/wardrobe/common-items/{item_id}/clone`
  (`routers/wardrobe.py:460`) → `WardrobeService.clone_common_item` (`services/wardrobe_service.py:196`).
- See-on-me needs **no backend work**: `routers/tryon.py:210` already allows `is_common_item` ids.
- Mobile try-on entry: `navigation.navigate('SeeThisOnMeConfirm', { outfit })` — worked example
  `auxi/src/screens/FavouriteScreen.tsx:260`.

## File ownership (no two parallel phases share a file)

| Phase | Owns |
|---|---|
| 01 | `models/discovery.py`, `migrations/env.py`, `migrations/versions/discovery1a2b_*.py` |
| 02 | `repositories/discovery_repository.py`, `services/discovery_admin_service.py`, `routers/admin/discovery_outfits.py`, `routers/admin/__init__.py` |
| 03 | `services/discovery_service.py`, `routers/discovery.py`, `routers/__init__.py`, `app.py`, `API_DOCUMENTATION.md` |
| 04 | `wardrobe-admin/src/{services/discoveryOutfitsService.ts, pages/DiscoveryOutfits.tsx, components/discovery-outfits/*, App.tsx, components/layout/Layout.tsx}` |
| 05 | `tests/test_discovery_*.py` |
| 06 | `auxi/src/services/discoveryService.ts`, `auxi/src/hooks/useDiscovery.ts` |
| 07 | `auxi/src/screens/discovery/{DiscoveryScreen,DiscoveryOutfitCard,DiscoveryFilterRow}.tsx`, `SidebarMenu.tsx` + **shared:** `types/navigation.ts`, `AppNavigator.tsx`, `analytics.ts`, `translations/*` |
| 08 | `auxi/src/screens/discovery/{DiscoveryOutfitDetailScreen,DiscoveryItemStrip}.tsx`, `hooks/useSaveCommonItemToWardrobe.ts`, `screens/ItemDetailScreen.tsx`, `screens/item-detail/ItemDetailReadPanel.tsx`, `services/wardrobeService.ts` + **same shared 4** |
| 09 | `auxi/src/services/deepLinkHandler.ts` + tests + **same shared 2** (`analytics.ts`, `translations/*`) |
| 10 | `auxi/docs/analytics/mixpanel-tracking-plan.md`, reports |

**Serialization rule:** 07 → 08 → 09 never run in parallel (they share `types/navigation.ts`,
`AppNavigator.tsx`, `analytics.ts`, `translations/*`). Backend phases 02/03 do not share files and
may run in parallel; 04/05/06 likewise.

## Release order

backend 01→03 deploy → admin 04 deploy → mobile 06→09 ship → admin publishes content last.
Runtime kill-switch: unpublish all outfits in the admin SPA (no deploy needed).

## Decisions (resolved 2026-08-27, stakeholder Hiệp)

1. **Cover image storage → public-read, not presigned.** Discovery gets its own upload path
   (`POST /api/admin/discovery-outfits/upload-cover`, phase 02) that calls
   `S3Manager.get_public_url(object_name)` (`utils/s3_utils.py:189`) instead of
   `generate_presigned_url(..., expiration=604800)` — never expires. Note:
   `generate_presigned_url` itself already delegates to `get_public_url` when `self.endpoint_url` is
   set (`s3_utils.py:175-176`), which is likely true in prod (R2 — see the checksum-workaround
   comment at the top of `s3_utils.py`), so the "7-day expiry" on the shared common-items path may
   already be stale in practice. Either way, Discovery's own path makes it explicit and
   environment-independent. **Devops pre-launch checklist item (not code):** confirm the bucket/
   `discovery_outfits/` prefix actually grants public read — `get_public_url` only *builds* the URL,
   it doesn't set the ACL/bucket policy.
2. **D1 resolved** — ship as a drawer row + stack route now (per plan default); a real bottom
   tab-bar is separate future work, not this ticket.
3. **D2 resolved** — 4 garments/outfit max is accepted (forced by `routers/tryon.py:168`).
4. **Logged-out access resolved** — accept the existing behavior: all routes require auth, so a
   social post opened by a logged-out user lands on login first, then continues to the outfit after
   auth (standard deep-link-after-login pattern, no new public surface).
5. **Trend tags resolved** — stay free-form for v1; already normalized (lowercase, trimmed, deduped)
   server-side in `DiscoveryAdminService.normalize_tags` (phase 02). Curated vocabulary is a later
   follow-up if drift becomes a problem.
