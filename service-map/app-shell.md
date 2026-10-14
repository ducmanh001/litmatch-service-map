# Service map: App shell (health, capabilities)

id: service-map:app-shell
module: module:app-shell
deployable: core-api
confidence: high

Health probe và hợp đồng capabilities

Cung cấp /health, /health/live, /health/ready (probe cho Docker/k8s) và GET /api/v1/capabilities (hợp đồng tính năng runtime cho web/admin). Không có bảng riêng, không chứa business logic; chỉ kiểm tra Postgres/Redis và phản chiếu cấu hình env của các module khác.

Owns: không sở hữu bảng/khoá Redis nào (readiness mở kết nối ioredis riêng chỉ để PING)

## Endpoints and events

| id | kind | method | path | auth | purpose |
|---|---|---|---|---|---|
| e1 | rest | GET | /health | public | Alias liveness cũ: trả { status: 'ok', uptimeSeconds }, không gọi dependency, @Public và @SkipThrottle. |
| e2 | rest | GET | /health/live | public | Liveness: chỉ cho biết process còn chạy (dùng cho Docker HEALTHCHECK và k8s livenessProbe), không chạm Postgres/Redis, @Public và @SkipThrottle. |
| e3 | rest | GET | /health/ready | public | Readiness: chạy song song SELECT 1 trên Postgres và PING Redis (timeout 1s), trả 503 nếu một trong hai down để orchestrator rút pod khỏi load balancer; @Public, throttle 30/phút. |
| e4 | rest | GET | /api/v1/capabilities | public | Trả CapabilitiesDto (auth.phoneOtp/google/apple/facebook/guest, topUp.web/native, video, notifications.push) suy ra từ env thật và danh sách bảo trì CAPABILITY_MAINTENANCE_FEATURES; @Public, @SkipThrottle. |

## Coordination edges (why this service talks to others)

### SELECT 1 kiểm sống (c50) with postgres [infra, out, shared-db]

- endpoints: e3
- what: ReadinessService.checkPostgres chạy dataSource.query('SELECT 1') với timeout 1s qua DataSource TypeORM dùng chung.
- why: core-api không thể phục vụ request nghiệp vụ nếu không có Postgres (nguồn sự thật duy nhất của mọi module), nên pod chỉ được nhận traffic khi truy vấn đơn giản này thành công; timeout 1s tránh probe treo làm kubelet kẹt.
- if missing: Pod mới lên hoặc mất DB vẫn được cân bằng tải vào, mọi request trả 500 thay vì bị rút khỏi rotation.
- evidence: apps/core-api/src/app/readiness.service.ts:57
- review: sound

### PING Redis kiểm sống (c51) with redis [infra, out, redis-state]

- endpoints: e3
- what: ReadinessService giữ một ioredis riêng (lazyConnect, không retry, không offline queue) và gọi PING với timeout 1s.
- why: Redis phục vụ rate limit, realtime publish và presence nên mất Redis làm core-api suy giảm; probe dùng client riêng không retry để kết quả phản ánh trạng thái thật ngay lập tức và không làm nhiễu client nghiệp vụ.
- if missing: Redis chết mà readiness vẫn xanh: rate limit/realtime hỏng im lặng và orchestrator không phát hiện.
- evidence: apps/core-api/src/app/readiness.service.ts:67
- review: sound

### đọc capabilities để bật nút (c52) with web [client, in, http]

- endpoints: e4
- what: apps/web useCapabilities gọi GET /api/v1/capabilities (staleTime Infinity, retry false); login-form dùng isCapabilityUsable để bật/tắt phoneOtp, guest, Google/Apple/Facebook (kèm clientId), top-up dùng topUp.*.
- why: Khả năng thật (provider có credential hay không, đang bảo trì không) chỉ server biết; contract yêu cầu frontend không tự suy từ build env, nên client phải hỏi để không hiện nút đăng nhập/nạp tiền mà server sẽ từ chối.
- if missing: Web hiện nút Google/OTP/nạp tiền chưa cấu hình rồi nhận lỗi 400/403 khi bấm; không thể bảo trì từng tính năng từ server.
- evidence: apps/web/src/shared/capabilities/api.ts:24
- review: sound

### capabilities cho trang login (c53) with admin-console [client, in, http]

- endpoints: e4
- what: apps/admin có bản useCapabilities riêng gọi cùng /api/v1/capabilities; login-page kiểm isCapabilityUsable(auth.phoneOtp) và các provider trước khi cho đăng nhập.
- why: Admin cũng đăng nhập qua module auth nên cần biết phương thức nào đang bật; dùng chung hợp đồng runtime thay vì env build để một lần deploy server đổi được cả hai frontend.
- if missing: Admin console hiện phương thức đăng nhập không dùng được hoặc phải hard-code theo env build.
- evidence: apps/admin/src/shared/capabilities/api.ts:24
- review: sound

### phản chiếu cờ đăng nhập (c54) with auth [service, out, config]

- endpoints: e4
- what: CapabilitiesService đọc AUTH_PHONE_OTP_ENABLED, AUTH_GOOGLE_CLIENT_ID, AUTH_APPLE_CLIENT_ID, AUTH_FACEBOOK_APP_ID/SECRET để tính auth.* và trả clientId cho client.
- why: Chính các env này quyết định AuthService có cho đăng nhập hay không (assertPhoneOtpEnabled trả 403, SocialVerifierService từ chối provider chưa cấu hình), nên capabilities phải đọc đúng cùng nguồn để không nói 'enabled' khi auth sẽ từ chối.
- if missing: Hai nơi lệch nhau: UI báo đăng nhập khả dụng nhưng /auth/* trả 403/400.
- evidence: apps/core-api/src/app/capabilities.service.ts:191
- review: sound

### phản chiếu cờ nạp tiền (c55) with economy [service, out, config]

- endpoints: e4
- what: CapabilitiesService đọc ECONOMY_IAP_VERIFIER, ECONOMY_APPLE_SHARED_SECRET, ECONOMY_GOOGLE_* và PAYOS_* để tính topUp.web/native/nativeApple/nativeGoogle.
- why: Economy chỉ xác nhận được giao dịch khi verifier (dev/store/disabled) và credential PayOS/Apple/Google đúng cấu hình; client phải biết trước kênh nạp nào dùng được để không dẫn người dùng vào luồng thanh toán server không xác nhận được (suy luận).
- if missing: Hiển thị gói nạp cho kênh chưa có credential, khiến giao dịch thất bại sau khi người dùng đã bắt đầu thanh toán.
- evidence: apps/core-api/src/app/capabilities.service.ts:47
- review: sound — Đúng với code: mode disabled/thiếu credential PayOS bị Economy từ chối; bỏ marker '(suy luận)'.

### phản chiếu cờ push (c56) with notification [service, out, config]

- endpoints: e4
- what: CapabilitiesService đọc NOTIFICATION_PUSH_PROVIDER (dev|fcm|apns|disabled) để tính notifications.push.
- why: Push di động được chọn bằng NOTIFICATION_PUSH_PROVIDER ở common/platform (dev adapter bị cấm ở production, dev-platform-adapters.ts:33-37); web-push là port riêng bật bằng WEB_PUSH_ENABLED mà capabilities không phản ánh. Hiện chỉ "dev" ngoài production ra Beta, mọi giá trị khác kể cả fcm/apns đã cấu hình ra Disabled (capabilities.service.ts:158-166).
- if missing: Hợp đồng báo push Disabled ở production dù FCM/APNs/web-push có thể đang gửi được; chưa có client đọc cờ này nên chưa có UI sai, nhưng client nào bắt đầu dựa vào nó sẽ ẩn tính năng push đang chạy.
- evidence: apps/core-api/src/app/capabilities.service.ts:63
- review: sound

### phản chiếu cờ upload video (c57) with short-video [service, out, config]

- endpoints: e4
- what: CapabilitiesService đọc VIDEO_UPLOAD_ENABLED và NODE_ENV để tính video.upload/video.transcode (chỉ Beta ngoài production).
- why: ShortVideoService từ chối upload khi VIDEO_UPLOAD_ENABLED tắt và production chưa có adapter storage/transcode thật; capabilities phải cùng điều kiện để client không mời người dùng tải video lên rồi bị từ chối.
- if missing: Nút tải video hiện ở production dù server không có adapter xử lý video.
- evidence: apps/core-api/src/app/capabilities.service.ts:60
- review: sound

### probe live/ready (c58) with external:kubernetes-docker [external, in, http]

- endpoints: e2, e3
- what: Orchestrator gọi trực tiếp /health/live và /health/ready (không có prefix /api/v1): Docker HEALTHCHECK (apps/core-api/Dockerfile) và k8s livenessProbe dùng live, k8s readinessProbe và docker-compose.dev.yml dùng ready.
- why: Probe tách làm hai: live không chạm Postgres/Redis (chỉ biết process còn sống), ready chạy SELECT 1 + PING Redis (timeout 1s) rồi trả 503 để orchestrator rút pod khỏi load balancer. Path phải khớp tường minh với route ngoài global prefix (comment k8s/base/core-api/deployment.yaml).
- if missing: Orchestrator không phát hiện pod treo hoặc mất DB/Redis: pod chết vẫn nhận traffic hoặc không được khởi động lại.
- evidence: k8s/base/core-api/deployment.yaml:68, k8s/base/core-api/deployment.yaml:76, apps/core-api/Dockerfile:15
- review: sound

## Notes

- Health/capabilities KHÔNG có prefix /api/v1 cho /health* (k8s/base/core-api/deployment.yaml ghi rõ); /capabilities nằm dưới /api/v1.
- Readiness chỉ kiểm Postgres (SELECT 1) và Redis (PING), timeout 1s mỗi cái; không kiểm Kafka, LiveKit hay nhà cung cấp ngoài. /health/live không gọi dependency nào.
- Docker HEALTHCHECK (apps/core-api/Dockerfile, deploy/hosted) dùng /health/live; k8s dùng live cho liveness và ready cho readiness; docker-compose.dev.yml dùng /health/ready.
- CapabilitiesService.notifications.push chỉ trả Beta khi NOTIFICATION_PUSH_PROVIDER="dev" ngoài production, còn lại luôn Disabled (capabilities.service.ts:155-167). Đã đối chiếu: push di động (fcm/apns/dev/disabled) chọn qua NOTIFICATION_PUSH_PROVIDER ở platform.module.ts, còn web-push là WebPushNotificationPort riêng bật bằng WEB_PUSH_ENABLED (web-push-notification.adapter.ts:23) và NotificationService.sendPush gọi cả hai. Capabilities không đọc WEB_PUSH_ENABLED và không có nhánh Enabled cho fcm/apns, nên trong production cờ luôn Disabled dù push có thể đang chạy. Hiện không client nào trong apps/web, apps/admin đọc notifications.push (chỉ auth.* và topUp.* dùng isCapabilityUsable), nên đây là lệch hợp đồng chưa gây lỗi UI; cần chủ module xác nhận ý định.
- Cấu hình topUp.web coi PayOS sẵn sàng khi đủ PAYOS_CLIENT_ID/API_KEY/CHECKSUM_KEY và có redirect URL; capabilities không gọi PayOS nên 'enabled' chỉ nghĩa là đã cấu hình, không phải PayOS đang sống.
- Không có docs/services cho app-shell; mô tả dựa trên code.
