# GENERAL ERRORS — DANH SÁCH LỖI CHUNG XUYÊN SUỐT DỰ ÁN (cross-task)

> 🎯 **Mục đích:** Sổ tay lỗi dùng CHUNG cho cả team. Mỗi người / mỗi máy chạy AI (Claude) phải đọc TRƯỚC khi bắt đầu, để **không lặp lại lỗi người khác đã gặp**, giữ cách làm **đồng bộ — nhất quán**, **đỡ mất thời gian chạy lại**.
>
> 🧭 **Cách trình bày (2 tầng — quan trọng):** mỗi lỗi viết theo **2 cách diễn đạt song song**:
> - 🟢 **Nói dễ hiểu (non-tech)** — cho BA / người không rành kỹ thuật / dùng để báo cáo nhanh.
> - 🔧 **Bản kỹ thuật (tech)** — cho dev / AI / người rành, có thuật ngữ + dấu hiệu + cách fix chính xác.
>
> → Nhờ vậy CÙNG một dòng lỗi vừa báo cáo được cho người **không rõ** vừa cho người **rõ**, và log ra **đọc nhanh**.
>
> 🗂️ **Phân loại theo NGUỒN GỐC** (A→E), không chỉ lỗi kỹ thuật: hiểu sai · kỹ thuật · môi trường-đồng bộ · dữ liệu · giao tiếp.
> ✍️ **Cập nhật:** xong mỗi việc → thêm lỗi MỚI chưa có vào đúng nhóm (xem §G). Không trùng.
> 📌 Nguồn ban đầu: `apis/v5.RemoveCommentRecording/BaoCao-Loi-KF28016.md` + phiên migrate v33/34/35/36 (2026-06-09). Mirror: https://github.com/ikarateam/errors_log

---

## A. HIỂU SAI ĐỀ BÀI & MINDSET (hay lặp nhất — cảnh giác cao)

| # | 🟢 Nói dễ hiểu (non-tech) | 🔧 Bản kỹ thuật (tech) | Cách tránh |
|---|---|---|---|
| A1 | Làm sai việc vì hiểu sai yêu cầu ngay từ đầu, phải làm lại từ đầu | Tưởng phải viết skeleton cho API microservice CHƯA tồn tại; thực ra phải test API legacy ĐANG CHẠY rồi mới clone sang MS | Xác nhận RÕ mục tiêu + đầu ra trước khi code; mơ hồ thì hỏi 1 câu chốt, đừng đoán |
| A2 | Test nhầm sang tính năng/bản khác | Nhắm sai servlet / sai version (`/v34` vs `/v35`…), test không phản ánh luồng thật | Đối chiếu đúng tên + version đang chạy; đọc source 2 phía |
| A3 | Làm quá tay, lo xa không cần thiết, đào lan man | Over-engineer; sợ "vỡ fixture" vô căn cứ, probe lan man ngoài scope | Bám đúng phạm vi; nghi "sợ vỡ" → kiểm tra bằng 1 probe, đừng tự suy diễn |
| A4 | Đang làm dở việc này lại nhảy sang việc khác | Mở "mặt trận" API khác khi cái đang làm chưa xong/chưa rà rule | Làm xong + verify 1 việc rồi mới sang việc kế |
| A5 | Làm liền tù tì nhiều bước không dừng hỏi | Bulldoze chuỗi ≥3 bước phụ thuộc data, không dừng confirm (sai NLLB pacing) | Việc ≥3 bước phụ thuộc nhau → làm từng bước, DỪNG chờ confirm |
| A6 | Tự ý làm nhiều hơn cái được giao | Chỉ được duyệt bỏ A nhưng tự bỏ luôn B → vượt phạm vi | Chỉ làm đúng cái đã duyệt; đổi gì khác → hỏi |
| A7 | Vội đổ lỗi cho dev trong khi lỗi là của mình | Kết luận "bug sản phẩm" khi thực ra assert/setup của test sai | Probe trực tiếp kiểm chứng trước khi gán lỗi cho dev |
| A8 | Tự bịa thêm trường hợp test không ai yêu cầu | Thấy 1 dòng quirk trong code → tự đẻ thành TC dù không có risk thật | Quirk → ghi RISK/ghi chú chờ dev xác nhận, KHÔNG thành TC |
| A9 | Tự tạo thứ mới khi chưa được phép rồi phải gỡ | Tự tạo env var mới (`Nllb_legacy_*`) → vi phạm anti-var-bloat | Reuse biến `Nllb_` sẵn (pool owner/member/guest/bot — có cặp deviceId+uid); chỉ tạo khi đã hỏi |
| A10 | Mang bộ test mẫu về dùng nhưng quên rà lại theo chuẩn | Clone reference nhưng giữ `to.have.status`, thiếu `responseTime`, tên không dấu | Clone xong PHẢI audit theo CLAUDE_NLLB (§4.1/§4.2/§8.1) ngay |
| A11 | "Rà toàn bộ xem chỗ nào dính" → tự nới sang khu vực KHÔNG phải việc mình | NLLB nói "rà trong đống GROUP", tôi quét + sửa cả `Micro Service/Paypal` (folder ngoài task) → ĐÚNG collection nhưng SAI sub-scope, NLLB hoảng "sao tôi đụng Paypal". Khác C3 (sai collection) | "Toàn bộ" = toàn bộ **trong phạm vi được giao** (chỉ folder GROUP / chỉ KF của mình), KHÔNG phải cả collection. Liệt kê đúng tập folder trước khi sửa; folder lạ → bỏ qua + báo, đừng tự ý vá |
| A12 | Bỏ sót 1 field optional khi probe → hiểu sai hành vi API → suýt báo bug nhầm | get-users-recommend có field `type` (không @NotNull); tôi probe THIẾU type → non-owner ra 500, suýt label "non-owner bị cấm = bug permission". Thực ra 500 = NPE do thiếu `type` (`type.equalsIgnoreCase(null)`); với type đúng (VIP/ADMIN) hành vi hoàn toàn khác | **Grep SOURCE controller/DTO để biết ĐỦ field (kể cả optional) TRƯỚC khi probe + thiết kế TC**; probe quét các giá trị field optional. Đừng dựng model hành vi từ 1 request thiếu field. (gắn A7 — grep source trước khi gán bug) |
| A13 | Đọc code trên máy mình rồi kết luận chắc nịch "thiếu dòng X / lỗi ở hàm Y", nhưng bản đang chạy trên server lại KHÁC bản mình đọc | **Source checkout local ≠ build đã deploy** (DEV/PROD lệch version). Tôi suýt gán "MS thiếu nhánh owner-self-delete" dựa trên file `PostService.java` local; thực tế message + hành vi runtime không khớp dòng code đó → nếu cite dòng code làm root-cause sẽ sai | KHÔNG cite đúng dòng/nhánh code local làm bằng chứng bug khi chưa chắc bản deploy trùng. Neo kết luận vào **fact quan sát được lúc chạy** (message thật + probe GQL/Datastore), code local chỉ để định hướng. Nghi version drift → nói rõ "theo source local…", đừng khẳng định |
| A14 | Tự dựng luồng test mà app thật không làm vậy (vd "xoá nhiều bài 1 lúc") chỉ vì API cho phép | API nhận `postIds=[nhiều id]` (batch) ⇒ tôi build TC batch-delete; nhưng app chỉ xoá TỪNG bài → test không phản ánh luồng business thật, NLLB bắt bỏ | API CHO PHÉP ≠ app DÙNG. Xác nhận luồng client/business thật trước khi thiết kế; test mô phỏng đúng cách người dùng thao tác, đừng tận dụng khả năng kỹ thuật mà luồng thật không chạy |
| A15 | ĐỌC rule rồi (hook xác nhận) nhưng khi GÕ tên folder/TC mới lại bỏ dấu tiếng Việt → "Nguoi khong phai chu phong" thay vì "Người không phải chủ phòng", khó đọc | Rule §4.6 (tên TC/sub-folder tiếng Việt CÓ DẤU) đã nằm trong context, folder gốc cũng đầy dấu, nhưng lúc SINH string tên *trong code Python/JSON* một thói quen generic "ASCII-safe cho code/JSON" tự bật, ghi đè rule. Lỗi ở **bước generate, không phải bước đọc**: đọc ≠ áp dụng; pre-flight scan chỉ soi range/uid, KHÔNG soi "tên đã có dấu chưa" → cái không có trong checklist thì lọt. Postman/Newman/JSON xử Unicode tốt, ASCII-hoá là vô căn cứ | (1) Khi consolidate/rename: **COPY nguyên phong cách tên folder GỐC** (đầy dấu), đừng viết lại từ đầu theo ý mình (gắn anti-pattern "tự đổi format thay vì theo template"). (2) Thêm vào **pre-flight scan SAU build**: flag tên TC/sub-folder + tên `pm.test` nếu chứa từ tiếng Việt mà THIẾU dấu (vd có "duyet/khong/nguoi/du lieu" trần). 0 cảnh báo mới push. (3) Rule generic kỹ thuật KHÔNG được override rule dự án đã ghi rõ — khi 2 cái xung đột, rule dự án thắng |

---

## B. KỸ THUẬT DỰNG & CHẠY TEST (Postman / Newman / legacy GAE)

| # | 🟢 Nói dễ hiểu (non-tech) | 🔧 Bản kỹ thuật (tech) — dấu hiệu & fix | Cách tránh |
|---|---|---|---|
| B1 | Quên "đóng dấu/ký" yêu cầu nên server từ chối | Mất bước ký: body phải là `parameters=<base64(json)>-<key>` (ký ở pre-request cấp collection/folder). Dấu hiệu: `{"status":"FAILED","message":"ERROR"}` / 322B thiếu field. Cần collection var `password=7123358024`, `password2=3564` | Giữ script ký; request tự ký per-request thì append ký vào CUỐI pre-request (sau khi set var) |
| B2 | Địa chỉ máy chủ ghép sai nên đường dẫn hỏng | `host:["{{base_url}}"]` với base_url đã chứa `https://` → newman dựng URL hỏng, trả `<html><head>` → JSONError | Để url dạng string `"{{base_url}}/path"` + truyền base_url qua `--environment`/`--env-var` |
| B3 | Bắn dồn quá nhanh nên bị chặn tốc độ | GAE throttle → trả HTML thay JSON, JSONError "Unexpected token '<'" hàng loạt | Newman `--delay-request 1200` trở lên |
| B4 | Phản hồi có "đuôi rác" bám theo dữ liệu | Response legacy có hậu tố `OriginalRequest=...` → `pm.response.json()` ném lỗi parse | Strip trước parse: `JSON.parse(pm.response.text().replace(/OriginalRequest=[\s\S]*$/,''))` |
| B5 | Trường hợp lỗi thiếu mất vài mục dữ liệu | Nhánh error trả `{status,message}` không có `users` → `j.users.length` ném TypeError | Assert null-safe: `pm.expect(!j.users || j.users.length===0).to.be.true` |
| B6 | Mỗi API trả cấu trúc khác nhau, không giống như tưởng | vd `v33.TopUsers` KHÔNG có `status/message`, chỉ `{users,cursor}` → assert `status` fail | Probe response THẬT trước, không giả định shape |
| B7 | Vài thông báo luôn tiếng Việt dù đặt ngôn ngữ khác | i18n: 1 số message hardcode VI bất kể `language` → assert chuỗi `en` fail | Lấy chuỗi message THẬT từ response, không tự dịch |
| B8 | Đổi tên biến chỗ này quên chỗ kia | Chỉ sửa `{{var}}` trong body, quên `pm.environment.get("var")` trong script → lệch | Swap biến phải thay CẢ 3 dạng: `{{var}}`, `"var"`, `'var'` |
| B9 | Môi trường thiếu một thông tin cấu hình quan trọng | Env thiếu `utility_admin_token` (rỗng) → admin/utility call 401 "Thiếu/sai header" | Kiểm env đích đủ biến hạ tầng trước khi chạy; token dùng chung → để collection variable |
| B10 | Điều kiện/quyền đọc từ chỗ khác nơi mình tưởng | VIP gate của `GetVisitHistory` đọc `UserVipScore`+`recountTime`=tháng hiện tại, KHÔNG phải `User.vipLevel` | Đọc SOURCE để biết gate đọc field/entity nào, set đúng nguồn |
| B11 | Lệnh "sửa" không tự tạo dữ liệu; lệnh "tạo" thì lỗi | `entity/edit` → 404 nếu chưa tồn tại; `entity/create/add/upsert` → 500 | Chỉ edit được entity ĐÃ tồn tại; cần fixture có sẵn (BannedObject/UserVipScore…) hoặc seed bằng API chuyên dụng |
| B12 | Ghi xong đọc lại chưa thấy ngay (hệ thống xử lý chậm) | Ban/VIP đọc qua memcache + ghi async taskqueue → read-after-write chưa phản ánh; chập chờn | Busy-wait đủ lâu (ban ~6s; visit-history taskqueue ~15s). Đây là rào BÀI TOÁN, không phải lỗi test |
| B13 | Tên tiêu đề viết hoa/thường nhạy cảm tuỳ máy chủ | Header case-sensitive (vd `X-Admin-Token`) → 401 dù có token | Dùng đúng y tên header như reference; 401 thì kiểm tên + giá trị token |
| B14 | Chạy lẻ một phần, cấu hình chung không tự nạp | `--folder` từ cloud collection: collection-level var đôi khi không resolve → base_url thành literal `{{base_url}}` | Truyền biến qua `--env-var`/`--environment` thay vì dựa collection variable |
| B15 | Dữ liệu test cần độc nhất mỗi lần chạy + dọn sạch | recordingIDLocal trùng giây → cùng record, đếm sai; để rác | localId unique theo mili-giây `new Date().getTime()` + teardown xóa data đã tạo |
| B16 | Sửa 1 folder nhưng lưu đè cả collection → dễ giẫm việc người khác | Patch bằng `putCollection` (ghi đè NGUYÊN collection) cho 1 thay đổi nhỏ: re-save MỌI folder (Paypal/AppApi/KF khác) theo bản đã fetch → ai sửa tay xen giữa fetch↔PUT bị nuốt (lost-update), và dễ vô tình chạm folder ngoài scope | PUT theo **request/folderId** (`PUT collections/{CID}/requests/{RID}`, bỏ field server-managed) — chỉ ghi đúng item cần. Nếu buộc PUT cả collection: refetch ngay sát lúc PUT + chỉ đổi item trong folder của mình |
| B17 | Test message tiếng Việt PASS ở DEV nhưng FAIL ở PROD (đúng chữ vẫn rớt) | Server resolve user qua `request.language` → link tương ứng (`linkedUserYokara` cho en.yokara, `linkedUser` generic cho vi). Account chỉ login `en.yokara` → request `language=vi` không tìm thấy user → trả nhầm "hết phiên" thay vì message cần test. DEV pass vì account đã từng login vi (có link); PROD chưa | Khi test message `vi` (request language=vi): [Setup] PHẢI login account đó với CẢ `en.yokara` lẫn `vi` (`login_setup(roles,"vi")`). Đừng giả định 1 lần login en đủ cho mọi ngôn ngữ |
| B18 | API nặng (ES/Redis) flaky responseTime ở lần gọi đầu (cold-start) | Lần call đầu trong run = cold (JIT + cache lạnh + ES query) → responseTime spike vượt ngưỡng, dù warm chỉ ~500ms; rớt giả không phải server chậm thật | Thêm **warm-up call** trong [Setup] (gọi chính endpoint 1 lần, chỉ assert HTTP 200) để TC đo lúc đã nóng; hoặc nâng ngưỡng có lý do ghi rõ. KHÔNG để cold-start làm test flaky |
| B19 | Để file tạm chạy Newman trong `/tmp` rồi giữa chừng nó biến mất → Newman báo không thấy file | macOS/sandbox có thể **dọn `/tmp` giữa phiên** → `newman run /tmp/coll_run.json` ENOENT; chạy newman với path tương đối ở sai cwd cũng ENOENT | Để file làm việc Newman (coll/env/output) ở **thư mục ổn định dưới home** (vd `~/.nm_tmp`), KHÔNG dùng `/tmp`; dùng path tuyệt đối hoặc `cd` vào đúng thư mục trước khi chạy. (vẫn giữ rule C4: refetch + xoá cache trước mỗi run) |

---

## C. MÔI TRƯỜNG & ĐỒNG BỘ NHIỀU MÁY – NHIỀU NGƯỜI ⭐ (lý do chính cần file dùng chung)

| # | 🟢 Nói dễ hiểu (non-tech) | 🔧 Bản kỹ thuật (tech) | Cách tránh |
|---|---|---|---|
| C1 | Mỗi máy đang chạy một bản test khác nhau | Bản local stale ≠ bản người khác sửa tay trên collection/Zephyr → cùng test khác kết quả | Trước khi chạy LẤY BẢN MỚI NHẤT từ nguồn chung (refetch collection+env, `rm /tmp/coll_*.json`); đừng tin bản đang lưu |
| C2 | Dùng chung tài khoản/biến rồi giẫm chân nhau | Env DEV/PROD shared nhiều tester (`linhltt_/nnn_/anhtd_`); đụng biến/fixture người khác → state đổi giữa chừng | Chỉ dùng prefix/tài nguyên được phân (vd `NLLB_`); không đụng của tester khác |
| C3 | Đẩy kết quả vào nhầm chỗ chung của người khác | Push nhầm collection (GAE shared `8e31842d…` thay vì `AutomationTest - Ngocllb`) → đè việc người khác | Xác nhận đúng COL_ID + đúng folderId trước khi PUT/push |
| C4 | Tin dữ liệu cũ trên máy thay vì dữ liệu thật hiện tại | Cache `/tmp/coll_*.json` cũ → báo cáo sai (vd báo 55 fail trong khi thật chỉ 1) | Xoá cache + refetch collection/env trước MỖI lần chạy Newman |
| C5 | Chạy máy/môi trường này ổn nhưng nơi khác hỏng | Dev pass nhưng prod fail do thiếu composite `index.yaml` / config drift dev↔prod | Verify cả dev & prod 3-run; coi prod là chuẩn cuối; lệch cấu hình → ghi rõ report |
| C6 | Giả định công cụ/khoá trên máy mình ai cũng có | Hardcode dựa API key / tool có ở máy mình; máy khác chạy thiếu → lỗi | Ghi rõ thứ cần chuẩn bị (key, tool); đừng giả định môi trường đồng nhất |
| C7 | Push qua API thành công nhưng lát sau bản cũ tự quay lại, người kia "chưa thấy gì" | App Postman đang MỞ collection với bản cũ → auto-sync đẩy bản local (cũ) **đè ngược** lên bản vừa PUT (lost-update từ phía app, KHÁC B16 là window fetch↔PUT của chính mình). Dấu hiệu: PUT OK + fetch-ngay-sau đúng, nhưng fetch lại sau vài phút ra bản cũ | Sau khi PUT: **fetch lại xác nhận**; nếu bị revert → người đang mở app phải **Pull/refresh (hoặc đóng) collection TRƯỚC** rồi mới push lại; đừng giữ tab có chỉnh sửa chưa lưu lúc đang push. Push↔mở-app đồng thời = đè qua đè lại |

---

## D. DỮ LIỆU & TRẠNG THÁI CHẬP CHỜN (test lúc đậu lúc rớt)

| # | 🟢 Nói dễ hiểu (non-tech) | 🔧 Bản kỹ thuật (tech) | Cách tránh |
|---|---|---|---|
| D1 | Xanh giả: test báo đậu nhưng không kiểm đúng cái cần | "Any entity matches filter" khớp nhầm entity CŨ → pass-giả lọt bug thật | Khoá id CỦA LẦN CHẠY NÀY (response id, hoặc `_t0=Date.now()-5000` filter `dateTime>t0`); unset stale env trước Send |
| D2 | Chạy 1 lần thấy xanh đã tưởng chắc | "1 lần pass" ≠ deterministic; flaky lần sau rớt | Verify lặp 3-run/env; xanh ổn định mới tính done |
| D3 | Không dọn dữ liệu nên lần sau bị nhiễu | Thiếu teardown → entity tích tụ qua các run, đếm sai | Mỗi test [Cleanup] bằng API delete chính thống cùng domain (KHÔNG xoá thẳng Firebase / fake flag) |
| D4 | Đếm/so ngay sau thao tác cần thời gian xử lý | So sánh ngay sau async commit (send-gift contrib lag ~2.3s) → số chưa cập nhật | Poll-based recheck (`pm.sendRequest`+setTimeout) hoặc self-loop `setNextRequest` tới khi settle |
| D5 | Run trước bị NGẮT giữa chừng để lại "lời mời/yêu cầu" sót → run sau rớt 1 case lạ | State sót (vd Firebase `roomUserScore/<room>/invites/<fb>` chưa clear) làm assertion recheck-NEGATIVE ("đã xoá") đỏ; thao tác chính (re-invite) VẪN success nên KHÔNG phải bug API | Khi 1 case fail sau khi đã pass: PROBE state thực tế (GQL+Firebase) TRƯỚC khi đào code; dọn leftover bằng API nghịch chính thống (DECLINE/cancel) rồi chạy lại; cân nhắc pre-clean ở [Setup] cho suite có invite/request |
| D7 | Gắn "thử lại tự động" cho nhiều request TRÙNG TÊN → nó nhảy nhầm về cái đầu, lặp loạn | `postman.setNextRequest(pm.info.requestName)` (self-loop retry kiểu D4) dựa vào TÊN; trong 1 folder có ≥2 request trùng tên (vd 5 recheck "[Recheck] all-group") thì Newman/Postman luôn nhảy về request ĐẦU TIÊN khớp tên → retry lặp sai chỗ, đếm/state loạn, có khi treo vòng | Trước khi gắn self-loop: ĐẢM BẢO mỗi request đặt **tên DUY NHẤT** trong folder (gắn hậu tố phân biệt: "(member STATUS)"/"(owner mode ALL)"…). Quy tắc chung: bất kỳ cơ chế điều hướng theo tên (`setNextRequest`) đều yêu cầu tên unique |
| D8 | TC "verify không đổi" (negative) rớt chập chờn vì THAO TÁC THÀNH CÔNG của TC TRƯỚC chưa kịp ghi xong (cache lag) lúc TC này chụp baseline | API microservice ghi xong (`update-properties` 200) nhưng read-endpoint (`groups/detail`) cache eventual-consistency → giá trị cập nhật landed trễ. TC sau gọi `[Before]` chụp baseline lúc write của TC trước CÒN settling → baseline lệch giá trị recheck đọc sau đó → assert "không đổi" đỏ (dù TC này 4 request đều 401/từ chối, KHÔNG hề đổi gì). Fixed-delay càng nhỏ càng dễ dính, DEV/PROD đều có | (1) MỌI recheck đọc-lại giá-trị-vừa-ghi → **poll-retry self-loop** (`setNextRequest`, MAX ~15, early-exit) thay fixed-delay. (2) TC có [Restore] → thêm **settle-poll** cuối TC (chờ giá trị revert đúng baseline) để TC kế khởi động trên state đã settled. (3) TC negative chụp baseline [Before] rồi so chính baseline đó — vẫn cần state vào TC đã ổn định. Burn KF-27805 2026-06-15 (TC05 reject-recheck bị nhiễm bởi restore TC04) |
| D6 | Chủ nhóm fixture "đúng là chủ" nhưng PROD lại không cho thao tác → tưởng bug code, thực ra DỮ LIỆU group thiếu dòng chủ nhóm trong bảng thành viên | Permission đọc từ membership table (`LiveRoomUser` role) chứ KHÔNG fallback `LiveRoom.ownerKey`. PROD có group mà `LiveRoom.ownerKey`=owner NHƯNG `LiveRoomUser` 0 row role=OWNER (owner thiếu hẳn khỏi member list) → owner-self-action (vd posts/delete) bị từ chối `FAILED "Không thể xoá bài đăng!"`. DEV data lành → pass. Message KHÁC nhau ("không thể xoá" vs "không phải thành viên") tuỳ version deploy lệch source → đừng over-claim đúng dòng code | Owner-action fail trên 1 env: GQL probe `LiveRoomUser WHERE liveRoomKey=KEY(...)` đối chiếu `LiveRoom.ownerKey` + `Device.linkedUser*` TRƯỚC khi kết luận. Lệch data = bug DATA (seed/migrate), không phải test sai; report đủ 3 fact (entity ownerKey / membership rows / post facebookOwnerKey). Fixture chủ nhóm phải có row OWNER trong CHÍNH group test |

---

## E. GIAO TIẾP, PHẠM VI & BÁO CÁO (cho người sau hiểu đúng)

| # | 🟢 Nói dễ hiểu (non-tech) | 🔧 Bản kỹ thuật (tech) | Cách tránh |
|---|---|---|---|
| E1 | Báo "xong" trong khi còn bước chưa làm/chưa kiểm | Declare done thiếu sub-step / chưa verify-this-run | Chỉ sign-off khi đã verify đủ; còn vướng → nói rõ pending cái gì |
| E2 | Viết báo cáo/đặt tên bằng thuật ngữ khó, BA không hiểu | Tên TC/log toàn thuật ngữ (R⊆S, cursor, phantom) → BA không nắm | Viết ngôn ngữ thường có dấu; cần thì kèm bản kỹ thuật riêng (đúng tinh thần file này) |
| E3 | Che giấu/nói giảm khi test rớt hoặc bỏ qua bước | Report đẹp che fail/skip → quyết định dựa thông tin sai | Báo trung thực: rớt thì nói rớt kèm output; bỏ bước thì nói đã bỏ |
| E4 | Tự quyết thay đổi lớn mà không nêu lý do | Silent decision không trace được, người sau không phản biện được | Nêu quyết định + lý do ngắn để người khác kiểm/phản biện |
| E5 | Đọc log test bằng "lấy N dòng cuối" → cắt mất bảng tổng kết → báo nhầm số fail | `tail -N` / cắt output Newman làm mất bảng `failures` + summary table (failures liệt kê SAU phần chi tiết) → đếm thiếu/sai số case rớt rồi báo cáo sai | Capture TRỌN output ra file (`> /tmp/run.txt 2>&1`) rồi đọc bảng summary + danh sách `# failure` đầy đủ; muốn lọc thì `grep -E "assertions|failures|AssertionError"`, KHÔNG `tail` cụt |

---

## F. CHECKLIST TRƯỚC KHI BẮT ĐẦU & SAU KHI XONG

**Trước khi bắt đầu:**
1. ✅ Đọc file này + `CLAUDE.md` + `CLAUDE_NLLB.md`; xác nhận đúng mục tiêu/đầu ra + đúng API/version. *(chống A1, A2)*
2. ✅ **Lấy bản mới nhất từ nguồn chung** (refetch + xoá cache), đừng tin bản cũ trên máy. *(chống C1, C4)*
3. ✅ Reuse tài nguyên/biến được phân (`Nllb_`/`NLLB_`); không tạo mới, không đụng của người khác. *(chống A9, C2)*
4. ✅ Làm từng bước, dừng confirm; không nhảy việc; không vượt phạm vi. *(chống A4, A5, A6)*
5. ✅ Probe response THẬT trước khi viết assert (shape, message, mã lỗi, nhánh thiếu field). *(chống B5, B6, B7)*

**Sau khi xong:**
6. ✅ Chạy lặp 3-run ở CẢ dev & prod → ổn định mới tính done. *(chống C5, D2)*
7. ✅ Đảm bảo ký + strip `OriginalRequest` + null-safe + delay chống throttle. *(chống B1, B3, B4)*
8. ✅ Dọn sạch data test đã tạo bằng API delete chính thống. *(chống B15, D3)*
9. ✅ Chưa chắc lỗi của ai → probe kiểm chứng trước khi gán cho dev. *(chống A7)*
10. ✅ Báo cáo trung thực, ngôn ngữ thường (kèm bản kỹ thuật nếu cần). *(chống E1, E2, E3)*

---

## G. CÁCH CẬP NHẬT DANH SÁCH (cuối mỗi session/task — BẮT BUỘC)

1. Nhìn lại mọi lỗi/trục trặc ĐÃ GẶP trong phiên — đủ mọi nguồn (hiểu sai · kỹ thuật · môi trường-đồng bộ · dữ liệu · giao tiếp).
2. Đối chiếu A→E. Lỗi nào CHƯA có → thêm dòng mới vào **ĐÚNG nhóm theo nguồn gốc**, đánh số tiếp (A11, B16, C7…).
3. Mỗi lỗi mới viết **cả 2 tầng**: 🟢 nói dễ hiểu (non-tech) + 🔧 bản kỹ thuật — để báo cáo được cho người không rõ lẫn người rõ.
4. Lỗi đã có nhưng tái phát → KHÔNG nhân bản; chỉ bổ sung dấu hiệu/cách tránh cho rõ hơn.
5. Mục tiêu: lần sau máy nào người nào đọc cũng né được — **đồng bộ, nhất quán, đỡ chạy lại.**
