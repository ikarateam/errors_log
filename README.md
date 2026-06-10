# GENERAL ERRORS — DANH SÁCH LỖI CHUNG XUYÊN SUỐT QUÁ TRÌNH LÀM (cross-task)

> **File ĐỌC BẮT BUỘC trước mọi task test** (cùng `Documents/CLAUDE.md` + `claude_docs/CLAUDE_NLLB.md`) — để KHÔNG lặp lại lỗi, đỡ mất thời gian.
> Quy tắc cập nhật: **chỉ thêm lỗi CHƯA có** (không trùng). Mỗi lỗi = mô tả + hậu quả + cách tránh.
> Nguồn: gom từ `apis/v5.RemoveCommentRecording/BaoCao-Loi-KF28016.md` + phiên làm 6 API migrate v33/34/35/36 (2026-06-09).

---

## A. LỖI QUY TRÌNH / HIỂU SAI (hay lặp nhất — cảnh giác cao)

| # | Lỗi | Hậu quả | Cách tránh |
|---|---|---|---|
| A1 | **Hiểu sai bản chất nhiệm vụ** (vd: tưởng viết "skeleton dự kiến cho API microservice chưa có", thực ra phải **test API CŨ đang chạy thật** rồi clone sang MS sau) | Làm lại từ đầu, mất nhiều phiên | Xác nhận RÕ mục tiêu + đầu ra trước khi code. Nếu mơ hồ → hỏi 1 câu chốt, đừng đoán |
| A2 | **Nhắm sai API / sai phiên bản** (viết test cho API khác / bản cũ) | Test không phản ánh luồng thật | Đối chiếu đúng tên + version app đang chạy; đọc source 2 phía |
| A3 | **Làm thừa / over-engineer / đào quá sâu** (vd sợ "vỡ fixture" vô căn cứ, probe lan man) | Tốn thời gian sai trọng tâm | Bám đúng phạm vi. Khi nghi "sợ vỡ" → kiểm tra giả định bằng 1 probe, đừng tự suy diễn |
| A4 | **Nhảy việc khi việc đang dở** (lo API khác khi cái đang làm chưa xong/chưa rà rule) — bị nhắc nhiều lần | Khó kiểm soát, rối | Làm xong + verify 1 việc rồi mới sang việc kế. Không mở mặt trận mới giữa chừng |
| A5 | **Làm dồn nhiều bước không dừng xin xác nhận** | Đi xa hướng | Quy trình ≥3 bước phụ thuộc data → làm từng bước, DỪNG chờ confirm (NLLB pacing) |
| A6 | **Tự ý vượt phạm vi** (chỉ được bảo bỏ A, lại tự bỏ cả B) | Sai yêu cầu | Chỉ làm đúng cái được duyệt; thay đổi khác → hỏi |
| A7 | **Kết luận vội "lỗi của dev"** khi thực ra là lỗi test của mình | Suýt báo nhầm dev | Kiểm chứng kỹ (probe trực tiếp) trước khi gán lỗi cho sản phẩm |
| A8 | **Tự đẻ testcase từ quirk code** (thấy 1 dòng quirk → biến thành TC) khi không có yêu cầu/rủi ro thật | TC thừa, rối | Quirk → ghi RISK/ghi chú chờ dev, KHÔNG thành TC (CLAUDE_NLLB §8.1) |
| A9 | **Tự tạo biến env mới khi chưa được phép** (vd `Nllb_legacy_*`) rồi phải gỡ | Vi phạm anti-var-bloat | Reuse biến `Nllb_` sẵn có; chỉ tạo khi đã hỏi. Tận dụng pool bot/guest/owner/user (có cặp deviceId+uid) |
| A10 | **Bê bộ test reference về nhưng KHÔNG rà rule** (giữ `to.have.status`, thiếu `responseTime`, tên không dấu) | Phải sửa lại sau | Clone xong PHẢI audit theo CLAUDE_NLLB (§4.1/§4.2/§8.1) trước khi coi là xong |

---

## B. LỖI KỸ THUẬT HẠ TẦNG (Postman / Newman / legacy GAE) — gotcha quý

| # | Lỗi / Bẫy | Dấu hiệu | Cách tránh / Fix |
|---|---|---|---|
| B1 | **Quên cơ chế KÝ của API legacy** — body gửi đi phải là `parameters=<base64(json)>-<key>` (ký ở **pre-request cấp collection/folder**). Clone folder lẻ → MẤT ký | Server trả `{"status":"FAILED","message":"ERROR"}` hoặc 322B thiếu field | Đảm bảo có script ký. Collection GAE ký ở collection-level; nếu request **tự ký per-request** thì phải append ký vào CUỐI pre-request (sau khi set var). Cần collection var `password=7123358024`, `password2=3564` |
| B2 | **URL host-split với `{{base_url}}` chứa sẵn `https://`** → newman dựng URL hỏng | Trả về HTML (`<html><head>`) → JSONError | Để URL dạng **string** `"{{base_url}}/path"` + truyền base_url qua **`--environment` hoặc `--env-var`**, KHÔNG để `host:["{{base_url}}"]`. (Lưu ý: Postman tự chuẩn hóa string url → object host-split khi lưu — vẫn chạy được MIỄN base_url resolve qua env) |
| B3 | **GAE throttle khi bắn dồn request** → trả HTML thay JSON | JSONError "Unexpected token '<'" hàng loạt | Chạy newman với **`--delay-request 1200`** trở lên |
| B4 | **Response legacy có hậu tố `OriginalRequest=...`** | `pm.response.json()` ném lỗi parse | **Strip trước parse:** `var j = JSON.parse(pm.response.text().replace(/OriginalRequest=[\s\S]*$/,''))` |
| B5 | **Nhánh lỗi của API THIẾU field** (vd error trả `{status,message}` không có `users`) | `j.users.length` ném TypeError | Assert null-safe: `pm.expect(!j.users || j.users.length===0).to.be.true` |
| B6 | **Response shape khác nhau giữa API** (vd v33.TopUsers KHÔNG có `status/message`, chỉ `{users,cursor}`) | Assert `status` fail | Probe response THẬT trước, không giả định cấu trúc |
| B7 | **i18n: vài message luôn trả tiếng Việt** bất kể `language` | Assert chuỗi en fail | Lấy chuỗi message THẬT từ response (gọi thử), không tự dịch |
| B8 | **Đổi biến chỉ sửa `{{var}}` trong body, quên `"var"` trong script** (`pm.environment.get("var")`) | Body dùng biến mới, script vẫn đọc biến cũ → lệch | Khi swap biến phải thay CẢ 3 dạng: `{{var}}`, `"var"`, `'var'` |
| B9 | **Env thiếu biến hạ tầng** (vd `utility_admin_token` rỗng ở 1 env) → mọi admin/utility call 401 | 401 "Thiếu hoặc sai header" | Kiểm env đích có ĐỦ biến hạ tầng trước khi chạy. Token dùng chung → để **collection variable** (khỏi đụng env team read-only) |
| B10 | **Nguồn quyền/điều kiện đọc từ entity KHÁC kỳ vọng** (vd VIP gate của GetVisitHistory đọc `UserVipScore` + `recountTime`=tháng hiện tại, KHÔNG phải `User.vipLevel`) | Set VIP qua User entity nhưng gate vẫn báo VIP<4 | Đọc SOURCE để biết gate đọc field/entity nào; set đúng nguồn |
| B11 | **`entity/edit` KHÔNG tự tạo** (404 nếu entity chưa tồn tại); `entity/create/add/upsert` → 500 | Setup state fail 404/500 | Chỉ edit được entity ĐÃ tồn tại. Cần user/fixture có sẵn entity (BannedObject/UserVipScore...), hoặc seed bằng API chuyên dụng |
| B12 | **Ban/VIP đọc qua memcache + ghi async taskqueue** → đọc-sau-ghi chưa phản ánh | Kết quả chập chờn (lúc đúng lúc sai) | Busy-wait đủ lâu sau khi ghi (ban ~6s; visit-history taskqueue ~15s). Đây là rào BÀI TOÁN, không phải lỗi test |
| B13 | **Header phân biệt hoa/thường tùy server** (vd `X-Admin-Token`) | 401 dù có token | Dùng đúng tên header như reference; nếu 401, kiểm tên header + giá trị token có rỗng không |
| B14 | **Chạy `--folder` từ cloud collection: biến collection-level đôi khi không resolve** | base_url thành literal `{{base_url}}` | Truyền biến qua `--env-var` hoặc `--environment` thay vì dựa collection variable |
| B15 | **Dữ liệu test cần unique mỗi run + dọn sạch** (vd recordingIDLocal trùng giây → cùng record, đếm sai) | Test chập chờn, để rác | localId unique theo **mili-giây** (`new Date().getTime()`), + teardown xóa data tạo ra |

---

## C. CHECKLIST TRƯỚC KHI BẮT ĐẦU (rút từ các lỗi trên)

1. ✅ Đọc CLAUDE.md + CLAUDE_NLLB.md + file này. Xác nhận ĐÚNG mục tiêu/đầu ra + đúng API/version.
2. ✅ Làm từng bước, dừng confirm; không nhảy việc; không vượt phạm vi.
3. ✅ Reuse biến `Nllb_` sẵn có (pool owner/member/admin/guest/bot/user — có cặp deviceId+uid); KHÔNG tự tạo biến.
4. ✅ Probe response THẬT trước khi viết assert (shape, message, mã lỗi, nhánh thiếu field).
5. ✅ Đảm bảo cơ chế KÝ + strip `OriginalRequest` + null-safe assert + delay chống throttle.
6. ✅ Clone test reference → AUDIT rule ngay (status exact, responseTime, tên có dấu giọng tester ≤80).
7. ✅ State async (memcache/taskqueue) → busy-wait; data test → unique + dọn rác.
8. ✅ Chưa chắc lỗi của ai → probe kiểm chứng trước khi gán cho dev.
