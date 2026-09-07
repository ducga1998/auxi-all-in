# F1 — Sửa contract `occasion`: khôi phục formality gate của V05

**Date:** 2026-08-22
**Branch (umbrella):** `claude/outfit-recommendation-analysis-o7x103`
**Scope:** `wardrobe-backend/` (chính) + `auxi/` (dọn dẹp, deploy sau)
**Owner routing:** backend → `backend-dev` · mobile → `mobile-dev` · contract sign-off → `tech-lead` · giá trị window → CEO
**Status:** Planned, not started
**Nguồn gốc:** issue **V05-3** (`plans/reports/v05-eval-260602-1353-hot-warmth-zero-exclusion.md:76`), mở từ 2026-06-02, chưa fix (`v05-eval-260602-1703-prod-fix-verification.md:47` liệt kê ở "Follow-ups — not yet run")

> **Artifact điều phối.** Submodule engine không reachable từ session này
> (`ducga1998/wardrobe-backend` khác owner; `auxi-wardrobe/auxi-backend` ngoài repo-scope).
> Cùng tình huống và cùng pattern với `plans/260717-0338-new-item-surfacing-boost/plan.md`.
> Implementation land trong submodule qua `backend-dev`. Mọi `file:line` dưới đây trích từ các
> report đã verify code thật — **phải verify lại trước khi sửa** (xem §7).

---

## 1. Vấn đề

Formality gate của V05 **đang bị vô hiệu hóa ngoài production** trên màn Home.

Chuỗi nhân quả:

```
auxi HomeScreen.tsx:610,629   user.occasion = mode   ('safe' | 'power' | 'creative')
        ↓
backend tra OCCASION_FORMALITY (engine_v05_constants.py:272-289)
        ↓  không khớp value nào
fallback 'unknown' = (3,6) ±2
        ↓
window = [1,8]  ←  TOÀN BỘ thang formality → gate không loại bỏ gì
```

**Bằng chứng prod:** `occasion='safe'` trên **75/124 event** (`v05-eval-260602-1353` §2).
Chính report đó tự refute giả thuyết ngược lại:
*"Formality window dropping survivors — REFUTED: occasion `safe`→unknown→`[1,8]`"*.

**Hệ quả:** engine không có cơ sở nào để từ chối ghép quần track (formality ~1) với giày lười
da (~5). Đây là guardrail duy nhất lẽ ra chặn được — nó đã tắt.

Cảnh báo đầu tiên: `code-review-260522-1020-home-screen-scan.md:97-100` (H7), từ 2026-05-22.

---

## 2. Phát hiện quyết định hướng fix — mode ĐÃ được truyền đúng chỗ

Payload mobile hiện tại (`plans/260611-1902-v05-tester-simple-mode/plan.md:83`, mirror của HomeScreen):

```jsonc
{
  "user":   { "gender": "U", "occasion": mode },              // ← ô nhiễm
  "intent": { "mood": MODE_TO_MOOD[mode] }                    // ← mode đã ở đây rồi
}
```

`MODE_TO_MOOD = { safe: 'calm', power: 'confident', creative: 'playful' }` (`:36`).

**Kết luận:** tín hiệu mode **đã đến engine đúng đường qua `intent.mood`**. `occasion: mode` là
dữ liệu thừa, và cái giá của nó là phá sập formality window.

→ **Bỏ `occasion: mode` KHÔNG mất bất kỳ tín hiệu nào.**

→ Do đó phương án "thêm `safe`/`power`/`creative` vào `OCCASION_FORMALITY`" bị **loại**: nó hợp
thức hóa việc trộn mood vào occasion, chiếm vĩnh viễn field occasion, và làm người dùng không bao
giờ chọn được occasion thật (work/date/gym). Xem §6.

---

## 3. Cái bẫy — bỏ field không đủ

Nếu mobile chỉ bỏ `occasion` mà backend giữ nguyên:

```
occasion = None  →  vẫn rơi về 'unknown'  →  vẫn [1,8]  →  bug y nguyên
```

**Fix thật nằm ở default phía backend, không phải ở mobile.** Mobile chỉ là dọn dẹp.
Đây cũng là lý do sửa backend trước: nó bảo vệ **mọi** client (app cũ, admin tester,
`RecommendationTest` sandbox) bất kể chúng gửi gì.

---

## 4. Thay đổi đề xuất

### 4.1 Backend — thu hẹp `unknown` default 🔴 đây là fix thật

**File:** `wardrobe-backend/blueprints/recommendation/engine_v05_constants.py` (~272-289)

Thu hẹp tuple fallback `unknown` từ `(3,6)` xuống một dải trung tính hợp lý.

| | tuple | sau ±widening | ý nghĩa |
|---|---|---|---|
| Hiện tại | `(3,6)` | `[1,8]` | không gate gì cả |
| **Đề xuất** | `(3,5)` | `[2,6]`¹ | vẫn rộng rãi, nhưng chặn được cực trị |

¹ **Giả định chưa verify** — phụ thuộc biên độ widening thật trong code (report ghi `±2`, nhưng
không rõ áp cho mọi occasion hay chỉ `unknown`). `backend-dev` phải đọc code thật rồi chốt số.
Mục tiêu là **kết quả** `[2,6]`, không phải tuple cụ thể.

Với `[2,6]`: quần track (f≈1) rớt khỏi window → bộ đồ trong ảnh không compose được nữa.

**Ràng buộc bắt buộc:** đưa sau **config flag** qua ML config versioning (AlgorithmCockpit —
`docs/system-architecture.md:168`), giống pattern `NEW_ITEM_BOOST_ENABLED`
(`plans/260717-0338-*` §2.3). Không hard-code đổi thẳng. Lý do ở §5.

### 4.2 Backend — chấp nhận occasion thật từ client (nếu chưa có)

Verify `OCCASION_FORMALITY` đã có các key occasion thật (`casual`/`work`/`date`/`sport`…). Nếu
thiếu, bổ sung — đây là điều kiện để mobile §4.3 gửi được giá trị có nghĩa. **Chỉ sửa bảng hằng
số, không đụng pipeline.**

### 4.3 Mobile — ngừng gửi mode-as-occasion (deploy SAU backend)

**File:** `auxi/src/screens/HomeScreen.tsx:610,629` (+ mirror trong admin V05 tester)

Bỏ `occasion: mode` khỏi payload `/v05/build`. `intent.mood` giữ nguyên.
Nếu sau này có UI chọn occasion thật thì gửi giá trị đó — ngoài scope vòng này (YAGNI).

**Thứ tự deploy:** backend trước (an toàn: app cũ gửi `safe` vẫn rơi về `unknown` nay đã hẹp
lại → được bảo vệ ngay, không cần chờ app release). Mobile sau, như dọn dẹp contract.

### 4.4 Trace + doc

- Thêm `occasion_resolved` + `formality_window` vào build trace (object trace đã có sẵn —
  `min_distance`, `seen_signatures_count` tại `engine_v05.py:~306`). Cần cho §5.
- `API_DOCUMENTATION.md`: ghi rõ vocabulary hợp lệ của `occasion` + hành vi khi unknown/absent.
  **Bắt buộc** theo two-repo contract (`CLAUDE.md`).
- `docs/system-architecture.md:194-300` mô tả pipeline V05 **sai** so với code thật (nêu
  Silhouette→Color→Layering→Footwear→Accessory; thực tế L1 hard gates → L2 compose → L5 novelty
  → L6 rank). Sửa trong cùng PR hoặc tách ticket — nhưng đừng để nguyên.

---

## 5. Rủi ro chính — thu hẹp window làm cạn pool ⚠️

Đây là rủi ro nghiêm trọng nhất và **là lý do phải có flag + eval gate**.

Pool đã cạn sẵn (`v05-eval-260602-1353` §3, số prod thật):

| | |
|---|---|
| Item `warmth_level=0`/unset | 96/356 (**27%**) |
| Anchor TOP/BOTTOM/OUTER bị loại ở mọi nhiệt độ | **25.9%** |
| User có 0 TOP sống sót ở HOT | **4/7 (57%)** |

Và Phase 0 Task 1.1 ("Formality cliff fix", `plans/260515-1530-v05-phase-0-foundation/plan.md`)
**đã từng phải nới** formality vì gate quá chặt làm 49 TOP → chỉ 8 pass L2.

→ **Siết lại formality lúc này có thể tái tạo đúng bug mà Task 1.1 đã sửa.**

**Mitigation (bắt buộc, không phải tùy chọn):**

1. Chạy `/v05-eval --hybrid` lấy **baseline trước** khi bật flag.
2. Bật flag cho cohort nhỏ trước.
3. Gate: `v05_pool_insufficient_rate` **không tăng**. Tăng → rollback flag, nới window một bậc, đo lại.
4. Nếu `[2,6]` làm cạn pool: cân nhắc soft-penalty thay hard gate (multiplier, giống
   `COMMON_INJECTED_PENALTY=0.9`) — cùng pattern engine đã có.

---

## 6. Phương án đã cân nhắc và loại

| Phương án | Lý do loại |
|---|---|
| Thêm `safe`/`power`/`creative` vào `OCCASION_FORMALITY` | Hợp thức hóa conflation. Mode đã có đường riêng qua `intent.mood` (§2). Chiếm vĩnh viễn field occasion → không bao giờ có occasion thật. |
| Chỉ sửa mobile, không đụng backend | Không đủ — `occasion=None` vẫn rơi về `unknown` `[1,8]` (§3). Và không bảo vệ client cũ. |
| Bỏ hẳn fallback, bắt buộc occasion | Breaking change trên wire. Vi phạm nguyên tắc additive của repo (xem `tech-lead-260528-0012` §1). |

---

## 7. Verify TRƯỚC khi code (gate)

Không viết dòng code nào trước khi trả lời xong:

- [ ] **V05-3 còn sống không?** Report ghi 06/2026, nay là 08/2026.
      → `SELECT occasion, COUNT(*) FROM v05_pool_insufficient_events WHERE created_at > NOW() - INTERVAL '14 days' GROUP BY 1;`
      Nếu không còn `safe`/`power`/`creative` → đã có người fix → **dừng, đóng plan này**.
- [ ] Đọc `OCCASION_FORMALITY` thật (`engine_v05_constants.py:272-289`): các key hiện có, tuple
      `unknown`, và **biên độ widening áp ở đâu** (mọi occasion hay chỉ unknown?). Quyết định số
      cuối ở §4.1 phụ thuộc câu này.
- [ ] Verify `HomeScreen.tsx:610,629` còn gửi `occasion: mode` không (line number có thể đã trôi
      sau refactor `GH-364-mobile-screen-refactor`).
- [ ] Confirm engine thật sự đọc `intent.mood` (nếu không, việc bỏ `occasion: mode` **có** làm mất
      tín hiệu → đổi hướng fix).

---

## 8. Phases

**Phase 0 — Verify (gate)** → §7. Không pass → không code.

**Phase 1 — Backend (`backend-dev`)**
- [ ] Thu hẹp `unknown` default (§4.1), sau config flag
- [ ] Bổ sung occasion thật vào `OCCASION_FORMALITY` nếu thiếu (§4.2)
- [ ] Trace `occasion_resolved` + `formality_window` (§4.4)
- [ ] Tests: unknown→window mới · occasion hợp lệ→window đúng · absent→window mới ·
      **regression: outfit spread formality > 4 không compose được** · flag off = hành vi cũ y nguyên
- [ ] `API_DOCUMENTATION.md` (bắt buộc)
- [ ] `pytest` + `python test_server.py` xanh

**Phase 2 — Eval gate (`v05-eval`)**
- [ ] Baseline trước khi bật flag
- [ ] Bật cohort nhỏ → so sánh: `pool_insufficient_rate` không tăng, coherence P2 tăng
- [ ] Regression fail → rollback flag, nới một bậc, lặp lại

**Phase 3 — Mobile (`mobile-dev`, sau khi Phase 2 pass)**
- [ ] Bỏ `occasion: mode` (§4.3) + admin V05 tester mirror
- [ ] `npx tsc --noEmit && yarn lint`
- [ ] Không có UI mới → không phát sinh event Mixpanel
      (`.claude/rules/analytics-tracking-required.md` — thuộc diện "refactor giữ nguyên behavior")

**Phase 4 — Đóng**
- [ ] `tech-lead` sign-off contract two-repo
- [ ] Bump submodule pin sau khi cả 2 repo merge
- [ ] Sửa `docs/system-architecture.md:194-300` (§4.4)

---

## 9. Success criteria

- `occasion` unknown/absent → window **không còn** `[1,8]`
- Bộ "sơ mi linen + quần track + giày lười" (spread formality ~4) không compose được ở window mới
- `v05_pool_insufficient_rate` **không tăng** so với baseline
- v05-eval coherence (P2) tăng đo được
- Không breaking change trên wire; client cũ degrade an toàn

---

## 10. Câu hỏi chưa giải quyết

1. Giá trị window `unknown` cuối cùng — chờ đọc bảng thật + eval Phase 2. `[2,6]` là đề xuất, chưa chốt.
2. Biên độ ±widening áp cho mọi occasion hay chỉ `unknown`? Quyết định §4.1.
3. Có cần UI chọn occasion thật trên Home không, hay `intent.mood` là đủ? → CEO. Ngoài scope vòng này.
4. Nếu window hẹp làm cạn pool: hard gate hay soft-penalty? → quyết theo số liệu Phase 2 (§5.4).
5. V05-3 còn sống không (§7) — **chưa verify được trong session này** vì không có DB/engine access.

---

## 11. Traceability

- **Nguồn:** V05-3 (`plans/reports/v05-eval-260602-1353-hot-warmth-zero-exclusion.md:76`)
- **Cảnh báo sớm:** H7 (`plans/reports/code-review-260522-1020-home-screen-scan.md:97-100`)
- **Follow-up chưa chạy:** `plans/reports/v05-eval-260602-1703-prod-fix-verification.md:47`
- **Branch:** `claude/outfit-recommendation-analysis-o7x103`
- **Linear ticket:** _TBD — PM tạo & gán_ (Linear MCP chưa auth ở session này)

---

## 12. Cập nhật ưu tiên

F6 (plan này) đứng **trước** F1 — xem `plans/260822-0958-v05-coherence-floor/plan.md` §2.4.
F1 vẫn nên làm, nhưng là lớp phòng thủ thứ hai, không phải lớp đầu.
