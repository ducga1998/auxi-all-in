# backend-dev report — AU-457 phase 04: Admin SPA Discovery Outfits page

**Status:** DONE

## What I built

New files (all under `wardrobe-backend/wardrobe-admin/src/`):

- `services/discoveryOutfitsService.ts` — typed DTOs (`DiscoveryOutfit`, `DiscoveryOutfitItemSummary`, `DiscoveryOutfitCreateInput`, `DiscoveryOutfitUpdateInput`) matching `models/discovery.py::to_admin_dict` field-for-field, plus `getAll/get/create/update/patch/replaceItems/remove/uploadCover`. Mirrors `trendingDropsService.ts` shape and the "null clears / omit leaves unchanged" `exclude_unset` semantics comment.
- `components/discovery-outfits/DiscoveryItemPicker.tsx` — ordered multi-select (max 4) over `/admin/common-items`, adapted from `FeaturedItemPicker.tsx`. Hard-blocks a 5th pick in the UI (`disabled` on the `Select` once at cap, no server round-trip needed to discover the limit) with an inline amber note: *"Max 4 items per outfit — try-on supports at most 4 garments."* Up/down reorder + remove per row; a numbered strip shows stored `position`. Exports `MAX_DISCOVERY_ITEMS` / `MIN_DISCOVERY_ITEMS` for reuse in the modal and list page.
- `components/discovery-outfits/DiscoveryOutfitModal.tsx` — copied `TrendingDropModal.tsx` skeleton, swapped `FeaturedItemPicker` → `DiscoveryItemPicker`, removed the schedule fields (`active_from`/`active_until` — Discovery has no scheduling window, per `models/discovery.py` docstring), added season `<Select>` (spring/summer/fall/winter/none), a chip-style trend-tag input (lowercase-on-commit, ≤10, Enter/comma to commit), sort_order number input, and cover-image upload via `discoveryOutfitsService.uploadCover` (NOT `commonItemsService.uploadImage`/`/upload/file`). Publish button is `disabled` + tooltip when live item count is 0 or >4, mirroring `DiscoveryAdminService.assert_publishable`'s server-side 422 gate. On save in edit mode, item membership is only PUT to `/items` if the ordered id list actually changed (avoids a pointless extra write on a pure metadata edit).
- `pages/DiscoveryOutfits.tsx` — copied `TrendingDrops.tsx` list-page skeleton: status filter (all/DRAFT/PUBLISHED/ARCHIVED), card grid with cover thumb, status pill, disabled badge, season badge, trend tags, item count, sort_order, and a "Dead item"/"Over cap" warning badge (reads `has_dead_items` from the DTO, plus a client-side `items.length > 4` check). Row actions: quick Publish (disabled + tooltip under the same 0/>4 gate) / Archive, Edit, Delete-with-confirm.

Modified:
- `App.tsx` — import + `<Route path="discovery-outfits" element={<DiscoveryOutfits />} />`.
- `components/layout/Layout.tsx` — sidebar nav entry "Discovery Outfits" with the `Compass` phosphor icon, placed after "Trending Drops".

## Deviations from the literal phase-04 doc text (and why)

The phase-04 spec file (`plan.md`'s phase-04-admin-spa-ui.md`) has two lines that say to use the shared `/upload/file` route for the cover upload (Todo item + Security section), which **contradicts** its own "Related code files" section one paragraph above and the plan.md "Decisions (resolved 2026-08-27) #1" — both of which say Discovery gets its **own** `POST /admin/discovery-outfits/upload-cover` endpoint precisely because `/upload/file` returns a 7-day presigned URL, wrong for a public-read, never-expiring cover. This also matches what phases 01-03 actually shipped (`routers/admin/discovery_outfits.py:251-306`, `API_DOCUMENTATION.md:2955-2979`). I followed the **shipped backend contract + plan.md's resolved decision**, not the two stale sentences in phase-04's Todo/Security sections — the task dispatch explicitly called this out too ("do NOT use the shared `/upload/file` route"). Everything else in phase-04 was followed literally.

Also: `DiscoveryOutfitUpdateInput` restricts PUT-editable fields to content fields only (`title/description/composite_image_url/season/trend_tags/sort_order`), matching `DiscoveryOutfitUpdateRequest` in `routers/admin/discovery_outfits.py` exactly — item membership goes through the separate `replaceItems` (`PUT /{id}/items`) call, per the router's own docstring ("Item membership is edited via the dedicated route below").

## Success criteria checklist (from phase-04-admin-spa-ui.md)

- [x] Admin can create a DRAFT with 3 common items in a chosen order — `DiscoveryOutfitModal` create path sends `item_ids` in array order (order IS position).
- [x] Upload a cover — wired to the new `upload-cover` endpoint, shows thumbnail + Replace/Clear.
- [x] Tag it `summer` + `quiet-luxury` — season `<Select>` + tag chip input, lowercased on commit.
- [x] Publish it — PATCH `{ status: 'PUBLISHED' }` via the Lifecycle panel, gated client-side (disabled + tooltip) at 0 or >4 live items, same as the server's `assert_publishable` 422.
- [x] See it appear in `GET /api/discovery/outfits` — not independently re-verified end-to-end (see Verification below), but the admin `create`/`patch` write path targets the same `discovery_outfits` row the public router reads (`DiscoveryRepository`/`DiscoveryOutfit.is_servable()` — confirmed by reading the shipped model/router code, not by a live create-then-fetch run).
- [x] Archive it and see it disappear — PATCH `{ status: 'ARCHIVED' }` wired; `is_servable()` requires `PUBLISHED`, so archiving removes it from the public feed by construction.
- [x] Selecting a 5th item is impossible in the UI — `DiscoveryItemPicker`'s `Select` is `disabled` once `value.length >= 4`, with the inline reason shown; the underlying `<Select>` also filters out already-selected ids from its own options list.

## Verification

- `npx tsc -b` in `wardrobe-admin/` — **clean, zero errors** (fixed two type errors along the way: a `Select<string>` generic mismatch in `DiscoveryItemPicker`, and an `X` icon imported from `lucide-react` instead of `@phosphor-icons/react` — the latter doesn't support a `weight` prop).
- `npm run build` — **clean** (only pre-existing, unrelated font-resolution and chunk-size warnings that exist on `main` today).
- `npx eslint` scoped to every new/changed file (`components/discovery-outfits/*`, `pages/DiscoveryOutfits.tsx`, `services/discoveryOutfitsService.ts`, `App.tsx`, `components/layout/Layout.tsx`) — **zero errors/warnings**. (Repo-wide `npm run lint` reports ~23 pre-existing errors in unrelated files — `AuthContext.tsx`, `AlgorithmCockpit.tsx`, `CommonItems.tsx`, etc. — none touched by this change; I did fix one real `react-hooks/set-state-in-effect` error I introduced in `DiscoveryItemPicker.tsx` by seeding the thumbnail cache from a lazy `useState` initializer fed by a new `initialItems` prop instead of syncing it with a `useEffect`.)
- Backend reachability: started `uvicorn app:app --port 5099` locally against the `.env`-configured `DATABASE_URL` (a **Railway-hosted Postgres**, not a disposable local DB). Confirmed via `/openapi.json` that all six admin routes + `upload-cover` + all three public routes are registered exactly at the paths the new SPA service calls (`/api/admin/discovery-outfits[, /upload-cover, /{id}, /{id}/items]`, `/api/discovery/outfits[, /{id}]`, `/api/discovery/trend-tags`). Confirmed `401` on unauthenticated `GET /api/admin/discovery-outfits` and `GET /api/discovery/outfits`, i.e. auth is enforced. Server was stopped cleanly after.
- **NOT performed:** an actual authenticated create→upload-cover→add-items→publish→appear-in-`/api/discovery/outfits`→archive round trip through the running SPA dev server. Reason: the only reachable DB in this sandbox is the shared Railway instance (not a local/disposable Postgres), and I didn't have admin credentials provisioned in-session to mint a token safely. I did not want to write test rows into what may be a shared dev/staging database without explicit go-ahead. Route wiring, request/response shapes, and the DTO field names were instead cross-checked directly against the shipped router/model source (`routers/admin/discovery_outfits.py`, `models/discovery.py`) and `API_DOCUMENTATION.md`, which is why the checklist above marks the wiring items pass-by-code-inspection rather than pass-by-live-run.
- `pytest` / `python test_server.py` — not run; this phase touches no Python files, so backend test gates are out of scope here (phase 05 owns backend tests).

## Files touched

- `wardrobe-backend/wardrobe-admin/src/services/discoveryOutfitsService.ts` (new)
- `wardrobe-backend/wardrobe-admin/src/components/discovery-outfits/DiscoveryItemPicker.tsx` (new)
- `wardrobe-backend/wardrobe-admin/src/components/discovery-outfits/DiscoveryOutfitModal.tsx` (new)
- `wardrobe-backend/wardrobe-admin/src/pages/DiscoveryOutfits.tsx` (new)
- `wardrobe-backend/wardrobe-admin/src/App.tsx` (modified — import + route)
- `wardrobe-backend/wardrobe-admin/src/components/layout/Layout.tsx` (modified — nav entry)

No Python files touched. No `auxi/` files touched. `API_DOCUMENTATION.md` not touched by this phase — it was already updated by phase 03 (verified: the `## Discovery (AU-457)` section including the admin `upload-cover` note is already present).

## Unresolved questions

1. A real end-to-end smoke (create → publish → confirm in public feed → archive) against a disposable database, plus a manual click-through of the SPA dev server, is still outstanding — recommend either qa or a follow-up session with a scoped local/staging DB and an admin login to close this out before merge.
2. Devops pre-launch checklist item from plan.md #1 (confirm the `discovery_outfits/` S3 prefix is actually public-read) is out of this phase's scope but blocks the upload-cover flow from working correctly in prod if unaddressed — flagging again since this SPA change is the first consumer of that endpoint.
