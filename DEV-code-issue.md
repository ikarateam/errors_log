# DEV CODE ISSUES — báo bug logic phía code dev

> File tổng hợp các bug **logic của code DEV** mà tester (NLLB) đào được trong quá trình test, để gửi dev.
> Mỗi bug ghi theo format: **API bị · Vấn đề thực tế (mô tả user) · Logic code đào được · Đề xuất hướng fix**.
> Mức độ: 🔴 nặng (sai data/đếm, user thấy) · 🟠 vừa · 🟡 nhẹ. Trạng thái: `OPEN` chờ dev / `FIXING` / `DONE`.

---

## #1 — `my-joined-groups` bỏ sót nhóm đã tham gia (ZSet cache lệch bảng membership)

- **API bị:** `POST /mc-live-room/mobile/v1/groups/my-joined-groups` (microservice `live-room`)
- **Mức độ / trạng thái:** 🔴 nặng · `OPEN` · phát hiện 2026-06-25
- **Device tái hiện:** `A2890CF1-074C-45B3-A6CD-889FE96C099F-7648-0000037D3081E0C2`

### Vấn đề thực tế (theo mô tả user)
User CÓ tham gia 1 nhóm (vd "Cùng hát cùng vui"), nhưng **thi thoảng** gọi API này thì nhóm đó **không xuất hiện** trong danh sách. Phải **vào phòng (nhóm) đó rồi out ra** thì lần gọi sau nhóm mới hiện lại.

### Logic code đào được
API **không** liệt kê nhóm từ bảng membership thật `LiveRoomUser` (nguồn sự thật). Nó đọc từ một **Redis Sorted Set** `user:{userId}:groups:unread_rank` (gọi `rankKey`), snapshot sang `tempKey` rồi trả về.
*(`GroupService.getMyJoinedGroups` — `service/GroupService.java:284-446`; key: `utils/RedisUtils.java:8`)*

→ **Tập nhóm hiển thị = nội dung ZSet `rankKey`.** Nhóm nào thiếu khỏi ZSet này thì vô hình, dù DB ghi là member.

1. **Ghi ZSet lossy:** mọi lệnh ghi đi qua `setUnreadCount` / `markGroupAsRead` / `deleteGroupInCache`, đều bọc `try/catch` chỉ `log.warn`, fire-and-forget, không retry; một số ghi Redis còn async.
   *(`PostService.java:296, 317, 341`; `DatastoreSharedRepository` dùng `setValueAsync`)*
   → Lúc user join (`LiveRoomActionService.java:207` gọi `setUnreadCount`), nếu Redis chớp/nghẽn 1 nhịp → **DB membership ghi OK nhưng ZADD vào rankKey im lặng thất bại** → nhóm không bao giờ vào ZSet.

2. **Cơ chế tự-sửa bị gate sai (LỖ HỔNG GỐC):** ở page 1 có reconcile *(`GroupService.java` ~305-335)*:
   ```java
   Long cacheSize = redisTemplate.opsForZSet().zCard(rankKey);   // số phần tử ZSet
   int  expectedSize = liveRoomUsers.size();                     // số row membership DB
   if (cacheSize == null || cacheSize != expectedSize) { rebuild → ZADD lại tất cả }
   ```
   Chỉ so **SỐ LƯỢNG (cardinality)**, KHÔNG so **NỘI DUNG (set)**. Giả định "cùng count ⇒ cùng tập" là sai:
   - Thiếu nhóm thật X **+ có 1 entry rác Y** (nhóm đã rời mà ZREM mất, hoặc nhóm đã DELETED mà row LiveRoomUser vẫn được đếm) → `cacheSize == expectedSize` → **rebuild bị bỏ qua** → X mãi thiếu.
   - `expectedSize` lấy từ `findAllGroupByUserKey` = **key-query Datastore eventually-consistent** *(`LiveRoomUserSharedRepository.java:175-190`)* → con số tự dao động giữa các lần gọi → **lúc khớp lúc không** → đúng nghĩa "thi thoảng".

3. **Vì sao "vào phòng → out ra" fix được:** vào phòng → GET posts → `markGroupAsRead` *(`PostService.java:652`)* phát **`ZADD rankKey groupId`** → chèn lại đúng nhóm đang thiếu. Tương tự khi có người **đăng bài mới** trong nhóm, `updateUnreadCount` *(`PostService.java:236-260`)* loop members ZADD lại. → Nhóm "im lặng" (không có bài mới) ở trạng thái thiếu cho tới khi user tự mở phòng — khớp tuyệt đối mô tả user.

> Đã loại trừ: (a) "LiveRoomUser cache trả null" — repo có fallback DB nên member thật luôn resolve được; (b) pagination top-20 — vào phòng làm score GIẢM, không đẩy lên top, không khớp triệu chứng; (c) `cacheSize != expectedSize` (Long vs int) — có 1 toán hạng primitive nên unbox so số đúng, KHÔNG phải bug reference.

### Cách xác nhận live (Redis + GQL)
1. Resolve `userId` từ `deviceId` (Device → User).
2. `GQL: SELECT * FROM LiveRoomUser WHERE userKey = KEY(User,'<userId>')` → xác nhận row nhóm "Cùng hát cùng vui" TỒN TẠI (DB = member).
3. `ZSCORE user:<userId>:groups:unread_rank <groupId>` lúc nhóm đang mất → nếu **nil** trong khi DB có row → CHỨNG MINH ZSet-drift. Thêm `ZCARD` so count DB để thấy có trùng-giả (phantom) không.
4. Vào phòng (hoặc nhờ ai đăng bài) → `ZSCORE` lại → có giá trị → nhóm hiện lại.

### Đề xuất hướng fix
- **Chính:** page 1 reconcile theo **SET, không theo COUNT** — so tập `{groupId}` của DB membership với tập ZSet, ZADD bù các id thiếu (hoặc đơn giản luôn union toàn bộ membership DB vào rankKey mỗi page 1). Bỏ điều kiện `cacheSize != expectedSize`.
- **Phụ:** các ZADD trọng yếu (join/leave) nên đồng bộ + có retry, đừng async/swallow lỗi.
- **Phụ:** rà điều kiện `if (page == 1)` — rebuild chỉ chạy khi cursor parse đúng `1`; nếu client gửi cursor đầu là `0`/`""` thì rebuild KHÔNG bao giờ chạy *(`GroupController.java:181` `Long.parseLong(getCursor())`)*. Nên rebuild theo "page đầu" thay vì so cứng `==1`.

---

## #2 — Số following / fan bị phình (sai bộ đếm `FriendCount` + FR rác)

- **API bị (hiển thị):** danh sách bạn / following-fan (legacy `v34` — `FollowingUsers` / `Fan`). **Nguồn sinh lệch:** `v34.DeleteRequestToFollow`, `v34.AddFriend`, `v34.ConfirmFriend` + bảng `FriendCount` / `FriendRelationship`.
- **Mức độ / trạng thái:** 🔴 nặng · `OPEN` · phát hiện 2026-06-24

### Vấn đề thực tế (theo mô tả user)
- Số **following / fan hiển thị phình cao hơn thực tế**.
- **Account đã xoá vẫn trả về trong list** + vẫn bị đếm.
- **Huỷ request lúc bên kia vừa accept** nhưng hệ thống vẫn ghi nhận follow.

### Bằng chứng lệch (thật, hệ thống)
- Owner: following thực = **7** nhưng `FriendCount.follow = 29`; `User.followingNo = 7` (ĐÚNG) trong khi `FriendCount.follow = 29` (SAI) → **2 bộ đếm đá nhau**, app hiển thị qua `FriendCount` nên user thấy số phồng.
- Admin: fan thực = **11** nhưng `FriendCount.fan = 87`.
- Quét 15 user mẫu → **13/15 lệch**, gần như luôn phình cao hơn thực tế.
- ⚠️ Bản thân API `following-users` trả list CHÍNH XÁC (đọc thẳng `FriendRelationship`, owner 7=7) → **không phải nơi sinh lệch**; lệch nằm ở counter `FriendCount` + FR rác (soft-deleted).

### Logic code đào được — 4 nguyên nhân gốc
1. ⭐ **Thủ phạm chính — `v34.DeleteRequestToFollow`** (API "unfollow sạch" mà app dùng): xoá `FriendRelationship` vô điều kiện nhưng **KHÔNG gọi `updateFriendCount`** → mỗi lần unfollow để lại **+1 phantom vĩnh viễn** trong follow/fan. `AddFriend` có guard "đã follow → return" nên re-follow KHÔNG bù lại → counter chỉ tăng không giảm (khớp owner phình +22).
2. **`AddFriend` non-atomic:** `updateFriendCount(ADD)` (transaction) chạy TRƯỚC, `put(FR)` tách riêng → crash giữa chừng = count tăng nhưng FR thiếu.
3. **Account xoá MỀM (`deleteTime`) không được dọn khỏi FR:** cleanup chỉ xử lý user hard-delete (`== null`); soft-delete lọt qua → followee xoá gần 1 năm (admin: `804246…`, `deleteTime 2025-07-02`) vẫn `status=NEW`, vẫn bị đếm + vẫn trả về trong list (`isWaitingToDelete=true`). ← đúng hiện tượng "account đã xoá vẫn trả data".
4. **Race private accept-vs-cancel:** `ConfirmFriend` (B duyệt) cộng count rồi mới set FR=NEW; `DeleteRequestToFollow` (A huỷ) xoá FR đồng thời → để lại FR-ma hoặc count treo. ← đúng hiện tượng "huỷ request lúc bên kia accept nhưng vẫn ghi nhận follow".

### Đề xuất hướng fix
- **`DeleteRequestToFollow`:** khi xoá FR phải gọi `updateFriendCount` (giảm follow/fan tương ứng) — đối xứng với `AddFriend`. Đây là fix quan trọng nhất, chặn nguồn phình mới.
- **`AddFriend` (atomic):** gộp `updateFriendCount` + `put(FR)` vào **cùng 1 transaction** để không lệch khi crash.
- **Cleanup account xoá:** xử lý cả **soft-delete** (`deleteTime != null`), không chỉ hard-delete (`== null`) — dọn FR + trừ count của các followee đã xoá.
- **Race accept-vs-cancel:** dùng transaction / version-check để `ConfirmFriend` và `DeleteRequestToFollow` không cùng tác động FR một lúc (set FR và update count phải atomic, hoặc khoá theo cặp user).
- **Backfill 1 lần:** chạy job đối soát `FriendCount` về đúng số FR thật hiện có (đã loại soft-deleted) cho toàn bộ user lệch — vì lỗi cũ đã để lại data sai tích luỹ, fix code không tự sửa số cũ.
