# Phase 04 — Admin SPA: Discovery Outfits page

**Repo:** `wardrobe-backend/wardrobe-admin/` · **Owner:** backend-dev · **Effort:** 5h
**Status:** pending · **Blocked by:** phase 02

## Context (verified templates)

- Service: `wardrobe-admin/src/services/trendingDropsService.ts` (89 LOC — typed DTOs, `api.get/post/put/patch/delete`, the `exclude_unset` null-vs-omit note `:67-70`).
- Page: `wardrobe-admin/src/pages/TrendingDrops.tsx` (275 LOC — list + status filter + row actions).
- Modal: `wardrobe-admin/src/components/trending-drops/TrendingDropModal.tsx` (386 LOC).
- Item picker: `wardrobe-admin/src/components/trending-drops/FeaturedItemPicker.tsx` (214 LOC) —
  single-select over common items; Discovery needs an **ordered multi-select (≤4)**.
- Image upload: mirror the CLIENT-side shape of `TrendingDropModal.tsx:96-108`
  (`commonItemsService.uploadImage(file)` → `{ url }` → stored on the DTO field), but point it at
  the NEW `discoveryOutfitsService.uploadCover(file)` → `POST /api/admin/discovery-outfits/upload-cover`
  (phase 02) — **not** the shared `/upload/file` route. Decision 2026-08-27: Discovery's cover needs
  a public-read, non-expiring URL; the shared route returns a 7-day presigned one. Keeping them
  separate avoids touching common-items' upload behavior.
- Route + nav registration: `wardrobe-admin/src/App.tsx:23,56` and
  `wardrobe-admin/src/components/layout/Layout.tsx:23`.

## Requirements

1. **List page** `/discovery-outfits`: table of outfits — cover thumb, title, season, trend tags,
   item count, status pill, enabled toggle, sort_order; status filter (all/DRAFT/PUBLISHED/ARCHIVED);
   actions Edit / Publish / Archive / Delete (delete behind a confirm).
2. **Create/Edit modal**: title, description, cover image upload (mirror `handlePromoUpload`),
   season `<Select>` (spring/summer/fall/winter/none), trend-tag chip input (free text, lowercase on
   commit, ≤10), sort_order number, and the ordered item picker.
3. **Item picker (`DiscoveryItemPicker`)**: multi-select over `/admin/common-items` with search;
   selected items render as an ordered strip with up/down (or drag) reorder and remove;
   **hard-block selection beyond 4** with the reason surfaced inline ("try-on supports max 4
   garments") — do not let the admin discover the cap only as a 422.
4. Publish button disabled (with tooltip) when live item count is 0 or >4, mirroring the server gate.

## Related code files

- CREATE `wardrobe-admin/src/services/discoveryOutfitsService.ts`
- CREATE `wardrobe-admin/src/pages/DiscoveryOutfits.tsx`
- CREATE `wardrobe-admin/src/components/discovery-outfits/DiscoveryOutfitModal.tsx`
- CREATE `wardrobe-admin/src/components/discovery-outfits/DiscoveryItemPicker.tsx`
- MODIFY `wardrobe-admin/src/App.tsx` (import + `<Route path="discovery-outfits" …>`)
- MODIFY `wardrobe-admin/src/components/layout/Layout.tsx` (nav entry, phosphor icon e.g. `Compass`)

## Implementation steps

1. Service file first — mirror `trendingDropsService.ts` shape exactly (typed interfaces, one object
   export with `getAll/get/create/update/patch/replaceItems/remove`). Keep the "null clears, omitted
   leaves unchanged" comment; it is the same backend semantics.
2. Copy `TrendingDropModal.tsx` as the starting skeleton for the modal, swap the single
   `FeaturedItemPicker` for the new multi-select, add season + tags + sort_order fields.
3. `DiscoveryItemPicker` starts from `FeaturedItemPicker.tsx` (same common-items fetch/search) and
   adds selection array state + reorder + the ≤4 guard.
4. Register route + nav.
5. `yarn build` (or `tsc -b`) in `wardrobe-admin/` must pass.

## Todo

- [ ] `discoveryOutfitsService.ts`
- [ ] list page with filters + row actions
- [ ] create/edit modal incl. cover upload via `/upload/file`
- [ ] ordered ≤4 item picker
- [ ] route + sidebar entries
- [ ] typecheck/build clean; manual end-to-end against local `:5001`

## Success criteria

Admin can, in one sitting: create a DRAFT with 3 common items in a chosen order, upload a cover,
tag it `summer` + `quiet-luxury`, publish it, see it appear in `GET /api/discovery/outfits`, then
archive it and see it disappear. Selecting a 5th item is impossible in the UI.

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Bucket/prefix not actually public-read despite `get_public_url()` building the URL | M×H | Devops pre-launch checklist item (plan.md §Decisions #1) — URL construction ≠ ACL/bucket policy |
| Admin picks a non-common item | L×M | Picker only lists `/admin/common-items`; server re-validates (phase 02) |
| Reorder UX drifts from stored `position` | M×M | `PUT /items` sends the array as displayed; refetch after save |
| Copy-paste from TrendingDropModal leaves dead schedule fields | M×L | Discovery has no active_from/until — delete those inputs, don't hide them |

## Security

Admin-only surface; role enforced server-side (`get_current_admin`). No secrets in the SPA. Upload
goes through the existing `/upload/file` route — do not add a new upload path.

## Next steps

Independent of mobile phases. Can ship before or after them.
