# Fix: beautify cutout punching holes into garment interior

## Bug report
User: some beautify (AI studio shot) results have transparent/semi-transparent
holes cut into the MIDDLE of the garment, not just at edges.

## Root cause (confirmed, reproduced)
`services/beautify_service.py` runs gpt-image-1's generated studio photo
through `remove_bg_service.cutout_bytes()` (rembg, `isnet-general-use`, no
alpha matting). The prompt asked for a **pure-white** background — near-zero
contrast against light/white/cream garment regions. On flat, low-texture
interior areas the model's confidence drops and the plain (non-matting) mask
leaves semi-transparent "holes" punched into the object interior, not just
soft edges.

Reproduced with a real item pulled from prod DB (Onitsuka Tiger sneakers,
cream/white body): default rembg → 46,184 partial-alpha px with visible holes
mid-shoe. Confirmed NOT an edge-touching/margin issue — holes appeared with
generous margin around the object.

Tested fixes (measured by partial-alpha pixel count + visual inspection):
- `alpha_matting=True` alone: ~50% reduction, one hole still visible.
- Gray background alone (no matting): hole moved, not eliminated.
- **Both together**: visually clean on the test sample.

## Fix (3 files, TDD)
1. `services/remove_bg_service.py` — `cutout_bytes(image_bytes, alpha_matting: bool = False)`.
   Default stays `False` — the user-upload pipeline (`ai_service.py`, real
   photos with varied backgrounds) is untouched. When `True`, passes
   `alpha_matting_foreground_threshold=240, background_threshold=10, erode_size=10`
   to `rembg.remove()`.
2. `services/beautify_service.py` — `generate_studio_shot()` now calls
   `cutout_bytes(png, alpha_matting=True)`.
3. `settings.py` — `BEAUTIFY_STUDIO_PROMPT`: background changed from
   "seamless pure-white" to "seamless light neutral gray (#E8E8E8)" +
   explicit margin instruction. Background color is invisible to end users
   (always stripped by rembg before storage) — this is a pure segmentation-
   accuracy lever, no product/UX change.

## Tests
- `tests/test_remove_bg_service.py` — new `TestCutoutBytesAlphaMatting`
  (2 tests: default omits matting kwargs, `alpha_matting=True` passes tuned
  thresholds to `rembg.remove`).
- `tests/test_beautify_service.py` — updated `mock_cutout.assert_called_once_with`
  to expect `alpha_matting=True`.
- Full suite: `29 passed` (beautify + remove_bg + endpoints + queue + model).
- Full repo `pytest -q`: 22 pre-existing failures, all unrelated (S3 URL /
  Groq key / openai_tryon_service / engine_v05 tests — local `.env` config
  bleed into isolated test fixtures; none touch beautify/remove_bg/settings
  prompt). Verified one (`test_s3_url.py`) fails identically on unrelated
  grounds (local R2 domain vs test's expected AWS domain).

## Not done / follow-ups
- Did not re-run the live OpenAI → rembg pipeline end-to-end (would cost a
  real gpt-image-1 call) — validated via a reconstructed proxy image from a
  real accepted beautify candidate instead. Recommend spot-checking the next
  few real beautify jobs in prod/staging for hole regressions.
- `alpha_matting=True` adds rembg CPU latency (extra refinement pass); not
  measured, but beautify is already a multi-second async job (OpenAI call
  dominates), so expected negligible relative impact.
- Residual risk: alpha matting improved but did not reach 0 partial-alpha
  pixels on the isolated no-gray-bg variant — the combination (gray bg +
  matting) is what got a clean result. If holes still appear on some
  garments, next lever is bumping `alpha_matting_erode_size` or switching
  rembg model.

## Unresolved questions
- None blocking. Confirm with user after a few real beautify runs whether
  the fix holds across garment types (leather/dark items unaffected either
  way since contrast was already high there).
