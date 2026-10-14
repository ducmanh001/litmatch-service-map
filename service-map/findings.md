# Observations raised while reading the code (service map)

id: service-map:findings
status updated for fixes through commit d3cb28c

### V1: Relay realtime có thể nhân bản event khi chạy nhiều pod?

status: fixed; services: signaling-gateway

ĐÃ SỬA (commit 40e2aa3): relay() giờ emit bằng server.local.to(room) nên mỗi pod chỉ phát cho socket của mình; test hai pod chạy @socket.io/redis-adapter thật. Đánh đổi: pod mất subscription thì socket của pod đó không nhận (ghi trong docs/services/realtime-gateway.md). Quan sát gốc ngày 09/10: Trùng event có thể xảy ra: mỗi pod PSUBSCRIBE realtime:user:* rồi relay() gọi server.to(userRoom(userId)).emit (không phải server.local) trong khi redis-adapter gắn vào cùng Server qua CorsIoAdapter, nên N pod emit N lần và client có thể nhận N lần. Không test nào assert số lần nhận qua relay; docs § 6 nói "chỉ emit cho room cục bộ" lệch với code.

evidence: apps/signaling-gateway/src/app/signaling.gateway.ts:332, apps/signaling-gateway/src/main.ts:46, docs/services/realtime-gateway.md:87

### V6: Đếm view video có race khi hai request song song?

status: fixed; services: short-video

ĐÃ SỬA (commit 4f0ae66): Đếm view dùng INSERT ... ON CONFLICT DO NOTHING rồi khoá hàng (pessimistic_write), không còn đường save()+catch unique violation. Quan sát gốc ngày 09/10: Insert video_views được unique (videoId, viewerId) chặn, nhưng cộng viewCount không có guard: transaction đọc view bằng findOne không khoá, tính shouldQualifyNow = nowQualifies && !view.qualified, update không điều kiện qualified=false rồi increment. Hai request song song của cùng viewer cùng đọc qualified=false đều tăng viewCount, nên có race.

evidence: apps/core-api/src/modules/short-video/short-video.service.ts:288, apps/core-api/src/modules/short-video/short-video.service.ts:298, apps/core-api/src/modules/short-video/entities/video-view.entity.ts:16

### V7: Sweeper video failed có thể bỏ đói video mới?

status: fixed; services: short-video

ĐÃ SỬA (commit 4f0ae66): Thêm cột videos.storage_cleaned_at: hàng đã dọn xong rời hàng đợi, hàng xoá lỗi bị đẩy về cuối nên không còn lặp 200 hàng cũ nhất. Quan sát gốc ngày 09/10: Mỗi tick lấy tối đa VIDEO_SWEEP_BATCH = 200 video failed, ORDER BY updated_at ASC, id ASC, và chỉ gọi storagePort.delete; không có cờ, không xoá, không đổi status hay updated_at sau khi xử lý. Vậy cùng 200 hàng cũ nhất bị đọc lại mỗi tick; nếu backlog failed >200 thì hàng mới hơn không bao giờ vào batch này (chỉ được thử một lần ở nhánh expired lúc chuyển sang failed).

evidence: apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:91, apps/core-api/src/modules/short-video/jobs/video-sweeper.service.ts:101

### V8: sort=ranked loại video chưa có rankScore dù comment nói fallback

status: confirmed; services: short-video

Lệch giữa comment và code: VideoRankingService nói fallback sort=recent khi rankScore IS NULL, nhưng listing với sort=ranked thêm qb.andWhere('v.rankScore IS NOT NULL') và chỉ order theo rankScore, nên video mới chưa được job tính điểm bị loại khỏi danh sách ranked thay vì rơi về thứ tự recent.

evidence: apps/core-api/src/modules/short-video/jobs/video-ranking.service.ts:23, apps/core-api/src/modules/short-video/short-video.service.ts:221

### V10: Hủy mini-game có thể ghi đè kết quả đã chốt?

status: fixed; services: mini-game

ĐÃ SỬA (commit 8799483): Huỷ ván là UPDATE có điều kiện status = waiting_moves; thua race thì trả 409 hoặc trạng thái cancelled idempotent, không ghi đè ván đã resolved. Quan sát gốc ngày 09/10: cancelSession đọc session và kiểm tra status ngoài transaction (getSessionForParticipant), rồi trong transaction chỉ update theo id, không WHERE status và không khoá dòng. Cancel đua với resolve có thể ghi đè 'resolved' bằng 'cancelled'. Chiều submitMove thì có WHERE status = WaitingMoves.

evidence: apps/core-api/src/modules/mini-game/mini-game.service.ts:252, apps/core-api/src/modules/mini-game/mini-game.service.ts:194, apps/core-api/src/modules/mini-game/mini-game.service.ts:160

### V11: Kiểm bundleId Apple bị bỏ qua khi biến môi trường rỗng?

status: fixed; services: economy

ĐÃ SỬA (commit 03ab95e/fb29385): Env validation bắt buộc ECONOMY_APPLE_BUNDLE_ID khi IAP verifier là store, hoặc khi webhook verifier là store và đã điền Apple Root CA; để trống ở các chế độ còn lại vẫn bỏ qua kiểm bundleId. Docs ghi rõ kiểm receipt hiện chưa so bundle_id. Quan sát gốc ngày 09/10: Controller bỏ qua việc đối chiếu bundleId khi ECONOMY_APPLE_BUNDLE_ID rỗng (if (expectedBundleId && ...)). Mặc định rỗng: Joi default(''), .env.example và k8s/base configmap đều để ''. deploy/production/compose.yml không set biến này nên cũng ra mặc định ''. Giá trị thật ngoài repo không biết được.

evidence: apps/core-api/src/modules/economy/webhooks/economy-webhooks.controller.ts:69, apps/core-api/src/config/env.validation.ts:439, k8s/base/core-api/configmap.yaml:94

### V12: getHiddenUserIds đọc mọi report vĩnh viễn?

status: confirmed; services: safety

getHiddenUserIds lấy mọi report 2 chiều trong bảng reports không có điều kiện thời gian (comment ghi ẩn vĩnh viễn). canPair dùng cutoff = now - SAFETY_REMATCH_COOLDOWN_DAYS và lọc createdAt >= cutoff cho report gần đây. Hai hàm khác cửa sổ thời gian.

evidence: apps/core-api/src/modules/safety/safety.service.ts:273, apps/core-api/src/modules/safety/safety.service.ts:262, apps/core-api/src/modules/safety/safety.service.ts:301

### V13: Lazy-expire lời mời có thể không được lưu?

status: confirmed; services: matching

assertNotExpired chỉ gán invite.status = Expired trong bộ nhớ rồi throw DomainException (409); decline/cancel/accept gọi nó bên trong dataSource.transaction nên throw làm rollback, Expired không được lưu. Chỉ InviteSweeperService (UPDATE ... SET status = Expired) ghi DB và giải phóng uq_match_invites_pending_pair.

evidence: apps/core-api/src/modules/matching/services/invite.service.ts:625, apps/core-api/src/modules/matching/services/invite.service.ts:245, apps/core-api/src/modules/matching/jobs/invite-sweeper.service.ts:65

### M1: Chú thích đầu file k8s của media-server lỗi thời so với cấu hình

status: confirmed; services: media-server

Đầu k8s/base/media-server/deployment.yaml vẫn viết 'CHƯA chọn phương án nào ở đây vì cần ADR', nhưng thân file đã đặt hostNetwork: true theo ADR 0005. Chú thích đầu file cần cập nhật để khỏi gây hiểu nhầm là mạng RTC chưa được chốt.

evidence: k8s/base/media-server/deployment.yaml:11, k8s/base/media-server/deployment.yaml:58

### M2: Sự kiện match.matched được khai báo nhưng không có nơi nào phát

status: confirmed; services: matching, signaling-gateway

RealtimeEvents.MatchMatched (match.matched) và kiểu MatchMatchedEventData có trong libs/common-dtos nhưng không có chỗ phát nào trong core-api và web không lắng nghe; luồng thật phát match.confirmed. Comment trong matching.metrics.ts vẫn nhắc match.matched. 25 sự kiện còn lại đều có chỗ phát.

evidence: libs/common-dtos/src/lib/realtime-events.ts:42, apps/core-api/src/modules/matching/matching.metrics.ts:31

### M3: Movie Match ẩn danh phát realtime bên trong transaction, trái với tài liệu

status: confirmed; services: movie-match, signaling-gateway

updateAnonState và endAnonWatch gọi publishAnonPlayback ngay trong dataSource.transaction (sau manager.save, trước COMMIT), còn docs/services/realtime-gateway.md ghi publish luôn chạy sau khi transaction commit. Hậu quả thực tế nhỏ vì đây là bước cuối của transaction và event là best-effort, nhưng quy tắc publish-sau-commit không được giữ ở đây.

evidence: apps/core-api/src/modules/movie-match/movie-match.service.ts:438, apps/core-api/src/modules/movie-match/movie-match.service.ts:459, docs/services/realtime-gateway.md:35

### M4: AUTH_MIN_AGE và DEPLOY_ENVIRONMENT có trong .env.example nhưng không có schema hay mã đọc

status: confirmed; services: auth, app-shell

Hai biến được đặt trong .env.example (AUTH_MIN_AGE còn được nhắc ở dòng chú thích phía dưới) nhưng không có trong schema Joi của core-api hay signaling-gateway và không có mã nào đọc; có thể là cấu hình mồ côi hoặc chưa nối dây.

evidence: .env.example:121, .env.example:75

### M5: Đặc tả AsyncAPI trong repo chỉ liệt kê 8 trong 26 sự kiện realtime

status: confirmed; services: signaling-gateway

specs/realtime.asyncapi.yaml khai 8 message và không có schema payload (ghi rõ payload nằm ở kiểu nguồn), trong khi libs/common-dtos khai 26 sự kiện kèm kiểu payload đầy đủ.

evidence: specs/realtime.asyncapi.yaml:18, libs/common-dtos/src/lib/realtime-events.ts:37
