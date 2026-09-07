# Item color edit does not persist — root-cause trace

**Date:** 2026-08-21 · **Branch:** `claude/item-color-persistence-ii08r0`
**Symptom (CEO report):** đổi màu một item → UI cập nhật ngay → mở lại item thì màu quay về màu cũ.
**Verdict:** mobile client đúng end-to-end. Ghi nhận thất bại nằm ở backend
`POST /api/wardrobe/items/{id}/attributes`. **Chưa xác nhận được dòng code backend** —
repo `auxi-wardrobe/auxi-backend` bị chặn ở lớp permission của session (xem §5).

---

## 1. Luồng đã trace (auxi-mobile @ `56c63dd`)

Write path — không có lỗi:

| Bước | File | Ghi chú |
|---|---|---|
| Chọn màu trong picker | `src/screens/ItemDetailScreen.tsx:317` | `draftSetters.color = setDraftColor`, commit đúng |
| Diff vs giá trị hiện tại | `ItemDetailScreen.tsx:487` | `draftColor !== currentColor`, `currentColor = normalizeColorLabel(item)` |
| Dựng payload | `ItemDetailScreen.tsx:488-490` | gửi cả 3: `dominant_color`, `colors[]`, `color_hex` |
| Gửi request | `src/services/wardrobeService.ts:239-248` | `POST /api/wardrobe/items/{id}/attributes` |

Read path (khi mở lại item) — cũng không có lỗi:

- `ItemDetailScreen.tsx:166` → `wardrobeService.getWardrobeItem(itemId)`
- → `wardrobeService.ts:232-237` `fetchWardrobeItemById` = `GET /api/wardrobe/items` rồi `.find()`
- **Không có cache client nào trên đường này.** TanStack Query chỉ cache ở màn Home/Wardrobe
  (`wardrobeKeys`, `wardrobeService.ts:98-101`) và đã được invalidate sau khi save
  (`ItemDetailScreen.tsx:539`). ItemDetail gọi HTTP trực tiếp, không qua query cache.
- `normalizeColorLabel` (`src/utils/wardrobeItemMappers.ts:126-145`) ưu tiên
  `dominant_color` → `colors[0]` → match `color_hex`. Đọc đúng field client vừa ghi.

→ Mở lại item = fetch tươi từ server. Màu cũ hiện lại ⇒ **server trả về màu cũ** ⇒ server không lưu
(hoặc lưu rồi bị ghi đè sau đó).

## 2. Bằng chứng gián tiếp mạnh nhất

`ItemDetailScreen.tsx:512-535`: sau khi nhận `updatedItem` từ server, client **spread response rồi
ghi đè lại từng field của payload lên trên**:

```ts
const mergedItem = { ...item, ...updatedItem };
if (payload.dominant_color) { mergedItem.dominant_color = payload.dominant_color; }
if (payload.colors)         { mergedItem.colors         = payload.colors; }
if (payload.color_hex)      { mergedItem.color_hex      = payload.color_hex; }
```

Đoạn override này chỉ có ý nghĩa khi response của server **không** phản ánh thay đổi. Nó chính là
thứ làm UI hiện màu mới ngay sau khi Save (che mất lỗi), rồi lộ ra khi user mở lại item.

## 3. Giả thuyết backend (cần xác nhận, xếp theo xác suất)

1. **`updatable_fields` whitelist thiếu color** — `repositories/wardrobe_repository.py:142-147`.
   Theo `plans/260626-0005-pr148-usage-frequency-backend/plan.md:29`, endpoint `/attributes`
   lọc payload qua whitelist này. Nếu `dominant_color`/`colors`/`color_hex` không nằm trong đó,
   request trả 200 nhưng field bị drop im lặng — khớp 100% với triệu chứng.
2. **Pydantic request schema `:43` (`routers/wardrobe.py:139`) thiếu 3 field color** → FastAPI
   loại bỏ trước khi tới repo. Cùng triệu chứng.
3. **Job auto-tagging ghi đè sau đó** — cùng plan nhắc tới cơ chế `user_edits` tracking. Nếu edit
   thủ công không được đánh dấu vào `user_edits`, pipeline re-tagging (bg-removal / beautify /
   `is_preparing`) sẽ ghi đè `dominant_color` bằng giá trị AI suy ra → màu cũ quay lại.

## 4. Fix đề xuất

**Backend (fix gốc, bắt buộc):**
- Thêm `dominant_color`, `colors`, `color_hex` vào `updatable_fields`
  (`repositories/wardrobe_repository.py:142-147`) và vào request schema của `/attributes`
  (`routers/wardrobe.py`).
- Đánh dấu 3 field này vào `user_edits` khi user sửa tay, để pipeline auto-tagging không ghi đè.
- Đảm bảo `to_dict()` trả về cả 3 field trong response của `/attributes` **và** của
  `GET /wardrobe/items` (list là nguồn đọc của ItemDetail).
- Test: PATCH màu → GET list → assert `dominant_color` mới; chạy lại re-tagging → assert vẫn giữ.
- Cập nhật `API_DOCUMENTATION.md` (bắt buộc theo backend rule).

**Mobile (dọn sau khi BE xanh):**
- Bỏ block override `:521-535` trong `ItemDetailScreen.tsx`, tin vào response server. Giữ lại
  override = tiếp tục che lỗi loại này trong tương lai.
- Cân nhắc: nếu response không echo field vừa gửi → hiện toast lỗi thay vì báo success.

### 4.1 Follow-up tracking

Hai việc dưới đây KHÔNG nằm trong PR này (PR này chỉ là artifact điều tra). Phải mở ticket
riêng, nếu không sẽ rơi mất sau khi backend xanh:

- [ ] **BE** — `auxi-backend`: color fields vào `updatable_fields` + request schema +
      `user_edits`; test PATCH → GET → re-tagging; cập nhật `API_DOCUMENTATION.md`.
      Đây là nơi 3 giả thuyết §3 được xác nhận/loại bỏ trên code thật.
- [ ] **Mobile** — `auxi-mobile`: gỡ block override `ItemDetailScreen.tsx:521-535`.
      **Gate: chỉ làm SAU khi BE ticket ở trên đã merge và verify.** Gỡ sớm sẽ khiến màu
      revert ngay tại màn hình thay vì lúc mở lại — lộ lỗi rõ hơn nhưng UX tệ hơn.

## 5. Blocker

`auxi-wardrobe/auxi-backend` (private) không đọc được từ session này:
`add_repo` bị **auto mode classifier** chặn (cả `read` lẫn `push`), kể cả sau khi CEO đồng ý trong
chat — đây là gate ở lớp harness, không phải lớp GitHub. `mcp__github__get_file_contents` cũng từ
chối: session chỉ scope `auxi-wardrobe/auxi-all-in`.

Cách gỡ (CEO chọn 1):
- mở session mới với `auxi-wardrobe/auxi-backend` làm source ngay từ đầu, hoặc
- thêm permission rule cho `add_repo` trong settings, rồi ping lại session này.

## Unresolved

- Nội dung thật của `updatable_fields` (`wardrobe_repository.py:142-147`) — 3 giả thuyết §3 chưa
  phân định được nếu không đọc file.
- `/attributes` có trả 200 kèm item nguyên vẹn (silent drop) hay 422? Nếu 422 thì client đã hiện
  toast lỗi — CEO xác nhận là **không** thấy toast lỗi ⇒ nghiêng về silent drop.
- Bug có xảy ra với mọi item hay chỉ item vừa upload (`is_preparing` / vừa beautify)? Nếu chỉ nhóm
  sau ⇒ giả thuyết #3 (auto-tagging ghi đè) thắng.
