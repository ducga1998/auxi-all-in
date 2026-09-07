# Phase 01 — Discovery models + Alembic migration

**Repo:** `wardrobe-backend/` · **Owner:** backend-dev · **Effort:** 2h · **Priority:** P1 (blocks all)
**Status:** pending · **Blocked by:** —

## Context

- Idiom source: `wardrobe-backend/models/trending_drop.py` (whole file — `DropStatus:29`,
  `_as_utc:48`, `String(36)` PKs, tz-aware timestamps, `is_active:109`, `to_admin_dict:147`).
- Migration source: `migrations/versions/trendingdrop1a2b_add_trending_drops.py`.
- Model registration for autogenerate/metadata: `migrations/env.py:9-28` (explicit per-module imports).
- Season vocabulary already in the schema: `models/wardrobe.py:102` —
  `season = db.Column(db.JSON)` holding lowercase `["spring","summer","fall"]`. Reuse that
  vocabulary verbatim (lowercase) — do NOT invent `SPRING`.

## Key insights

- Discovery is a **distinct** model from `TrendingDrop` (multi-item browsable grid vs single-item
  one-active push card with dismiss semantics). Do NOT extend `TrendingDrop`.
- The whole schema is `String(36)` UUID-as-string, not Postgres UUID. SQLite is the test harness →
  no Postgres-only types, no `ARRAY`, no `JSONB`.
- `wardrobe_items` rows can be **soft-deleted** (`is_deleted`) — `RESTRICT` on the FK only blocks
  hard-delete. Servability must re-check `is_deleted` at read time (mirrors `TrendingDrop.is_active:120-125`).

## Requirements

### `DiscoveryOutfit` → table `discovery_outfits`

| Column | Type | Notes |
|---|---|---|
| `id` | String(36) PK | `default=lambda: str(uuid.uuid4())` |
| `title` | String(120) NOT NULL | |
| `description` | Text NOT NULL default `""` | optional caption on the detail screen |
| `composite_image_url` | String(1024) NULL | admin-uploaded cover art; never generated in code |
| `season` | String(16) NULL, indexed | fixed enum `spring\|summer\|fall\|winter`; NULL = all-season |
| `trend_tags` | JSON NULL | free-form admin-authored string array, lowercase, deduped |
| `status` | String(16) NOT NULL default `DRAFT`, indexed | DRAFT/PUBLISHED/ARCHIVED |
| `is_enabled` | Boolean NOT NULL default true | |
| `sort_order` | Integer NOT NULL default 0 | feed ordering, ASC then `created_at` DESC |
| `created_by` | String(36) FK users.id NULL | |
| `created_at` / `updated_at` | DateTime(timezone=True) NOT NULL | `onupdate` on `updated_at` |

Class-level constants (enum-like, mirroring `DropStatus`):
`DiscoveryStatus.{DRAFT,PUBLISHED,ARCHIVED,ALL}` and `DiscoverySeason.{SPRING,SUMMER,FALL,WINTER,ALL}`
with lowercase values.

Methods:
- `is_servable(self) -> bool` — `status == PUBLISHED and is_enabled and len(self.live_items()) >= 1`.
- `live_items(self)` — ordered items whose `item` exists and `not item.is_deleted`.
- `to_card_dict()` — public list DTO: `id, title, composite_image_url, season, trend_tags, item_count`.
- `to_public_dict(items)` — detail DTO: card fields + `description` + embedded `items`.
- `to_admin_dict()` — every column + `items` (id, position, item id/name/image_url).

### `DiscoveryOutfitItem` → table `discovery_outfit_items`

| Column | Type | Notes |
|---|---|---|
| `id` | String(36) PK | |
| `discovery_outfit_id` | String(36) FK `discovery_outfits.id` ondelete=CASCADE, NOT NULL, indexed | |
| `item_id` | String(36) FK `wardrobe_items.id` ondelete=**RESTRICT**, NOT NULL, indexed | must be a SYSTEM common item |
| `position` | Integer NOT NULL default 0 | render order |

`__table_args__ = (UniqueConstraint("discovery_outfit_id", "item_id", name="uq_discovery_outfit_item"),)`
Relationship: `item = db.relationship("WardrobeItem", foreign_keys=[item_id])`;
`outfit.items = db.relationship(..., order_by="DiscoveryOutfitItem.position", cascade="all, delete-orphan")`.

## Related code files

- CREATE `wardrobe-backend/models/discovery.py`
- MODIFY `wardrobe-backend/migrations/env.py` — add `import models.discovery` in the alphabetical block (after `models.decision`)
- CREATE `wardrobe-backend/migrations/versions/discovery1a2b_add_discovery_outfits.py`

## Implementation steps

1. Write `models/discovery.py` copying the header-comment discipline of `models/trending_drop.py`.
   Re-use its `_as_utc` idea only if a timestamp comparison is actually needed — Discovery has **no
   scheduling window**, so `is_servable()` takes no `now` argument. (YAGNI: do not port `active_from`/`active_until`.)
2. Register in `migrations/env.py`.
3. `alembic heads` (or read `migrations/versions/`) to confirm the CURRENT head before writing
   `down_revision`. As of 2026-08-27 the head is **`clonedflag1a2`**
   (`clonedflag1a2_add_is_cloned_from_common.py:27-28`). **Re-verify — do not trust this line.**
4. Hand-write the migration (`revision = "discovery1a2b"`), copying the type/server_default style of
   `trendingdrop1a2b` (`server_default=sa.text("true")` for booleans, `sa.func.now()` for timestamps).
   Indexes: `ix_discovery_outfits_status`, `ix_discovery_outfits_season`,
   `ix_discovery_outfit_items_discovery_outfit_id`, `ix_discovery_outfit_items_item_id`.
   `downgrade()` drops indexes then tables, children first.
5. Round-trip locally: `alembic upgrade head` then `alembic downgrade -1` then `upgrade head` again.

## Todo

- [ ] `models/discovery.py` with both classes + DTO methods
- [ ] `migrations/env.py` import added
- [ ] `discovery1a2b` migration written against the RE-VERIFIED head
- [ ] upgrade/downgrade/upgrade round-trip green on local Postgres
- [ ] `python -c "import models.discovery"` clean; `pytest -m unit -x` still green

## Success criteria

- Both tables created with the exact columns above; `alembic heads` returns a SINGLE head after merge.
- `DiscoveryOutfit(...).is_servable()` returns False for DRAFT, for `is_enabled=False`, and for an
  outfit whose only item is soft-deleted.

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Migration parented on a stale head → two heads on main | M×H | Re-verify head at write time (step 3); this exact trap is documented in `trendingdrop1a2b:16-19` |
| `trend_tags` JSON column can't be SQL-filtered portably | H×M | Filter tags in Python (phase 03); documented ceiling ~few hundred outfits, follow-up = tag junction table |
| Postgres rejects `server_default=1` for Boolean | M×M | Use `sa.text("true")` as `trendingdrop1a2b:57` does |
| Admin hard-deletes a common item referenced by an outfit | L×M | FK RESTRICT blocks it; soft-delete handled by `live_items()` |

## Security

No user input reaches this layer. `season` and `status` are validated at the router (phase 02), not
by a DB CHECK constraint (keeps SQLite/Postgres parity — same choice `trending_drops` made).

## Next steps

Unblocks phase 02 (admin API) and phase 03 (public API) — those two can run in parallel.
