# Phase 03 — Public API `/api/discovery/*` + contract sign-off

**Repo:** `wardrobe-backend/` · **Owner:** backend-dev, then **tech-lead sign-off** · **Effort:** 3h
**Status:** pending · **Blocked by:** phase 01 · **BLOCKS: phase 06 (mobile)**

## Context

- Public router template: `routers/trending_drop.py` (whole file — `SimpleRateLimiter` usage
  `:34-36`, `_enforce_rate_limit` helper `:39-53` including the `RATE_LIMIT_ENABLED` early-return,
  `Depends(get_current_user)` + `Depends(get_db)` `:65-66`, service exceptions → HTTP `:95-121`).
- Registration: `routers/__init__.py:26,52` and `app.py:269,328`.
- Contract rule: `wardrobe-backend/CLAUDE.md` §Rules — `API_DOCUMENTATION.md` update is MANDATORY;
  root `CLAUDE.md` "Two-Repo Contract" — tech-lead signs off before mobile builds.

## Endpoints

### `GET /api/discovery/outfits`

Auth required. Rate limit 60/min (read tier). Query: `season?`, `trend_tag?`, `limit` (default 20,
max 50), `offset` (default 0).

```json
{
  "outfits": [
    { "id": "…", "title": "…", "composite_image_url": "…|null",
      "season": "summer|null", "trend_tags": ["quiet-luxury"], "item_count": 3 }
  ],
  "count": 20, "total": 47, "limit": 20, "offset": 0
}
```

Only `is_servable()` outfits. Order: `sort_order` ASC, then `created_at` DESC. Unknown `season`
value → 422 (closed enum). Unknown `trend_tag` → empty list, **not** an error (tags are free-form).

### `GET /api/discovery/outfits/{outfit_id}`

Auth required. Rate limit 60/min.

```json
{
  "id": "…", "title": "…", "description": "…",
  "composite_image_url": "…|null", "season": "summer|null", "trend_tags": ["…"],
  "items": [
    { "id": "…", "position": 0, "name": "…", "image_url": "…", "image_png": "…|null",
      "category": "top", "category_code": "TEE", "layer_code": "L2", "is_common_item": true }
  ]
}
```

**404** when the id doesn't exist OR the outfit isn't servable (DRAFT / ARCHIVED / disabled / all
items soft-deleted). This 404 is the contract the mobile deep-link fallback keys on — the mobile
client MUST NOT distinguish "missing" from "unpublished" (avoids leaking draft existence).
Soft-deleted items are dropped from `items`; if that empties the outfit → 404.

### `GET /api/discovery/trend-tags` (small, worth it)

Returns the distinct tags across servable outfits: `{ "tags": ["quiet-luxury", "y2k"] }`.
Without it the mobile filter row would have to derive chips from page 1 only — which silently hides
tags that live on page 2. ~15 LOC, removes a whole class of UI bug.

## Related code files

- CREATE `wardrobe-backend/services/discovery_service.py` (public read service; `DiscoveryNotFoundError`)
- CREATE `wardrobe-backend/routers/discovery.py`
- MODIFY `wardrobe-backend/routers/__init__.py` (import + `__all__`)
- MODIFY `wardrobe-backend/app.py` (import + `include_router`)
- MODIFY `wardrobe-backend/API_DOCUMENTATION.md` (new `## Discovery` section — MANDATORY)

## Implementation steps

1. `DiscoveryService(db)`: `list_servable(season, trend_tag, limit, offset)` and
   `get_servable(outfit_id)` raising `DiscoveryNotFoundError`.
2. **Filtering strategy (deliberate):** narrow in SQL by `status == PUBLISHED`, `is_enabled`, and
   `season` (indexed column). Filter `trend_tags` in **Python** — the column is portable JSON and
   `JSON_CONTAINS`/`@>` diverge between SQLite (tests) and Postgres (prod). Same rationale as
   `trending_drop_repository.py:8-12`. Paginate AFTER the Python filter so `total` is correct.
   Ceiling: fine to a few hundred outfits; beyond that, migrate to a `discovery_outfit_tags` table
   (follow-up, not now — YAGNI).
3. Router: copy `routers/trending_drop.py`'s rate-limit helper verbatim, two `SimpleRateLimiter(60)`
   instances. `DiscoveryNotFoundError` → 404 with `{"error","message","request_id"}` (same envelope
   as `trending_drop.py:96-103`).
4. Register in `routers/__init__.py` and `app.py`.
5. Write the `API_DOCUMENTATION.md` section following `.claude/rules/api-documentation.md` format
   (method, auth, rate limit, request, response, errors, example).

## Todo

- [ ] service + router + registration
- [ ] `API_DOCUMENTATION.md` §Discovery written (all 3 endpoints, both 404 causes documented)
- [ ] `uvicorn` boots, `/docs` shows the routes
- [ ] **tech-lead sign-off recorded** (report in `plans/reports/`) → unblocks phase 06

## Success criteria

- DRAFT outfit id → detail 404, and absent from list.
- Publish it → appears in list, detail 200.
- Soft-delete its last item → back to 404 + absent from list, **no 500**.
- `?season=autumn` (not in enum) → 422; `?season=fall` → filtered list.
- Unauthenticated → 401. 61st read in a minute → 429 (when `RATE_LIMIT_ENABLED`).

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Mobile built against an unsigned contract → drift | M×H | Hard gate: phase 06 starts only after tech-lead sign-off |
| Python-side tag filter degrades at scale | L×M | Documented ceiling + follow-up junction table; admin content is curated (tens of rows) |
| Detail leaks a soft-deleted item's dead image URL | M×M | `live_items()` filter in the DTO builder, covered by a phase-05 test |
| `season` free-typed by admin diverges from the mobile chip list | M×M | Closed `Literal` at BOTH admin (phase 02) and public routers; lowercase, matching `models/wardrobe.py:102` |
| Naming drift `composite_image_url` vs `promo_image_url` | L×L | One name across model/DTO/SPA/mobile — never rename between layers |

## Security

`Depends(get_current_user)` on all three routes (Discovery is not anonymous). 404-not-403 for
non-servable outfits so DRAFT titles can't be enumerated. Rate-limited per user id.

## Next steps

Unblocks phase 05 (tests) and, after sign-off, phase 06 (mobile service layer).
