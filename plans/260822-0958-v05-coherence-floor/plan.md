# F6 — Coherence score + floor: không bao giờ phục vụ bộ đồ ngớ ngẩn

**Date:** 2026-08-22
**Branch (umbrella):** `claude/outfit-recommendation-analysis-o7x103`
**Scope:** `wardrobe-backend/` (Phase 1-4) + `auxi/` (Phase 5 — nhãn trung thực, ship SAU khi đo)
**Owner routing:** backend → `backend-dev` · mobile → `mobile-dev` · design gate → `designer` · tuning sign-off → `tech-lead` · ngưỡng + copy + Figma → CEO
**Status:** Planned, not started
**Ưu tiên:** 🔴 **P0 — cao hơn F1** (`plans/260822-0951-v05-occasion-contract-fix/`), xem §2.4

> **Artifact điều phối.** Engine submodule không reachable từ session này
> (`ducga1998/wardrobe-backend` khác owner; `auxi-wardrobe/auxi-backend` ngoài repo-scope).
> Cùng pattern với `plans/260717-0338-new-item-surfacing-boost/plan.md`.
> Mọi `file:line` trích từ report đã verify code thật — **verify lại trước khi sửa** (§7).

---

## 1. Yêu cầu sản phẩm (CEO, 2026-08-22)

> *"Nếu hết lựa chọn cũng không nên đưa ra gợi ý quá ngớ ngẩn."*
> *"Trong tủ đồ rõ ràng còn nhiều lựa chọn phù hợp hơn, nhưng nó lại đưa ra lựa chọn không phù hợp này quá sớm."*
> *"Chỉ đưa ra lựa chọn miễn cưỡng để đủ tạo thành 1 bộ đồ khi nó thực sự dùng hết các lựa chọn
> phù hợp về mặt thời trang."*

Ba vế, ba vấn đề khác nhau:

| Vế | Vấn đề | Cần |
|---|---|---|
| "ra **quá sớm** dù tủ còn đồ tốt hơn" | **Lỗi xếp hạng** — bộ dở thắng bộ hay | Coherence là **term trong ranking** |
| "hết lựa chọn cũng không được ngớ ngẩn" | **Thiếu sàn chất lượng** | Coherence là **floor** |
| "miễn cưỡng **chỉ khi** đã dùng hết lựa chọn hợp thời trang" | Floor cứng sẽ tạo dead-end oan | Floor là **thang phân tầng**, không phải cắt cụt |

→ Cần **cả ba**, dùng chung một hàm coherence.

**Vế 3 là ràng buộc quan trọng nhất về mặt thiết kế:** floor **không xóa** ứng viên — nó **hạ
tầng** ứng viên. Bộ miễn cưỡng vẫn được phục vụ, nhưng **chỉ sau khi** tầng hợp thời trang đã
cạn thật sự. Xem §3.3.

---

## 2. Chẩn đoán — vì sao bộ dở THẮNG bộ hay

Quan sát "tủ còn nhiều lựa chọn hợp hơn" **bác bỏ giả thuyết pool starvation**. Đây không
phải vét đáy — bộ này thắng ranking sòng phẳng. Ba cơ chế:

### 2.1 Anchor không bao giờ được chấm điểm 🔴 gốc rễ

`architecture-260531-1439` §2.1(3), verify tại `engine_v05_layers.py:465-466,561-562`:

> *Outfit score = mean of non-anchor slot scores; **anchor is never scored***

Nếu quần track là anchor → bản thân nó **không bị đánh giá**. Engine chỉ chấm `sơ mi ↔ quần`
và `giày ↔ quần`. Trên 4 chiều nó nhìn được (màu / dáng / formality / length-rise):
be↔đen tương phản tốt, tan↔đen tốt. **Từng cặp điểm cao → mean cao → lọt top.**

### 2.2 Không có term "cả bộ có đọc thành một look không"

Chấm điểm là **pairwise với anchor**, không phải all-pairs, và **không có chỉ số cấp outfit**.
Không chỗ nào hỏi *"3 món này có cùng một ngôn ngữ không?"*. `mean` còn che giấu một cặp lệch.

### 2.3 Novelty THƯỞNG cho sự kỳ lạ

L5 novelty (`engine_v05_layers.py:1007-1031`) chỉ phạt lặp màu/dáng/formality/tag.
Một bộ khác thường → đọc là "fully novel" → **được cộng điểm**.

Đúng lỗi cấu trúc đã bắt ở bug dress-bias: **novelty không phân biệt được "tươi mới" với "sai"**.

### 2.4 Vì sao F6 phải đứng trước F1

| | F1 (thu hẹp formality window) | **F6 (coherence floor)** |
|---|---|---|
| Sửa ở đâu | **Đầu vào** — lọc pool | **Đầu ra** — trước khi serve |
| Trị được ca này? | Một phần | ✅ Trực tiếp |
| Rủi ro cạn pool | ⚠️ Có — thu pool → ép ghép bừa | ✅ Không thu pool |
| Phụ thuộc occasion đúng? | ✅ Có | ❌ Không — chặn mọi nguyên nhân |

F6 là backstop ở cửa ra: đúng sai gì ở thượng nguồn, **bộ ngớ ngẩn vẫn không lọt**.
F1 vẫn nên làm, nhưng sau.

---

## 3. Thiết kế

### 3.1 Hàm coherence cấp outfit (mới, thuần)

Module leaf mới `blueprints/recommendation/engine_v05_coherence.py` — cùng ràng buộc leaf như
`engine_v05_distance.py` (không import services/engine internals).

```
coherence(outfit) -> float [0,1]
  = 1 - penalty(formality_spread)      # tín hiệu chính, dễ đọc nhất
      - penalty(archetype_conflict)     # từ style_tags đã có sẵn
```

- `formality_spread = max(formality) - min(formality)` trên **TẤT CẢ** item, **kể cả anchor** →
  vá thẳng §2.1 mà không phải viết lại kiến trúc chấm điểm.
- `archetype_conflict`: bảng nhỏ đọc `style_tags` (field đã tồn tại, đã được đọc ở
  `engine_v05_signature.py:61-62`). Ví dụ `athleisure × dressy → conflict`.
  **Không schema change, không migration, không backfill.**

Bộ trong ảnh: spread ≈ 4 (track ~1 → loafer ~5) + athleisure×smart-casual → coherence rất thấp.

### 3.2 Dùng ở HAI chỗ (đây là điểm mấu chốt)

**(a) Term trong ranking** → trị vế "ra quá sớm"

Nhân vào điểm outfit trước khi rank (cùng pattern multiplier engine đã dùng —
`COMMON_INJECTED_PENALTY=0.9`). Bộ dở **tự động tụt xuống dưới** bộ hay đang có sẵn.
Không cần floor cũng đã giải quyết được ca của anh.

**(b) Floor phân tầng khi serve** → trị vế "không ngớ ngẩn" + "miễn cưỡng chỉ khi hết cách"

Floor **KHÔNG xóa** ứng viên. Nó chia ứng viên thành 2 tầng, luôn vét cạn tầng trên trước:

```
TIER 1  coherence >= V05_MIN_COHERENCE     "hợp thời trang"  → phục vụ trước, LUÔN LUÔN
TIER 2  coherence <  V05_MIN_COHERENCE     "miễn cưỡng"      → chỉ khi TIER 1 cạn sạch
```

Áp ở **cả `/build` lẫn `try_another`** (ca này ra sớm ⇒ có thể ở build — đừng chỉ vá try_another).

### 3.3 Thang leo — tái dùng nguyên pattern đã có

Engine **đã có sẵn** đúng shape này ở `try_another`: `distance floor → relaxed (0.5·floor) +
flag → terminal` (`plans/260611-2012-v05-diversity-try-another/phase-03-engine-recompose.md`).
Coherence dùng lại **y hệt**, không xây cơ chế mới:

```
1. TIER 1 — ứng viên coherence >= floor            → phục vụ  ✅ hợp thời trang
2. TIER 1 cạn → recompose (đã có) tìm thêm TIER 1  → phục vụ  ✅
3. TIER 1 thật sự cạn sạch
   → TIER 2: bộ coherence CAO NHẤT còn lại         → phục vụ  ⚠️ miễn cưỡng
     + fallback_flags += 'relaxed_coherence'
     + trace.coherence_score  (mobile hiển thị nhãn trung thực nếu muốn)
4. Không compose nổi bất kỳ bộ nào (pool rỗng thật)
   → wardrobe_gap 200 + reason  (đã có: outfit=None, CTA)
```

**Điểm mấu chốt của bước 3:** ngay cả khi miễn cưỡng, vẫn phục vụ bộ **ít tệ nhất** — vì
coherence là thang liên tục, không phải nhị phân. Bộ trong ảnh (spread ≈4) sẽ thua bộ spread 3
ngay trong chính tầng 2.

Cơ chế đã tồn tại và đã wire tới mobile — `flow-map-260529-1428-try-another-cross-repo.md:128-140`
(`fallback_flags` / `wardrobe_gap` / `wardrobe_gap_reason`; mobile `setIsWardrobeGap(true)` tại
`HomeScreen.tsx:649-686`). `relaxed_coherence` là flag **additive**, client cũ bỏ qua an toàn —
đúng chuẩn additive của repo (`tech-lead-260528-0012` §1).

**Tuyệt đối không trả 422** — `v05-eval-260602-1353` §V05-8 đã ghi 422 là UX sai.

**Hệ quả tốt:** thiết kế này gần như **không thể tăng dead-end** so với hiện tại — mọi bộ hôm nay
serve được thì mai vẫn serve được, chỉ khác **thứ tự** và **có nhãn**. Rủi ro lớn nhất ở §4 bị
triệt tiêu.

### 3.4 Observability (bắt buộc)

- **`relaxed_coherence` rate** — % lần serve phải rơi xuống TIER 2. Đây là **metric sản phẩm
  quan trọng nhất** của F6: nó đo đúng câu "tủ này có đủ đồ để phối tử tế không".
- `coherence_score` + `formality_spread` vào build trace (object trace đã có —
  `min_distance`, `seen_signatures_count` tại `engine_v05.py:~306`).
- Log TIER 2 serve vào `v05_pool_insufficient_events`, `failure_reason='coherence_relaxed'`
  (tái dùng bảng đã có, không tạo bảng mới).

→ Biến một lỗi chất lượng **vô hình** thành tín hiệu **đo được** — và đó chính là dữ liệu để nói
với user "tủ bạn thiếu món gì", hoặc để gợi ý mua sắm.

### 3.5 Mobile — hiển thị trung thực khi `relaxed_coherence` (CEO chốt 2026-08-22)

**Nguyên tắc:** không đổ lỗi cho user, không xin lỗi vòng vo, phải **hành động được**.

| Kiểu | Ví dụ | Đánh giá |
|---|---|---|
| ❌ Báo lỗi | "Không tìm được bộ đồ phù hợp" | Đọc như app hỏng |
| ❌ Tự phủ nhận | "Bộ này có thể không hợp" | Làm mất tin, không giúp gì |
| ✅ Trung thực + chỉ nguyên nhân | "Hết bộ thật hợp rồi — đây là phương án gần nhất" | Đặt kỳ vọng đúng, quy về **tủ** chứ không phải gu của app |

**Phiên bản mạnh nhất:** nối nhãn với **CTA bổ sung tủ đã có sẵn** (pattern `wardrobe_gap` →
"add items", `HomeScreen.tsx:649-686`). `relaxed_coherence` **chính là** tín hiệu thiếu đồ —
biến điểm yếu thành vòng lặp giá trị của sản phẩm thay vì một lời thú nhận cụt.

**Ràng buộc quy trình (đừng bỏ qua — đây là UI mới):**

- 🔴 **`designer` design-review là HARD GATE** (`.claude/rules/design-review-required.md`) — thêm
  pill/label mới trên outfit card thuộc diện "component với visual treatment riêng". FAIL chặn PR.
- Cần **Figma từ CEO** (CEO là designer) trước khi `mobile-dev` code — hoặc chấp nhận MVP text-only
  rồi polish sau, như tiền lệ WearPromptModal (`260515-1530-v05-phase-0` Task 2.5).
- **Mixpanel event** (`.claude/rules/analytics-tracking-required.md`): nhãn là hiển thị thụ động,
  không phải handler — nhưng **nên** track impression để đo tương quan với hành vi
  (user thấy nhãn → bấm "try another" nhiều hơn? bỏ đi?). Đề xuất `outfit_compromise_shown`
  (`object_verb`, quá khứ, snake_case) + property `coherence_score` (số, không PII).
  Cập nhật `auxi/docs/analytics/mixpanel-tracking-plan.md`.
- i18n: chuỗi tiếng Việt + tiếng Anh, không hardcode.

### 3.6 Hằng số

| Hằng số | Đề xuất | Ghi chú |
|---|---|---|
| `V05_MIN_COHERENCE` | `0.5` | Ranh giới TIER 1 / TIER 2. Sàn "không ngớ ngẩn", không phải "hoàn hảo" |
| `MAX_FORMALITY_SPREAD` | `3` | Spread ≤2 không phạt; 3 phạt nhẹ; ≥4 rớt xuống TIER 2 |
| `V05_COHERENCE_ENABLED` | `false` | Feature flag qua AlgorithmCockpit, giống `NEW_ITEM_BOOST_ENABLED` |

---

## 4. Rủi ro & mitigation

| Rủi ro | Mitigation |
|---|---|
| ~~Floor quá cao → tăng dead-end~~ | ✅ **Đã triệt tiêu bằng thiết kế** — TIER 2 luôn phục vụ được (§3.3). Bộ nào hôm nay serve được thì mai vẫn serve được, chỉ khác thứ tự + có nhãn |
| Chặn nhầm bộ cố tình lệch chuẩn (high-low styling) | Không "chặn" — chỉ hạ tầng. Bộ táo bạo vẫn ra khi TIER 1 cạn. Mood `creative` có thể hạ floor |
| `style_tags` thưa → archetype vô dụng | Đo độ phủ ở Phase 0. Thưa → chạy formality-spread trước, archetype sau |
| Chọi với novelty (§2.3) | Test bắt buộc: coherence **nhân sau** novelty, không để novelty bù lại được |
| TIER 2 rate cao → user thấy toàn bộ miễn cưỡng | Đó là **sự thật về cái tủ**, không phải lỗi engine. Đo được ⇒ chuyển thành gợi ý bổ sung tủ. Trước F6 sự thật này bị giấu |
| **Nhãn trung thực gây nhiễu** nếu `relaxed_coherence` rate cao | 🔴 **Đo TRƯỚC khi ship nhãn.** Phase 5 bị chặn bởi số liệu Phase 4. rate thấp (<15%) → nhãn rõ + CTA. rate cao → nhãn kín đáo, và ưu tiên thật sự là vá tủ/catalog chứ không phải nhãn |

---

## 5. Phases

**Phase 0 — Đo (GATE)**
- [ ] Độ phủ `style_tags` + `formality_level` trên item thật (thiếu data → hàm vô nghĩa)
- [ ] Baseline `/v05-eval --hybrid`: phân bố `formality_spread` của outfit đang serve
- [ ] **Verify §2.1 trên code thật**: anchor có thật sự không được chấm không?
- [ ] Bộ trong ảnh đến từ `/build` hay `try_another` lần mấy? → xác nhận "ra sớm"

**Phase 1 — Hàm coherence**
- [ ] `engine_v05_coherence.py` (leaf, thuần dict)
- [ ] Unit test: bộ trong ảnh → điểm thấp; bộ coherent → điểm cao; spread=0 → 1.0

**Phase 2 — Ranking term** ← *trị "ra quá sớm"*
- [ ] Nhân vào điểm outfit trước rank, sau flag `V05_COHERENCE_ENABLED`
- [ ] Test: cho pool có cả bộ hay lẫn bộ dở → bộ hay phải rank cao hơn (regression của chính ca này)

**Phase 3 — Phân tầng TIER 1 / TIER 2** ← *trị "miễn cưỡng chỉ khi hết cách"*
- [ ] Phân tầng ở **cả build lẫn try_another**
- [ ] Thang leo §3.3; TIER 2 phục vụ bộ coherence cao nhất + flag `relaxed_coherence`
- [ ] `wardrobe_gap` **chỉ** khi không compose nổi bộ nào; **không bao giờ 422**
- [ ] Observability §3.4 (đặc biệt `relaxed_coherence` rate)
- [ ] `API_DOCUMENTATION.md`: flag mới `relaxed_coherence` (bắt buộc, additive)
- [ ] Test then chốt: **TIER 2 không bao giờ serve khi TIER 1 còn ứng viên**

**Phase 4 — Eval gate**
- [ ] Coherence P2 tăng
- [ ] `pool_insufficient_rate` + `wardrobe_gap` rate **không tăng** (thiết kế nói phải bằng — lệch ⇒ có bug)
- [ ] **`relaxed_coherence` rate: ghi nhận baseline** (số mới, chưa từng đo) — 🔴 **gate của Phase 5**
- [ ] `pytest` + `python test_server.py` xanh
- [ ] CEO chốt `V05_MIN_COHERENCE` sau khi đọc eval

**Phase 5 — Nhãn trung thực trên mobile** ← *CEO chốt "nên thành thật", 2026-08-22*

> 🔴 **BỊ CHẶN bởi số liệu Phase 4.** Không ship nhãn khi chưa biết tần suất nó xuất hiện.

- [ ] Đọc `relaxed_coherence` rate từ Phase 4 → quyết mức độ nổi bật:
      **rate thấp (<15%)** → nhãn rõ + CTA bổ sung tủ ·
      **rate cao** → nhãn kín đáo, và ưu tiên thật sự chuyển sang vá tủ/catalog, không phải nhãn
- [ ] CEO chốt copy (VN + EN) + Figma cho nhãn
- [ ] `mobile-dev`: đọc `fallback_flags.includes('relaxed_coherence')` trong `v05Api.ts`
      (cùng chỗ đã map `cycled`/`wardrobe_gap`), render nhãn trên outfit card
- [ ] Mixpanel `outfit_compromise_shown` + cập nhật `auxi/docs/analytics/mixpanel-tracking-plan.md`
- [ ] i18n VN + EN, không hardcode chuỗi
- [ ] 🔴 `designer` design-review — **HARD GATE**, FAIL chặn PR
- [ ] `npx tsc --noEmit && yarn lint` xanh

---

## 6. Success criteria

- Bộ "sơ mi linen + quần track + giày lười" **không lọt top-3 ở build** khi tủ có bộ hợp hơn
- **TIER 2 không bao giờ serve khi TIER 1 còn ứng viên** (ràng buộc cứng)
- Khi buộc phải miễn cưỡng: serve bộ **ít tệ nhất** + flag `relaxed_coherence`
- `wardrobe_gap` **chỉ** khi không compose nổi bộ nào; **không bao giờ** 422
- `pool_insufficient_rate` + `wardrobe_gap` rate **không tăng** so với baseline
- Khi TIER 2: user được **nói thật** — nhãn trung thực, chỉ đúng nguyên nhân (tủ thiếu đồ),
  kèm hành động, không đổ lỗi cũng không xin lỗi vòng vo

---

## 7. Verify trước khi code

1. §2.1 "anchor never scored" — trích từ report 2026-05-31, **phải đọc code thật lại**. Nếu đã sửa → chẩn đoán đổi.
2. Độ phủ `style_tags`/`formality_level` (Phase 0) — quyết định archetype có khả thi không.
3. Ca này ở build hay try_another → quyết định Phase 3 ưu tiên chỗ nào.

## 8. Câu hỏi chưa giải quyết

1. `V05_MIN_COHERENCE` cuối — chờ eval Phase 4.
2. Mood `creative` có được nới floor không? (táo bạo vs ngớ ngẩn — ranh giới là quyết định của CEO)
3. ~~Serve bộ dưới floor kèm nhãn hay ẩn hẳn?~~ ✅ **CEO chốt 2026-08-22:** serve, nhưng **chỉ khi
   đã dùng hết lựa chọn hợp thời trang** → thiết kế TIER 1/TIER 2 (§3.3).
4. ~~Mobile có nên hiển thị nhãn khi `relaxed_coherence=true`?~~ ✅ **CEO chốt 2026-08-22: "nên
   thành thật"** → Phase 5 (§3.5). Còn treo: **copy chính xác** + **mức độ nổi bật** — cả hai chờ
   `relaxed_coherence` rate từ Phase 4.
5. Sửa gốc §2.1 (chấm cả anchor) thay vì vá bằng coherence cấp outfit — sâu hơn nhiều, đáng một
   epic riêng. F6 cố tình **không** đụng vào để giữ minimal.

## 9. Liên quan

- `plans/260822-0951-v05-occasion-contract-fix/` — F1, làm **sau** F6 (§2.4)
- `plans/reports/architecture-260531-1439-v05-engine-ceo-analysis-verification.md` — §2.1, §2.3
- `plans/reports/flow-map-260529-1428-try-another-cross-repo.md:128-140` — cơ chế `wardrobe_gap` đã có
- Linear ticket: _TBD — PM tạo_ (Linear MCP chưa auth)
