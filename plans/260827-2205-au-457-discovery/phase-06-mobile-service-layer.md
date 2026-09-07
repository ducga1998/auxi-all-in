# Phase 06 — Mobile service layer + types

**Repo:** `auxi/` · **Owner:** mobile-dev · **Effort:** 2h
**Status:** pending · **Blocked by:** phase 03 **tech-lead sign-off** (hard gate)

## Context

- Service template: `auxi/src/services/trendingDropService.ts` — wraps the shared `apiClient`
  (never a new axios instance), documents wire quirks in the file header, exports typed DTOs.
- Hook template: `auxi/src/hooks/useActiveTrendingDrop.ts:16-40` — TanStack Query + a module-level
  query-key constant.
- Rule (`auxi/CLAUDE.md`): services wrap `apiClient`; screens/hooks never import axios.
- Contract source of truth: `wardrobe-backend/API_DOCUMENTATION.md` §Discovery (phase 03).

## Requirements

`auxi/src/services/discoveryService.ts`:

```ts
export type DiscoverySeason = 'spring' | 'summer' | 'fall' | 'winter';

export interface DiscoveryOutfitCard {
  id: string; title: string;
  composite_image_url: string | null;
  season: DiscoverySeason | null;
  trend_tags: string[];
  item_count: number;
}

export interface DiscoveryOutfitItem {
  id: string; position: number; name: string;
  image_url: string; image_png: string | null;
  category: string; category_code: string; layer_code: string;
  is_common_item: boolean;
}

export interface DiscoveryOutfitDetail extends Omit<DiscoveryOutfitCard, 'item_count'> {
  description: string;
  items: DiscoveryOutfitItem[];
}

export const discoveryService = {
  listOutfits(params: { season?; trendTag?; limit?; offset? }): Promise<{ outfits; total; limit; offset }>,
  getOutfit(id: string): Promise<DiscoveryOutfitDetail | null>,   // null on 404, never throws on 404
  listTrendTags(): Promise<string[]>,
};
```

`getOutfit` returning **`null` on HTTP 404** (rethrowing everything else) is deliberate: the deep
link (phase 09) and the detail screen both need "gone or unpublished → fall back", and the backend
returns an identical 404 for both causes (phase 03).

Hooks (`auxi/src/hooks/useDiscovery.ts`): `useDiscoveryOutfits(filters)` (infinite or paged query),
`useDiscoveryOutfit(id)`, `useDiscoveryTrendTags()`. Query keys under a
`DISCOVERY_QUERY_KEY = 'discovery'` root so filters invalidate cleanly.

## Related code files

- CREATE `auxi/src/services/discoveryService.ts`
- CREATE `auxi/src/hooks/useDiscovery.ts`
- No modifications to existing files in this phase (clean ownership boundary).

## Implementation steps

1. Read `API_DOCUMENTATION.md` §Discovery — types must be transcribed from it, not guessed.
2. Write the service; document any wire asymmetry in the file header (the trendingDropService habit).
3. Write hooks with `staleTime` ~60s (curated content changes rarely) and no focus-refetch.
4. `npx tsc --noEmit`.

## Todo

- [ ] service with 3 methods + 404→null semantics
- [ ] hooks with a shared query-key root
- [ ] `npx tsc --noEmit` clean

## Success criteria

Types compile against the documented contract; `getOutfit('nonexistent')` resolves `null` (verified
against a locally running backend, per the umbrella "no mocked backend" rule).

## Risk assessment

| Risk | L×I | Mitigation |
|---|---|---|
| Built against an unsigned/changed contract | M×H | Hard gate on phase 03 sign-off; types transcribed from the doc |
| 404-swallowing hides real errors | M×M | Only `error.response.status === 404` maps to null; everything else rethrows |
| Cover URL is a 7-day presigned S3 link → image 403s later | H×M | Same root issue as phase 04; mobile shows a token-styled placeholder on image error (never a broken frame) |

## Next steps

Unblocks phases 07 and 08.
