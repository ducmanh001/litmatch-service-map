# Giới hạn tốc độ và chống lạm dụng?

id: qa:rate-limit-abuse
category: abuse
services: module:matching, module:signaling-gateway, module:gift, module:auth

Nhiều lớp: ThrottlerGuard toàn cục đứng trước JWT và @Throttle riêng cho route nhạy cảm; rate limit nghiệp vụ bằng Redis (Lua) dùng chung nhiều instance; gateway giữ quota kết nối bằng lease Lua và fail-closed khi Redis lỗi; khách có hạn mức theo ngày lưu ở DB. [10/10] ThrottlerGuard đếm ở bộ nhớ từng pod mặc định; THROTTLE_STORAGE=redis (K8s) dùng RedisThrottlerStorage chung nhiều pod, khoá theo req.ip nên cần HTTP_TRUST_PROXY_HOPS đúng.

### Throttler

ThrottlerGuard toàn cục là guard đầu tiên; các route nhạy cảm như tạo invite hay tặng quà thêm @Throttle với hạn mức riêng.

services: matching, gift; evidence: apps/core-api/src/app/app.module.ts:154, apps/core-api/src/modules/matching/controllers/invite.controller.ts:53

### Redis Lua

Bộ đếm nghiệp vụ chạy bằng script Lua trong Redis nên đúng khi có nhiều instance; gateway dùng Lua tương tự để lấy, gia hạn và trả lease quota kết nối.

services: matching, signaling-gateway; evidence: apps/core-api/src/common/redis/redis-rate-limit.adapter.ts:79, apps/signaling-gateway/src/app/connection-quota.service.ts:126

### Fail-closed và hạn mức ngày

Redis hoặc quota lỗi thì handshake bị từ chối thay vì cho qua; khách có hạn mức ghép đôi theo ngày lưu ở DB và gắn với token thiết bị.

services: signaling-gateway, matching; evidence: apps/signaling-gateway/src/app/signaling.gateway.ts:36, apps/core-api/src/modules/matching/entities/guest-match-quota.entity.ts:4

## Alternatives not chosen

- Chỉ rate limit ở ingress/gateway: Suy luận: ingress thấy IP chứ không thấy user hay nghiệp vụ; hạn mức theo user và theo hành động phải nằm ở ứng dụng.
