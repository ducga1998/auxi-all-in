# Phase 02 — Admin API: repository, service, router

**Repo:** `wardrobe-backend/` · **Owner:** backend-dev · **Effort:** 4h
**Status:** pending · **Blocked by:** phase 01

## Context

- Router template: `routers/admin/trending_drops.py` (whole file — `Annotated`/`StringConstraints`
  types `:31-34`, `model_dump(exclude_unset=True)` PUT semantics `:149`, PATCH transitions `:168`,
  404-before-write guard `:153`, `get_db_context` usage `:100`).
- Repo template: `repositories/trending_drop_repository.py` (`__init__(db)` `:38`, `update()`
  applying explicit `None` `:116-133`, app-side child delete for SQLite portability `:135-149`).
- Service template: `services/trending_drop_admin_service.py` (`is_valid_featured_item:34` — the
  exact common-item validation to copy).
- Router mount: `routers/admin/__init__.py:13,31` (parent prefix `/api/admin`).
- Auth: `deps.auth.get_current_admin`.

## Key insights

- **D2 constraint (hard):** `routers/tryon.py:168` rejects try-on payloads outside 1..4 items. An
  outfit with 5+ items is un-"See-on-me"-able. Enforce `1 <= len(item_ids) <= 4` on publish, and
  `<= 4` on any item-add. This is the single most important validation in this phase.
- Item ids must be SYSTEM common items (`is_common_item and not is_deleted`) — same rule as
  `TrendingDropAdminService.is_valid_featured_item:34-39`. Copy the predicate, do not re-derive.
- Publish must be blocked (422), not soft-warned, when the outfit has 0 live items or >4 —
  unlike TrendingDrop's soft `overlaps_active_drop` warning, an unservable published outfit is a
  user-visible 404 from a social deep link.

## Architecture / data flow

```
admin SPA ──/api/admin/discovery-outfits──► router (HTTP: validate, 404 guard)
                                              │
                                    DiscoveryAdminService (rules: common-item check,
                                              │              1..4 cap, publish gate)
                                    DiscoveryRepository (ORM only)
                                              │
                                        discovery_outfits (+ _items)
```

## Endpoints (prefix `/discovery-outfits` under `/api/admin`)

| Method | Path | Body / Query | Notes |
|---|---|---|---|
| GET | `` | `?status=` | list + item counts, `sort_order` ASC then `created_at` DESC |
| GET | `/{id}` | — | full admin DTO incl. ordered items |
| POST | `` | title, description, composite_image_url?, season?, trend_tags[]?, sort_order?, item_ids[] | always creates DRAFT; validates items |
| PUT | `/{id}` | partial content fields | `exclude_unset`; nullable fields accept explicit `null` |
| PATCH | `/{id}` | `status?`, `is_enabled?` | publish gate runs here |
| PUT | `/{id}/items` | `{ item_ids: [...] }` | **full replace, ordered** — single write path for add/remove/reorder |
| DELETE | `/{id}` | — | hard delete, children first |
| POST | `/upload-cover` | multipart `file` | returns `{ "url": "…" }`, **public-read**, never expires |

**Cover upload (decision 2026-08-27):** do NOT reuse the shared `/upload/file` route
(`routers/admin/utils.py:19-44 _handle_image_upload`, which returns a 7-day presigned URL) — that
route stays untouched for common-items to avoid any blast radius on an existing feature. Discovery
gets its own tiny handler: same temp-file + `s3.upload_file(...)` pattern, prefix
`discovery_outfits/`, but return `s3.get_public_url(unique_filename)` (`utils/s3_utils.py:189`)
instead of `generate_presigned_url`. Devops must confirm the bucket/prefix is actually public-read
before launch (URL construction ≠ ACL) — tracked in plan.md, not blocking dev.

**KISS decision:** one ordered `PUT /items` full-replace instead of three endpoints
(add / remove / reorder). The list is ≤4 elements; three endpoints would triple the surface, the
tests, and the SPA state for zero benefit. Reorder = re-send the array.

Pydantic constraints (mirror column widths, per `trending_drops.py:31-34`):
`title` 1..120 strip_whitespace · `description` ≤4000 · `composite_image_url` ≤1024 ·
`season: Optional[Literal["spring","summer","fall","winter"]]` · `trend_tags: List[str]` each
1..40 chars, list ≤10, lowercased + deduped server-side · `item_ids: List[str]` len 1..4, each ≤36.

## Related code files

- CREATE `wardrobe-backend/repositories/discovery_repository.py`
- CREATE `wardrobe-backend/services/discovery_admin_service.py`
- CREATE `wardrobe-backend/routers/admin/discovery_outfits.py`
- MODIFY `wardrobe-backend/routers/admin/__init__.py` (import + `router.include_router(...)`)

Cover upload lives in `discovery_outfits.py` itself (not `admin/utils.py`) — it's a ~15-line
handler, not worth a shared module for one caller.

## Implementation steps

1. Repository: `get`, `get_with_items`, `list(status=None)`, `create(**fields)`,
   `update(id, **fields)` (apply explicit `None`, like `trending_drop_repository.py:116`),
   `replace_items(outfit_id, item_ids)` (delete children by filter, re-insert with `position=idx`,
   single commit), `delete(id)` (children first — SQLite doesn't enforce CASCADE, see
   `trending_drop_repository.py:135-141`).
2. Service `DiscoveryAdminService(db)`:
   - `validate_item_ids(ids) -> list[str]` — returns the offending ids (empty = OK): non-existent,
     `is_common_item is False`, or `is_deleted`. Reuse the predicate from
     `trending_drop_admin_service.py:34-39`.
   - `assert_publishable(outfit)` — raises `ValueError` when live item count is 0 or >4.
   - `normalize_tags(tags)` — strip, lowercase, dedupe, preserve order.
3. Router: HTTP-only. 422 on invalid item ids (`{"error": "...", "invalid_item_ids": [...]}`),
   404 before any write, 422 on publish gate failure.
4. Register in `routers/admin/__init__.py`.

## Todo

- [ ] repository with `replace_items` transactional (one commit)
- [ ] service with the three rules above
- [ ] router with 7 routes, all `Depends(get_current_admin)`
- [ ] registered under the admin parent router
- [ ] manual curl: create DRAFT → PUT items → PATCH publish → GET list

## Success criteria

- `POST` with a non-common item id → 422 naming the id; nothing written.
- `PUT /items` with 5 ids → 422; with 4 → 200 and positions 0..3 in order.
- `PATCH {"status":"PUBLISHED"}` on an outfit with 0 live items → 422, row stays DRAFT.
- `DELETE` removes children then parent; a referenced common item is untouched.

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| >4-item outfit published → mobile See-on-me 400s | H×H | 1..4 cap enforced at add AND publish (D2) |
| `replace_items` partially applied on error | M×H | Single transaction, one commit at the end; roll back on IntegrityError |
| Admin publishes an outfit whose items were soft-deleted later | M×M | `is_servable()` re-checks at read time (phase 03); admin list shows a `has_dead_items` flag in the DTO |
| PUT clearing a non-nullable column with explicit `null` | L×M | Non-`Optional` type + `None` default, as `TrendingDropUpdateRequest:73-75` |

## Security

Every route `Depends(get_current_admin)`. ORM only. Sizes bounded by `StringConstraints` so a
malformed payload is a 4xx, never a DB 500 (`trending_drops.py:29-30` rationale).

## Next steps

Unblocks phase 04 (admin SPA) and phase 05 (tests).
